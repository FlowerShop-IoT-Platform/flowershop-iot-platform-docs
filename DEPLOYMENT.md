# Deployment Guide — FlowerShop IoT Platform

## Live Environment

| Resource | Value |
|---|---|
| API URL | https://api.findmyflowers.pl (also https://flower-shop-backend-core.fly.dev) |
| Fly app name | `flower-shop-backend-core` |
| Fly Postgres app | `flower-shop-postgres` |
| Region | `fra` (Frankfurt) |
| Deployed from | `main` branch via `.github/workflows/deploy-api.yml` (`flyctl deploy --config fly.toml`) |

---

## Infrastructure

### Fly.io App

| Setting | Value |
|---|---|
| VM size | shared-1x-cpu @ 1 GB RAM |
| Machines | Always-on (`auto_stop_machines = false`, `min_machines_running = 1`) |
| HTTPS | Enforced (`force_https = true`) |
| Config | `fly.toml` |

> **Note:** `fly.api.toml` (app `flowershop-api`, region `waw`) is a stale, unused orphan config left over from an earlier plan. The live API deploys from `fly.toml` (app `flower-shop-backend-core`, region `fra`).

### PostgreSQL

| Setting | Value |
|---|---|
| Fly app | `flower-shop-postgres` |
| Region | `fra` (Frankfurt) |
| Engine | PostgreSQL 17 (Fly unmanaged) |
| Internal hostname | `flower-shop-postgres.internal` |
| Internal port | `5432` |
| Database | `flower_shop_backend_core` |
| Username | `flower_shop_backend_core` |

> The password is stored as a Fly secret (`DATABASE_URL`) on the app. Do not commit it.

---

## Inspecting the Production Database

The database is only reachable from within the Fly private network. Use `flyctl proxy` to open a local tunnel.

**Step 1 — open the tunnel** (keep this terminal open):
```powershell
flyctl proxy 5433:5432 -a flower-shop-postgres
```

**Step 2 — connect with pgAdmin or any Postgres client:**

| Field | Value |
|---|---|
| Host | `host.docker.internal` (from inside Docker) / `localhost` (from native client) |
| Port | `5433` |
| Database | `flower_shop_backend_core` |
| Username | `flower_shop_backend_core` |
| Password | see Fly secret `DATABASE_URL` or ask a team member |

**pgAdmin** (local dev, via Docker):
```powershell
docker-compose -f docker/docker-compose.dev.yml --profile tools up pgadmin-dev -d
# then open http://localhost:5050  (admin@flowershop.dev / admin123)
```

---

## Secrets

Managed via `flyctl secrets` — never committed to the repo.

| Secret | Purpose |
|---|---|
| `DATABASE_URL` | Full Postgres connection URL (set automatically by `postgres attach`) |
| `ConnectionStrings__DefaultConnection` | Npgsql connection string for EF Core |
| `ConnectionStrings__ReadConnection` | Read replica (same as primary for now) |
| `JWT__SecretKey` | JWT signing key — randomly generated 64-char hex string |
| `Authentication__UseKeycloak` | `true` — real Keycloak auth live at `auth.findmyflowers.pl` |
| `Stripe__SecretKey` / `Stripe__WebhookSecret` | Stripe payments (wired) |
| `FileStorage__R2__AccessKeyId` / `FileStorage__R2__SecretAccessKey` | Cloudflare R2 photo storage |

To view which secrets are set (values are never shown):
```powershell
flyctl secrets list --app flower-shop-backend-core
```

To update a secret (triggers automatic redeploy):
```powershell
flyctl secrets set KEY="value" --app flower-shop-backend-core
```

---

## Deploying

Deployments are triggered automatically on every push to `main` via the GitHub integration.

To deploy manually:
```powershell
flyctl deploy --app flower-shop-backend-core
```

To check deployment status:
```powershell
flyctl status --app flower-shop-backend-core
```

To tail live logs:
```powershell
flyctl logs --app flower-shop-backend-core
```

---

## Companion Services

| Service | Status | Notes |
|---|---|---|
| Keycloak | ✅ Deployed | Fly app `flowershop-keycloak` at `auth.findmyflowers.pl`; `Authentication__UseKeycloak=true` |
| MQTT broker | ✅ Deployed | HiveMQ Cloud, TLS port 8883 (replaces dev-only Mosquitto) |
| Admin Portal | ✅ Deployed | Fly app `flowershop-admin-portal` at `admin.findmyflowers.pl` (`fly.admin-portal.toml`) |
| Vendor Portal | ✅ Deployed | Fly app `flowershop-vendor-portal` at `vendors.findmyflowers.pl` (`fly.vendor-portal.toml`) |
| Customer App | ✅ Deployed | Fly app `flowershop-customer-app` at `app.findmyflowers.pl` (`fly.customer-app.toml`) |
| Cloudflare R2 | ✅ Wired | Bouquet photos, bucket `flowershop-bouquets`, served from `img.findmyflowers.pl` |
| Stripe | ✅ Wired | `Stripe__SecretKey` / `Stripe__WebhookSecret` via Fly secrets |
| Redis | Not provisioned in prod | Falls back to in-memory distributed cache (dev-only in `docker-compose.dev.yml`) |
| RabbitMQ | Not provisioned in prod | Using `InMemoryEventBus` (dev-only in `docker-compose.dev.yml`) |
