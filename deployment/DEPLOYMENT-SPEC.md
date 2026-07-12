# FlowerShop IoT Platform — Free-Tier Deployment Spec

> **Target**: `findmyflowers.pl` demo / early-access deployment
> **Cost target**: low-cost showcase, always-on where it matters
> **Status**: shipped — live at `findmyflowers.pl`. The final topology diverged from this proposal in two big ways: **all three portals run on Fly.io (not Render)** and **PostgreSQL runs on Fly (not Aiven)**. Redis and RabbitMQ were never provisioned in prod (dev-only). The sections below have been updated to reflect what actually shipped.

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
| `FlowerShop.API` | Fly.io VM (`shared-cpu-1x`, 1 GB, region `fra`) | always-on | Hosts in-process MQTT subscriber + 5 background services (`MqttDeviceCommunicationService`, `VaseHealthMonitoringService`, `BouquetFreshnessUpdateService`, `PerformanceAnalyticsService`, `OrderReservationExpiryService`, `SubscriptionMatchingService`). Must stay awake; kept warm with `auto_stop_machines=false` / `min_machines_running=1`. App `flower-shop-backend-core`, config `fly.toml`. |
| Keycloak 23 | Fly.io VM (`shared-cpu-1x`, 1 GB, region `fra`) | always-on | JVM cold-start makes every login painful, so Fly keeps it warm. App `flowershop-keycloak` at `auth.findmyflowers.pl`. Uses the `keycloak` database on Fly Postgres. |
| `FlowerShop.AdminPortal` (Razor) | Fly.io VM (region `fra`) | always-on | App `flowershop-admin-portal` at `admin.findmyflowers.pl`, config `fly.admin-portal.toml`. |
| `FlowerShop.VendorPortal` (Razor) | Fly.io VM (region `fra`) | always-on | App `flowershop-vendor-portal` at `vendors.findmyflowers.pl` (plural), config `fly.vendor-portal.toml`. |
| `FlowerShop.CustomerApp` (MVC + OIDC) | Fly.io VM (region `fra`) | always-on | App `flowershop-customer-app` at `app.findmyflowers.pl`, config `fly.customer-app.toml`. `min_machines_running=1` keeps it warm — no keep-warm cron needed. |
| PostgreSQL (`flower_shop_backend_core` + `keycloak` DBs) | Fly.io Postgres (`flower-shop-postgres`, PostgreSQL 17, region `fra`) | — | App DB `flower_shop_backend_core` + `keycloak` DB, on one Fly Postgres cluster. Reachable in-cluster via `flower-shop-postgres.flycast:5432`. |
| Redis cache | **Not provisioned in prod** | — | `RedisCacheService` falls back to `AddDistributedMemoryCache()` when `ConnectionStrings:Redis` is unset. Redis only runs in `docker-compose.dev.yml`. |
| RabbitMQ | **Not provisioned in prod** | — | `InMemoryEventBus` is the active implementation. RabbitMQ only runs in `docker-compose.dev.yml`. |
| MQTT broker | HiveMQ Cloud (TLS, port 8883) | Free (100 conn, 10 GB/mo) | Vases are MQTT **clients**; API subscribes as another client. `MQTT__BrokerHost` points at HiveMQ + TLS. Replaces dev-only Mosquitto. |
| Bouquet photo storage | Cloudflare R2 | Free (10 GB, zero egress) | Implemented (EP-15): `FileStorage:R2:Provider=R2` → `R2FileStorageService`. Bucket `flowershop-bouquets`; photos served from the custom domain `img.findmyflowers.pl` (R2 → bucket → Settings → Custom Domains) so image bandwidth bypasses the API. Account ID + bucket + public URL live in `appsettings.Production.json`; the two credentials come from Fly secrets `FileStorage__R2__AccessKeyId` / `FileStorage__R2__SecretAccessKey`. `findmyflowers.pl` DNS is on Cloudflare (done). |
| Stripe webhook | Served by API — no separate host | — | `POST /api/stripe/webhook` piggybacks on Fly's auto-HTTPS. Wired via `Stripe__SecretKey` / `Stripe__WebhookSecret`. |

## DNS layout (all CNAME → provider apex, auto-TLS)

| Host | Target |
|---|---|
| `api.findmyflowers.pl` | Fly app — `FlowerShop.API` (`flower-shop-backend-core`) |
| `auth.findmyflowers.pl` | Fly app — Keycloak (`flowershop-keycloak`) |
| `admin.findmyflowers.pl` | Fly app — AdminPortal (`flowershop-admin-portal`) |
| `vendors.findmyflowers.pl` | Fly app — VendorPortal (`flowershop-vendor-portal`) |
| `app.findmyflowers.pl` | Fly app — CustomerApp (`flowershop-customer-app`) |
| `img.findmyflowers.pl` | Cloudflare R2 bucket `flowershop-bouquets` |

## Code changes required before first deploy — DONE

These were blockers at proposal time. All are now shipped.

1. ✅ **`src/backend/FlowerShop.API/Program.cs`** — `app.UseForwardedHeaders(...)` with `XForwardedFor | XForwardedProto` is applied before `UseHttpsRedirection()`, so `Request.Scheme` is correct behind Fly's proxy.
2. ✅ **`src/backend/FlowerShop.API/Program.cs`** — `app.UseWebSockets()` runs before `MapHub<BouquetHub>()` for SignalR over the reverse proxy.
3. ✅ **CORS** — prod `Cors:AllowedOrigins` lists the real portal / mobile origins; the dev wildcard is gone.
4. ✅ **Secrets externalized to env vars** on each host via Fly secrets: `ConnectionStrings__DefaultConnection`, `Authentication__Keycloak__*`, `MQTT__*`, `Stripe__SecretKey`, `Stripe__WebhookSecret`, `FileStorage__R2__*`. None baked into images. (Redis/RabbitMQ connection strings are unset in prod — those services aren't provisioned.)
5. ✅ **Known VendorPortal bugs fixed** before exposing to a real vendor (EP-12 T-12-003 / T-12-004).
6. ✅ **Keycloak realm committed** at `docker/keycloak/flowershop-realm.json` and imported into the Fly Keycloak VM. Realm `flowershop` boots with clients `flowershop-api`, `flowershop-customer-app`, `ESP32-40C86C` and a Google IdP. Note: there is **no** dedicated admin/vendor OIDC client — both portals authenticate through the `flowershop-api` client via password grant.

## Runtime configuration per host

### Fly — `FlowerShop.API`

```
ASPNETCORE_URLS=http://+:8080
ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
ConnectionStrings__DefaultConnection=Host=flower-shop-postgres.flycast;Database=flower_shop_backend_core;Username=...;Password=...
# Redis and RabbitMQ are NOT provisioned in prod — leave these unset:
#   ConnectionStrings__Redis   (RedisCacheService falls back to in-memory)
#   RabbitMQ__ConnectionString (InMemoryEventBus is the active impl)
MQTT__BrokerHost=<cluster>.hivemq.cloud
MQTT__BrokerPort=8883
MQTT__UseTls=true
MQTT__Username=<hivemq user>
MQTT__Password=<hivemq pass>
Authentication__UseKeycloak=true
Authentication__Keycloak__Authority=https://auth.findmyflowers.pl/realms/flowershop
Authentication__Keycloak__ClientId=flowershop-api
Authentication__Keycloak__ClientSecret=<secret>
Stripe__SecretKey=<sk_..._>
Stripe__WebhookSecret=<whsec_...>
FileStorage__R2__AccessKeyId=<r2 access key id>
FileStorage__R2__SecretAccessKey=<r2 secret access key>
Cors__AllowedOrigins__0=https://findmyflowers.pl
Cors__AllowedOrigins__1=https://admin.findmyflowers.pl
Cors__AllowedOrigins__2=https://vendors.findmyflowers.pl
```

Migrations run automatically on startup (`Program.cs` calls `Database.MigrateAsync()` with 10-attempt retry).

### Fly — Keycloak

```
KC_DB=postgres
KC_DB_URL=jdbc:postgresql://flower-shop-postgres.flycast:5432/keycloak
KC_DB_USERNAME=<keycloak_db_user>
KC_DB_PASSWORD=<...>
KC_HOSTNAME=auth.findmyflowers.pl
KC_PROXY=edge
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=<strong>
# command: start --optimized --import-realm
```

### Fly — each portal

```
ASPNETCORE_URLS=http://+:8080
ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
ApiSettings__BaseUrl=https://api.findmyflowers.pl
Authentication__Authority=https://auth.findmyflowers.pl/realms/flowershop   # CustomerApp only
Authentication__ClientId=flowershop-customer-app                            # CustomerApp only
Authentication__ClientSecret=<secret>                                       # CustomerApp only
Authentication__PostLoginRedirectUri=https://app.findmyflowers.pl/          # CustomerApp only
```

The Admin and Vendor portals authenticate through the `flowershop-api` client via password grant (there is no dedicated portal OIDC client); they only need `ApiSettings__BaseUrl` plus the configurable Keycloak base URL.

## Known risks at this topology

| Risk | Impact | Mitigation |
|---|---|---|
| Fly VM eviction (rare but possible) | API + MQTT subscriber goes down | All Fly apps run `min_machines_running=1` so they stay warm; graduate to a dedicated VM before onboarding many real vendors |
| HiveMQ 100-device cap | Demo fleet >100 vases | Upgrade plan or self-host Mosquitto on a $4 DO droplet |
| Fly Postgres backups | Data-loss risk | ✅ Mitigated — `.github/workflows/db-backup.yml` runs weekly (Mondays 03:17 UTC), `pg_dump -F c` of both `flower_shop_backend_core` and `keycloak` DBs to R2 |
| Stripe webhook replay attacks | Double-processing | Signature-verified in `StripeController`; idempotency key checked in `CreateOrderCommandHandler` |
| Keycloak realm drift between dev and Fly | Auth fails after redeploy | Realm export JSON committed at `docker/keycloak/flowershop-realm.json`; imported on boot |

## Alternatives considered (historical)

- **Render free for the portals** — the original proposal put the three portals on Render free. In the end **all three portals shipped on Fly.io** (region `fra`, `min_machines_running=1`) so cold starts never enter the picture and everything lives under one provider. Render is no longer used anywhere.
- **Aiven free PG** — the proposal called for Aiven. In the end PostgreSQL shipped on **Fly Postgres** (`flower-shop-postgres`, PG 17), co-located with the apps and reachable over `.flycast`.
- **Railway end-to-end** — better DX but only $5 one-time credit; not truly free long-term.
- **Supabase Auth instead of Keycloak** — removes a VM from the diagram, but the code already integrates with Keycloak (`KeycloakAdminClient`, `VendorOnboardingService`). Re-platforming auth is a separate epic, not a deployment task.
- **Azure Container Apps free tier** — generous free grant but requires a credit card and the 180k-vCPU-sec cap runs out fast with 5 background workers ticking. Rejected.

## Upgrade path (when demo graduates)

1. Move `FlowerShop.API` to a dedicated Fly VM → SLA-backed uptime.
2. HiveMQ Starter ($5/mo) → 1k device cap.
3. Provision real Redis (`ConnectionStrings:Redis`) and RabbitMQ (`RabbitMQ:ConnectionString`) if/when the cache and event-bus load warrant it — the code already supports both.
