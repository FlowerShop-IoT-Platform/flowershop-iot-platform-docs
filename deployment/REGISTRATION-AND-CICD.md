# Registration Checklist & CI/CD Plan

> **Purpose**: step-by-step accounts to create, where each component lands, what it costs, and how CI deploys each one on change.
> **Prerequisites**: GitHub repo access, a payment card (Fly requires one even on free usage), the domain `findmyflowers.pl`.

Related documents:
- [DEPLOYMENT-SPEC.md](DEPLOYMENT-SPEC.md) — higher-level architecture rationale
- Diagrams: [topology](01-hosting-topology.svg) · [purchase flow](02-customer-purchase-flow.svg) · [IoT flow](03-iot-ingest-flow.svg) · [network/DNS](04-network-and-dns.svg)

---

## 1. Component → Host → Price Matrix

| # | Component | Public URL | Host | Plan / Machine | Monthly cost | What you register |
|---|-----------|------------|------|----------------|--------------|-------------------|
| 1 | `FlowerShop.API` (.NET 8, REST + SignalR + MQTT client + 5 hosted services + Stripe webhook) | `https://api.findmyflowers.pl` | **Fly.io** | `shared-cpu-1x` 1 GB, always-on, region `waw` | **~$5.70/mo** (offset by $5 Hobby credit) | Fly.io Hobby account + app `flowershop-api` |
| 2 | Keycloak 23 (OIDC + admin API) | `https://auth.findmyflowers.pl` | **Fly.io** | `shared-cpu-1x` 1 GB, always-on, region `waw` | **~$5.70/mo** | Fly.io app `flowershop-keycloak` |
| 3 | `FlowerShop.CustomerApp` (MVC + OIDC) | `https://app.findmyflowers.pl` | **Fly.io** | `shared-cpu-1x` 512 MB, `min_machines_running=1`, region `waw` | **~$3.89/mo** | Fly.io app `flowershop-customer-app` |
| 4 | `FlowerShop.AdminPortal` (Razor) | `https://admin.findmyflowers.pl` | **Render free** | Docker web service, 512 MB, sleeps after 15 min | **$0** | Render account + service `flowershop-admin-portal` |
| 5 | `FlowerShop.VendorPortal` (Razor) | `https://vendor.findmyflowers.pl` | **Render free** | Docker web service, 512 MB, sleeps after 15 min | **$0** | Render service `flowershop-vendor-portal` |
| 6 | PostgreSQL (DBs: `flowershop_iot`, `keycloak`) | internal — Aiven host | **Aiven** | Free PG-1, Frankfurt (`aws-eu-central-1`), 5 GB disk / 1 GB RAM | **$0** | Aiven account + service `flowershop-pg` |
| 7 | Redis cache | internal — Upstash REST/TCP | **Upstash** | Free serverless, 500k cmd/mo, 256 MB | **$0** | Upstash account + DB `flowershop-cache` |
| 8 | RabbitMQ (event bus) | internal — CloudAMQP | **CloudAMQP** | Little Lemur (free), 1M msg/mo, 20 conn | **$0** | CloudAMQP account + instance `flowershop-bus` |
| 9 | MQTT broker | `<cluster>.hivemq.cloud:8883` | **HiveMQ Cloud** | Serverless free, 100 conn, 10 GB/mo | **$0** | HiveMQ Cloud account + cluster `flowershop-mqtt` |
| 10 | Bouquet photo storage | internal — R2 API | **Cloudflare R2** | 10 GB free, zero egress | **$0** | Cloudflare account + R2 bucket `flowershop-photos` |
| 11 | Domain `findmyflowers.pl` | DNS zone | **OVHcloud** | .pl registration | **~10 PLN y1 / ~65 PLN renew** (~€15/y) | OVHcloud account + domain |
| 12 | Keep-warm cron (ping Render portals every 10 min) | — | **cron-job.org** | Free | **$0** | cron-job.org account + 2 jobs |
| 13 | Container registry | `ghcr.io/<owner>/<image>` | **GitHub Container Registry** | Free for public repos / 500 MB for private | **$0** | Already part of your GitHub org |

**Totals**: **~$15.30/mo** Fly-side **− $5 Hobby credit = ~$10.30/mo cash** + **~€15/yr** domain. Everything else: $0.

---

## 2. Registration order (30–45 minutes end to end)

Do them in this order — each step gives you a credential the next step needs.

### 2.1 GitHub (you already have this)
Confirm the repo is pushed. Enable **Actions** in Settings → Actions → General → "Allow all actions".

### 2.2 Cloudflare (account + DNS)
1. Sign up at https://dash.cloudflare.com.
2. Add site `findmyflowers.pl` (even before you've bought it — Cloudflare will give you two nameservers).
3. **Noted for later**: you'll paste these NS at OVH.
4. While here: create API token (scoped to Zone → DNS → Edit) for the GHA workflow that sets DNS records. Store as `CLOUDFLARE_API_TOKEN`.
5. (Later, after R2 is needed) Enable R2, create bucket `flowershop-photos`.

### 2.3 Domain — OVHcloud
1. Sign up at https://www.ovhcloud.com/en/domains/.
2. Register `findmyflowers.pl` (~10 PLN first year).
3. In the OVH control panel, change nameservers to the two Cloudflare ones from step 2.2.
4. Propagation: 15 min – 2 h.

### 2.4 Aiven (PostgreSQL)
1. Sign up at https://console.aiven.io.
2. Create service → PostgreSQL → **Free plan** → cloud `AWS`, region `eu-central-1` (Frankfurt).
3. Service name: `flowershop-pg`.
4. Wait ~5 min for `RUNNING`.
5. On the service page, **Databases** tab → add second database named `keycloak` (the default one `defaultdb` will be for the app; create a third named `flowershop_iot` if you prefer symmetric naming).
6. Grab the **service URI** (looks like `postgres://avnadmin:xxx@xxx.aivencloud.com:12345/defaultdb?sslmode=require`).
7. Create two users if you want isolation (`flowershop_user`, `keycloak_user`) — not required for demo.

### 2.5 Upstash (Redis)
1. Sign up at https://console.upstash.com.
2. Create database `flowershop-cache`, region `eu-central-1`, TLS enabled.
3. Copy the `rediss://...` connection string.

### 2.6 CloudAMQP (RabbitMQ)
1. Sign up at https://www.cloudamqp.com.
2. Create instance → **Little Lemur (Free)** → region `EU-West-1` (Ireland) or `EU-Central-1`.
3. Instance name: `flowershop-bus`.
4. Copy the `amqps://...` URL from the instance details page.

### 2.7 HiveMQ Cloud (MQTT)
1. Sign up at https://www.hivemq.com/mqtt-cloud-broker/.
2. Create **Serverless** cluster, region EU.
3. Cluster name: `flowershop-mqtt`.
4. In **Access Management**, create a credential for the backend (`flowershop-backend`) and one per vase (or one shared for demo).
5. Copy cluster URL (e.g. `xxx.s1.eu.hivemq.cloud`), port `8883`.

### 2.8 Fly.io
1. Sign up at https://fly.io. **Add a payment card.** Free usage stays inside the $5/mo Hobby credit.
2. Install `flyctl` locally: https://fly.io/docs/flyctl/install/.
3. `fly auth login`.
4. Generate a deploy token: `fly tokens create deploy -x 999999h` → store as GitHub secret `FLY_API_TOKEN`.
5. Don't create apps manually — the GHA workflows do it with `fly launch --copy-config --no-deploy` on first run.

### 2.9 Render
1. Sign up at https://render.com (GitHub OAuth).
2. In **Account Settings → API Keys**, create key → store as GitHub secret `RENDER_API_KEY`.
3. Don't create services manually. The GHA workflow creates them via API on first run, or you can create them once through the dashboard and then let GHA deploy on push — see §4.2 for both options. The dashboard path is simpler; pick that unless you want full IaC.

### 2.10 cron-job.org
1. Sign up at https://cron-job.org.
2. After portals are live, add two jobs pinging `https://admin.findmyflowers.pl/health` and `https://vendor.findmyflowers.pl/health` every 10 min. (CustomerApp stays warm via Fly's `min_machines_running=1`.)

### 2.11 GitHub secrets to set (one place)
After 2.1–2.9, go to `Settings → Secrets and variables → Actions` and set:

| Secret | Source | Used by |
|---|---|---|
| `FLY_API_TOKEN` | §2.8 | API, Keycloak, CustomerApp workflows |
| `RENDER_API_KEY` | §2.9 | AdminPortal, VendorPortal workflows |
| `RENDER_ADMIN_SERVICE_ID` | Render dashboard after creating service | AdminPortal workflow |
| `RENDER_VENDOR_SERVICE_ID` | Render dashboard after creating service | VendorPortal workflow |
| `CLOUDFLARE_API_TOKEN` | §2.2 | DNS workflow (optional) |
| `AIVEN_PG_CONNECTION` | §2.4, paste URI | Fly `secrets set` bootstrap |
| `UPSTASH_REDIS_URL` | §2.5 | Fly secrets |
| `CLOUDAMQP_URL` | §2.6 | Fly secrets |
| `HIVEMQ_HOST`, `HIVEMQ_USER`, `HIVEMQ_PASS` | §2.7 | Fly secrets |
| `KEYCLOAK_ADMIN_PASSWORD` | you pick | Keycloak workflow |
| `KEYCLOAK_CLIENT_SECRET` | after realm import | API + CustomerApp secrets |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | Stripe dashboard | API secrets |

---

## 3. Path-triggered CI/CD

GHA workflows live under `.github/workflows/`. Each is triggered by **changes to the files that component compiles from**, so a portal-only change doesn't redeploy the API and vice-versa.

### 3.1 Path filters per component

| Workflow file | Triggers on push to `main` when these paths change |
|---|---|
| `deploy-api.yml` | `src/backend/**`, `Dockerfile`, `.github/workflows/deploy-api.yml` |
| `deploy-admin-portal.yml` | `src/portals/FlowerShop.AdminPortal/**`, `.github/workflows/deploy-admin-portal.yml` |
| `deploy-vendor-portal.yml` | `src/portals/FlowerShop.VendorPortal/**`, `.github/workflows/deploy-vendor-portal.yml` |
| `deploy-customer-app.yml` | `src/portals/FlowerShop.CustomerApp/**`, `.github/workflows/deploy-customer-app.yml` |
| `deploy-keycloak.yml` | `infra/keycloak/**` (realm exports, Dockerfile), `.github/workflows/deploy-keycloak.yml` |

Shared code in `src/backend/FlowerShop.Domain`, `.Application`, `.Infrastructure` sits under `src/backend/**` — any change there triggers **only the API workflow** (which is correct; portals don't reference backend assemblies, they call the API over HTTP).

Each workflow also has `workflow_dispatch:` so you can trigger a manual redeploy from the Actions UI.

### 3.2 Workflow outputs / expected duration

| Workflow | Typical runtime | Deploy target |
|---|---|---|
| `deploy-api.yml` | ~4–6 min | Fly.io app `flowershop-api` |
| `deploy-keycloak.yml` | ~3 min | Fly.io app `flowershop-keycloak` |
| `deploy-customer-app.yml` | ~3–4 min | Fly.io app `flowershop-customer-app` |
| `deploy-admin-portal.yml` | ~2–3 min | Render service `flowershop-admin-portal` |
| `deploy-vendor-portal.yml` | ~2–3 min | Render service `flowershop-vendor-portal` |

Fly uses `--remote-only` (build on Fly's builders); Render pulls from GHCR image.

---

## 4. First-time bootstrap sequence

Run once, then CI takes over.

### 4.1 Fly apps
```bash
# For each of: flowershop-api, flowershop-keycloak, flowershop-customer-app
fly launch --no-deploy --copy-config --name flowershop-api --region waw
fly secrets set \
  ConnectionStrings__DefaultConnection="$AIVEN_PG_CONNECTION" \
  ConnectionStrings__Redis="$UPSTASH_REDIS_URL" \
  RabbitMQ__ConnectionString="$CLOUDAMQP_URL" \
  MQTT__BrokerHost="$HIVEMQ_HOST" \
  MQTT__Username="$HIVEMQ_USER" \
  MQTT__Password="$HIVEMQ_PASS" \
  Authentication__Keycloak__ClientSecret="$KEYCLOAK_CLIENT_SECRET" \
  Stripe__SecretKey="$STRIPE_SECRET_KEY" \
  Stripe__WebhookSecret="$STRIPE_WEBHOOK_SECRET" \
  --app flowershop-api

fly certs add api.findmyflowers.pl --app flowershop-api
# Repeat pattern for keycloak and customer-app with the right secrets
```

Add DNS CNAMEs in Cloudflare: `api`, `auth`, `app` → `<app>.fly.dev`.

### 4.2 Render services (dashboard path — recommended)
1. New → **Web Service** → connect repo → select `src/portals/FlowerShop.AdminPortal/Dockerfile`, root dir `.` (build context is repo root).
2. Plan: Free. Region: Frankfurt.
3. Environment vars:
   - `ASPNETCORE_URLS=http://+:8080`
   - `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`
   - `ApiSettings__BaseUrl=https://api.findmyflowers.pl`
4. Custom domain: `admin.findmyflowers.pl`.
5. Once created, copy the service ID from the URL (e.g. `srv-xxxxx`) → set as `RENDER_ADMIN_SERVICE_ID` in GitHub secrets.
6. Disable **Auto-Deploy** in Render dashboard — GHA will trigger deploys via API so path filtering works. Render's own auto-deploy fires on any push, which defeats the point.
7. Repeat for VendorPortal.

### 4.3 Keycloak realm bootstrap
1. After first `deploy-keycloak.yml` run, `fly ssh console -a flowershop-keycloak`.
2. Import the realm: upload `infra/keycloak/flowershop-realm.json` via admin UI at `https://auth.findmyflowers.pl`, or pre-mount via `KC_IMPORT_REALM=true` + `start --import-realm`.
3. After import, read the `flowershop-api` client secret from Credentials tab → set as `KEYCLOAK_CLIENT_SECRET` in GitHub secrets → re-run API + CustomerApp workflows to pick it up.

---

## 5. Workflow files — see `.github/workflows/`

- [`deploy-api.yml`](../../.github/workflows/deploy-api.yml)
- [`deploy-keycloak.yml`](../../.github/workflows/deploy-keycloak.yml)
- [`deploy-customer-app.yml`](../../.github/workflows/deploy-customer-app.yml)
- [`deploy-admin-portal.yml`](../../.github/workflows/deploy-admin-portal.yml)
- [`deploy-vendor-portal.yml`](../../.github/workflows/deploy-vendor-portal.yml)

Each one:
1. Checks out the repo
2. (Render portals) Builds the Docker image and pushes to GHCR
3. (Fly apps) Runs `flyctl deploy --remote-only --config <app>.fly.toml`
4. (Render portals) Triggers Render deploy via `POST https://api.render.com/v1/services/{id}/deploys`
5. Waits for health check

---

## 6. What to do today (go-live checklist)

- [ ] Register domain at OVH (§2.3)
- [ ] Create Cloudflare account + add zone (§2.2)
- [ ] Create Aiven PG (§2.4)
- [ ] Create Upstash Redis (§2.5)
- [ ] Create CloudAMQP instance (§2.6)
- [ ] Create HiveMQ cluster (§2.7)
- [ ] Create Fly.io account, install flyctl, generate token (§2.8)
- [ ] Create Render account, generate API key (§2.9)
- [ ] Set all GitHub secrets (§2.11)
- [ ] Commit the 5 workflow files + `fly.*.toml` configs
- [ ] Run `fly launch` for each of 3 Fly apps (§4.1)
- [ ] Create 2 Render services via dashboard + copy service IDs (§4.2)
- [ ] Push a change → watch path-filtered workflow deploy only the affected component
- [ ] Add Cloudflare CNAMEs for all 5 subdomains
- [ ] Add cron-job.org pings for admin + vendor portals (§2.10)
- [ ] Bootstrap Keycloak realm (§4.3)
- [ ] Smoke test: login via CustomerApp, list bouquets, place a test order with Stripe test key
