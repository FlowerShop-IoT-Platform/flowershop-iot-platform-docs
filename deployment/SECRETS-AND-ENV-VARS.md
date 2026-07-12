# Secrets & Environment Variables — Source of Truth

> **Scope**: every secret, credential, and configuration value needed across Fly (API, Keycloak, all 3 portals, Postgres), GitHub Actions, Cloudflare (DNS + R2), Stripe and HiveMQ.
> **Rule #1**: secrets never live in the repo. `appsettings.*.json` may contain *defaults* but never production values.
> **Rule #2**: rotate anything listed here if it ever lands in a chat log, screenshot, ticket, or PR diff.

## Where secrets live

| Store | What belongs here | How set |
|---|---|---|
| **Fly secrets** (per app) | Runtime config for `flower-shop-backend-core` (API), `flowershop-keycloak`, `flowershop-customer-app`, `flowershop-admin-portal`, `flowershop-vendor-portal` | `fly secrets set KEY=value --app <name>` — triggers a rolling restart |
| **GitHub Actions secrets** | Build/deploy-time credentials (Fly token, docs deploy token, backup creds) | `Settings → Secrets and variables → Actions` |
| **Cloudflare zone** | DNS records, R2 credentials | Dashboard; no secrets stored in code |
| **Local `.env`** (never committed) | Your laptop's copy when running `fly secrets set` bootstrap | `.gitignore` already covers `.env` |

Do **not** use `fly.*.toml` `[env]` blocks for secrets — those are committed to git. Only put non-sensitive public config there (URLs, feature flags, ASP.NET env names).

---

## Master table — every value, where it comes from, who consumes it

Abbreviations in the "Consumer" column: **A** = `FlowerShop.API` (`flower-shop-backend-core`) · **K** = Keycloak · **C** = CustomerApp · **Ad** = AdminPortal · **Ve** = VendorPortal · **GHA** = GitHub Actions · **DNS** = Cloudflare.

### Data plane

| Key | Value format | Source | Consumer | Rotates |
|---|---|---|---|---|
| `ConnectionStrings__DefaultConnection` | `Host=flower-shop-postgres.flycast;Database=flower_shop_backend_core;Username=...;Password=...` | Fly Postgres (`flower-shop-postgres`, PG 17) | A | on credential rotation |
| `ConnectionStrings__ReadConnection` | same as above (or read replica if promoted) | Fly Postgres | A | same |
| `ConnectionStrings__Redis` | *(unset in prod — Redis not provisioned; falls back to in-memory)* | — | A | — |
| `RabbitMQ__ConnectionString` | *(unset in prod — RabbitMQ not provisioned; `InMemoryEventBus` active)* | — | A | — |
| `MQTT__BrokerHost` | `xxxxxx.s1.eu.hivemq.cloud` | HiveMQ cluster URL | A | — |
| `MQTT__BrokerPort` | `8883` | constant | A | — |
| `MQTT__UseTls` | `true` | constant | A | — |
| `MQTT__Username` | per-client credential | HiveMQ → Access management | A | on credential rotation |
| `MQTT__Password` | per-client credential | HiveMQ → Access management | A | on credential rotation |

### Keycloak — DB & admin

| Key | Value | Consumer |
|---|---|---|
| `KC_DB_URL` | `jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak` | K |
| `KC_DB_USERNAME` | Fly Postgres user (shared with app today — create a separate role for prod isolation) | K |
| `KC_DB_PASSWORD` | — | K |
| `KEYCLOAK_ADMIN` | `admin` | K |
| `KEYCLOAK_ADMIN_PASSWORD` | strong password, 20+ chars | K |
| `KC_HOSTNAME` | `auth.findmyflowers.pl` (in `fly.keycloak.toml` `[env]`) | K |
| `KC_PROXY` | `edge` (in `fly.keycloak.toml` `[env]`) | K |

### Auth — API & CustomerApp

| Key | Value | Consumer |
|---|---|---|
| `Authentication__UseKeycloak` | `true` in prod, `false` for dev-token mode | A |
| `Authentication__Keycloak__Authority` | `https://auth.findmyflowers.pl/realms/flowershop` | A |
| `Authentication__Keycloak__ClientId` | `flowershop-api` | A |
| `Authentication__Keycloak__ClientSecret` | from Keycloak client credentials tab | A |
| `Authentication__Authority` | same as API's Authority | C |
| `Authentication__ClientId` | `flowershop-customer-app` | C |
| `Authentication__ClientSecret` | from Keycloak client credentials tab | C |
| `Authentication__PostLoginRedirectUri` | `https://findmyflowers.pl/` | C |

### Stripe

| Key | Value | Consumer |
|---|---|---|
| `Stripe__SecretKey` | `sk_test_...` → `sk_live_...` on launch | A |
| `Stripe__WebhookSecret` | `whsec_...` — **must match** the webhook endpoint registered in the Stripe dashboard | A |
| `Stripe__PublishableKey` | `pk_test_...` → `pk_live_...` | C (if used), A |

### CORS & forwarded headers

| Key | Value | Consumer |
|---|---|---|
| `ASPNETCORE_FORWARDEDHEADERS_ENABLED` | `true` | A, C, Ad, Ve |
| `Cors__AllowedOrigins__0` | `https://findmyflowers.pl` | A |
| `Cors__AllowedOrigins__1` | `https://admin.findmyflowers.pl` | A |
| `Cors__AllowedOrigins__2` | `https://vendors.findmyflowers.pl` | A |

### Portals (Fly secrets/env)

| Key | Value | Service |
|---|---|---|
| `ASPNETCORE_URLS` | `http://+:8080` | Ad, Ve |
| `ASPNETCORE_FORWARDEDHEADERS_ENABLED` | `true` | Ad, Ve |
| `ApiSettings__BaseUrl` | `https://api.findmyflowers.pl` | Ad, Ve |
| `Authentication__KeycloakBaseUrl` | `https://auth.findmyflowers.pl` | Ad, Ve |

AdminPortal and VendorPortal authenticate through the **`flowershop-api` client via password grant** — there is **no** dedicated `flowershop-vendor-portal` OIDC client. Only the CustomerApp uses OIDC (with the `flowershop-customer-app` client, above).

### Photo storage (EP-15)

| Key | Value | Consumer |
|---|---|---|
| `FileStorage__R2__Provider` (config `FileStorage:R2:Provider`) | `R2` | A |
| `FileStorage__R2__AccessKeyId` | R2 API token key (Fly secret) | A |
| `FileStorage__R2__SecretAccessKey` | R2 API token secret (Fly secret) | A |
| `FileStorage__R2__BucketName` | `flowershop-bouquets` | A |
| `FileStorage__R2__PublicUrl` | `https://img.findmyflowers.pl` | A |

Account ID + bucket + public URL live in `appsettings.Production.json`; only the two credentials are Fly secrets.

---

## GitHub Actions secrets

These drive the deploy workflows under `.github/workflows/`.

There are 8 workflow files: 5 deploy workflows (`deploy-api`, `deploy-admin-portal`, `deploy-vendor-portal`, `deploy-customer-app`, `deploy-keycloak`) plus `db-backup`, `docs`, and one disabled `ci-cd.yml.disabled`.

| Secret | Source | Used by |
|---|---|---|
| `FLY_API_TOKEN` | `fly tokens create deploy -x 999999h` | all 5 `deploy-*` workflows |
| `PG_PASSWORD` | Fly Postgres password | `db-backup` (via `flyctl proxy`) |
| `R2_ACCESS_KEY_ID` | R2 API token key | `db-backup` (upload dumps) |
| `R2_SECRET_ACCESS_KEY` | R2 API token secret | `db-backup` |
| `R2_ENDPOINT` | `https://<account>.r2.cloudflarestorage.com` | `db-backup` |
| `R2_BACKUP_BUCKET` | R2 bucket for backups | `db-backup` |
| `DOCS_DEPLOY_TOKEN` | PAT with push access to the docs gh-pages repo | `docs` |
| `CLOUDFLARE_API_TOKEN` *(optional)* | Cloudflare → My profile → API Tokens → "Zone / DNS / Edit" | Any DNS automation workflow |
| `GITHUB_TOKEN` *(auto)* | Built-in | GHA default token |

The following are optional GHA secrets used only during bootstrap (to `fly secrets set` in an ad-hoc workflow, instead of from your laptop). After bootstrap they can be deleted from GHA:

- `HIVEMQ_HOST`, `HIVEMQ_USER`, `HIVEMQ_PASS`
- `KEYCLOAK_ADMIN_PASSWORD`
- `KEYCLOAK_CLIENT_SECRET_API`, `KEYCLOAK_CLIENT_SECRET_CUSTOMER`
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`

---

## Fly Postgres connection string

The API and Keycloak reach Fly Postgres over the private `.flycast` network:

- **API (Npgsql):** `Host=flower-shop-postgres.flycast;Database=flower_shop_backend_core;Username=<user>;Password=<pwd>`
- **Keycloak (JDBC):** `jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak`

From a laptop, reach it via a tunnel: `flyctl proxy 5433:5432 -a flower-shop-postgres`, then connect to `localhost:5433`.

---

## Rotation procedure

Any compromised value, rotate in this order:

1. **Rotate at the source** (Keycloak client secret regen / Stripe key roll / R2 token reset).
2. Update the value in every consumer's secret store (see Consumer column above).
3. For Fly apps: `fly secrets set ...` auto-rolls the machine.
4. For GitHub Actions: overwrite the secret, then rerun the last deploy of every consumer workflow so the value is live.
5. Audit — `fly secrets list --app <name>` shows fingerprints; confirm they changed.

**Do not** delete the old value until the new one has been verified in prod — a rollback may need it.

---

## What never goes into a secret store

- Git history (even once, even in a revert — the commit is still reachable via SHA).
- `appsettings.*.json` in the repo.
- Docker image layers (baked-in env vars leak to anyone with pull access).
- Error logs / Sentry breadcrumbs — scrub before sending.
- Support-ticket attachments to SaaS vendors.
