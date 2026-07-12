# FlowerShop Deployment Runbook — First Live Deployment

> **Audience**: operator performing the first production deploy of `findmyflowers.pl`.
> **Goal**: bring the platform up end-to-end, in order, with a health-check gate at every stage. No stage proceeds until the previous one passes.
> **Related docs**: [DEPLOYMENT-SPEC.md](DEPLOYMENT-SPEC.md) (topology rationale) · [REGISTRATION-AND-CICD.md](REGISTRATION-AND-CICD.md) (account creation) · [SECRETS-AND-ENV-VARS.md](SECRETS-AND-ENV-VARS.md) · [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md) · [ROLLBACK-AND-RECOVERY.md](ROLLBACK-AND-RECOVERY.md) · [OBSERVABILITY.md](OBSERVABILITY.md)

## How to use this runbook

- Work top-to-bottom. Every stage has **Pre-reqs → Do → Verify → Gate**.
- The **Gate** is a literal command whose success is the single criterion for moving on. If the gate fails, consult the *Troubleshoot* row, then retry — do not skip ahead.
- Commands assume `bash` (Git Bash on Windows is fine). All `$VAR` placeholders map to the secrets table in [SECRETS-AND-ENV-VARS.md](SECRETS-AND-ENV-VARS.md).
- Expected total time for a clean run: **~2.5 h** (most of it waiting for DNS propagation and Keycloak's first boot).

---

## Stage 0 — Pre-flight (T-24h)

The accounts and the code must be ready **before** you sit down to deploy.

| # | Check | How |
|---|---|---|
| 0.1 | All 11 accounts from [REGISTRATION-AND-CICD.md §2](REGISTRATION-AND-CICD.md#2-registration-order-3045-minutes-end-to-end) created | Walk that section |
| 0.2 | GitHub Actions secrets set | `Settings → Secrets and variables → Actions` — all rows from [SECRETS-AND-ENV-VARS.md §GitHub Actions](SECRETS-AND-ENV-VARS.md#github-actions-secrets) present |
| 0.3 | The five code changes from [DEPLOYMENT-SPEC.md §"Code changes required before first deploy"](DEPLOYMENT-SPEC.md) are merged to `main` | `git log --oneline main` contains those commits |
| 0.4 | `flyctl` installed and authenticated | `fly version && fly auth whoami` |
| 0.5 | `psql` available for DB smoke tests | `psql --version` |
| 0.6 | Stripe account in **test mode** for first deploy | dashboard shows orange "Test mode" banner |

**Gate 0:**
```bash
fly auth whoami && psql --version && gh secret list | grep -E 'FLY_API_TOKEN' | wc -l
# must print at least 1
```

> **Hosting reality:** everything runs on **Fly.io** in region **`fra`** — the API, all three portals, Keycloak, and PostgreSQL. There is no Render, Aiven, Upstash, or CloudAMQP. Redis and RabbitMQ are dev-only (docker-compose) and are **not provisioned in prod**.

---

## Stage 1 — Domain & DNS zone (T-2h)

DNS propagation is the slowest step — start it before anything else.

**Pre-reqs:** §0 green.

**Do:**
1. OVHcloud → register `findmyflowers.pl`.
2. Cloudflare → **Add site** `findmyflowers.pl` → note the two nameservers (e.g. `xxx.ns.cloudflare.com`, `yyy.ns.cloudflare.com`).
3. OVH → Domain → **DNS Servers** → replace OVH's with Cloudflare's two.
4. In Cloudflare zone, set **SSL/TLS → Overview → Full (strict)**. Under **Edge Certificates**, ensure "Always use HTTPS" is on.
5. Do **not** create subdomain CNAMEs yet — they point at apps we haven't created. Stage 8 adds them.

**Verify:**
```bash
dig +short NS findmyflowers.pl
# Expected: the two Cloudflare NS records (may take 15 min – 2 h to propagate)
```

**Gate 1:**
```bash
until dig +short NS findmyflowers.pl | grep -q cloudflare.com; do echo "waiting for NS propagation"; sleep 60; done
echo "NS delegated to Cloudflare — proceed"
```

| Troubleshoot | Fix |
|---|---|
| Still OVH NS after 2 h | Re-check OVH DNS servers panel; some OVH accounts require explicit "Apply changes" click |
| Cloudflare says "zone pending" | You forgot step 3 at OVH |

---

## Stage 2 — Data plane (T-90m)

Everything that stores state, in one pass. Order inside the stage does not matter — they're independent.

**Pre-reqs:** §1 green.

### 2a. PostgreSQL on Fly

**Do:** provision a Fly Postgres app `flower-shop-postgres` (PostgreSQL **17**), region `fra`, then create the two databases the platform needs: `flower_shop_backend_core` (app) and `keycloak`.

```bash
# Create the cluster (skip if it already exists).
fly postgres create --name flower-shop-postgres --region fra --vm-size shared-cpu-1x --initial-cluster-size 1

# Create the two databases (via psql over a local proxy).
fly proxy 15432:5432 -a flower-shop-postgres &
psql "postgres://postgres:$PG_PASSWORD@localhost:15432/postgres" -c "CREATE DATABASE flower_shop_backend_core;"
psql "postgres://postgres:$PG_PASSWORD@localhost:15432/postgres" -c "CREATE DATABASE keycloak;"
```

**Verify:** reach both DBs through the Fly proxy (don't commit any URI).
```bash
# flyctl proxy 15432:5432 -a flower-shop-postgres  must be running in another shell
psql "postgres://postgres:$PG_PASSWORD@localhost:15432/flower_shop_backend_core" -c "SELECT version();"
psql "postgres://postgres:$PG_PASSWORD@localhost:15432/keycloak"                  -c "SELECT 1;"
```

**Gate 2a:** both `psql` calls return without error. Note that within Fly's private network the DB is reached at `flower-shop-postgres.flycast:5432` — that is what Keycloak and the API use (Stages 3 & 5), not the local proxy.

### 2b. Redis — not provisioned in prod

Redis is a **dev-only** dependency (docker-compose.dev.yml). There is no managed Redis in production. `RedisCacheService` falls back to an in-memory distributed cache when `ConnectionStrings:Redis` is unset, so no action is required here. Skip to 2c.

### 2c. RabbitMQ — not provisioned in prod

RabbitMQ is likewise **dev-only**. Prod `appsettings` sets `EventBus:Type=RabbitMQ`, but `RabbitMQEventBus.PublishAsync` is a stub; the working event path in production is the **Outbox pattern**, with `InMemoryEventBus` as the effective in-process fallback. No managed broker is provisioned — skip to 2d.

### 2d. HiveMQ Cloud

**Do:** [REGISTRATION-AND-CICD.md §2.7](REGISTRATION-AND-CICD.md#27-hivemq-cloud-mqtt). This is the one external managed service — it replaces the dev-only Mosquitto broker.

**Verify:**
```bash
# Any MQTT client works; mosquitto_pub shown here. TLS on port 8883.
mosquitto_pub -h "$HIVEMQ_HOST" -p 8883 --cafile /etc/ssl/certs/ca-certificates.crt \
  -u "$HIVEMQ_USER" -P "$HIVEMQ_PASS" -t "test/deploy" -m "hello" -d
# Expected: "Client sent CONNECT" then "Client received CONNACK (0)"
```

**Gate 2:** Postgres (both DBs) and HiveMQ are reachable with the credentials you just saved.

---

## Stage 3 — Keycloak on Fly (T-60m)

Keycloak must come up **before** the API, because the API validates tokens against its JWKS URL.

**Pre-reqs:** §2a (needs the `keycloak` DB).

**Do:**

```bash
cd <repo root>

# 1. Launch the app (idempotent — skips create if it exists).
fly launch --no-deploy --copy-config --name flowershop-keycloak --region fra --yes

# 2. Set secrets. Keycloak reaches Postgres over Fly's private network (.flycast).
fly secrets set --app flowershop-keycloak \
  KC_DB_URL="jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak" \
  KC_DB_USERNAME="postgres" \
  KC_DB_PASSWORD="$PG_PASSWORD" \
  KEYCLOAK_ADMIN="admin" \
  KEYCLOAK_ADMIN_PASSWORD="$KEYCLOAK_ADMIN_PASSWORD"

# 3. Attach the custom hostname (cert provisioning happens async).
fly certs add auth.findmyflowers.pl --app flowershop-keycloak

# 4. Deploy.
fly deploy --config fly.keycloak.toml
```

**Verify — allow up to 2 min for first boot (JVM + schema migrate):**
```bash
# Step A: Fly health check passing (uses /health/ready set via KC_HEALTH_ENABLED).
fly status --app flowershop-keycloak | grep -E 'started|passing'

# Step B: via Fly's *.fly.dev domain while cert propagates.
curl -fsS https://flowershop-keycloak.fly.dev/health/ready
# Expected: {"status":"UP","checks":[...]}
```

**Gate 3:** `curl -fsS https://flowershop-keycloak.fly.dev/realms/master/.well-known/openid-configuration >/dev/null && echo OK` prints `OK`.

| Troubleshoot | Fix |
|---|---|
| `FATAL: role "..." does not exist` | Wrong `KC_DB_USERNAME`; the Fly Postgres superuser is `postgres` |
| Stuck in "starting" > 3 min | `fly logs -a flowershop-keycloak`; usually the DB isn't reachable — confirm the `flower-shop-postgres.flycast:5432/keycloak` host in `KC_DB_URL` and that both apps share the same Fly org/private network |
| `KC_HOSTNAME` mismatch warnings | Cert hasn't attached yet — keep deploying via `.fly.dev` until Stage 8 adds the CNAME |

---

## Stage 4 — Keycloak realm bootstrap

**Pre-reqs:** §3 green.

**Do:**
1. Browser → `https://flowershop-keycloak.fly.dev`, login as `admin / $KEYCLOAK_ADMIN_PASSWORD`.
2. **Realm** dropdown → **Create realm** → *Resource file* → upload `docker/keycloak/flowershop-realm.json` (committed during Stage 0). The realm ships with a **Google identity provider** already configured.
3. Verify the `flowershop-api` and `flowershop-customer-app` clients exist. The three portals do **not** have their own OIDC clients — they authenticate against the `flowershop-api` client using the password (Direct Access) grant.
4. For each **confidential** client (`flowershop-api`, `flowershop-customer-app`): **Credentials** tab → copy the secret.
5. Store as GitHub Actions secrets and also locally for the next stage:
   - `KEYCLOAK_CLIENT_SECRET_API`
   - `KEYCLOAK_CLIENT_SECRET_CUSTOMER`
6. In each client's **Settings**, update the redirect URIs to production hosts (they likely point at `localhost` in the dev export):
   - `flowershop-customer-app` → the customer app is served on `app.findmyflowers.pl`, so Valid redirect URIs: `https://app.findmyflowers.pl/signin-oidc`, Post logout: `https://app.findmyflowers.pl/`
   - `flowershop-api` → no browser redirects needed; confirm **Direct access grants** is **on** (the portals and dev tokens depend on it).

**Gate 4:**
```bash
curl -fsS https://flowershop-keycloak.fly.dev/realms/flowershop/.well-known/openid-configuration \
  | jq -r '.issuer, .jwks_uri, .token_endpoint'
# Expected: three non-null URLs under /realms/flowershop
```

---

## Stage 5 — API on Fly

**Pre-reqs:** §2 (all four data-plane items) + §4 green.

**Do:**

The API is the Fly app **`flower-shop-backend-core`**, deployed via the repo's **`fly.toml`** (region `fra`, 1 GB VM, always-on). Ignore the stale, unused `fly.api.toml` orphan.

```bash
fly launch --no-deploy --copy-config --name flower-shop-backend-core --region fra --yes

# Redis and RabbitMQ are NOT provisioned in prod — do not set ConnectionStrings__Redis
# or RabbitMQ__ConnectionString. The API's cache falls back to in-memory and the working
# event path is the Outbox pattern.
fly secrets set --app flower-shop-backend-core \
  ConnectionStrings__DefaultConnection="postgres://postgres:$PG_PASSWORD@flower-shop-postgres.flycast:5432/flower_shop_backend_core" \
  ConnectionStrings__ReadConnection="postgres://postgres:$PG_PASSWORD@flower-shop-postgres.flycast:5432/flower_shop_backend_core" \
  MQTT__BrokerHost="$HIVEMQ_HOST" \
  MQTT__BrokerPort="8883" \
  MQTT__UseTls="true" \
  MQTT__Username="$HIVEMQ_USER" \
  MQTT__Password="$HIVEMQ_PASS" \
  Authentication__UseKeycloak="true" \
  Authentication__Keycloak__Authority="https://auth.findmyflowers.pl/realms/flowershop" \
  Authentication__Keycloak__ClientId="flowershop-api" \
  Authentication__Keycloak__ClientSecret="$KEYCLOAK_CLIENT_SECRET_API" \
  Stripe__SecretKey="$STRIPE_SECRET_KEY" \
  Stripe__WebhookSecret="$STRIPE_WEBHOOK_SECRET" \
  FileStorage__Provider="R2" \
  FileStorage__R2__AccessKeyId="$R2_ACCESS_KEY_ID" \
  FileStorage__R2__SecretAccessKey="$R2_SECRET_ACCESS_KEY" \
  Cors__AllowedOrigins__0="https://findmyflowers.pl" \
  Cors__AllowedOrigins__1="https://admin.findmyflowers.pl" \
  Cors__AllowedOrigins__2="https://vendors.findmyflowers.pl"

fly certs add api.findmyflowers.pl --app flower-shop-backend-core

fly deploy --config fly.toml
```

Photos are stored in **Cloudflare R2** (bucket `flowershop-bouquets`, served publicly at `https://img.findmyflowers.pl`). Only the two R2 access-key secrets need setting here; the bucket name, public URL and provider live in committed config.

The API runs `Database.MigrateAsync()` on startup (Program.cs ~line 248, 10-attempt retry). Expect the first boot to spend ~30 s creating tables on the empty `flower_shop_backend_core` DB.

**Verify:**
```bash
# Logs should show "Applied migration" entries and then "Now listening on: http://[::]:8080".
fly logs --app flower-shop-backend-core | grep -E 'Applied migration|Now listening'

# Fly's built-in health check polls /health/ready every 15 s.
fly status --app flower-shop-backend-core | grep passing

# Hit /health/ready directly.
curl -fsS https://flower-shop-backend-core.fly.dev/health/ready | jq .
# Expected: {"status":"Healthy","results":{"...":{"status":"Healthy"},...}}
```

**Gate 5:** `/health/ready` returns `{"status":"Healthy"}` with **all** entries (DB, MQTT) also Healthy.

| Troubleshoot | Fix |
|---|---|
| `"status":"Unhealthy"` with DB entry red | Wrong connection string; confirm the `flower-shop-postgres.flycast:5432/flower_shop_backend_core` host, user `postgres`, and `$PG_PASSWORD` |
| Keycloak JWKS fetch fails | Keycloak cert not ready yet — use `https://flowershop-keycloak.fly.dev` in the Authority env var temporarily, redeploy, then switch back after Stage 8 |
| MQTT check "Unhealthy" | HiveMQ credentials — test with `mosquitto_pub` again (§2d) |

---

## Stage 6 — CustomerApp on Fly

**Pre-reqs:** §5 green.

**Do:**

The customer app is served on **`app.findmyflowers.pl`** (Fly app `flowershop-customer-app`, region `fra`).

```bash
fly launch --no-deploy --copy-config --name flowershop-customer-app --region fra --yes

fly secrets set --app flowershop-customer-app \
  Authentication__ClientId="flowershop-customer-app" \
  Authentication__ClientSecret="$KEYCLOAK_CLIENT_SECRET_CUSTOMER"

fly certs add app.findmyflowers.pl --app flowershop-customer-app

fly deploy --config fly.customer-app.toml
```

**Verify:**
```bash
curl -si https://flowershop-customer-app.fly.dev/ | head -n 1
# Expected: HTTP/2 200 OR HTTP/2 302 (302 if it redirects unauthenticated users to Keycloak)
```

**Gate 6:** status code is `200` or `302`. Navigating the `.fly.dev` URL in a browser loads the homepage or redirects cleanly to `auth.findmyflowers.pl`. Once DNS is attached (Stage 8) it's live at `app.findmyflowers.pl`.

---

## Stage 7 — Admin & Vendor portals on Fly

These are the last surfaces because they only matter once the API is serving. Both portals deploy to **Fly.io** (region `fra`), same as everything else — there is no Render.

**Pre-reqs:** §5 green.

**Do:** launch and deploy each portal from its committed Fly config. Fly keeps them warm via `min_machines_running=1`, so no external keep-warm cron is needed.

```bash
# Admin portal — app flowershop-admin-portal, domain admin.findmyflowers.pl
fly launch --no-deploy --copy-config --name flowershop-admin-portal --region fra --yes
fly certs add admin.findmyflowers.pl --app flowershop-admin-portal
flyctl deploy --config fly.admin-portal.toml

# Vendor portal — app flowershop-vendor-portal, domain vendors.findmyflowers.pl (PLURAL)
fly launch --no-deploy --copy-config --name flowershop-vendor-portal --region fra --yes
fly certs add vendors.findmyflowers.pl --app flowershop-vendor-portal
flyctl deploy --config fly.vendor-portal.toml
```

Both portals authenticate against the `flowershop-api` Keycloak client using the password grant — they have no OIDC client of their own, so no extra Keycloak secrets are needed here.

**Verify:** the portal health-check path is **`/Login`** (not `/health`).
```bash
curl -fsS https://flowershop-admin-portal.fly.dev/Login  -o /dev/null -w '%{http_code}\n'
curl -fsS https://flowershop-vendor-portal.fly.dev/Login -o /dev/null -w '%{http_code}\n'
```

**Gate 7:** both `/Login` return 200.

---

## Stage 8 — Attach subdomains

Now every surface is live on its `*.fly.dev` hostname — point DNS at them. Everything is on Fly, so every record follows Fly's DNS/TLS conventions.

**Pre-reqs:** §3, §5, §6, §7 green.

**Do:** in Cloudflare DNS, create these records (**Proxy status: DNS only** — orange cloud *off*, since Fly manages its own TLS). Every record is a plain subdomain CNAME to its `.fly.dev` host, including the customer app on `app`:

| Name | Target |
|---|---|
| `api` | `flower-shop-backend-core.fly.dev` |
| `auth` | `flowershop-keycloak.fly.dev` |
| `app` | `flowershop-customer-app.fly.dev` |
| `admin` | `flowershop-admin-portal.fly.dev` |
| `vendors` | `flowershop-vendor-portal.fly.dev` |

Wait for each host to issue its cert (Fly does this automatically once the record exists — 1–5 min typical).

**Verify:**
```bash
for host in api.findmyflowers.pl auth.findmyflowers.pl app.findmyflowers.pl admin.findmyflowers.pl vendors.findmyflowers.pl; do
  echo "=== $host ==="
  curl -sI "https://$host/" | head -n 1
done
```

**Gate 8:** every host returns `HTTP/2 200` or `HTTP/2 302` with a valid cert (no TLS errors).

| Troubleshoot | Fix |
|---|---|
| `SSL_ERROR_NO_CYPHER_OVERLAP` / cert still old | Cloudflare proxy is on — switch the record to DNS-only (grey cloud) |
| Fly says "awaiting certificate" > 10 min | `fly certs check <host> --app <app>` — usually a stale AAAA record left by the proxy |

---

## Stage 9 — Stripe webhook registration

**Pre-reqs:** §5, §8 green.

**Do:**
1. Stripe Dashboard → **Developers → Webhooks → Add endpoint**.
2. URL: `https://api.findmyflowers.pl/api/stripe/webhook`.
3. Events: `payment_intent.succeeded`, `payment_intent.payment_failed`, `charge.refunded`.
4. Copy the signing secret → set on Fly:
   ```bash
   fly secrets set --app flower-shop-backend-core Stripe__WebhookSecret="$NEW_WEBHOOK_SECRET"
   # fly auto-rolls the app on secret change; wait for passing status
   ```
5. In Stripe dashboard → **Send test webhook** → choose `payment_intent.succeeded` → click Send.

**Verify:**
```bash
# Tail API logs while the test fires. You want to see a 2xx response and NO signature errors.
fly logs --app flower-shop-backend-core | grep -iE 'stripe|webhook'
```

**Gate 9:** Stripe dashboard shows the test delivery with a `200` response.

---

## Stage 10 — Keep-warm & operational cron jobs

**Pre-reqs:** §8 green.

**Do:** no external keep-warm service is needed — every Fly app runs with `min_machines_running=1`, so machines stay warm without pinging. There is no cron-job.org dependency.

Database backup is already automated in-repo as **`.github/workflows/db-backup.yml`** — it runs every **Monday at 03:17 UTC**, does a `pg_dump -F c` of **both** databases (`flower_shop_backend_core` and `keycloak`), and uploads the dumps to Cloudflare R2. Restores use `pg_restore` (see [ROLLBACK-AND-RECOVERY.md §"Database backup"](ROLLBACK-AND-RECOVERY.md#database-backup)). Just confirm the workflow is present and its R2 secrets are set.

**Verify:**
```bash
# Confirm all Fly apps report min_machines_running >= 1.
for app in flower-shop-backend-core flowershop-admin-portal flowershop-vendor-portal flowershop-customer-app; do
  fly status --app "$app" | grep -E 'started'
done
# Confirm the backup workflow exists and has run (or run it once manually).
gh workflow view db-backup.yml
```

**Gate 10:** all apps show `started` machines and `gh workflow view db-backup.yml` resolves.

---

## Stage 11 — End-to-end smoke test

**Pre-reqs:** §1–§10 all green.

Run the scripted smoke suite in [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md). At minimum:

- [ ] `curl https://api.findmyflowers.pl/health/ready` → all Healthy
- [ ] Log in to admin portal as platform admin
- [ ] Create a vendor in admin portal; log in to vendor portal with the new vendor account
- [ ] Upload a bouquet in vendor portal (mock vase for now)
- [ ] Browse as anonymous customer on app; see bouquet in list/map
- [ ] Register a customer via Keycloak; complete a Stripe **test-mode** order end-to-end
- [ ] Publish a test MQTT message to `vase/<serial>/heartbeat`; confirm API log shows ingest

**Gate 11 (go-live):** all six check-boxes green.

---

## Timing summary

| Stage | Expected | Blocker if missing |
|---|---|---|
| 0 Pre-flight | prior day | Code changes merged, secrets in GH |
| 1 DNS | 15 min – 2 h wait | Nothing else can finish without NS propagated |
| 2 Data plane | 15 min | — |
| 3 Keycloak | 10 min | Fly Postgres must be up |
| 4 Realm | 10 min | — |
| 5 API | 10 min (inc. migrations) | Keycloak JWKS reachable |
| 6 CustomerApp | 5 min | — |
| 7 Portals | 10 min | Fly deploy of both portals |
| 8 DNS attach | 10–15 min (cert issue) | — |
| 9 Stripe | 5 min | — |
| 10 Crons | 5 min | — |
| 11 Smoke | 30 min | — |

**Clean-run wall clock:** ~2.5 h.

---

## One-line status snapshot

Useful when re-opening the runbook on day two:

```bash
echo "== Fly =="
fly status --app flower-shop-backend-core  | grep -E 'passing|failing'
fly status --app flowershop-keycloak       | grep -E 'passing|failing'
fly status --app flowershop-customer-app   | grep -E 'passing|failing'
fly status --app flowershop-admin-portal   | grep -E 'passing|failing'
fly status --app flowershop-vendor-portal  | grep -E 'passing|failing'
echo "== HTTP =="
for h in api.findmyflowers.pl auth.findmyflowers.pl app.findmyflowers.pl admin.findmyflowers.pl vendors.findmyflowers.pl; do
  printf "%-30s %s\n" "$h" "$(curl -s -o /dev/null -w '%{http_code}' https://$h/)"
done
```

All `passing` and all `200/302` = system is up.
