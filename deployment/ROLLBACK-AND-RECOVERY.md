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

### 1.1 Fly apps (`flower-shop-backend-core`, `flowershop-keycloak`, and the three portals)

```bash
# List releases (newest first).
fly releases --app flower-shop-backend-core

# Roll back to previous.
fly releases rollback --app flower-shop-backend-core   # interactive — picks N-1
# or explicit:
fly image show --app flower-shop-backend-core          # get current image sha
fly releases --app flower-shop-backend-core | head -5  # find the one before
fly deploy --app flower-shop-backend-core --image registry.fly.io/flower-shop-backend-core:<previous-tag>
```

Fly performs a rolling deploy; the old machine is drained cleanly. Expected downtime: 0.

### 1.2 Portal Fly apps (`flowershop-admin-portal`, `flowershop-vendor-portal`, `flowershop-customer-app`)

Same as §1.1 — the portals are Fly apps too:

```bash
fly releases rollback --app flowershop-admin-portal
fly releases rollback --app flowershop-vendor-portal
fly releases rollback --app flowershop-customer-app
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
| **Fly Postgres** | `/health/ready` shows DB `Unhealthy`; `psql` hangs | `fly status --app flower-shop-postgres`; `fly machines restart --app flower-shop-postgres` if a machine is stuck. If regional outage, wait it out — site goes read-mostly. See §5.2 for read-only fallback. |
| **Redis** | not applicable in prod | Redis isn't provisioned; `RedisCacheService` falls back to in-memory. |
| **RabbitMQ** | not applicable in prod | RabbitMQ isn't provisioned; `InMemoryEventBus` is the active impl. |
| **HiveMQ** | MQTT health check fails; no vase heartbeats in logs | Restart API subscriber: `fly machines restart --app flower-shop-backend-core`. If still down, check status page. Vases will reconnect automatically. |
| **Keycloak** | All auth'd endpoints 401 | See §4. |
| **Stripe** | Checkout returns errors | Stripe has no "rollback" — check [status.stripe.com](https://status.stripe.com). While down: disable checkout via feature flag; existing bouquets still browsable. |
| **Fly region `fra`** | `fly status` shows all machines down | Deploy to an alternate region: `fly deploy --region ams`. Update `primary_region` in `fly.*.toml` if this is sustained. |

---

## §3 Per-component recovery

### API machine crashed / memory pressure

```bash
fly status --app flower-shop-backend-core              # state?
fly logs --app flower-shop-backend-core | tail -200    # crash reason
fly machines list --app flower-shop-backend-core       # which machine
fly machines restart <id> --app flower-shop-backend-core
```

Persistent OOM → bump memory:
```bash
fly scale memory 1536 --app flower-shop-backend-core   # from 1gb to 1.5gb (costs ~$1 extra/mo)
```

### Portal machine stuck / unhealthy

```bash
fly status --app flowershop-admin-portal
fly machines restart --app flowershop-admin-portal
```

Then re-run the GHA workflow if a bad image shipped.

### CustomerApp redirect loop after deploy

Usually `Authentication__PostLoginRedirectUri` mismatch with Keycloak valid redirect URIs.
```bash
# Check what the app is sending:
fly secrets list --app flowershop-customer-app
# In Keycloak admin, ensure https://findmyflowers.pl/signin-oidc is in "Valid redirect URIs".
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

Rare, but if the `keycloak` DB on Fly Postgres is damaged:
1. Create a fresh DB: `CREATE DATABASE keycloak_v2;`
2. Update `KC_DB_URL` (`jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak_v2`).
3. Redeploy Keycloak — it will migrate schema from scratch.
4. Import realm (§4.1).
5. Regenerate client secrets, update `Authentication__Keycloak__ClientSecret` on API and CustomerApp.

**Result:** every user must log in again (their sessions are gone), but user accounts persist if you export/import `users` — see [Keycloak export docs](https://www.keycloak.org/server/importExport).

---

## §5 Database restore

### 5.1 Restore from backup

**Backups come from the weekly pipeline** (`.github/workflows/db-backup.yml`, see §7 below). Dumps are `pg_dump -F c` (PostgreSQL custom format), so restore with `pg_restore`, not `psql`.

```bash
# Open a tunnel to Fly Postgres (keep this terminal open):
flyctl proxy 5433:5432 -a flower-shop-postgres

# Download the most recent dump from R2 (files are named <db>-<STAMP>.dump):
aws --endpoint-url "$R2_ENDPOINT" s3 cp \
  "s3://$R2_BACKUP_BUCKET/flower_shop_backend_core-<STAMP>.dump" ./

# Restore into a fresh DB (never overwrite prod without a staging copy first):
createdb -h localhost -p 5433 -U postgres flower_shop_backend_core_restore
pg_restore -h localhost -p 5433 -U postgres \
  -d flower_shop_backend_core_restore --no-owner flower_shop_backend_core-<STAMP>.dump

# The keycloak DB dump restores the same way from keycloak-<STAMP>.dump.
# After verifying data looks correct, point the app connection string at the restored DB.
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

# Common cause: Cloudflare proxy (orange cloud) turned on for a domain Fly manages.
# Fix: Cloudflare DNS → click the cloud icon to grey (DNS only).
```

### 6.2 Cloudflare zone deleted / NS removed

If the whole zone goes, re-add it in Cloudflare (free tier). NS records at OVH should already point here — no OVH action needed.

### 6.3 Subdomain points at a stale `*.fly.dev`

Happens after app rename. Update the CNAME in Cloudflare to the current Fly host, then:
```bash
fly certs check api.findmyflowers.pl --app flower-shop-backend-core
```

---

## §7 Database backup (existing pipeline)

Backups are **already automated** by `.github/workflows/db-backup.yml`.

### 7.1 What the pipeline does

- **Schedule:** Mondays at **03:17 UTC** (plus `workflow_dispatch` for manual runs).
- **Method:** reaches Fly Postgres via `flyctl proxy`, then `pg_dump -F c` (PostgreSQL custom format) of **both** databases:
  - `flower_shop_backend_core-<STAMP>.dump`
  - `keycloak-<STAMP>.dump`
- **Destination:** uploaded to Cloudflare R2 (via the AWS S3 CLI against `$R2_ENDPOINT` / `$R2_BACKUP_BUCKET`).
- **Secrets:** `PG_PASSWORD`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT`, `R2_BACKUP_BUCKET`.

Restore with `pg_restore` (custom-format dumps, not `psql`) — see §5.1. Retention: manually delete anything older than 90 days from R2.

### 7.2 Restore drill

Practice it. Once a quarter, pick a dump at random, `pg_restore` it into a scratch DB on Fly Postgres, and run a smoke query. A backup you've never restored is not a backup.

---

## §8 Runbook for total loss

If the Fly org is deleted (billing lapse, hijack, human error) — this takes out the API, Keycloak, the three portals, and Postgres, since everything runs on Fly:

1. Treat domain and Keycloak realm JSON as the only irreplaceable assets — both should be in the git repo or a personal password manager.
2. Rebuild from scratch using [DEPLOYMENT-RUNBOOK.md](DEPLOYMENT-RUNBOOK.md).
3. Restore database from R2 (§5.1).
4. Expected recovery time: 3–4 h if all credentials are in the password manager.

Mitigations against this scenario:
- Enable 2FA on every SaaS account (Fly, Cloudflare, GitHub, Stripe, HiveMQ).
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
- [ ] Verify backups are running (`db-backup.yml` last success — Mondays 03:17 UTC)
