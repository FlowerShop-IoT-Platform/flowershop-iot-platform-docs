# FlowerShop Deployment Runbook — First Live Deployment

> **Audience**: operator performing the first production deploy of `findmyflowers.pl`.
> **Goal**: bring the platform up end-to-end, in order, with a health-check gate at every stage. No stage proceeds until the previous one passes.
> **Related docs**: [DEPLOYMENT-SPEC.md](DEPLOYMENT-SPEC.md) (topology rationale) · [REGISTRATION-AND-CICD.md](REGISTRATION-AND-CICD.md) (account creation) · [SECRETS-AND-ENV-VARS.md](SECRETS-AND-ENV-VARS.md) · [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md) · [ROLLBACK-AND-RECOVERY.md](ROLLBACK-AND-RECOVERY.md) · [OBSERVABILITY.md](OBSERVABILITY.md)

## How to use this runbook

- Work top-to-bottom. Every stage has **Pre-reqs → Do → Verify → Gate**.
- The **Gate** is a literal command whose success is the single criterion for moving on. If the gate fails, consult the *Troubleshoot* row, then retry — do not skip ahead.
- Commands assume `bash` (Git Bash on Windows is fine). All `$VAR` placeholders map to the secrets table in [SECRETS-AND-ENV-VARS.md](SECRETS-AND-ENV-VARS.md).
- Expected total time for a clean run: **~2.5 h** (most of it waiting for DNS, Aiven provisioning, and Keycloak's first boot).

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
fly auth whoami && psql --version && gh secret list | grep -E 'FLY_API_TOKEN|RENDER_API_KEY|AIVEN_PG_CONNECTION' | wc -l
# must print at least 3
```

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

### 2a. Aiven PostgreSQL

**Do:** [REGISTRATION-AND-CICD.md §2.4](REGISTRATION-AND-CICD.md#24-aiven-postgresql). Create service `flowershop-pg`, then on the **Databases** tab add `flowershop_iot` and `keycloak` (leave `defaultdb` alone).

**Verify:** copy the service URI into `$AIVEN_BASE_URI` locally (don't commit).
```bash
export APP_DB="${AIVEN_BASE_URI/\/defaultdb/\/flowershop_iot}"
export KC_DB="${AIVEN_BASE_URI/\/defaultdb/\/keycloak}"

psql "$APP_DB" -c "SELECT version();"
psql "$KC_DB"  -c "SELECT 1;"
```

**Gate 2a:** both `psql` calls return without error.

### 2b. Upstash Redis

**Do:** [REGISTRATION-AND-CICD.md §2.5](REGISTRATION-AND-CICD.md#25-upstash-redis). Copy the `rediss://...` URL.

**Verify:**
```bash
redis-cli --tls -u "$UPSTASH_REDIS_URL" ping
# Expected: PONG
```
(If `redis-cli` isn't installed, skip — we'll re-verify during Stage 5 via the API `/health` which tests Redis connectivity.)

### 2c. CloudAMQP

**Do:** [REGISTRATION-AND-CICD.md §2.6](REGISTRATION-AND-CICD.md#26-cloudamqp-rabbitmq).

**Verify:** open the instance's **RabbitMQ Manager** link — it should load the mgmt UI. Note the `amqps://` URI.

### 2d. HiveMQ Cloud

**Do:** [REGISTRATION-AND-CICD.md §2.7](REGISTRATION-AND-CICD.md#27-hivemq-cloud-mqtt).

**Verify:**
```bash
# Any MQTT client works; mosquitto_pub shown here.
mosquitto_pub -h "$HIVEMQ_HOST" -p 8883 --cafile /etc/ssl/certs/ca-certificates.crt \
  -u "$HIVEMQ_USER" -P "$HIVEMQ_PASS" -t "test/deploy" -m "hello" -d
# Expected: "Client sent CONNECT" then "Client received CONNACK (0)"
```

**Gate 2:** all four services are reachable from your laptop with the credentials you just saved.

---

## Stage 3 — Keycloak on Fly (T-60m)

Keycloak must come up **before** the API, because the API validates tokens against its JWKS URL.

**Pre-reqs:** §2a (needs the `keycloak` DB).

**Do:**

```bash
cd <repo root>

# 1. Launch the app (idempotent — skips create if it exists).
fly launch --no-deploy --copy-config --name flowershop-keycloak --region waw --yes

# 2. Set secrets.
fly secrets set --app flowershop-keycloak \
  KC_DB_URL="jdbc:postgresql://${AIVEN_PG_HOST}:${AIVEN_PG_PORT}/keycloak?sslmode=require" \
  KC_DB_USERNAME="$AIVEN_PG_USER" \
  KC_DB_PASSWORD="$AIVEN_PG_PASSWORD" \
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
| `FATAL: role "..." does not exist` | Wrong `KC_DB_USERNAME`; check Aiven service users |
| Stuck in "starting" > 3 min | `fly logs -a flowershop-keycloak`; usually DB TLS issue — confirm `?sslmode=require` in `KC_DB_URL` |
| `KC_HOSTNAME` mismatch warnings | Cert hasn't attached yet — keep deploying via `.fly.dev` until Stage 8 adds the CNAME |

---

## Stage 4 — Keycloak realm bootstrap

**Pre-reqs:** §3 green.

**Do:**
1. Browser → `https://flowershop-keycloak.fly.dev`, login as `admin / $KEYCLOAK_ADMIN_PASSWORD`.
2. **Realm** dropdown → **Create realm** → *Resource file* → upload `infra/keycloak/flowershop-realm.json` (committed during Stage 0).
3. Verify the four clients exist: `flowershop-api`, `flowershop-customer-app`, `flowershop-admin-portal`, `flowershop-vendor-portal`.
4. For each **confidential** client (`flowershop-api`, `flowershop-customer-app`): **Credentials** tab → copy the secret.
5. Store as GitHub Actions secrets and also locally for the next stage:
   - `KEYCLOAK_CLIENT_SECRET_API`
   - `KEYCLOAK_CLIENT_SECRET_CUSTOMER`
6. In each client's **Settings**, update the redirect URIs to production hosts (they likely point at `localhost` in the dev export):
   - `flowershop-customer-app` → Valid redirect URIs: `https://app.findmyflowers.pl/signin-oidc`, Post logout: `https://app.findmyflowers.pl/`
   - `flowershop-api` → no browser redirects needed; confirm "Direct access grants" on if you use dev tokens.

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

```bash
fly launch --no-deploy --copy-config --name flowershop-api --region waw --yes

fly secrets set --app flowershop-api \
  ConnectionStrings__DefaultConnection="$APP_DB" \
  ConnectionStrings__ReadConnection="$APP_DB" \
  ConnectionStrings__Redis="$UPSTASH_REDIS_URL" \
  RabbitMQ__ConnectionString="$CLOUDAMQP_URL" \
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
  Cors__AllowedOrigins__0="https://app.findmyflowers.pl" \
  Cors__AllowedOrigins__1="https://admin.findmyflowers.pl" \
  Cors__AllowedOrigins__2="https://vendor.findmyflowers.pl"

fly certs add api.findmyflowers.pl --app flowershop-api

fly deploy --config fly.api.toml
```

The API runs `Database.MigrateAsync()` on startup (Program.cs ~line 248, 10-attempt retry). Expect the first boot to spend ~30 s creating tables on an empty Aiven DB.

**Verify:**
```bash
# Logs should show "Applied migration" entries and then "Now listening on: http://[::]:8080".
fly logs --app flowershop-api | grep -E 'Applied migration|Now listening'

# Fly's built-in health check polls /health/ready every 15 s.
fly status --app flowershop-api | grep passing

# Hit /health/ready directly.
curl -fsS https://flowershop-api.fly.dev/health/ready | jq .
# Expected: {"status":"Healthy","results":{"...":{"status":"Healthy"},...}}
```

**Gate 5:** `/health/ready` returns `{"status":"Healthy"}` with **all** entries (DB, MQTT) also Healthy.

| Troubleshoot | Fix |
|---|---|
| `"status":"Unhealthy"` with DB entry red | Wrong Aiven URI; check `?sslmode=require` + user/DB names |
| Keycloak JWKS fetch fails | Keycloak cert not ready yet — use `https://flowershop-keycloak.fly.dev` in the Authority env var temporarily, redeploy, then switch back after Stage 8 |
| MQTT check "Unhealthy" | HiveMQ credentials — test with `mosquitto_pub` again (§2d) |

---

## Stage 6 — CustomerApp on Fly

**Pre-reqs:** §5 green.

**Do:**

```bash
fly launch --no-deploy --copy-config --name flowershop-customer-app --region waw --yes

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

**Gate 6:** status code is `200` or `302`. Navigating the `.fly.dev` URL in a browser loads the homepage or redirects cleanly to `auth.findmyflowers.pl`.

---

## Stage 7 — Admin & Vendor portals on Render

These are the last surfaces because they only matter once the API is serving.

**Pre-reqs:** §5 green.

**Do:** [REGISTRATION-AND-CICD.md §4.2](REGISTRATION-AND-CICD.md#42-render-services-dashboard-path--recommended) — create both Render services via dashboard, set env vars as listed, add custom domains `admin.findmyflowers.pl` and `vendor.findmyflowers.pl`, **disable Render's auto-deploy**, copy both service IDs into GitHub secrets.

Then trigger both workflows manually:
```bash
gh workflow run deploy-admin-portal.yml
gh workflow run deploy-vendor-portal.yml
gh run watch
```

**Verify:**
```bash
# Render gives each service a *.onrender.com hostname. Substitute yours.
curl -fsS https://flowershop-admin-portal.onrender.com/health
curl -fsS https://flowershop-vendor-portal.onrender.com/health
```

**Gate 7:** both `/health` return 200.

---

## Stage 8 — Attach subdomains

Now every backend is live on its `*.fly.dev` / `*.onrender.com` hostname — point DNS at them.

**Pre-reqs:** §3, §5, §6, §7 green.

**Do:** in Cloudflare DNS, create these CNAMEs (**Proxy status: DNS only** — orange cloud *off*, since Fly and Render manage their own TLS):

| Name | Target |
|---|---|
| `api` | `flowershop-api.fly.dev` |
| `auth` | `flowershop-keycloak.fly.dev` |
| `app` | `flowershop-customer-app.fly.dev` |
| `admin` | `<your-admin-service>.onrender.com` |
| `vendor` | `<your-vendor-service>.onrender.com` |

Wait for each host to issue its cert (Fly and Render both do this automatically once the CNAME exists — 1–5 min typical).

**Verify:**
```bash
for sub in api auth app admin vendor; do
  echo "=== $sub ==="
  curl -sI "https://$sub.findmyflowers.pl/" | head -n 1
done
```

**Gate 8:** every subdomain returns `HTTP/2 200` or `HTTP/2 302` with a valid cert (no TLS errors).

| Troubleshoot | Fix |
|---|---|
| `SSL_ERROR_NO_CYPHER_OVERLAP` / cert still old | Cloudflare proxy is on — switch the CNAME to DNS-only (grey cloud) |
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
   fly secrets set --app flowershop-api Stripe__WebhookSecret="$NEW_WEBHOOK_SECRET"
   # fly auto-rolls the app on secret change; wait for passing status
   ```
5. In Stripe dashboard → **Send test webhook** → choose `payment_intent.succeeded` → click Send.

**Verify:**
```bash
# Tail API logs while the test fires. You want to see a 2xx response and NO signature errors.
fly logs --app flowershop-api | grep -iE 'stripe|webhook'
```

**Gate 9:** Stripe dashboard shows the test delivery with a `200` response.

---

## Stage 10 — Keep-warm & operational cron jobs

**Pre-reqs:** §8 green.

**Do:** at [cron-job.org](https://cron-job.org) create two pings (every 10 min, HTTP GET, expect 200):

- `https://admin.findmyflowers.pl/health`
- `https://vendor.findmyflowers.pl/health`

Also add a weekly backup job — see [ROLLBACK-AND-RECOVERY.md §"Database backup"](ROLLBACK-AND-RECOVERY.md#database-backup).

**Gate 10:** two "Success" executions visible on cron-job.org within 20 min.

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
| 3 Keycloak | 10 min | Aiven must be up |
| 4 Realm | 10 min | — |
| 5 API | 10 min (inc. migrations) | Keycloak JWKS reachable |
| 6 CustomerApp | 5 min | — |
| 7 Portals | 10 min | Render dashboard steps |
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
fly status --app flowershop-api        | grep -E 'passing|failing'
fly status --app flowershop-keycloak   | grep -E 'passing|failing'
fly status --app flowershop-customer-app | grep -E 'passing|failing'
echo "== HTTP =="
for h in api auth app admin vendor; do
  printf "%-10s %s\n" "$h" "$(curl -s -o /dev/null -w '%{http_code}' https://$h.findmyflowers.pl/)"
done
```

All `passing` and all `200/302` = system is up.
