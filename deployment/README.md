# `docs/deployment/` — Index

All deployment documentation for `findmyflowers.pl`. Read in roughly this order.

## Before first deploy

| Doc | What it's for | Read when |
|---|---|---|
| [DEPLOYMENT-SPEC.md](DEPLOYMENT-SPEC.md) | Why components live where; topology rationale; code changes required before deploy | First — establishes the mental model |
| [REGISTRATION-AND-CICD.md](REGISTRATION-AND-CICD.md) | Account setup (Fly, Render, Aiven, Cloudflare, OVH, etc.), path-triggered CI/CD, bootstrap commands | Before touching any SaaS console |
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

## Diagrams

| # | View | Render |
|---|---|---|
| 1 | Hosting topology (who lives where) | [01-hosting-topology.svg](01-hosting-topology.svg) |
| 2 | Customer purchase flow (OIDC → cart → Stripe) | [02-customer-purchase-flow.svg](02-customer-purchase-flow.svg) |
| 3 | IoT ingest → freshness push to browsers | [03-iot-ingest-flow.svg](03-iot-ingest-flow.svg) |
| 4 | Network boundaries + DNS | [04-network-and-dns.svg](04-network-and-dns.svg) |

## One-sentence role reference

- **Fly.io** hosts the always-on VMs: API, Keycloak, CustomerApp.
- **Render** hosts the cold-start-OK Razor portals: AdminPortal, VendorPortal.
- **Aiven** hosts PostgreSQL (two DBs: `flowershop_iot`, `keycloak`).
- **Upstash** provides Redis (cache; real impl pending EP-09).
- **CloudAMQP** provides RabbitMQ (event bus; real impl pending EP-09).
- **HiveMQ Cloud** provides MQTT broker (vase telemetry).
- **Cloudflare R2** stores bouquet photos (post-EP-15).
- **Cloudflare DNS** is authoritative for `findmyflowers.pl`.
- **OVHcloud** is the domain registrar only.
- **Stripe** processes payments; webhook lands on `api.findmyflowers.pl/api/stripe/webhook`.
- **GitHub Actions** runs path-triggered deploys; **GHCR** stores portal images.
- **cron-job.org** keeps Render portals warm and pings health endpoints.

## Quick links for operators

- Deploy API manually: `gh workflow run deploy-api.yml`
- Deploy portal manually: `gh workflow run deploy-admin-portal.yml` / `deploy-vendor-portal.yml`
- Tail API logs: `fly logs --app flowershop-api`
- Rollback API: `fly releases rollback --app flowershop-api`
- Smoke test all: `./scripts/smoke.sh` (see [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md))
