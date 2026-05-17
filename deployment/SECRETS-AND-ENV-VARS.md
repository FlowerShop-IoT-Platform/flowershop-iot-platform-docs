# Secrets & Environment Variables — Source of Truth

> **Scope**: every secret, credential, and configuration value needed across Fly, Render, GitHub Actions, Cloudflare, Stripe and the four SaaS providers.
> **Rule #1**: secrets never live in the repo. `appsettings.*.json` may contain *defaults* but never production values.
> **Rule #2**: rotate anything listed here if it ever lands in a chat log, screenshot, ticket, or PR diff.

## Where secrets live

| Store | What belongs here | How set |
|---|---|---|
| **Fly secrets** (per app) | Runtime config for `FlowerShop.API`, `flowershop-keycloak`, `flowershop-customer-app` | `fly secrets set KEY=value --app <name>` — triggers a rolling restart |
| **Render env vars** (per service) | Runtime config for admin + vendor portals | Dashboard → service → Environment |
| **GitHub Actions secrets** | Build/deploy-time credentials (Fly token, Render API key, GHCR PAT if private) | `Settings → Secrets and variables → Actions` |
| **Cloudflare zone** | DNS records, R2 credentials | Dashboard; no secrets stored in code |
| **Local `.env`** (never committed) | Your laptop's copy when running `fly secrets set` bootstrap | `.gitignore` already covers `.env` |

Do **not** use `fly.*.toml` `[env]` blocks for secrets — those are committed to git. Only put non-sensitive public config there (URLs, feature flags, ASP.NET env names).

---

## Master table — every value, where it comes from, who consumes it

Abbreviations in the "Consumer" column: **A** = `FlowerShop.API` · **K** = Keycloak · **C** = CustomerApp · **Ad** = AdminPortal · **Ve** = VendorPortal · **GHA** = GitHub Actions · **DNS** = Cloudflare.

### Data plane

| Key | Value format | Source | Consumer | Rotates |
|---|---|---|---|---|
| `ConnectionStrings__DefaultConnection` | `Host=...;Database=flowershop_iot;Username=...;Password=...;SslMode=Require` | Aiven service URI (converted Npgsql format) | A | on credential rotation |
| `ConnectionStrings__ReadConnection` | same as above (or read replica if promoted) | Aiven | A | same |
| `ConnectionStrings__Redis` | `rediss://default:<pwd>@<host>:<port>` | Upstash | A | on Upstash rotation |
| `RabbitMQ__ConnectionString` | `amqps://<user>:<pwd>@<host>/<vhost>` | CloudAMQP | A | on CloudAMQP rotation |
| `MQTT__BrokerHost` | `xxxxxx.s1.eu.hivemq.cloud` | HiveMQ cluster URL | A | — |
| `MQTT__BrokerPort` | `8883` | constant | A | — |
| `MQTT__UseTls` | `true` | constant | A | — |
| `MQTT__Username` | per-client credential | HiveMQ → Access management | A | on credential rotation |
| `MQTT__Password` | per-client credential | HiveMQ → Access management | A | on credential rotation |

### Keycloak — DB & admin

| Key | Value | Consumer |
|---|---|---|
| `KC_DB_URL` | `jdbc:postgresql://<aiven-host>:<port>/keycloak?sslmode=require` | K |
| `KC_DB_USERNAME` | Aiven user (shared with app today — create a separate role in Aiven for prod) | K |
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
| `Authentication__PostLoginRedirectUri` | `https://app.findmyflowers.pl/` | C |

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
| `Cors__AllowedOrigins__0` | `https://app.findmyflowers.pl` | A |
| `Cors__AllowedOrigins__1` | `https://admin.findmyflowers.pl` | A |
| `Cors__AllowedOrigins__2` | `https://vendor.findmyflowers.pl` | A |

### Portals (Render env vars)

| Key | Value | Service |
|---|---|---|
| `ASPNETCORE_URLS` | `http://+:8080` | Ad, Ve |
| `ASPNETCORE_FORWARDEDHEADERS_ENABLED` | `true` | Ad, Ve |
| `ApiSettings__BaseUrl` | `https://api.findmyflowers.pl` | Ad, Ve |

VendorPortal additionally uses Keycloak for vendor login (confidential client `flowershop-vendor-portal`). Set `Authentication__Authority`, `Authentication__ClientId`, `Authentication__ClientSecret` the same way as CustomerApp, but with the vendor-portal client.

### Photo storage (post-EP-15)

| Key | Value | Consumer |
|---|---|---|
| `ObjectStorage__Provider` | `CloudflareR2` | A |
| `ObjectStorage__AccessKeyId` | R2 API token key | A |
| `ObjectStorage__SecretAccessKey` | R2 API token secret | A |
| `ObjectStorage__BucketName` | `flowershop-photos` | A |
| `ObjectStorage__PublicUrlPrefix` | `https://<bucket-id>.r2.cloudflarestorage.com` or Cloudflare public URL | A |

---

## GitHub Actions secrets

These drive the deploy workflows under `.github/workflows/`.

| Secret | Source | Used by |
|---|---|---|
| `FLY_API_TOKEN` | `fly tokens create deploy -x 999999h` | `deploy-api`, `deploy-keycloak`, `deploy-customer-app` |
| `RENDER_API_KEY` | Render → Account → API keys | `deploy-admin-portal`, `deploy-vendor-portal` |
| `RENDER_ADMIN_SERVICE_ID` | Render dashboard URL (`srv-xxxxx`) | `deploy-admin-portal` |
| `RENDER_VENDOR_SERVICE_ID` | Render dashboard URL | `deploy-vendor-portal` |
| `CLOUDFLARE_API_TOKEN` *(optional)* | Cloudflare → My profile → API Tokens → "Zone / DNS / Edit" | Any DNS automation workflow |
| `GITHUB_TOKEN` *(auto)* | Built-in | Push images to GHCR |

The following are optional GHA secrets used only during bootstrap (to `fly secrets set` in an ad-hoc workflow, instead of from your laptop). After bootstrap they can be deleted from GHA:

- `AIVEN_PG_CONNECTION`
- `UPSTASH_REDIS_URL`
- `CLOUDAMQP_URL`
- `HIVEMQ_HOST`, `HIVEMQ_USER`, `HIVEMQ_PASS`
- `KEYCLOAK_ADMIN_PASSWORD`
- `KEYCLOAK_CLIENT_SECRET_API`, `KEYCLOAK_CLIENT_SECRET_CUSTOMER`
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`

---

## Converting an Aiven URI to Npgsql format

Aiven gives you `postgres://user:pass@host:12345/defaultdb?sslmode=require`.
.NET Npgsql wants `Host=host;Port=12345;Database=defaultdb;Username=user;Password=pass;SslMode=Require`.

```bash
aiven_uri_to_npgsql() {
  python3 - <<PY
from urllib.parse import urlparse, unquote
u = urlparse("$1")
print(f"Host={u.hostname};Port={u.port};Database={u.path.lstrip('/')};Username={unquote(u.username)};Password={unquote(u.password)};SslMode=Require")
PY
}

aiven_uri_to_npgsql "postgres://avnadmin:xxx@xxx.aivencloud.com:12345/flowershop_iot?sslmode=require"
```

---

## Rotation procedure

Any compromised value, rotate in this order:

1. **Rotate at the source** (Keycloak client secret regen / Stripe key roll / Upstash password reset).
2. Update the value in every consumer's secret store (see Consumer column above).
3. For Fly apps: `fly secrets set ...` auto-rolls the machine.
4. For Render services: a redeploy picks up new env vars (`curl -X POST ...deploys` from the workflow is enough).
5. For GitHub Actions: overwrite the secret, then rerun the last deploy of every consumer workflow so the value is live.
6. Audit — `fly secrets list --app <name>` shows fingerprints; confirm they changed.

**Do not** delete the old value until the new one has been verified in prod — a rollback may need it.

---

## What never goes into a secret store

- Git history (even once, even in a revert — the commit is still reachable via SHA).
- `appsettings.*.json` in the repo.
- Docker image layers (baked-in env vars leak to anyone with pull access).
- Error logs / Sentry breadcrumbs — scrub before sending.
- Support-ticket attachments to SaaS vendors.
