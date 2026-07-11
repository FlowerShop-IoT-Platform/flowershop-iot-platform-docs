# Showcase Deployment Runbook

> Goal: take the platform from "API + R2 live on `.fly.dev`" to a **full showcase**
> — real Keycloak login and all three portals on branded `findmyflowers.pl` URLs.
>
> **Companion to** `DEPLOYMENT-SPEC.md` (topology + rationale). This file is the
> ordered, executable checklist. Do the phases in order — each depends on the previous.

**Last updated:** 2026-07-06

---

## Current state (verified 2026-07-06)

| Component | Status |
|---|---|
| API `flower-shop-backend-core` | ✅ Deployed on Fly (region `iad`), healthy at `flower-shop-backend-core.fly.dev`. Running `Authentication__UseKeycloak=false` (dev-auth mode). |
| Postgres `flower-shop-postgres` | ✅ Deployed on Fly (chosen over Aiven from the spec) |
| R2 photo storage + `img.findmyflowers.pl` | ✅ Fully working (EP-15) |
| DNS zone `findmyflowers.pl` | ✅ On Cloudflare (nameservers nora/owen) |
| Keycloak | ❌ Not deployed (`auth.findmyflowers.pl` → 000) |
| `api.findmyflowers.pl` custom domain | ❌ Not set (API only on `.fly.dev`) |
| AdminPortal / VendorPortal / CustomerApp | ❌ Not deployed (`admin`/`vendor`/`app` → 000) |
| Stripe / MQTT prod config | ❌ Background services erroring each poll |

**Realm facts** (`docker/keycloak/flowershop-realm.json`): realm `flowershop`; clients
`flowershop-api`, `flowershop-customer-app`, `ESP32-40C86C`; Google IdP `google`.
There is **no** dedicated admin/vendor client — both portals authenticate through
the `flowershop-api` client via password grant.

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

## Phase 4 — Deploy the three portals to Render

Each has a Dockerfile (`src/portals/<Portal>/Dockerfile`, EXPOSE 8080). Repeat per portal.
Ref: `DEPLOYMENT-SPEC.md` §"Render — each portal".

For **each** of AdminPortal / VendorPortal / CustomerApp:
1. Render → **New Web Service** → connect this repo → **Docker** runtime → set the Dockerfile path
   to the portal's Dockerfile (root context, so it can COPY the shared projects).
2. Env vars (common):
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
3. Add the custom domain in Render (`admin` / `vendor` / `app`.findmyflowers.pl) → Render gives a
   `CNAME` target → add it in **Cloudflare DNS**, proxy **OFF** (grey) so Render's TLS works.
4. **CustomerApp:** register `https://app.findmyflowers.pl/signin-oidc` as a valid redirect URI on
   the `flowershop-customer-app` client in Keycloak (and post-logout `.../signout-callback-oidc`).
5. Keep-warm: add a cron-job.org ping every 10 min on CustomerApp + API (Render free sleeps at 15 min).

---

## Phase 5 — Wire CORS to the real origins

1. In `appsettings.Production.json` set:
   ```json
   "Cors": { "AllowedOrigins": [
     "https://app.findmyflowers.pl",
     "https://admin.findmyflowers.pl",
     "https://vendor.findmyflowers.pl"
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
- [ ] Vendor portal login → bouquet upload → `photoUrl` is `img.findmyflowers.pl/...` and image loads
- [ ] Customer app → "Sign in with Google" → returns to `app.findmyflowers.pl` logged in → map loads
- [ ] No CORS errors in browser console on any portal

---

## Known risks (from DEPLOYMENT-SPEC.md, still apply)

Fly free-VM eviction · Render cold-start chain (~90 s first login) · CloudAMQP 20-conn cap ·
HiveMQ 100-device cap · Aiven/Fly PG has no auto-backups (weekly `pg_dump` → R2) ·
Keycloak realm drift dev↔prod (commit the realm export; the showcase imports it).
