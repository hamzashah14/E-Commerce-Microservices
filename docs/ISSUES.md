# Troubleshooting

Known problems with this platform, with the cause and the fix. Infrastructure-side issues (node capacity, EBS permissions) are in the [E-Commerce-Infrastructure issues log](https://github.com/hamzashah14/E-Commerce-Infrastructure/blob/main/docs/ISSUES.md).

## Kubernetes and GitOps

### Pods stay in `ImagePullBackOff` on the first deploy

- **Symptom:** every application pod fails to pull its image.
- **Cause:** the manifests reference the placeholder account `<AWS_ACCOUNT_ID>` and a commit tag that was never pushed to your registry.
- **Fix:** replace the placeholder with your account ID, push, and run the CI workflow once so images exist and the tags are updated. Steps in the [deployment guide](deployment-guide.md#prepare-the-manifests-for-your-aws-account).

### Argo CD shows `OutOfSync` but nothing deploys

- **Cause:** `syncPolicy` in `gitops/argo-cd.yml` is `{}`, which means manual sync.
- **Fix:** press **Sync** in the UI, or enable the `automated` policy with `prune` and `selfHeal`.

### Argo CD UI does not load over HTTPS

- **Cause:** Argo CD is installed with `server.insecure = true`, so the server speaks plain HTTP.
- **Fix:** `kubectl port-forward svc/argocd-server 8443:80 -n argocd` and open `http://localhost:8443`.

### The frontend loads but shows no data in the cluster

- **Cause:** the API URL is compiled into the React bundle at build time (default `http://localhost:3001/api`). The `REACT_APP_API_URL` variable in the Deployment does not change a finished build, and the in-cluster name `gateway` is not resolvable from a browser.
- **Fix:** forward the gateway to local port 3001 (`kubectl port-forward svc/gateway 3001:3001 -n ecommerce`). To use another URL, set `REACT_APP_API_URL` when building the image.

### Products page is empty or the product service logs `database "products_db" does not exist`

- **Cause:** the PostgreSQL StatefulSet starts with an empty cluster on EKS. The EBS volume root contains `lost+found`, which breaks `initdb` when `PGDATA` is the volume root; the manifest sets `PGDATA=/var/lib/postgresql/data/pgdata` to avoid that. No init scripts are mounted, so the databases do not exist until data is loaded.
- **Fix:** wait for `ecommerce-postgres-0` to be Ready, then apply the restore Job. If it failed because PostgreSQL was not ready, delete the Job and apply it again. Steps in the [deployment guide](deployment-guide.md#load-the-database).

### Application metrics are missing in Prometheus and Grafana

- **Symptom:** cluster and node metrics appear, application metrics do not, although they work under Docker Compose.
- **Cause:** kube-prometheus-stack only scrapes targets described by a `ServiceMonitor` with the label `release: kube-prometheus-stack`, and the target Service must have a named port that the monitor references (`http`).
- **Fix:** `gitops/k8s/backend/service-monitor.yml` selects the Service labelled `app: gateway`, port `http`, path `/metrics`. The gateway Service names its port `http`. Add the same pattern for other services you want scraped.

## Docker Compose

### The storefront is blank or nginx returns 403/404

- **Cause:** the `frontend` container is plain `nginx:alpine` serving `./frontend/build`, which does not exist until you build the React app.
- **Fix:** `npm --prefix frontend install && npm --prefix frontend run build`, then `docker compose up -d`.

### `/api/users` fails under Compose

- **Cause:** the gateway reads `USERS_SERVICE_URL`, but `docker-compose.yml` sets `USER_SERVICE_URL`. The gateway falls back to its default `http://localhost:3005`, which does not exist inside the container. (`ORDER_SERVICE_URL` and `PORT` in the same block are also not read; the gateway uses `ORDERS_SERVICE_URL` and `GATEWAY_PORT`.)
- **Fix:** in the gateway `environment` list, use `USERS_SERVICE_URL=http://user-service:3006`. The Kubernetes manifest already does.

### Database changes do not apply after editing `database/init/`

- **Cause:** PostgreSQL runs the init scripts only when its data directory is empty.
- **Fix:** `docker compose down -v` removes the `postgres_data` volume, then start again.

## CI

### The workflow does not start on push

- **Cause:** `ci.yml` is triggered by `workflow_dispatch` only.
- **Fix:** run it from the Actions tab, or change the trigger to `push` on `main` (see the README).

### `update-manifests` commits nothing

- **Cause:** the `sed` in the job matches `<account>.dkr.ecr.<region>.amazonaws.com/<service>:` using the `AWS_ACCOUNT_ID` and `AWS_REGION` secrets. If the manifests still contain the placeholder, nothing matches.
- **Fix:** replace the placeholder with your real account ID first.
