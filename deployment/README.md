# `docs/deployment/` — Index

All deployment documentation for `findmyflowers.pl`. Read in roughly this order.

## Before first deploy

| Doc | What it's for | Read when |
|---|---|---|
| [DEPLOYMENT-SPEC.md](DEPLOYMENT-SPEC.md) | Why components live where; topology rationale; what shipped | First — establishes the mental model |
| [SHOWCASE-RUNBOOK.md](SHOWCASE-RUNBOOK.md) | Ordered, executable checklist that took the platform live on `findmyflowers.pl` (Keycloak, api domain, all 3 portals, Stripe, MQTT) | The go-live record |
| [REGISTRATION-AND-CICD.md](REGISTRATION-AND-CICD.md) | Account setup (Fly, Cloudflare, OVH, etc.), path-triggered CI/CD, bootstrap commands | Before touching any SaaS console |
| [SECRETS-AND-ENV-VARS.md](SECRETS-AND-ENV-VARS.md) | Master table of every secret/env var, where it lives, who consumes it, rotation procedure | While registering accounts — populate as you go |

## During first deploy

| Doc | What it's for |
|---|---|
| [DEPLOYMENT-RUNBOOK.md](DEPLOYMENT-RUNBOOK.md) | **Ordered** stage-by-stage runbook with health-check gates. This is the doc you follow live on deploy day. |
| [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md) | Scripted end-to-end validation. Run after the runbook says "done". |

## Day-two operations

| Doc | What it's for |
|---|---|
| [OBSERVABILITY.md](OBSERVABILITY.md) | Where logs, health endpoints, and metrics live. First thing to read when something is wrong. |
| [ROLLBACK-AND-RECOVERY.md](ROLLBACK-AND-RECOVERY.md) | Rollback procedures per component, dependency outage playbook, DB backup/restore, incident checklist |

## Automation

| Workflow | What it does |
|---|---|
| [`.github/workflows/db-backup.yml`](../../.github/workflows/db-backup.yml) | Weekly (Mondays 03:17 UTC) `pg_dump -F c` of both DBs to Cloudflare R2, via `flyctl proxy` |
| [`.github/workflows/docs.yml`](../../.github/workflows/docs.yml) | Publishes these docs (Jekyll → gh-pages) to `docs.findmyflowers.pl` (custom domain via `docs/CNAME`) |

## Diagrams

| # | View | Render |
|---|---|---|
| 1 | Hosting topology (who lives where) | [01-hosting-topology.svg](01-hosting-topology.svg) |
| 2 | Customer purchase flow (OIDC → cart → Stripe) | [02-customer-purchase-flow.svg](02-customer-purchase-flow.svg) |
| 3 | IoT ingest → freshness push to browsers | [03-iot-ingest-flow.svg](03-iot-ingest-flow.svg) |
| 4 | Network boundaries + DNS | [04-network-and-dns.svg](04-network-and-dns.svg) |

## One-sentence role reference

- **Fly.io** hosts every always-on VM: API (`flower-shop-backend-core`), Keycloak, and all three portals (AdminPortal, VendorPortal, CustomerApp) — all in region `fra`.
- **Fly Postgres** (`flower-shop-postgres`, PostgreSQL 17) hosts both DBs: `flower_shop_backend_core` (app) and `keycloak`.
- Redis and RabbitMQ are **not provisioned in prod** — the code falls back to in-memory cache and `InMemoryEventBus` (both are dev-only in `docker-compose.dev.yml`).
- **HiveMQ Cloud** provides the MQTT broker (vase telemetry), TLS port 8883.
- **Cloudflare R2** stores bouquet photos (bucket `flowershop-bouquets`, served from `img.findmyflowers.pl`).
- **Cloudflare DNS** is authoritative for `findmyflowers.pl`.
- **OVHcloud** is the domain registrar only.
- **Stripe** processes payments; webhook lands on `api.findmyflowers.pl/api/stripe/webhook`.
- **GitHub Actions** runs path-triggered deploys (`flyctl deploy`); a weekly `db-backup.yml` and a `docs.yml` publisher round out the 8 workflows.

## Quick links for operators

- Deploy API manually: `gh workflow run deploy-api.yml`
- Deploy portal manually: `gh workflow run deploy-admin-portal.yml` / `deploy-vendor-portal.yml` / `deploy-customer-app.yml`
- Tail API logs: `fly logs --app flower-shop-backend-core`
- Tail portal logs: `fly logs --app flowershop-admin-portal` (or `-vendor-portal` / `-customer-app`)
- Rollback API: `fly releases rollback --app flower-shop-backend-core`
- Smoke test all: `./scripts/smoke.sh` (see [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md))
