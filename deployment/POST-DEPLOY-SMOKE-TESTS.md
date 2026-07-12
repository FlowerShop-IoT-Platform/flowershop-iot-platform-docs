# Post-Deploy Smoke Tests

> **Purpose**: once every stage of [DEPLOYMENT-RUNBOOK.md](DEPLOYMENT-RUNBOOK.md) is green, these scripted tests prove the platform actually *works* end-to-end — not just that services respond to `/health`.
> **When to run**: after first deploy, after any deploy that touches >1 component, after any Keycloak realm change, after any Stripe key rotation.

## Fast sanity layer — 60 seconds

Stick this in your shell and run whenever you want a quick "is prod up" pulse:

```bash
#!/usr/bin/env bash
set -u
BASE="findmyflowers.pl"
pass=0; fail=0
check() {
  local label="$1" url="$2" expect="$3"
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "$url")
  if [[ " $expect " == *" $code "* ]]; then
    printf "[OK]   %-30s %s\n" "$label" "$code"; ((pass++))
  else
    printf "[FAIL] %-30s got %s expected %s\n" "$label" "$code" "$expect"; ((fail++))
  fi
}
check "API /health/ready"   "https://api.$BASE/health/ready"    "200"
check "API /health/live"    "https://api.$BASE/health/live"     "200"
check "Keycloak OIDC disc." "https://auth.$BASE/realms/flowershop/.well-known/openid-configuration" "200"
check "CustomerApp root"    "https://app.$BASE/"                "200 302"
check "AdminPortal /Login"  "https://admin.$BASE/Login"         "200"
check "VendorPortal /Login" "https://vendors.$BASE/Login"       "200"
echo "---- $pass pass / $fail fail ----"
exit $fail
```

Save as `scripts/smoke.sh`. Fails with non-zero when anything is down — wire it into an external uptime monitor for passive monitoring.

---

## Deep smoke matrix

Run these in order; each depends on the previous succeeding. Every step has a **Pass** criterion that's literal enough to script.

### 1. Auth loop — Keycloak issues a working token

```bash
TOKEN=$(curl -fsS -X POST \
  "https://auth.findmyflowers.pl/realms/flowershop/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=flowershop-customer-app" \
  -d "client_secret=$KEYCLOAK_CLIENT_SECRET_CUSTOMER" \
  -d "username=smoketest@findmyflowers.pl" \
  -d "password=$SMOKETEST_PASSWORD" \
  -d "scope=openid" | jq -r .access_token)

[[ -n "$TOKEN" && "$TOKEN" != "null" ]] && echo PASS || echo FAIL
```

**Prereq:** create a one-off `smoketest@findmyflowers.pl` user in Keycloak realm (role: `User`).
**Pass:** non-empty JWT printed.

### 2. Authenticated API call — tenant context resolves

```bash
curl -fsS -H "Authorization: Bearer $TOKEN" \
  https://api.findmyflowers.pl/api/v1/customer/me | jq .
```

**Pass:** 200 with a `customerId` in the response. If 401, Keycloak's JWKS isn't trusted by the API (re-check `Authentication__Keycloak__Authority`). If 404 on the route, the customer profile endpoints aren't deployed yet — substitute any `[Authorize(Policy = "CustomerAccess")]` endpoint.

### 3. Database round-trip

```bash
curl -fsS -H "Authorization: Bearer $TOKEN" \
  "https://api.findmyflowers.pl/api/v1/bouquets?limit=1" | jq '.items | length'
```

**Pass:** numeric `0` or more (proves API→PostgreSQL read path works).

### 4. Write path + domain event

```bash
# As a vendor: create a bouquet.
# NOTE: there is no flowershop-vendor-portal client — the portals authenticate
# through the flowershop-api client via password grant, using a vendor user.
VENDOR_TOKEN=$(...same as step 1 with the flowershop-api client + vendor user...)

curl -fsS -X POST -H "Authorization: Bearer $VENDOR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"smoke-rose","priceMinor":1000,"currency":"PLN"}' \
  https://api.findmyflowers.pl/api/v1/vendor/bouquets | jq .
```

**Pass:** 201 with a `bouquetId` GUID. Confirms: Postgres write, MediatR pipeline, domain event publish (visible in logs as `Published BouquetCreated to InMemoryEventBus` — `InMemoryEventBus` is the active impl; RabbitMQ is not provisioned in prod).

### 5. IoT ingest — MQTT → API → SignalR

Terminal A (API log tail):
```bash
fly logs --app flower-shop-backend-core | grep -iE 'mqtt|vase'
```

Terminal B (simulate a vase):
```bash
mosquitto_pub -h "$HIVEMQ_HOST" -p 8883 --cafile /etc/ssl/certs/ca-certificates.crt \
  -u "$HIVEMQ_USER" -P "$HIVEMQ_PASS" \
  -t "vase/SMOKE-VASE-001/heartbeat" \
  -m '{"timestamp":"2026-05-11T10:00:00Z","uptimeSeconds":120,"firmware":"smoke-1.0"}'
```

**Pass:** within 5 s, terminal A shows a log line containing `SMOKE-VASE-001` handled by `MqttDeviceCommunicationService`.

### 6. Stripe order loop (test-mode)

```bash
# Add a cart item, preview, create order, confirm payment via test card.
# The full 4-call sequence is in docs/API-CONTRACTS.md — the critical thing to
# smoke-test is that /confirm-payment returns 200 with a `paid` status.

STRIPE_TEST_PM="pm_card_visa"   # canonical Stripe test payment method

# (Full scripted version lives in scripts/stripe-smoke.sh — see below.)
```

**Pass:** order transitions to `Paid`, bouquet status to `Sold`, and in Stripe dashboard → Payments the test charge is visible.

### 7. SignalR freshness push

```bash
# Install a quick SignalR client (any language works). Node example:
npm i -g @microsoft/signalr@latest
node -e "
const s = require('@microsoft/signalr');
const c = new s.HubConnectionBuilder()
  .withUrl('https://api.findmyflowers.pl/hubs/bouquet', { accessTokenFactory: () => process.env.TOKEN })
  .build();
c.on('FreshnessUpdated', b => { console.log('PASS', b); process.exit(0); });
c.start().then(() => console.log('connected'));
setTimeout(() => { console.log('FAIL: no event in 30s'); process.exit(1); }, 30000);
"
```

Then, in another terminal, publish an MQTT sensor reading for a bouquet's vase.
**Pass:** the Node client prints `PASS` with payload within 30 s.

### 8. Rate-limit enforcement

```bash
for i in {1..120}; do
  curl -s -o /dev/null -w "%{http_code}\n" https://api.findmyflowers.pl/api/v1/bouquets
done | sort | uniq -c
```

**Pass:** you see a mix of `200` and `429`. All `200` means the limiter isn't wired (check `app.UseRateLimiter()` in `Program.cs` — should be present). All `429` means limits are too aggressive — relax in config.

### 9. CORS from the real customer origin

```bash
curl -si -X OPTIONS https://api.findmyflowers.pl/api/v1/bouquets \
  -H "Origin: https://findmyflowers.pl" \
  -H "Access-Control-Request-Method: GET" \
  | grep -i "access-control-allow-origin"
```

**Pass:** `Access-Control-Allow-Origin: https://findmyflowers.pl`. If it echoes `*`, the CORS config still has the dev wildcard — fix in `appsettings.Production.json`.

### 10. Browser-driven critical path

Do this manually in an incognito window:

- [ ] `https://app.findmyflowers.pl/` (the CustomerApp) loads without TLS warning.
- [ ] "Sign in" redirects to `auth.findmyflowers.pl`, shows the flowershop realm login.
- [ ] Login succeeds and returns to `/` authenticated.
- [ ] Map page shows at least one bouquet pin (or, if DB empty, empty-state copy — not an error).
- [ ] Click pin → details page loads with image, price, vendor name.
- [ ] Add to cart → checkout → Stripe test card `4242 4242 4242 4242` → redirected to success page.
- [ ] Order appears under "My Orders".

**Pass:** every box checked.

---

## Scripted suite (optional)

Consolidate all 10 checks into `scripts/smoke-full.sh`:

```bash
#!/usr/bin/env bash
set -e
./scripts/smoke.sh                     # fast sanity
./scripts/smoke-auth.sh                # step 1
./scripts/smoke-api-authenticated.sh   # steps 2–4
./scripts/smoke-mqtt.sh                # step 5
./scripts/stripe-smoke.sh              # step 6
./scripts/smoke-signalr.js             # step 7
./scripts/smoke-ratelimit.sh           # step 8
./scripts/smoke-cors.sh                # step 9
echo "=== Now run §10 manually in incognito ==="
```

Add it to CI as a **post-deploy job** in `deploy-api.yml`, gated on a `SMOKE_ENABLED` var so PRs don't churn through it.

---

## What failure modes each test catches

| Test | Protects against |
|---|---|
| 1 Auth loop | Keycloak realm not imported, client secrets rotated, realm disabled |
| 2 Authenticated call | Wrong `Authority`, `TenantResolutionMiddleware` broken, JWT mapping regression |
| 3 DB read | Fly Postgres creds wrong, migration missing, `DbContext` DI broken |
| 4 Write path | Handler DI, validation pipeline, outbox / event publish wiring |
| 5 MQTT ingest | HiveMQ creds, broker TLS, `MqttDeviceCommunicationService` not subscribed (common after redeploy) |
| 6 Stripe loop | Webhook signing secret mismatch, bouquet reservation regression |
| 7 SignalR | `UseWebSockets()` missing, CORS blocks hub, forwarded headers misconfig |
| 8 Rate limit | Limiter removed or accidentally disabled in config |
| 9 CORS | Wildcard leak, allowed origins drift from DNS |
| 10 Browser | End-to-end integration catches anything the API tests can't reproduce |

---

## Incident playbook hook

If *any* smoke test fails after a deploy that was previously green:

1. Don't start debugging — **roll back first** (see [ROLLBACK-AND-RECOVERY.md](ROLLBACK-AND-RECOVERY.md)). A failing deploy is a production incident.
2. Rerun smoke against the rolled-back version. If it passes, you have your bisect baseline.
3. Investigate on a branch, not on `main`.
