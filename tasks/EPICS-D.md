# Epics, Stories & Tasks — Option D (Showcase Deployment & Go-Live)

> Companion to EPICS.md (EP-01–09, Mobile API), EPICS-B.md (EP-10–18, Portals & Analytics),
> and EPICS-C.md (EP-19–23, Vendor onboarding / subscription / vase / SSO).
> Epic is numbered EP-24 to continue the sequence.
> Requirement refs point to REQ-GAPS-D.md.
> Task tracker (ground truth): tasks-ep24-showcase-deployment.json.
> Analysis date: 2026-07-06

---

## Dependency Map

```
Existing live infra (API + Fly Postgres + R2/img + Cloudflare DNS)
  │
  ├── S-24-01 (code blockers: portal auth config, prod CORS)   ← must merge first
  │
  ├── S-24-02 (Keycloak @ auth. → API @ api. → flip to real auth)   🔴 critical path
  │       └── S-24-03 (three portals: admin. / vendors. / app.)
  │               └── (CORS wired to real origins — REQ-DEP-10, done in S-24-01's T-24-002)
  │
  ├── S-24-04 (docs.findmyflowers.pl)          ← independent, parallelizable
  ├── S-24-05 (HiveMQ MQTT [required], Stripe [optional])   ← after api domain
  └── S-24-06 (keep-warm crons + backup, then go-live smoke test)   ← last
```

**Critical path:** S-24-01 → S-24-02 → S-24-03 → S-24-06.
**Parallelizable:** S-24-04 (docs) and the API-domain half of S-24-02 (T-24-007) can run alongside
the Keycloak work.

---

## EP-24: Full-Fledged Showcase Deployment & Go-Live

**Priority**: P0-critical
**Dependencies**: EP-02 (Keycloak auth), EP-15 (IoT/MQTT + R2), EP-23 (Google SSO realm)
**Requirement refs**: REQ-DEP-01 through REQ-DEP-15

> Take the platform live at `findmyflowers.pl` for a pilot flower shop, including a live demo of one
> physical smart vase, at ~$10/mo. A complete plan already exists in `docs/deployment/`; this epic
> **executes** it, with three deltas: (1) the requested URL layout — Customer App on a dedicated
> subdomain (the plan first floated the bare apex; as-built it shipped on `app.findmyflowers.pl`),
> Vendor Portal on `vendors.` (plural), public docs on `docs.` (new); (2) Postgres is on Fly, not
> Aiven; (3) two code blockers merge first — portals hardcode the Keycloak URL + client secret, and
> prod CORS is a wildcard that breaks credentialed calls.

**Live state (updated 2026-07-12): DONE.** The platform is fully deployed and live at
`findmyflowers.pl`. API on `api.findmyflowers.pl` with **real Keycloak auth** (`UseKeycloak=true`);
Fly Postgres (PG 17) live; R2 + `img.findmyflowers.pl` working; Cloudflare zone live. Keycloak
deployed at `auth.findmyflowers.pl`; **all three portals deployed to Fly.io** (not Render) — Admin
`admin.`, Vendor `vendors.`, Customer App on `app.findmyflowers.pl`. Docs on `docs.findmyflowers.pl`. HiveMQ Cloud
MQTT (TLS 8883) live; weekly Postgres backup to R2 (`db-backup.yml`). All apps run in region `fra`,
always-on (`min_machines_running=1`).

### S-24-01 — Pre-deploy code blockers *(P0)*
These ship in the image, so they merge before any (re)deploy.
- **T-24-001** — De-hardcode portal Keycloak base URL + client secret → config (`Authentication:KeycloakBaseUrl`, `Authentication:ClientSecret`), dev defaults preserved. Both portals use the `flowershop-api` client via password grant.
- **T-24-002** — Replace prod CORS `["*"]` with the three real origins (app. + admin. + vendors.). This is also where REQ-DEP-10 lands.
- **T-24-003** — Scrub docker-hostname placeholders in `appsettings.Production.json` (DB/Redis/MQTT/JWT); confirm each is Fly-secret-overridden or genuinely unused.

### S-24-02 — Keycloak + API on real auth *(P0, critical blocker)*
- **T-24-004** — `CREATE DATABASE keycloak` on Fly Postgres (`flower-shop-postgres.flycast`).
- **T-24-005** — Deploy `flowershop-keycloak` Fly app (keycloak:23, ≥1 GB, always-on) importing the committed realm. A `fly.keycloak.toml` already exists — reuse/verify.
- **T-24-006** — Attach `auth.findmyflowers.pl` (grey-cloud CNAME) + set the real Google IdP secret + update Google's OAuth redirect. Read the `flowershop-api` client secret here.
- **T-24-007** — Attach `api.findmyflowers.pl` to the API (parallelizable).
- **T-24-008** — Flip API to `UseKeycloak=true` (+ Authority/ClientId/ClientSecret) **only after** T-24-006 verifies; confirm `dev-token` → 404.

### S-24-03 — Three portals on branded domains *(P0)* — DONE (deployed to Fly.io, not Render)
- **T-24-009** — Admin Portal → Fly app `flowershop-admin-portal` → `admin.findmyflowers.pl`.
- **T-24-010** — Vendor Portal → Fly app `flowershop-vendor-portal` → **`vendors.`** (plural).
- **T-24-011** — Customer App → Fly app `flowershop-customer-app` at **`app.findmyflowers.pl`** (plain subdomain CNAME to `flowershop-customer-app.fly.dev`) + OIDC env + Keycloak redirect URIs. (The plan first floated the bare apex; as-built it shipped on the `app.` subdomain, so no CNAME-flattening was needed.)

> **Delta from plan:** the three portals were deployed to **Fly.io** (region `fra`, always-on), not
> Render. There is no Render in the live stack; keep-warm crons are unnecessary
> (`min_machines_running=1`).

### S-24-04 — Public docs *(P1, independent)*
- **T-24-012** — Wire `docs.findmyflowers.pl` to the existing GitHub Pages docs publish (durable `docs/CNAME` + Cloudflare CNAME + Pages custom domain/HTTPS). New work, not in the runbook.

### S-24-05 — Backing services *(P1/P2)*
- **T-24-013** — Point MQTT at HiveMQ Cloud for the vase demo (**required** — physical vase in scope). Flash the vase firmware with the HiveMQ credential.
- **T-24-014** — Wire Stripe test-mode + webhook (**optional** — only if checkout is demoed).

### S-24-06 — Hardening & go-live *(P1)* — DONE
- **T-24-015** — Weekly `pg_dump` (custom format, both DBs) → R2 via `.github/workflows/db-backup.yml` (Mondays 03:17 UTC). Keep-warm crons dropped — all Fly apps run always-on (`min_machines_running=1`).
- **T-24-016** — End-to-end smoke test across every branded domain (all green → go-live).

---

## Cost summary (pilot showcase)

> **As deployed:** all three portals run on Fly.io (region `fra`, always-on), not Render.

| Component | Host | Monthly |
|---|---|---|
| API (always-on, hosts MQTT + background services) | Fly | ~$5.70 |
| Keycloak (always-on) | Fly | ~$5.70 |
| Customer App | Fly | ~$3.89 |
| Admin + Vendor portals | Fly | ~$3.89 ea. |
| Postgres | Fly | (existing) |
| R2 photos + img. | Cloudflare | $0 |
| MQTT | HiveMQ Cloud free | $0 |
| DNS | Cloudflare | $0 |
| Docs | GitHub Pages | $0 |
| Domain | (already registered) | ~€15/yr |

Cash cost scales with how many portal VMs stay always-on (offset by Fly's $5 Hobby credit).

---

## Notes & risks

- **Customer App subdomain** (T-24-011): the plan first floated a bare apex (which can't be a plain
  CNAME to Fly, and would have needed Cloudflare CNAME-flattening). As-built, the Customer App
  shipped on **`app.findmyflowers.pl`** — a plain subdomain CNAME to `flowershop-customer-app.fly.dev`,
  exactly like the other Fly apps, so no CNAME-flattening was needed.
- **Domain string divergence**: every DNS record, CORS entry, OIDC redirect URI, and workflow
  health-check URL uses `app.` (Customer App) and `vendors.` (not `vendor.`).
- **Ordering guard** (T-24-008): the API must not flip to real auth before Keycloak's JWKS is
  reachable, or everything 401s.
- **Realm has no admin/vendor client** — both portals authenticate through `flowershop-api` via
  password grant, which is why T-24-001 (config-driven secret) is a hard prerequisite for portal
  login in prod.
- Inherited risks (from DEPLOYMENT-SPEC.md): Fly VM eviction, HiveMQ 100-connection cap. Automated
  PG backups are now in place (`db-backup.yml`, weekly to R2).
