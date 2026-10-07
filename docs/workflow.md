# Delivery Workflow

---

## Overview

How a change moves from a developer's machine to the EKS cluster, and how it is observed once it runs.

```mermaid
flowchart LR
    A[👩‍💻 Developer] --> B[Local Docker]
    B --> C[Git Push]
    C --> D[GitHub Actions CI]
    D --> E[ECR Images]
    E --> F[ArgoCD GitOps]
    F --> G[EKS Cluster]
    G --> H[Prometheus + Grafana]
    H --> I[CloudWatch Logs - optional]
```

---

## Stage 1: Local Development

Every change starts on a developer's machine. The full application stack runs locally using Docker Compose — no cloud account required.

```mermaid
flowchart TD
    Dev[Developer writes code] --> DC[docker compose up]
    DC --> F[Frontend :3000]
    DC --> G[Gateway :3001]
    DC --> A[Auth :3002]
    DC --> P[Product Service :3003]
    DC --> OS[Order Service :3004]
    DC --> O[Orders :3005]
    DC --> U[User Service :3006]
    DC --> DB[(PostgreSQL :5432)]
    DC --> PR[Prometheus :9090]
    DC --> GR[Grafana :3007]

    G --> A
    G --> P
    G --> O
    G --> U
    A --> DB
    P --> DB
    O --> DB
    U --> DB
```

Each service has its own `Dockerfile`. Docker Compose wires them together on a shared network, so the full system can be tested locally before touching any cloud infrastructure. The frontend container serves `./frontend/build` with nginx, so run `npm --prefix frontend run build` first. The gateway proxies four prefixes; `order-service` runs standalone and is not routed by the gateway.

**What to verify locally:**
- All containers show `Up` in `docker ps`
- Frontend loads at http://localhost:3000
- `/api/products` returns data via the gateway
- Prometheus scrapes metrics from `/metrics` endpoints
- Grafana dashboards show live data

---

## Stage 2: Source Control

Once a change is tested locally, it goes into Git.

```mermaid
gitGraph
   commit id: "initial"
   branch feature/new-endpoint
   commit id: "add endpoint"
   commit id: "add tests"
   checkout main
   merge feature/new-endpoint id: "PR merged"
   commit id: "ci: update image tags"
```

**The flow:**
1. Developer creates a feature branch
2. Makes changes, commits with clear messages
3. Opens a Pull Request on GitHub
4. PR is reviewed and merged into `main`
5. Run the CI pipeline from the Actions tab (the workflow is manual; it can be switched to run on every push to `main`)

Everything is tracked — who changed what, when, and why. This is the foundation of GitOps.

---

## Stage 3: CI Pipeline — GitHub Actions

When the workflow runs, GitHub Actions builds Docker images for all 7 services in parallel and pushes them to Amazon ECR. The trigger is `workflow_dispatch` (manual); `on: push: branches: [main]` makes it run on every push.

```mermaid
flowchart TD
    Push[Run workflow] --> Trigger[GitHub Actions triggered]

    Trigger --> B1[Build auth]
    Trigger --> B2[Build gateway]
    Trigger --> B3[Build product-service]
    Trigger --> B4[Build order-service]
    Trigger --> B5[Build orders]
    Trigger --> B6[Build user-service]
    Trigger --> B7[Build frontend]

    B1 & B2 & B3 & B4 & B5 & B6 & B7 --> Push2[Push all images to ECR]
    Push2 --> UM[update-manifests job]
    UM --> |Updates image tags in gitops/k8s/| Commit[Commits back to main]
```

**Key concepts:**
- Each service is a separate matrix job — they all build in parallel
- Images are tagged with the commit SHA for full traceability
- The `update-manifests` job patches the image tag in every Kubernetes manifest and commits the change back
- Argo CD detects this commit; with the manual sync policy you then press Sync to roll it out

**Where to check:** GitHub repo → **Actions** tab → **E-Commerce CI Pipeline**

---

## Stage 4: Infrastructure — Terraform on AWS

Before the cluster can run anything, the infrastructure must exist. The Terraform in [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure) provisions everything from scratch.

```mermaid
flowchart TD
    TF[terraform apply] --> VPC[VPC — 3 AZs]
    VPC --> Sub1[Subnet us-east-1a]
    VPC --> Sub2[Subnet us-east-1b]
    VPC --> Sub3[Subnet us-east-1c]

    TF --> EKS[EKS Cluster]
    EKS --> NG[Node Group\nm7i-flex.large]
    Sub1 & Sub2 & Sub3 --> NG

    TF --> ECR1[ECR: frontend]
    TF --> ECR2[ECR: gateway]
    TF --> ECR3[ECR: auth]
    TF --> ECR4[ECR: ...]

    TF --> Helm1[Helm: ArgoCD\nnamespace: argocd]
    TF --> Helm2[Helm: kube-prometheus-stack\nnamespace: monitoring]
```

Terraform also installs ArgoCD and the Prometheus/Grafana stack into the cluster via Helm — so the entire platform is ready to receive workloads the moment `terraform apply` finishes.

---

## Stage 5: GitOps Deployment — ArgoCD

Argo CD runs inside the cluster and watches the `main` branch. When the CI pipeline commits updated image tags back to Git, Argo CD detects the change and, once synced, rolls out the new version.

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub (main)
    participant CI as GitHub Actions
    participant ECR as Amazon ECR
    participant Argo as ArgoCD
    participant EKS as EKS Cluster

    Dev->>GH: git push
    GH->>CI: trigger pipeline
    CI->>ECR: docker push (new image)
    CI->>GH: commit updated image tag
    Argo->>GH: polls every 3 mins / webhook
    GH-->>Argo: detects new commit
    Argo->>EKS: sync (rolling update)
    EKS-->>Argo: sync complete
```

**What ArgoCD does:**
- Continuously compares the desired state in Git against the live state in the cluster
- If they differ, it syncs — applying only what changed
- With `selfHeal` enabled, a manual change in the cluster is reverted to match Git (the sync policy in this repo is manual)
- Every deployment is auditable — it's just a Git commit

**Key files:**
- `gitops/argo-cd.yml` — registers the repo and branch with ArgoCD
- `gitops/kustomization.yml` — lists all Kubernetes resources to apply
- `gitops/k8s/` — all service deployments, services, database, secrets

---

## Stage 6: Observability

Once the application is running in EKS, three layers of observability keep watch.

```mermaid
flowchart LR
    subgraph Services [ecommerce namespace]
        GW[gateway /metrics]
        AU[auth /metrics]
        PS[product-service /metrics]
        OS[order-service /metrics]
        OR[orders /metrics]
        US[user-service /metrics]
    end

    subgraph Monitoring [monitoring namespace]
        SM[ServiceMonitor] -->|scrape every 15s| PR[Prometheus]
        PR --> GR[Grafana\nDashboard]
    end

    subgraph Logging [amazon-cloudwatch namespace]
        FB[Fluent Bit] -->|pod logs| CW[CloudWatch\n/eks/ecommerce/pods]
    end

    GW & AU & PS & OS & OR & US --> SM
    GW & AU & PS & OS & OR & US --> FB
```

**Metrics — Prometheus + Grafana**
- Every backend service exposes a `/metrics` endpoint using `prom-client`
- A `ServiceMonitor` resource tells the Prometheus Operator which Services to scrape; the one in this repo targets the gateway
- Grafana is pre-loaded with the **E-Commerce Microservices** dashboard via a ConfigMap labelled `grafana_dashboard: "1"`; the Grafana sidecar imports it automatically

**Logs — Fluent Bit + CloudWatch**
- Optional, installed with Helm (see the deployment guide): Fluent Bit runs as a DaemonSet in `amazon-cloudwatch`
- Captures stdout from every pod and ships logs to CloudWatch
- Log group: `/eks/ecommerce/pods`

**What to check in Grafana:**
- Request rate by service
- p95 / p99 response times
- 4xx and 5xx error rates
- Pod CPU and memory usage
- Pod restart count — surfaces crash loops early

---

## The Complete Picture

```mermaid
flowchart TD
    Dev[👩‍💻 Developer] -->|writes code| Local[Docker Compose\nLocal Testing]
    Local -->|git push| GH[GitHub main branch]
    GH -->|triggers| CI[GitHub Actions\nBuild + Push to ECR]
    CI -->|commits image tags| GH
    GH -->|ArgoCD detects change| Argo[ArgoCD\nRolling Deploy to EKS]
    Argo --> EKS[EKS Cluster\n7 microservices]
    EKS -->|metrics /metrics| Prom[Prometheus]
    EKS -->|pod logs| FB[Fluent Bit]
    Prom --> Grafana[Grafana\nDashboards]
    FB --> CW[CloudWatch\nLog Groups]
    Grafana --> Eng[Engineer]
    CW --> Eng

    subgraph IaC [Infrastructure as Code]
        TF[Terraform\nVPC + EKS + ECR + Helm]
    end
    TF --> EKS
```

---

## Key Files Reference

| File | Stage | Purpose |
|------|-------|---------|
| `docker-compose.yml` | Stage 1 | Local stack |
| `.github/workflows/ci.yml` | Stage 3 | Build and push images |
| [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure) | Stage 4 | Terraform for AWS (separate repository) |
| `gitops/argo-cd.yml` | Stage 5 | ArgoCD application definition |
| `gitops/k8s/` | Stage 5 | All Kubernetes manifests |
| `gitops/k8s/backend/service-monitor.yml` | Stage 6 | Prometheus scrape config |
| `gitops/k8s/grafana-dashboard.yml` | Stage 6 | Pre-loaded Grafana dashboard |
