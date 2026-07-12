# Showcase Deployment Runbook

> Goal: take the platform from "API + R2 live on `.fly.dev`" to a **full showcase**
> — real Keycloak login and all three portals on branded `findmyflowers.pl` URLs.
>
> **Companion to** `DEPLOYMENT-SPEC.md` (topology + rationale). This file is the
> ordered, executable checklist. Do the phases in order — each depends on the previous.

**Last updated:** 2026-07-12 (go-live complete)

---

## Current state (all phases DONE)

| Component | Status |
|---|---|
| API `flower-shop-backend-core` | ✅ Deployed on Fly (region `fra`), live at `api.findmyflowers.pl`. Running `Authentication__UseKeycloak=true` (real auth). |
| Postgres `flower-shop-postgres` | ✅ Deployed on Fly, PostgreSQL 17 (chosen over Aiven from the spec) |
| R2 photo storage + `img.findmyflowers.pl` | ✅ Fully working (EP-15) |
| DNS zone `findmyflowers.pl` | ✅ On Cloudflare (nameservers nora/owen) |
| Keycloak | ✅ Deployed as Fly app `flowershop-keycloak` at `auth.findmyflowers.pl` |
| `api.findmyflowers.pl` custom domain | ✅ Live |
| AdminPortal / VendorPortal / CustomerApp | ✅ Deployed on Fly at `admin.` / `vendors.` (plural) / `app.findmyflowers.pl` |
| Stripe / MQTT prod config | ✅ Stripe wired; MQTT on HiveMQ Cloud (TLS 8883) |

**Realm facts** (`docker/keycloak/flowershop-realm.json`): realm `flowershop`; clients
`flowershop-api`, `flowershop-customer-app`, `ESP32-40C86C`; Google IdP `google`.
There is **no** dedicated admin/vendor client — both portals authenticate through
the `flowershop-api` client via password grant.

> This runbook is preserved as the go-live record. The phases below describe what was
> done, in order; they are all complete.

---

## Phase 0 — Prerequisites & blockers to fix in code first

These are hardcoded dev values that will break in prod. Fix + merge before deploying portals.

- [ ] **AdminPortal & VendorPortal hardcode Keycloak URL.**
  `Services/AuthenticationService.cs` posts to `http://localhost:8090/realms/flowershop/...`.
  Make the Keycloak base URL configurable (e.g. `Authentication:KeycloakBaseUrl`), defaulting
  to localhost for dev. Without this, portal login fails in prod even after Keycloak is up.
- [ ] **CORS.** `appsettings.Production.json` → `Cors:AllowedOrigins` must list the real portal
  origins (see Phase 4). Empty/`*` currently falls back to `AllowAnyOrigin` — insecure for a
  public showcase, and blocks credentialed requests.
- [ ] Confirm a `keycloak` database exists on `flower-shop-postgres` (Phase 1 creates it).

---

## Phase 1 — Keycloak on `auth.findmyflowers.pl`  🔴 critical blocker

Nothing user-facing works until this is done. Ref: `DEPLOYMENT-SPEC.md` §"Fly — Keycloak".

1. **Create the `keycloak` database** on the existing Fly Postgres:
   ```bash
   fly postgres connect -a flower-shop-postgres
   # in psql:
   CREATE DATABASE keycloak;
   \q
   ```
2. **Create a Keycloak Fly app** (separate app, so it scales/roll independently):
   ```bash
   fly apps create flowershop-keycloak
   ```
   Author a `fly.keycloak.toml` (image `quay.io/keycloak/keycloak:23.0`, internal port 8080,
   health check `/health/ready`, ≥1 GB VM — JVM needs headroom, and `min_machines_running=1`
   so logins aren't cold-started).
3. **Set secrets/env** (mirror `docker-compose.dev.yml` lines 130-140, but prod hostname + HTTPS):
   ```bash
   fly secrets set -a flowershop-keycloak \
     KC_DB=postgres \
     KC_DB_URL="jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak" \
     KC_DB_USERNAME=postgres \
     KC_DB_PASSWORD=<pg password> \
     KC_HOSTNAME=auth.findmyflowers.pl \
     KC_PROXY=edge \
     KEYCLOAK_ADMIN=admin \
     KEYCLOAK_ADMIN_PASSWORD=<strong password>
   ```
   Start command: `start --optimized --import-realm` (production mode, not `start-dev`).
   Mount/bake `docker/keycloak/flowershop-realm.json` into `/opt/keycloak/data/import`.
4. **Custom domain + cert:**
   ```bash
   fly certs add auth.findmyflowers.pl -a flowershop-keycloak
   ```
   Then in **Cloudflare DNS** add a `CNAME auth → flowershop-keycloak.fly.dev`, proxy **OFF**
   (grey cloud — Fly terminates TLS itself; orange-cloud proxying double-terminates and breaks OIDC).
5. **Google IdP secret:** the realm imports the Google IdP but its client secret comes from env
   (see `docker/keycloak/README.md`). Set the real Google client-id/secret so SSO works.
6. **Verify:**
   ```bash
   curl -sI https://auth.findmyflowers.pl/realms/flowershop   # expect 200
   ```
   Confirm clients `flowershop-api` + `flowershop-customer-app` and the Google IdP exist in the
   admin console. **Update the Google Cloud OAuth consent** redirect URI to
   `https://auth.findmyflowers.pl/realms/flowershop/broker/google/endpoint`.

---

## Phase 2 — `api.findmyflowers.pl` custom domain

Portals + mobile point here, not the raw `.fly.dev`.

1. ```bash
   fly certs add api.findmyflowers.pl -a flower-shop-backend-core
   ```
2. **Cloudflare DNS:** `CNAME api → flower-shop-backend-core.fly.dev`, proxy **OFF** (grey).
3. Verify: `curl -sI https://api.findmyflowers.pl/health` → `200`.

---

## Phase 3 — Flip the API to real auth  🔴

Only after Phase 1 is verified (else the API can't validate any token and everything 401s).

1. ```bash
   fly secrets set -a flower-shop-backend-core \
     Authentication__UseKeycloak=true \
     Authentication__Keycloak__Authority=https://auth.findmyflowers.pl/realms/flowershop \
     Authentication__Keycloak__ClientId=flowershop-api \
     Authentication__Keycloak__ClientSecret=<from realm/admin console>
   ```
2. Deploy picks these up automatically (or `fly deploy`). This also **closes the dev-token
   endpoint** via both guards (`UseKeycloak=true` AND the new `IsDevelopment()` gate).
3. Verify: `POST /api/v1/auth/dev-token` → `404`; a real Keycloak-issued token is accepted on a
   protected endpoint.

---

## Phase 4 — Deploy the three portals to Fly

Each has a Dockerfile (`src/portals/<Portal>/Dockerfile`, EXPOSE 8080) and a committed Fly config
(`fly.admin-portal.toml`, `fly.vendor-portal.toml`, `fly.customer-app.toml`, all region `fra`,
`min_machines_running=1`). Repeat per portal.
Ref: `DEPLOYMENT-SPEC.md` §"Fly — each portal".

For **each** of AdminPortal / VendorPortal / CustomerApp:
1. `fly apps create <app>` (`flowershop-admin-portal` / `flowershop-vendor-portal` / `flowershop-customer-app`),
   then `flyctl deploy --config fly.<portal>.toml` (GHA `deploy-<portal>.yml` does this on push).
2. Secrets/env (common):
   ```
   ASPNETCORE_URLS=http://+:8080
   ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
   ApiSettings__BaseUrl=https://api.findmyflowers.pl
   ```
   **AdminPortal / VendorPortal also need** the Keycloak base URL from Phase 0's fix:
   ```
   Authentication__KeycloakBaseUrl=https://auth.findmyflowers.pl
   ```
   **CustomerApp also needs** (OIDC — reads `Authentication:*` in its Program.cs):
   ```
   Authentication__Authority=https://auth.findmyflowers.pl/realms/flowershop
   Authentication__ClientId=flowershop-customer-app
   Authentication__ClientSecret=<secret>
   Authentication__PostLoginRedirectUri=https://app.findmyflowers.pl/
   ```
3. `fly certs add <host> -a <app>` for `admin.` / `vendors.` (plural) / `app.findmyflowers.pl` → add
   the matching record in **Cloudflare DNS**, proxy **OFF** (grey) so Fly's TLS works.
4. **CustomerApp:** register `https://app.findmyflowers.pl/signin-oidc` as a valid redirect URI on
   the `flowershop-customer-app` client in Keycloak (and post-logout `.../signout-callback-oidc`).
5. No keep-warm cron needed — `min_machines_running=1` keeps every Fly app awake.

---

## Phase 5 — Wire CORS to the real origins

1. In `appsettings.Production.json` set:
   ```json
   "Cors": { "AllowedOrigins": [
     "https://findmyflowers.pl",
     "https://admin.findmyflowers.pl",
     "https://vendors.findmyflowers.pl"
   ] }
   ```
   (The API reads this into `FlowerShopPolicy`; a non-empty list without `*` switches it from
   `AllowAnyOrigin` to the explicit allow-list.)
2. Commit + `fly deploy`.

---

## Phase 6 — Optional: Stripe & MQTT (only if demoing checkout / IoT)

Not blockers for a login+browse showcase, but they error every poll cycle until set.
- **Stripe:** `fly secrets set Stripe__SecretKey=sk_test_... Stripe__WebhookSecret=whsec_...`
  (test-mode is fine for a showcase). Point the Stripe webhook at `https://api.findmyflowers.pl/api/stripe/webhook`.
- **MQTT:** point `MQTT__BrokerHost`/`Port`/`Username`/`Password`/`UseTls` at HiveMQ Cloud
  (spec's chosen broker) instead of dev `mqtt:1883`. Only needed for vase/IoT demos.

---

## Final end-to-end verification

- [ ] `https://auth.findmyflowers.pl/realms/flowershop` → 200
- [ ] `https://api.findmyflowers.pl/health` → 200; `dev-token` → 404
- [ ] Admin portal login (platformadmin) → dashboard loads, data via `api.findmyflowers.pl`
- [ ] Vendor portal (`vendors.findmyflowers.pl`) login → bouquet upload → `photoUrl` is `img.findmyflowers.pl/...` and image loads
- [ ] Customer app (`app.findmyflowers.pl`) → "Sign in with Google" → returns to `app.findmyflowers.pl` logged in → map loads
- [ ] No CORS errors in browser console on any portal

---

## Known risks (from DEPLOYMENT-SPEC.md, still apply)

Fly VM eviction (mitigated by `min_machines_running=1`) · HiveMQ 100-device cap ·
Fly PG backups handled by `.github/workflows/db-backup.yml` (weekly `pg_dump -F c` of both DBs → R2) ·
Keycloak realm drift dev↔prod (realm export committed at `docker/keycloak/flowershop-realm.json`; the showcase imports it).
