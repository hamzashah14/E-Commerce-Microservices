# Database

One PostgreSQL 15 instance hosts four databases, one per backend service.

| Database | Used by |
|----------|---------|
| `auth_db` | auth |
| `products_db` | product-service |
| `orders_db` | orders, order-service |
| `users_db` | user-service |

## Files

| File | Purpose |
|------|---------|
| `init/10-create-databases.sh` | creates the four databases |
| `init/20-init-schema.sql` | creates tables and loads sample rows (users, categories, products) |
| `boutique_full.sql` | full `pg_dump` of the cluster, used by the Kubernetes restore Job |
| `quick-seed.sql` | extra product rows for manual loading |
| `setup.sh` | helper for setting up a local database by hand |
| `.env.example` | connection variables for local use |

The dump keeps its original file name because the Kubernetes ConfigMap (`ecommerce-db-dump`) and the restore Job reference it. Inside the dump, the extra database is named `boutique_db`.

## How data is loaded

- **Docker Compose:** the Postgres image runs everything in `init/` the first time the data directory is empty. To run it again, `docker compose down -v` and start again.
- **Kubernetes:** the StatefulSet starts empty. Apply `gitops/k8s/database/restore-job.yml`; the Job waits for `ecommerce-postgres`, restores `boutique_full.sql` and creates the auth and orders tables. See the [deployment guide](../docs/deployment-guide.md#load-the-database).

The sample catalogue is five products (Silk Evening Gown, Cashmere Coat, Leather Handbag, Diamond Necklace, Designer Heels) in five categories.
