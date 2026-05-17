# Rollback & Recovery Playbook

> **Purpose**: bring the platform back to a known-good state when a deploy goes wrong, a dependency falls over, or data gets corrupted.
> **Principle**: rollback is always faster than root-cause analysis. Restore service first, investigate on a branch.

## Decision tree (60-second read)

```
Something is wrong in prod
├── Just deployed?                         → §1 Revert the deploy
├── Data plane (DB/Redis/Mq/Mqtt) offline? → §2 Dependency outage
├── One subdomain broken, others fine?     → §3 Per-component recovery
├── Auth broken (nobody can log in)?       → §4 Keycloak recovery
├── Data corrupted / bad migration?        → §5 Database restore
└── Whole site unreachable, DNS level?     → §6 DNS & TLS emergency
```

---

## §1 Revert the most recent deploy

Each component has an independent release stream — revert only the one that broke.

### 1.1 Fly apps (`flowershop-api`, `flowershop-keycloak`, `flowershop-customer-app`)

```bash
# List releases (newest first).
fly releases --app flowershop-api

# Roll back to previous.
fly releases rollback --app flowershop-api          # interactive — picks N-1
# or explicit:
fly image show --app flowershop-api                 # get current image sha
fly releases --app flowershop-api | head -5         # find the one before
fly deploy --app flowershop-api --image registry.fly.io/flowershop-api:<previous-tag>
```

Fly performs a rolling deploy; the old machine is drained cleanly. Expected downtime: 0.

### 1.2 Render services (admin + vendor portals)

```bash
# Via Render API: list deploys.
curl -sH "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/services/$RENDER_ADMIN_SERVICE_ID/deploys?limit=5" | jq .

# Trigger redeploy of a past commit:
curl -fsS -X POST -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"imageUrl":"ghcr.io/<owner>/flowershop-admin-portal:<prev-sha>"}' \
  "https://api.render.com/v1/services/$RENDER_ADMIN_SERVICE_ID/deploys"
```

Or simpler: re-run the previous green GitHub Actions run from the Actions UI — "Re-run all jobs".

### 1.3 Git-level revert (if the deploy shipped but the code is the problem)

```bash
# Revert the bad commit — this is a PR, not a force-push.
git revert <bad-sha>
git push origin main
# The path-filtered workflow redeploys only affected components.
```

**Never** `git push --force` to `main`. Always prefer `git revert`.

---

## §2 Dependency outage (the SaaS vendor is down)

| Dependency | How to detect | Immediate action |
|---|---|---|
| **Aiven PG** | `/health/ready` shows DB `Unhealthy`; `psql` hangs | Check [status.aiven.io](https://status.aiven.io). If regional outage, wait it out — site goes read-mostly (no writes) until DB returns. See §5.2 for read-only fallback. |
| **Upstash Redis** | `/health/ready` shows Redis unhealthy | Non-fatal today — `CacheService` falls back to in-memory. Once real Redis impl (EP-09 T-09-003) lands, consider a circuit breaker. |
| **CloudAMQP** | `RabbitMQEventBus.PublishAsync` logs connection errors | Currently a no-op stub; not impactful. Once EP-09 S-09-06 lands, outage means events queue in-process — risk of memory growth. Monitor. |
| **HiveMQ** | MQTT health check fails; no vase heartbeats in logs | Restart API subscriber: `fly machines restart --app flowershop-api`. If still down, check status page. Vases will reconnect automatically. |
| **Keycloak** | All auth'd endpoints 401 | See §4. |
| **Stripe** | Checkout returns errors | Stripe has no "rollback" — check [status.stripe.com](https://status.stripe.com). While down: disable checkout via feature flag; existing bouquets still browsable. |
| **Fly region `waw`** | `fly status` shows all machines down | Deploy to an alternate region: `fly deploy --region ams`. Update `primary_region` in `fly.*.toml` if this is sustained. |

---

## §3 Per-component recovery

### API machine crashed / memory pressure

```bash
fly status --app flowershop-api              # state?
fly logs --app flowershop-api | tail -200    # crash reason
fly machines list --app flowershop-api       # which machine
fly machines restart <id> --app flowershop-api
```

Persistent OOM → bump memory:
```bash
fly scale memory 1024 --app flowershop-api   # from 1gb to 1.5gb (costs ~$1 extra/mo)
```

### Render service stuck "deploying"

```bash
# Cancel the stuck deploy:
curl -fsS -X POST -H "Authorization: Bearer $RENDER_API_KEY" \
  "https://api.render.com/v1/services/$RENDER_ADMIN_SERVICE_ID/deploys/<deploy-id>/cancel"
```

Then re-run the GHA workflow.

### CustomerApp redirect loop after deploy

Usually `Authentication__PostLoginRedirectUri` mismatch with Keycloak valid redirect URIs.
```bash
# Check what the app is sending:
fly secrets list --app flowershop-customer-app
# In Keycloak admin, ensure https://app.findmyflowers.pl/signin-oidc is in "Valid redirect URIs".
```

---

## §4 Keycloak recovery

Losing Keycloak = losing login for everyone. Recovery priority is highest.

### 4.1 Keycloak up but realm broken

Symptom: `/realms/flowershop/.well-known/openid-configuration` 404s.

```bash
# SSH into the container:
fly ssh console --app flowershop-keycloak

# Re-import realm from the mounted file:
/opt/keycloak/bin/kc.sh import --file /opt/keycloak/data/import/flowershop-realm.json --override true
```

### 4.2 Admin lockout

If `$KEYCLOAK_ADMIN_PASSWORD` is lost:
```bash
fly ssh console --app flowershop-keycloak
/opt/keycloak/bin/kcadm.sh config credentials --server http://localhost:8080 \
  --realm master --user admin --password $KEYCLOAK_ADMIN_PASSWORD

# OR nuclear option — reset via env:
fly secrets set KEYCLOAK_ADMIN=admin KEYCLOAK_ADMIN_PASSWORD="$NEW_PASSWORD" --app flowershop-keycloak
fly machines restart --app flowershop-keycloak   # admin user is re-created on boot if missing
```

### 4.3 Keycloak DB schema corruption

Rare, but if `keycloak` schema on Aiven is damaged:
1. Create a fresh DB: `CREATE DATABASE keycloak_v2;`
2. Update `KC_DB_URL` to point at it.
3. Redeploy Keycloak — it will migrate schema from scratch.
4. Import realm (§4.1).
5. Regenerate client secrets, update `Authentication__Keycloak__ClientSecret` on API and CustomerApp.

**Result:** every user must log in again (their sessions are gone), but user accounts persist if you export/import `users` — see [Keycloak export docs](https://www.keycloak.org/server/importExport).

---

## §5 Database restore

### 5.1 Restore from backup

**Backups come from the weekly cron** (see §7 below). If Aiven has point-in-time recovery enabled (paid tiers), prefer that.

```bash
# Download the most recent dump from R2:
aws --endpoint-url https://<r2-account>.r2.cloudflarestorage.com s3 cp \
  s3://flowershop-backups/flowershop_iot-$(date -d yesterday +%F).sql.gz ./

gunzip flowershop_iot-*.sql.gz

# Restore into a fresh DB (never overwrite prod without a staging copy first):
psql "$AIVEN_RESTORE_URI" < flowershop_iot-*.sql

# After verifying data looks correct, swap connection strings.
fly secrets set --app flowershop-api \
  ConnectionStrings__DefaultConnection="$RESTORED_URI"
```

### 5.2 Read-only fallback

When write path is broken but reads work (bad migration half-applied, etc.):

1. Disable write endpoints at the edge — Cloudflare → Rules → create a rule blocking `POST/PUT/DELETE/PATCH` to `api.findmyflowers.pl`, return custom 503 JSON.
2. Roll back the API (§1.1) to the version before the bad migration.
3. EF migration rollback — `dotnet ef database update <previous-migration-name>` (run from a developer machine pointing at prod — risky; prefer §5.1 restore).

### 5.3 Bad migration detection

Every deploy that adds a migration must be tested against a staging DB first. Red flag: `fly logs` shows `RelationDoesNotExistException` after rollback — the new migration added/renamed a column, rollback kept old DB schema but old code now expects it.

**Prevention:** follow the expand-contract migration pattern (add column in release N, backfill in N+1, make required in N+2).

---

## §6 DNS & TLS emergency

### 6.1 Subdomain returning TLS error

```bash
# Check the cert chain:
openssl s_client -connect api.findmyflowers.pl:443 -servername api.findmyflowers.pl < /dev/null

# Common cause: Cloudflare proxy (orange cloud) turned on for a domain Fly/Render manages.
# Fix: Cloudflare DNS → click the cloud icon to grey (DNS only).
```

### 6.2 Cloudflare zone deleted / NS removed

If the whole zone goes, re-add it in Cloudflare (free tier). NS records at OVH should already point here — no OVH action needed.

### 6.3 Subdomain points at a stale `*.fly.dev`

Happens after app rename. Update the CNAME in Cloudflare to the current Fly host, then:
```bash
fly certs check api.findmyflowers.pl --app flowershop-api
```

---

## §7 Database backup (set-and-forget)

Aiven free tier has **no automated backups**. Set this up during Stage 10 of the runbook.

### 7.1 GitHub Actions weekly dump → Cloudflare R2

Create `.github/workflows/backup-db.yml`:

```yaml
name: backup-db
on:
  schedule:
    - cron: "0 3 * * 0"   # Sun 03:00 UTC
  workflow_dispatch:

jobs:
  dump:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - name: Install pg_dump
        run: sudo apt-get update && sudo apt-get install -y postgresql-client
      - name: Dump flowershop_iot
        env:
          PGURI: ${{ secrets.AIVEN_PG_CONNECTION_FLOWERSHOP }}
        run: |
          pg_dump "$PGURI" | gzip > flowershop_iot-$(date +%F).sql.gz
      - name: Upload to R2
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_ACCESS_KEY }}
        run: |
          aws --endpoint-url https://<r2-account>.r2.cloudflarestorage.com \
            s3 cp flowershop_iot-*.sql.gz s3://flowershop-backups/
```

Retention: manually delete anything older than 90 days from R2.

### 7.2 Restore drill

Practice it. Once a quarter, pick a dump at random, restore to a scratch Aiven DB, and run a smoke query. A backup you've never restored is not a backup.

---

## §8 Runbook for total loss

If the Fly org, Render account, or Aiven project is deleted (billing lapse, hijack, human error):

1. Treat domain and Keycloak realm JSON as the only irreplaceable assets — both should be in the git repo or a personal password manager.
2. Rebuild from scratch using [DEPLOYMENT-RUNBOOK.md](DEPLOYMENT-RUNBOOK.md).
3. Restore database from R2 (§5.1).
4. Expected recovery time: 3–4 h if all credentials are in the password manager.

Mitigations against this scenario:
- Enable 2FA on every SaaS account (Fly, Render, Aiven, Cloudflare, GitHub, Stripe).
- Keep billing alerts on — Fly free credit exhaustion triggers machine shutdown.
- Export Keycloak realm monthly: `fly ssh console -a flowershop-keycloak` → `kc.sh export --dir /tmp --realm flowershop` → `fly sftp shell` → `get`.

---

## §9 Post-incident checklist

After any rollback:

- [ ] Write a 1-page incident note in `docs/deployment/incidents/YYYY-MM-DD-<summary>.md`
- [ ] Identify the root cause (not just the fix)
- [ ] Add a regression test or smoke check that would have caught it
- [ ] If a runbook step was ambiguous, fix the runbook
- [ ] Rotate any credential that was exposed during the incident
- [ ] Verify backups are running (weekly cron last success)
