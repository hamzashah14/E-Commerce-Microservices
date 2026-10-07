# E-Commerce Microservices

A DevOps learning project. A seven-service e-commerce sample app serves as the workload for practising containerisation, CI/CD, GitOps, Kubernetes on AWS EKS and observability. The application code is deliberately small; the delivery platform around it is the point.

The AWS side (VPC, EKS, ECR, Argo CD, Prometheus and Grafana) is provisioned from the companion repository [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure).

## What this project covers

| Area | What is implemented | Where |
|------|---------------------|-------|
| Containers | One Dockerfile per service on `node:20-alpine`; multi-stage builds and a non-root user on the backend services | `backend/services/*/Dockerfile`, `frontend/Dockerfile` |
| Local environment | Docker Compose: 7 app containers, PostgreSQL 15, Prometheus, Grafana | `docker-compose.yml` |
| CI | GitHub Actions builds the 7 images in parallel and pushes them to Amazon ECR, tagged with the commit SHA | `.github/workflows/ci.yml` |
| GitOps | Kustomize manifests; CI bumps the image tags; Argo CD syncs `gitops/` to the cluster | `gitops/` |
| Kubernetes | Deployments and ClusterIP Services, PostgreSQL StatefulSet on EBS (gp2), Secrets, database restore Job | `gitops/k8s/` |
| Observability | `/metrics` on every backend service, ServiceMonitor, Grafana dashboard shipped as a ConfigMap | `gitops/k8s/`, `prometheus/`, `grafana/` |
| Infrastructure as code | Terraform: VPC, EKS 1.34, ECR, Argo CD and kube-prometheus-stack via Helm | [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure) |
| AI-assisted operations | Claude Code with project rules and AWS MCP servers | `CLAUDE.md`, [docs/claude-setup.md](docs/claude-setup.md) |

## Delivery flow

```
 git push ──▶ GitHub ──(run workflow)──▶ GitHub Actions ──▶ build 7 images ──▶ Amazon ECR
                ▲                              │
                └── commit new image tags ◀────┘
                │
        Argo CD watches gitops/ ──(sync)──▶ EKS, namespace "ecommerce"
                                               │
                         Prometheus ◀── /metrics ──┘ ──▶ Grafana dashboard
```

The workflow is triggered manually. See [CI/CD](#cicd) to switch it to run on every push.

## The workload

```
 Browser ──▶ Frontend :3000          (static React build)
    │
    └─ /api/* ──▶ Gateway :3001 ──▶ Auth :3002 ───────────▶ auth_db
                               ├──▶ Product Service :3003 ─▶ products_db
                               ├──▶ Orders :3005 ─────────▶ orders_db
                               └──▶ User Service :3006 ───▶ users_db
                  Order Service :3004 (standalone, not routed by the gateway)

 One PostgreSQL 15 instance (:5432) hosts all four databases.
```

| Service | Port | Role |
|---------|------|------|
| frontend | 3000 | React storefront |
| gateway | 3001 | Reverse proxy for `/api/auth`, `/api/products`, `/api/orders`, `/api/users`; exposes `/metrics` |
| auth | 3002 | Registration and login |
| product-service | 3003 | Product catalogue |
| order-service | 3004 | Minimal order endpoint |
| orders | 3005 | Order management |
| user-service | 3006 | User profiles |

Every backend service exposes `/metrics` and `/health`.

## Quick start (Docker Compose)

Prerequisites: Docker with Compose v2, Node.js 20+.

```bash
git clone https://github.com/hamzashah14/E-Commerce-Microservices.git
cd E-Commerce-Microservices

# Compose serves ./frontend/build with nginx, so build the frontend first
npm --prefix frontend install
npm --prefix frontend run build

docker compose up -d --build
```

| What | URL |
|------|-----|
| Storefront | http://localhost:3000 |
| Gateway (API, `/metrics`) | http://localhost:3001 |
| Prometheus | http://localhost:9090 |
| Grafana (`admin` / `admin`) | http://localhost:3007 |

On first start PostgreSQL runs the scripts in `database/init/`, which create the four databases and load sample data. `./health-check.sh` probes the frontend, product service and database.

```bash
docker compose down        # stop; add -v to also delete the database volume
```

## Deploying to AWS

Provision the cluster with [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure), then follow the [deployment guide](docs/deployment-guide.md). Short version:

1. Replace the `<AWS_ACCOUNT_ID>` placeholder in `gitops/k8s/**` with your account ID and push.
2. Add the four GitHub Actions secrets and run the **E-Commerce CI Pipeline** workflow.
3. Register the Argo CD application: `kubectl apply -f gitops/argo-cd.yml -n argocd`, then sync it.
4. Load the sample data with the restore Job: `kubectl apply -f gitops/k8s/database/restore-job.yml`.

## CI/CD

`.github/workflows/ci.yml` has two jobs:

1. **build-and-push**: a matrix over `auth`, `gateway`, `orders`, `order-service`, `product-service`, `user-service` and `frontend`. Each entry builds its Dockerfile and pushes `<account>.dkr.ecr.<region>.amazonaws.com/<service>:<commit-sha>`.
2. **update-manifests**: rewrites the image tags in `gitops/k8s/backend/*.yml` and `gitops/k8s/frontend/deployment.yml`, then commits `ci: update image tags to <sha>` back to the branch.

Required repository secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_ACCOUNT_ID`.

The trigger is `workflow_dispatch` (manual). To run on every push to `main`, change the `on:` block:

```yaml
on:
  push:
    branches:
      - main
```

## GitOps

`gitops/kustomization.yml` is the entry point and sets the `ecommerce` namespace. `gitops/argo-cd.yml` defines the Argo CD Application: repository this repo, branch `main`, path `gitops`, destination namespace `ecommerce`. Sync is manual. To let Argo CD apply and self-heal automatically:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Observability

- **Compose:** `prometheus/prometheus.yml` scrapes Prometheus itself and all six backend services (gateway included); Grafana is provisioned with Prometheus as its default data source.
- **EKS:** kube-prometheus-stack (installed by Terraform) in the `monitoring` namespace. `gitops/k8s/backend/service-monitor.yml` scrapes `/metrics` every 15 seconds. `gitops/k8s/grafana-dashboard.yml` carries the **E-Commerce Microservices** dashboard; the label `grafana_dashboard: "1"` makes the Grafana sidecar import it.

## Repository structure

```
.
├── .github/workflows/ci.yml     build and push images, bump manifest tags
├── backend/
│   ├── services/                gateway, auth, product-service, order-service, orders, user-service
│   └── shared/                  shared TypeScript types
├── frontend/                    React app and nginx config
├── database/                    init scripts, SQL dump, seed data
├── gitops/                      Kustomize manifests and Argo CD Application
├── grafana/provisioning/        Grafana data source for Compose
├── prometheus/prometheus.yml    scrape config for Compose
├── docs/                        guides and write-ups
├── docker-compose.yml           local stack
├── health-check.sh              local smoke check
├── CLAUDE.md                    Claude Code project rules
└── package.json                 npm workspaces and helper scripts
```

`image-service.js`, `mock-product-service.js`, `simple-product-service.js` and `update-service.js` in the repository root are standalone development utilities. They are not part of Compose, CI or the manifests.

## Documentation

| Document | Contents |
|----------|----------|
| [docs/deployment-guide.md](docs/deployment-guide.md) | End-to-end deployment: Compose, EKS, CI, Argo CD, observability, cleanup |
| [docs/workflow.md](docs/workflow.md) | How a change moves from laptop to cluster, stage by stage |
| [docs/system-design.md](docs/system-design.md) | System design concepts and where each shows up in this project |
| [docs/ISSUES.md](docs/ISSUES.md) | Known problems with causes and fixes |
| [docs/claude-setup.md](docs/claude-setup.md) | Claude Code and MCP server configuration |
| [database/README.md](database/README.md) | Database layout and how data is loaded |
| [frontend/README.md](frontend/README.md) | Frontend build and runtime configuration |

## Known limitations

This is a learning project, not a production system.

- Argo CD sync is manual (`syncPolicy: {}`).
- Every Deployment runs one replica, with no liveness or readiness probes and no resource requests or limits. There is no HPA or PodDisruptionBudget.
- The ServiceMonitor selects only the `gateway` Service, so in the cluster Prometheus scrapes the gateway only. Compose scrapes all services.
- Demo credentials are committed (`postgres123` in `gitops/secrets.yml` and `docker-compose.yml`, Grafana `admin` / `admin`). Use a secrets manager for anything real.
- CI authenticates to AWS with long-lived IAM access keys instead of GitHub OIDC.
- `docker-compose.yml` sets `USER_SERVICE_URL`, `ORDER_SERVICE_URL` and `PORT` on the gateway, but the gateway reads `USERS_SERVICE_URL`, `ORDERS_SERVICE_URL` and `GATEWAY_PORT`. In Compose, `/api/users` therefore falls back to `http://localhost:3005` and fails. Fix: rename the variable to `USERS_SERVICE_URL` with value `http://user-service:3006`. The Kubernetes manifest already uses the right names.
