# FlowerShop IoT Platform — Free-Tier Deployment Spec

> **Target**: `findmyflowers.pl` demo / early-access deployment
> **Cost target**: $0/month, accepting cold starts on non-critical surfaces
> **Status**: proposal — no code shipped against this yet

## Diagrams

| # | View | Source | Render |
|---|------|--------|--------|
| 1 | Hosting topology (who lives where) | [01-hosting-topology.mmd](01-hosting-topology.mmd) | [01-hosting-topology.svg](01-hosting-topology.svg) |
| 2 | Customer purchase flow (OIDC → cart → Stripe) | [02-customer-purchase-flow.mmd](02-customer-purchase-flow.mmd) | [02-customer-purchase-flow.svg](02-customer-purchase-flow.svg) |
| 3 | IoT ingest → freshness push to browsers | [03-iot-ingest-flow.mmd](03-iot-ingest-flow.mmd) | [03-iot-ingest-flow.svg](03-iot-ingest-flow.svg) |
| 4 | Network boundaries + DNS | [04-network-and-dns.mmd](04-network-and-dns.mmd) | [04-network-and-dns.svg](04-network-and-dns.svg) |

---

## Component → host matrix

| Component | Host | Tier | Why it goes there |
|---|---|---|---|
| `FlowerShop.API` | Fly.io VM (`shared-cpu-1x`, 512 MB) | Free allowance | Hosts in-process MQTT subscriber + 5 background services (`MqttDeviceCommunicationService`, `VaseHealthMonitoringService`, `BouquetFreshnessUpdateService`, `PerformanceAnalyticsService`, `OrderReservationExpiryService`, `SubscriptionMatchingService`). Must stay awake; Render free sleep after 15 min would kill IoT ingest. |
| Keycloak 23 | Fly.io VM (`shared-cpu-1x`, 1 GB) | Free allowance | JVM cold-start on Render free is ~60 s and makes every login painful. Fly keeps it warm. Uses the `keycloak` database on Aiven. |
| `FlowerShop.AdminPortal` (Razor) | Render free Docker web service | Free | Platform-admin traffic is rare — cold starts acceptable. |
| `FlowerShop.VendorPortal` (Razor) | Render free Docker web service | Free | Vendor logs in a few times a day — cold starts acceptable. |
| `FlowerShop.CustomerApp` (MVC + OIDC) | Render free Docker web service + cron-job.org ping every 10 min | Free | Customer-facing, so keep-warm ping mitigates UX hit. |
| PostgreSQL (`flowershop_iot` + `keycloak` DBs) | Aiven free PG | Free | 5 GB disk / 1 GB RAM covers both logical DBs — same split as `docker-compose.dev.yml` lines 130–133. |
| Redis cache | Upstash serverless | Free (500k cmd/mo, 256 MB) | Pay-per-command, scales to zero. Wire in once EP-09 T-09-003 lands; today `CacheService` still uses `IMemoryCache`. |
| RabbitMQ | CloudAMQP Little Lemur | Free (1M msg/mo, 20 conn) | Domain-event fan-out once EP-09 S-09-06 replaces `InMemoryEventBus`. Share the connection pool — 20-conn cap is the binding constraint. |
| MQTT broker | HiveMQ Cloud Serverless | Free (100 conn, 10 GB/mo) | Vases are MQTT **clients**; API subscribes as another client. Point `MQTT__BrokerHost` at HiveMQ + TLS. |
| Bouquet photo storage | Cloudflare R2 | Free (10 GB, zero egress) | Relevant after EP-15 ships real photo upload (replacing the `http://localhost:8080/api/mock-vase/...` hardcode, EP-12 T-12-004). |
| Stripe webhook | Served by API — no separate host | — | `POST /api/stripe/webhook` piggybacks on Fly's auto-HTTPS. Live signing secret only. |

## DNS layout (all CNAME → provider apex, auto-TLS)

| Host | Target |
|---|---|
| `api.findmyflowers.pl` | Fly app — `FlowerShop.API` |
| `auth.findmyflowers.pl` | Fly app — Keycloak |
| `admin.findmyflowers.pl` | Render service — AdminPortal |
| `vendor.findmyflowers.pl` | Render service — VendorPortal |
| `app.findmyflowers.pl` | Render service — CustomerApp |

## Code changes required before first deploy

These are blockers, not nice-to-haves.

1. **`src/backend/FlowerShop.API/Program.cs`** — add `app.UseForwardedHeaders(...)` with `XForwardedFor | XForwardedProto` **before** `UseHttpsRedirection()` (currently line 190). Without it, `Request.Scheme` stays `http` behind Fly's proxy and Keycloak rejects the OIDC reply URL.
2. **`src/backend/FlowerShop.API/Program.cs`** — add `app.UseWebSockets()` before the `MapHub<BouquetHub>()` call (line 227). Belt-and-suspenders for SignalR over reverse proxies.
3. **CORS** — replace `Cors:AllowedOrigins: ["*"]` in prod appsettings with the four portal / mobile origins above. The portals aren't in the dev allow-list — they only work in dev because of the wildcard.
4. **Secrets externalized to env vars** on each host: `ConnectionStrings__DefaultConnection`, `ConnectionStrings__Redis`, `MQTT__*`, `RabbitMQ__*`, `Authentication__Keycloak__*`, `Stripe__SecretKey`, `Stripe__WebhookSecret`. Never bake into Docker images.
5. **Fix known VendorPortal bugs** before exposing to a real vendor:
   - `Bouquets/Index.cshtml.cs` — reads `VendorId` from session instead of JWT claim (EP-12 T-12-003).
   - `Vases/Details.cshtml.cs` — POSTs to hardcoded `http://localhost:8080/api/mock-vase/...` (EP-12 T-12-004).
6. **Keycloak realm export** — run `kc.sh export --dir /opt/keycloak/data/import --realm flowershop` on the dev instance, commit the JSON, and mount it into the Fly Keycloak VM so it boots with the same clients (`flowershop-api`, `flowershop-customer-app`, `flowershop-admin-portal`, `flowershop-vendor-portal`) the code expects.

## Runtime configuration per host

### Fly — `FlowerShop.API`

```
ASPNETCORE_URLS=http://+:8080
ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
ConnectionStrings__DefaultConnection=Host=<aiven>;Database=flowershop_iot;Username=...;Password=...;SslMode=Require
ConnectionStrings__ReadConnection=<same host, read-only user>
ConnectionStrings__Redis=<upstash rediss:// url>
MQTT__BrokerHost=<cluster>.hivemq.cloud
MQTT__BrokerPort=8883
MQTT__UseTls=true
MQTT__Username=<hivemq user>
MQTT__Password=<hivemq pass>
RabbitMQ__ConnectionString=<amqps cloudamqp url>
Authentication__UseKeycloak=true
Authentication__Keycloak__Authority=https://auth.findmyflowers.pl/realms/flowershop
Authentication__Keycloak__ClientId=flowershop-api
Authentication__Keycloak__ClientSecret=<secret>
Stripe__SecretKey=<sk_live_...>
Stripe__WebhookSecret=<whsec_...>
Cors__AllowedOrigins__0=https://app.findmyflowers.pl
Cors__AllowedOrigins__1=https://admin.findmyflowers.pl
Cors__AllowedOrigins__2=https://vendor.findmyflowers.pl
```

Migrations run automatically on startup (`Program.cs` calls `Database.MigrateAsync()` with 10-attempt retry).

### Fly — Keycloak

```
KC_DB=postgres
KC_DB_URL=jdbc:postgresql://<aiven>:5432/keycloak?sslmode=require
KC_DB_USERNAME=<keycloak_db_user>
KC_DB_PASSWORD=<...>
KC_HOSTNAME=auth.findmyflowers.pl
KC_PROXY=edge
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=<strong>
# command: start --optimized --import-realm
```

### Render — each portal

```
ASPNETCORE_URLS=http://+:8080
ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
ApiSettings__BaseUrl=https://api.findmyflowers.pl
Authentication__Authority=https://auth.findmyflowers.pl/realms/flowershop   # CustomerApp only
Authentication__ClientId=flowershop-customer-app                            # CustomerApp only
Authentication__ClientSecret=<secret>                                       # CustomerApp only
Authentication__PostLoginRedirectUri=https://app.findmyflowers.pl/          # CustomerApp only
```

## Known risks at this topology

| Risk | Impact | Mitigation |
|---|---|---|
| Fly free VM eviction (rare but possible) | API + MQTT subscriber goes down | Accept for demo; graduate API to paid Fly (~$2/mo) before onboarding real vendors |
| Render portal cold starts chain | CustomerApp cold + API cold = ~90 s first login | cron-job.org keep-warm ping every 10 min on CustomerApp and API |
| CloudAMQP 20-connection cap | New MassTransit consumers exceed cap once EP-05 lands | Single shared connection in `IPublishEndpoint` registration; audit before EP-05 merge |
| HiveMQ 100-device cap | Demo fleet >100 vases | Upgrade plan or self-host Mosquitto on a $4 DO droplet |
| Aiven free PG has no automated backups | Data-loss risk | Weekly `pg_dump` from a GitHub Actions cron → R2 |
| Stripe webhook replay attacks | Double-processing | Already signature-verified in `StripeController`; confirm idempotency key in `CreateOrderCommandHandler` before live keys |
| Keycloak realm drift between dev and Fly | Auth fails after redeploy | Commit realm export JSON; CI check that dev and prod clients match |

## Alternatives considered

- **Render free for everything** — simplest, one provider, but cold starts break MQTT ingest. Rejected.
- **Railway end-to-end** — better DX than Render but only $5 one-time credit; not truly free long-term. Keep as upgrade path.
- **Neon Postgres** — 0.5 GB too tight once orders + analytics + Keycloak sessions share a box. Aiven's 5 GB wins.
- **Supabase Auth instead of Keycloak** — removes a VM from the diagram, but the code already integrates with Keycloak (`KeycloakAdminClient`, `VendorOnboardingService`). Re-platforming auth is a separate epic, not a deployment task.
- **Azure Container Apps free tier** — generous free grant but requires a credit card and the 180k-vCPU-sec cap runs out fast with 5 background workers ticking. Rejected.

## Upgrade path ($7–20/mo when demo graduates)

1. Move `FlowerShop.API` to Fly paid (`shared-cpu-1x` dedicated, ~$2/mo) → SLA-backed uptime.
2. Move CustomerApp to Fly (~$2/mo) → no cold starts, drop the keep-warm cron.
3. Upgrade Aiven PG to Startup tier (or move to Supabase Pro $25/mo) → automated backups.
4. HiveMQ Starter ($5/mo) → 1k device cap.
5. Keep AdminPortal + VendorPortal on Render free — internal traffic doesn't need SLA.
