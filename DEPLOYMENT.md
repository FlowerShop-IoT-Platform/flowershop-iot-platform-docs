# Deployment Guide — FlowerShop IoT Platform

## Live Environment

| Resource | Value |
|---|---|
| API URL | https://flower-shop-backend-core.fly.dev |
| Fly app name | `flower-shop-backend-core` |
| Fly Postgres app | `flower-shop-postgres` |
| Region | `iad` (Virginia) — target `fra` (Frankfurt) on next machine resize |
| Deployed from | `main` branch via GitHub integration |

---

## Infrastructure

### Fly.io App

| Setting | Value |
|---|---|
| VM size | shared-1x-cpu @ 1 GB RAM (256 MB until next redeploy — see note) |
| Machines | 2 |
| Auto-stop | Yes (scales to zero when idle, wakes on first request) |
| HTTPS | Enforced (`force_https = true`) |

> **Note:** The machines were provisioned before the `fly.toml` memory fix (PR #8) was merged. They will pick up `memory = '1gb'` on the next triggered redeploy.

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
| `Authentication__UseKeycloak` | `false` — Keycloak stubbed out, dev-token auth active |

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

## Services Not Yet Deployed

| Service | Status | Notes |
|---|---|---|
| MQTT broker | Not deployed | App retries in background, non-fatal |
| Redis | Not deployed | Falls back to in-memory cache |
| RabbitMQ | Not deployed | Using `InMemoryEventBus` |
| Keycloak | Not deployed | `Authentication__UseKeycloak=false` — dev-token auth active |
| Admin Portal | Not deployed | Separate app needed (EP-11) |
| Vendor Portal | Not deployed | Separate app needed (EP-12) |
| Customer App | Not deployed | Separate app needed (EP-13) |
