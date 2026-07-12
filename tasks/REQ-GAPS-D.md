# Requirement Gaps — Option D (Showcase Deployment & Go-Live)

> Companion to REQ-GAPS.md (Mobile API, REQ-DB/AUTH/ORD/…), REQ-GAPS-B.md (Portals/Analytics/IoT,
> REQ-ANA/ADM/VND/CWA/IOT), and REQ-GAPS-C.md (Vendor onboarding/subscription/vase, REQ-VONB/VSUB/VORD/VASR/SSO).
> Requirement refs point at the tasks in `tasks-ep24-showcase-deployment.json`.
> Analysis date: 2026-07-06

> **Status (2026-07-12): SHIPPED.** The platform is fully live at `findmyflowers.pl`. The gaps below
> are historical. Two deltas from the criteria as written: **all three portals shipped on Fly.io, not
> Render** (so the "Render web service" / `RENDER_*` secret / cron-job.org keep-warm requirements do
> not apply — Fly `min_machines_running=1` keeps apps warm), and **the API runs in region `fra`** (not
> `iad`) with **real Keycloak auth** (`UseKeycloak=true`). See EPICS-D.md and SHOWCASE-RUNBOOK.md for
> the as-built state.

## Context

The platform is being taken live at **`findmyflowers.pl`** for a pilot showcase to a first flower
shop, with one physical smart vase to demonstrate. The design target is minimum cash cost
(~$10/mo Fly + free tiers elsewhere), accepting cold starts on non-critical surfaces.

A full deployment plan already exists (`docs/deployment/` — spec, runbook, secrets, registration,
rollback, observability) plus `fly.*.toml` and `.github/workflows/deploy-*.yml`. **Most of it is
written but not executed.** Verified live state (2026-07-06):

| Component | Status |
|---|---|
| API `flower-shop-backend-core` | ✅ Live on Fly (region `iad`), healthy at `flower-shop-backend-core.fly.dev`, running `Authentication__UseKeycloak=false` (dev-auth mode) |
| Postgres `flower-shop-postgres` | ✅ Live on Fly (chosen over the spec's Aiven) |
| R2 photo storage + `img.findmyflowers.pl` | ✅ Fully working (EP-15) |
| DNS zone `findmyflowers.pl` | ✅ On Cloudflare |
| Keycloak | ❌ Not deployed (`auth.findmyflowers.pl` → no response) |
| `api.findmyflowers.pl` custom domain | ❌ Not set (API only on `.fly.dev`) |
| AdminPortal / VendorPortal / CustomerApp | ❌ Not deployed |
| Stripe / MQTT prod config | ❌ Background services error each poll |

## Requested public URL layout (differs from existing docs)

| Surface | Requested (this epic) | As-built (actual) |
|---|---|---|
| Customer App | (plan floated the bare apex `findmyflowers.pl`) | **`app.findmyflowers.pl`** (plain subdomain CNAME) |
| Vendor Portal | **`vendors.findmyflowers.pl`** (plural) | `vendors.findmyflowers.pl` |
| Admin Portal | `admin.findmyflowers.pl` | `admin.findmyflowers.pl` ✓ |
| Public docs | **`docs.findmyflowers.pl`** | `docs.findmyflowers.pl` |
| API | `api.findmyflowers.pl` | `api.findmyflowers.pl` ✓ |
| Auth | `auth.findmyflowers.pl` | `auth.findmyflowers.pl` ✓ |

One divergence carried real work: **`docs.` is new** — the docs workflow publishes to GitHub Pages
but no custom domain is wired. (The plan initially floated a bare-apex customer domain, which would
have needed Cloudflare CNAME-flattening; as-built the Customer App shipped on the plain
`app.findmyflowers.pl` subdomain CNAME, so that wrinkle never materialised.)

---

## Code blockers (must merge before deploy)

### REQ-DEP-01 `[BUG]` Portals hardcode the Keycloak URL and client secret — P0

`FlowerShop.AdminPortal/Services/AuthenticationService.cs:30` and
`FlowerShop.VendorPortal/Services/AuthenticationService.cs:30` post to a hardcoded
`http://localhost:8090/realms/flowershop/protocol/openid-connect/token` with a hardcoded
`client_secret=flowershop-secret`. Both portals' login therefore cannot work in production.
Required: make the Keycloak base URL and client secret configuration-driven
(`Authentication:KeycloakBaseUrl`, `Authentication:ClientSecret`), defaulting to the dev values so
local behaviour is unchanged.

### REQ-DEP-02 `[BUG]` Production CORS is a wildcard that breaks credentialed calls — P0

`appsettings.Production.json` sets `Cors:AllowedOrigins: ["*"]`. The policy in
`ServiceCollectionExtensions.cs` intentionally drops `AllowCredentials()` on the wildcard branch,
so a wildcard prod config is both insecure and blocks credentialed browser requests from the
portals. Required: set the real production origins (app + admin + vendors) so the policy switches
to the explicit-allow-list branch with `AllowCredentials()`.

### REQ-DEP-03 `[CONFIG]` Production appsettings placeholders — P1

`appsettings.Production.json` still carries docker-compose defaults: `Host=postgres` connection
strings, `Redis=redis:6379`, `MQTT.BrokerHost=mqtt:1883`, and `JWT:SecretKey="…Change_This"`.
Fly secrets already override the DB (the API is healthy), but the remaining values must be real or
clearly env-sourced so nothing silently falls back to a docker hostname in prod.

---

## Deployment tasks

### REQ-DEP-04 `[MISSING]` Keycloak deployed at `auth.findmyflowers.pl` — P0

Nothing user-facing works until Keycloak is live and the API validates real tokens against it.
Required: a `keycloak` database on the existing Fly Postgres; a `flowershop-keycloak` Fly app
(image `keycloak:23`, ≥1 GB, `min_machines_running=1`, health `/health/ready`, `KC_PROXY=edge`,
`start --optimized --import-realm`); the committed realm `docker/keycloak/flowershop-realm.json`
imported; custom domain + cert via `fly certs add` and a grey-cloud Cloudflare CNAME; and the real
Google IdP client id/secret set so SSO works. The realm has clients `flowershop-api` and
`flowershop-customer-app` and a Google IdP; there is **no** dedicated admin/vendor client (both
portals authenticate through `flowershop-api` via password grant, per REQ-DEP-01).

### REQ-DEP-05 `[MISSING]` API custom domain `api.findmyflowers.pl` — P1

The API is reachable only at `flower-shop-backend-core.fly.dev`. Portals and the mobile app must
target a branded, stable host. Required: `fly certs add api.findmyflowers.pl` and a grey-cloud
Cloudflare CNAME to the `.fly.dev` host; verify `/health` → 200 on the custom domain.

### REQ-DEP-06 `[MISSING]` API switched to real Keycloak auth — P0

The API runs `Authentication__UseKeycloak=false` (dev-token mode). Required (only after REQ-DEP-04
verifies): set `Authentication__UseKeycloak=true` plus `Authority`, `ClientId`, `ClientSecret` Fly
secrets; confirm `POST /api/v1/auth/dev-token` → 404 and a real Keycloak-issued token is accepted
on a protected endpoint.

### REQ-DEP-07 `[MISSING]` Admin Portal deployed at `admin.findmyflowers.pl` — P0

Required: a Render web service (Docker, portal Dockerfile, root context) with env
`ApiSettings__BaseUrl=https://api.findmyflowers.pl` and
`Authentication__KeycloakBaseUrl=https://auth.findmyflowers.pl` (from REQ-DEP-01); custom domain
`admin.findmyflowers.pl` with a grey-cloud Cloudflare CNAME to the Render host; service ID recorded
as the `RENDER_ADMIN_SERVICE_ID` GitHub secret; Render auto-deploy disabled so the path-filtered
workflow owns deploys.

### REQ-DEP-08 `[MISSING]` Vendor Portal deployed at `vendors.findmyflowers.pl` — P0

As REQ-DEP-07 for the Vendor Portal, on the **plural** `vendors.findmyflowers.pl` host, with
`RENDER_VENDOR_SERVICE_ID` recorded. Note the divergence from existing docs/config that use
`vendor.` (singular) — DNS, CORS, and the deploy workflow references must all use `vendors.`.

### REQ-DEP-09 `[MISSING]` Customer App deployed at `app.findmyflowers.pl` — P0

Required: deploy the CustomerApp (Render or Fly) with OIDC env
(`Authentication__Authority`, `ClientId=flowershop-customer-app`, `ClientSecret`,
`PostLoginRedirectUri=https://app.findmyflowers.pl/`); serve it on **`app.findmyflowers.pl`** via a
plain subdomain CNAME to the Fly host (as-built — the plan first floated the bare apex, but that
would have needed Cloudflare CNAME-flattening, so a normal `app.` subdomain was used instead); and
register `https://app.findmyflowers.pl/signin-oidc` + post-logout `https://app.findmyflowers.pl/` as
valid redirect URIs on the `flowershop-customer-app` Keycloak client.

### REQ-DEP-10 `[MISSING]` CORS wired to the real showcase origins — P0

Depends on REQ-DEP-02, -08, -09. Set `Cors:AllowedOrigins` to
`https://findmyflowers.pl`, `https://admin.findmyflowers.pl`, `https://vendors.findmyflowers.pl`;
commit and deploy the API.

### REQ-DEP-11 `[MISSING]` Public docs at `docs.findmyflowers.pl` — P1

`.github/workflows/docs.yml` publishes `docs/` to a GitHub Pages repo but no custom domain is
wired. Required: add a `CNAME` (value `docs.findmyflowers.pl`) to the Pages publish, a Cloudflare
CNAME `docs → <owner>.github.io`, and enable the custom domain + enforce HTTPS on the Pages repo.

### REQ-DEP-12 `[MISSING]` MQTT pointed at HiveMQ Cloud for the vase demo — P1

The showcase includes one physical smart vase, so IoT ingest is in scope (not optional). Required:
create the HiveMQ Cloud free cluster + one backend credential + one vase credential; set
`MQTT__BrokerHost/Port/Username/Password/UseTls` Fly secrets on the API; confirm the API's MQTT
health/ingest works and the vase heartbeat lands.

### REQ-DEP-13 `[MISSING]` Stripe test-mode wired for the checkout demo — P2

Optional unless demoing checkout. Required: set `Stripe__SecretKey` (sk_test) +
`Stripe__WebhookSecret` Fly secrets and register the webhook at
`https://api.findmyflowers.pl/api/stripe/webhook`; confirm a test `payment_intent.succeeded`
delivery returns 200.

### REQ-DEP-14 `[MISSING]` Keep-warm & operational crons — P2

Render free sleeps after 15 min. Required: cron-job.org pings every 10 min on the Render portals
(and the API if desired), plus a weekly `pg_dump` → R2 backup job (Fly PG has no automated
backups).

### REQ-DEP-15 `[VERIFY]` End-to-end showcase smoke test — P1

Run the full path on the branded domains: `auth` realm → 200; `api/health` → 200 and `dev-token`
→ 404; admin login → dashboard; vendor login → bouquet upload with an `img.findmyflowers.pl` photo;
customer "Sign in with Google" → returns to `app.findmyflowers.pl` logged in → map loads; no CORS errors; and a
vase heartbeat visible in API logs. Green across all = go-live.

---

## Summary Table

> Status reflects the as-built deployment (2026-07-12) — all items shipped. Two diverged from the
> original wording: portals are on **Fly.io** (not Render) and the Customer App is on
> **`app.findmyflowers.pl`** (not the bare apex).

| ID | Area | Priority | Status |
|---|---|---|---|
| REQ-DEP-01 | Portals hardcode Keycloak URL + client secret | P0 | ✅ DONE |
| REQ-DEP-02 | Prod CORS wildcard breaks credentialed calls | P0 | ✅ DONE |
| REQ-DEP-03 | Prod appsettings placeholders (DB/Redis/MQTT/JWT) | P1 | ✅ DONE |
| REQ-DEP-04 | Keycloak deployed at auth.findmyflowers.pl | P0 | ✅ DONE |
| REQ-DEP-05 | API custom domain api.findmyflowers.pl | P1 | ✅ DONE |
| REQ-DEP-06 | API switched to real Keycloak auth | P0 | ✅ DONE |
| REQ-DEP-07 | Admin Portal at admin.findmyflowers.pl | P0 | ✅ DONE (Fly, not Render) |
| REQ-DEP-08 | Vendor Portal at vendors.findmyflowers.pl | P0 | ✅ DONE (Fly, not Render) |
| REQ-DEP-09 | Customer App at app.findmyflowers.pl | P0 | ✅ DONE (Fly; `app.`, not apex) |
| REQ-DEP-10 | CORS wired to real showcase origins | P0 | ✅ DONE |
| REQ-DEP-11 | Public docs at docs.findmyflowers.pl | P1 | ✅ DONE |
| REQ-DEP-12 | MQTT pointed at HiveMQ Cloud for vase demo | P1 | ✅ DONE |
| REQ-DEP-13 | Stripe test-mode wired for checkout demo | P2 | ✅ DONE |
| REQ-DEP-14 | Keep-warm & operational crons | P2 | ✅ DONE (Fly always-on + weekly R2 backup) |
| REQ-DEP-15 | End-to-end showcase smoke test | P1 | ✅ DONE |
