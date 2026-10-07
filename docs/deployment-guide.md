# Deployment Guide

End-to-end deployment of the E-Commerce platform: local Docker Compose, then AWS EKS with GitHub Actions, Argo CD and the Prometheus and Grafana stack.

## Contents

1. [Prerequisites](#prerequisites)
2. [Run locally with Docker Compose](#run-locally-with-docker-compose)
3. [Provision the AWS infrastructure](#provision-the-aws-infrastructure)
4. [Prepare the manifests for your AWS account](#prepare-the-manifests-for-your-aws-account)
5. [Configure CI](#configure-ci)
6. [Deploy to EKS](#deploy-to-eks)
7. [Load the database](#load-the-database)
8. [Access the applications](#access-the-applications)
9. [Observability](#observability)
10. [Optional: forward logs to CloudWatch](#optional-forward-logs-to-cloudwatch)
11. [Credentials](#credentials)
12. [Cleanup](#cleanup)

## Prerequisites

| Tool | Needed for |
|------|-----------|
| Docker with Compose v2 | local stack |
| Node.js 20+ | building the frontend for Compose |
| AWS CLI, configured (`aws sts get-caller-identity` must succeed) | EKS and ECR |
| kubectl | cluster access |
| Terraform 1.5+ | infrastructure, see [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure) |
| A GitHub repository containing this code | CI and Argo CD |

## Run locally with Docker Compose

```bash
npm --prefix frontend install
npm --prefix frontend run build      # Compose serves ./frontend/build through nginx
docker compose up -d --build
docker compose ps
```

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Gateway, metrics | http://localhost:3001, http://localhost:3001/metrics |
| Prometheus | http://localhost:9090 |
| Grafana (`admin` / `admin`) | http://localhost:3007 |

Containers: `ecommerce-postgres`, `ecommerce-prometheus`, `ecommerce-grafana`, `ecommerce-gateway`, `ecommerce-auth`, `ecommerce-products`, `ecommerce-orders`, `ecommerce-order-management`, `ecommerce-users`, `ecommerce-frontend`.

PostgreSQL runs `database/init/` on first start (databases `auth_db`, `products_db`, `orders_db`, `users_db`, tables and sample rows). Stop with `docker compose down`; add `-v` to wipe the data volume so the init scripts run again.

## Provision the AWS infrastructure

Terraform lives in a separate repository, [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure). Apply it first. It creates:

- a VPC with three public subnets, an EKS 1.34 cluster `eks-cluster` and a managed node group
- one ECR repository per service (7)
- Argo CD (namespace `argocd`) and kube-prometheus-stack (namespace `monitoring`) through Helm

Then connect kubectl:

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
kubectl get nodes
```

## Prepare the manifests for your AWS account

The image references in `gitops/k8s/**` use a placeholder account ID:

```
<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/gateway:<tag>
```

The CI job that bumps image tags matches your real account ID, so replace the placeholder once, before the first CI run:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
grep -rl '<AWS_ACCOUNT_ID>' gitops/k8s | xargs sed -i '' "s/<AWS_ACCOUNT_ID>/${ACCOUNT_ID}/g"   # macOS
# Linux: use  sed -i "s/..." without the empty '' argument
git add gitops && git commit -m "chore: set AWS account id in manifests" && git push
```

The region `us-east-1` is hardcoded in the manifest image references; change it there if you deploy elsewhere (CI takes the region from the `AWS_REGION` secret). The tags currently in the manifests refer to images that do not exist in your registry. Pods stay in `ImagePullBackOff` until the CI workflow has run once.

## Configure CI

1. In AWS IAM create a user (for example `github-actions-ci`) with the managed policy `AmazonEC2ContainerRegistryFullAccess` and create an access key (use case: application running outside AWS).
2. In the GitHub repository open **Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | access key ID |
| `AWS_SECRET_ACCESS_KEY` | secret access key |
| `AWS_REGION` | `us-east-1` or your region |
| `AWS_ACCOUNT_ID` | 12-digit account ID |

3. Open **Actions → E-Commerce CI Pipeline → Run workflow**. The workflow is manual (`workflow_dispatch`).
   - `build-and-push` runs seven matrix jobs, one per image.
   - `update-manifests` rewrites the image tags in `gitops/k8s/` and commits `ci: update image tags to <sha>`.
4. Check the images:

```bash
aws ecr describe-images --repository-name frontend --region us-east-1 \
  --query 'imageDetails[*].imageTags' --output table
```

The tag is the commit SHA of the run.

## Deploy to EKS

Argo CD fetches the repository over HTTPS without credentials, so the repository must be public (or you add repository credentials in Argo CD).

Register the application:

```bash
kubectl apply -f gitops/argo-cd.yml -n argocd
kubectl get application -n argocd
```

Sync is manual, so the application shows `OutOfSync` until you sync it. Open the Argo CD UI (see [Access the applications](#access-the-applications)) and press **Sync**, or apply the manifests directly:

```bash
kubectl apply -k gitops/
```

To make Argo CD sync on its own, replace `syncPolicy: {}` in `gitops/argo-cd.yml` with:

```yaml
syncPolicy:
  automated:
    prune: true        # delete resources removed from Git
    selfHeal: true     # revert manual changes in the cluster
```

Check the pods (the database pod takes a little longer):

```bash
kubectl get pods -n ecommerce
```

## Load the database

On EKS the PostgreSQL StatefulSet starts empty. A Kubernetes Job loads the SQL dump (`database/boutique_full.sql`, shipped as the ConfigMap `ecommerce-db-dump`) and creates the auth and orders tables.

```bash
kubectl get pods -n ecommerce -l app=postgres        # wait for READY 1/1
kubectl apply -f gitops/k8s/database/restore-job.yml
kubectl get pods -n ecommerce -l job-name=ecommerce-db-restore
kubectl logs -n ecommerce -l job-name=ecommerce-db-restore
```

The Job ends in `Completed`. If it failed because PostgreSQL was not ready, delete it and apply it again:

```bash
kubectl delete job ecommerce-db-restore -n ecommerce
kubectl apply -f gitops/k8s/database/restore-job.yml
```

## Access the applications

All Services are `ClusterIP`; use port-forwarding.

```bash
kubectl port-forward svc/frontend 3000:3000 -n ecommerce &
kubectl port-forward svc/gateway 3001:3001 -n ecommerce &
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring &
kubectl port-forward svc/kube-prometheus-stack-grafana 8080:80 -n monitoring &
kubectl port-forward svc/argocd-server 8443:80 -n argocd &
```

| Application | URL |
|-------------|-----|
| Frontend | http://localhost:3000 |
| Gateway metrics | http://localhost:3001/metrics |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:8080 |
| Argo CD | http://localhost:8443 |

Two details matter here:

- The frontend calls the API at `http://localhost:3001/api`. That URL is compiled into the bundle at build time, so the browser needs the gateway forwarded to local port 3001 as above. The `REACT_APP_API_URL` value in the Deployment is not applied at runtime.
- Argo CD is installed with `server.insecure = true`, so it serves plain HTTP. Forward service port 80 and use `http://`, not `https://`.

## Observability

### Prometheus

kube-prometheus-stack discovers scrape targets through `ServiceMonitor` objects carrying the label `release: kube-prometheus-stack`. `gitops/k8s/backend/service-monitor.yml` selects the Service labelled `app: gateway`, port `http`, path `/metrics`, every 15 seconds. It targets the gateway only; add further Services (or label selectors) to cover the other backends.

Useful queries:

```promql
# Request rate per service
sum by (job) (rate(http_requests_total[5m]))

# 95th percentile latency
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# 5xx rate per service
sum by (job) (rate(http_requests_total{status_code=~"5.."}[5m]))

# Pod CPU and memory in the application namespace
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="ecommerce"}[5m]))
sum by (pod) (container_memory_working_set_bytes{namespace="ecommerce"})

# Pod restarts
kube_pod_container_status_restarts_total{namespace="ecommerce"}

# Node.js heap
nodejs_heap_size_used_bytes
```

### Grafana

The dashboard **E-Commerce Microservices** lives in `gitops/k8s/grafana-dashboard.yml` as a ConfigMap labelled `grafana_dashboard: "1"`. The Grafana sidecar from kube-prometheus-stack imports it automatically. Panels:

- Request Rate, Response Time, Active Requests and Error Rate for the selected service (the `service` variable at the top)
- Request Rate by Service, HTTP Error Rate by Service (4xx / 5xx)
- Node.js Heap Memory and Event Loop Lag by Service
- Pod CPU Usage, Pod Memory Usage, Pod Restart Count (`ecommerce` namespace)
- Service Health (UP/DOWN)

## Optional: forward logs to CloudWatch

Fluent Bit can ship pod logs to CloudWatch:

```bash
helm repo add aws https://aws.github.io/eks-charts
helm repo update

helm upgrade --install aws-for-fluent-bit aws/aws-for-fluent-bit \
  --namespace amazon-cloudwatch --create-namespace \
  --set cloudWatch.enabled=true \
  --set cloudWatch.region=us-east-1 \
  --set cloudWatch.logGroupName=/eks/ecommerce/pods \
  --set cloudWatch.logStreamPrefix=from-fluent-bit- \
  --set firehose.enabled=false \
  --set kinesis.enabled=false \
  --set elasticsearch.enabled=false
```

The Terraform in this setup attaches only `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy` and `AmazonEC2ContainerRegistryReadOnly` to the node role. Attach a CloudWatch Logs permission (for example `CloudWatchAgentServerPolicy`) to the node role first, or Fluent Bit cannot write. Logs then appear under **CloudWatch → Log groups → /eks/ecommerce/pods**.

## Credentials

```bash
# Grafana (user: admin)
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Argo CD (user: admin)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode
```

## Cleanup

Delete the workloads first so the EBS volume behind the PostgreSQL claim is released, then destroy the infrastructure:

```bash
kubectl delete -f gitops/argo-cd.yml -n argocd
kubectl delete namespace ecommerce
```

Then run `terraform destroy` in [E-Commerce-Infrastructure](https://github.com/hamzashah14/E-Commerce-Infrastructure). The ECR repositories are created with `force_delete = true`, so their images are removed too.
