# Observability — Logs, Health, Metrics

> **Purpose**: when something goes wrong in prod, this is the map of where to look first.
> **Scope**: what exists today on the free tier. Upgrade paths to structured logging / APM are listed at the end.

## TL;DR — "the site is broken"

| Symptom | First place to look |
|---|---|
| "I can't log in" | Keycloak logs: `fly logs --app flowershop-keycloak` |
| "Map is empty / 500s" | API logs: `fly logs --app flowershop-api` + `/health/ready` |
| "Bouquets don't refresh" | MQTT subscriber status: API logs → grep `Mqtt` |
| "Checkout fails" | Stripe Dashboard → Events, then API logs → grep `stripe` |
| "Admin portal won't load" | Render dashboard → flowershop-admin-portal → Logs |
| "Vendor portal login fails" | VendorPortal logs + Keycloak `flowershop-vendor-portal` client settings |
| "API intermittently slow" | Aiven metrics + Fly metrics on the API app |
| "Customer app 502" | Fly logs on customer-app; usually an API cold start |

---

## Health endpoints

Every component exposes at least one. Wire them into cron-job.org (§Stage 10 of runbook) for passive monitoring.

| Component | Endpoint | What it checks | Returns |
|---|---|---|---|
| `FlowerShop.API` | `GET /health` | All registered checks (DB, MQTT, any future Redis/RabbitMQ) | `{"status":"Healthy"}` 200 / `{"status":"Unhealthy"}` 503 |
| `FlowerShop.API` | `GET /health/ready` | Only checks tagged `ready` — used by Fly for routing decisions | 200 / 503 |
| `FlowerShop.API` | `GET /health/live` | Liveness only — `Predicate = _ => false` means it always returns 200 if the process is up | 200 |
| Keycloak | `GET /health/ready` | Built-in, requires `KC_HEALTH_ENABLED=true` (already set in `fly.keycloak.toml`) | 200 / 503 |
| Keycloak | `GET /realms/flowershop/.well-known/openid-configuration` | App-specific — proves realm loaded | 200 with JSON |
| `AdminPortal` / `VendorPortal` | `GET /health` | ASP.NET default — process liveness | 200 |
| `CustomerApp` | `GET /` | Home page; Fly uses it as health probe | 200/302 |

**Check all of them in one shell:**
```bash
for h in api auth app admin vendor; do
  printf "%-8s %s\n" "$h" "$(curl -s -o /dev/null -w '%{http_code}' https://$h.findmyflowers.pl/health 2>/dev/null || echo x)"
done
```

---

## Logs

### Fly apps — live tail

```bash
fly logs --app flowershop-api              # follow
fly logs --app flowershop-api --no-tail    # one shot (last ~100 lines)
```

Filter by grep client-side:
```bash
fly logs --app flowershop-api | grep -iE 'error|warn|stripe|mqtt'
```

**Retention:** Fly free tier keeps ~24 h of logs. For longer retention, forward with `fly log-shipper` (costs extra) or add Serilog sink → Seq / Loki.

### Render services

- Dashboard → service → **Logs** tab. Streaming UI, 7 days of retention on free tier.
- CLI alternative: Render doesn't ship a first-party CLI for logs yet; use the web UI or the REST API (`GET /v1/services/{id}/events`).

### Serilog structured output

The API already uses Serilog (see `src/backend/FlowerShop.API/Program.cs`). Fields automatically added:

- `RequestId` — from `RequestIdMiddleware`
- `TenantId` / `VendorId` — from `TenantResolutionMiddleware` (after auth)
- `UserId`, `Role` — from JWT claims
- `Elapsed` — from `UseSerilogRequestLogging()`

Grep a specific request across logs:
```bash
fly logs --app flowershop-api | grep "RequestId=abc-123"
```

### What to grep for

| What you want | grep pattern |
|---|---|
| Failed requests | `"StatusCode":5\|"level":"Error"` |
| Slow requests (>1s) | `Elapsed=[12][0-9]{3}` |
| Auth failures | `401\|AuthenticationFailed\|JwtBearer` |
| MQTT ingest | `MqttDevice\|vase/.*heartbeat` |
| Stripe | `stripe\|webhook\|payment_intent` |
| DB errors | `Npgsql\|EntityFramework\|RelationDoesNotExist` |
| Migrations | `Applied migration\|Pending migration` |

---

## Metrics

### Fly built-in

Dashboard → app → **Metrics**. Shows CPU, memory, network, connections, HTTP status breakdown per machine. Free. 7 days retention.

### Aiven PostgreSQL

Console → service → **Metrics** tab. Shows:
- Connections in use vs. max
- Disk usage (5 GB limit on free plan)
- Query throughput
- Replication lag (irrelevant on free tier; no replicas)

**Watch:** disk hitting 4 GB → start pruning old order data or upgrade plan.

### Upstash Redis

Console → database → **Usage**. Shows commands/month against the 500k free cap.

### CloudAMQP

Instance → **Metrics**. Shows message rate, queue depth, connection count.

### HiveMQ Cloud

Cluster → **Monitoring**. Shows connected clients (100 cap on free), messages/hour, storage.

### Cron-job.org (uptime)

Per job, shows "Last execution" and success history. Free-tier dashboards are a reasonable UptimeRobot substitute.

---

## Tracing a request end-to-end

A customer clicks "Add to cart" → three services touch the request:

1. **CustomerApp** (Fly) → look for the session cookie in its logs.
2. **API** (Fly) → `fly logs --app flowershop-api | grep <request-id>` — `RequestId` is forwarded from CustomerApp if present, otherwise generated.
3. **PostgreSQL** (Aiven) → `pg_stat_statements` shows the query; enable it in the Aiven console under *Advanced configuration*.

For IoT path: vase → HiveMQ → API → SignalR → CustomerApp. Correlate by `VaseSerialNumber` — it's stamped on every log line in `MqttDeviceCommunicationService`.

---

## Alerting

**None configured today.** The cheapest stack that works:

1. **cron-job.org** → email on 3 consecutive failures. Free. Covers uptime only.
2. **Cloudflare** → Health Checks (one free check) → emails if an origin goes down.
3. **Stripe** → Dashboard → Developers → Webhooks → failing deliveries trigger email automatically.
4. **Aiven** → service → Integrations → "Email notifications" → alerts on disk/CPU thresholds.

For anything richer (paging, PagerDuty, Slack webhooks) you need one of:
- BetterStack / Logtail (free tier with 1 GB logs/mo) — Serilog sink available
- Sentry (5k events/mo free) — `Sentry.AspNetCore` NuGet, one-line wire-up in `Program.cs`
- Seq self-hosted on Fly — ~$2/mo extra, full-text log search

Pick one when on-call becomes a real thing. Before then, cron-job.org + email is adequate.

---

## Known noisy log lines (safe to ignore)

| Pattern | Source | Why it's safe |
|---|---|---|
| `HealthReport with status Healthy` (every 15s) | Fly probes `/health/ready` | Just liveness; mute in Serilog filter if annoying |
| `Keycloak: realm 'master' - Password policy not configured` | Keycloak boot | `master` realm unused in prod |
| `Idempotency key not found, proceeding` | `IdempotencyMiddleware` on GETs | GETs don't need idempotency; check only on mutations |
| `InMemoryEventBus: Published <event>` | Events system (EP-09 pending) | Stubbed — replace when RabbitMQ is real |

---

## Upgrade path when the above isn't enough

1. **Structured logs off-site**: add a Serilog sink to BetterStack (free tier) — 10 min of work, immediate searchable history.
2. **APM**: Sentry Performance (free tier) or Datadog free trial. Wire via middleware; catches slow queries and exceptions with stack traces.
3. **Distributed tracing**: OpenTelemetry → Grafana Cloud (free 50 GB traces). Requires instrumentation changes, but correlates Stripe/DB/MQTT calls with UI requests.
4. **Synthetic checks**: Checkly or UptimeRobot Pro for multi-step journeys (login → checkout). Replaces manual §10 of [POST-DEPLOY-SMOKE-TESTS.md](POST-DEPLOY-SMOKE-TESTS.md).

Estimated upgrade cost to "production-grade observability": $0–25/mo depending on retention.
