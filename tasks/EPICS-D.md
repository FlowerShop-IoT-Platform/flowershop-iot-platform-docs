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
  │       └── S-24-03 (three portals: admin. / vendors. / apex)
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
> **executes** it, with three deltas: (1) the requested URL layout — Customer App on the bare apex,
> Vendor Portal on `vendors.` (plural), public docs on `docs.` (new); (2) Postgres is on Fly, not
> Aiven; (3) two code blockers merge first — portals hardcode the Keycloak URL + client secret, and
> prod CORS is a wildcard that breaks credentialed calls.

**Verified live state (2026-07-06):** API healthy on `flower-shop-backend-core.fly.dev` in dev-auth
mode; Fly Postgres live; R2 + `img.findmyflowers.pl` working; Cloudflare zone live. Keycloak, the
`api.` custom domain, and all three portals are undeployed.

### S-24-01 — Pre-deploy code blockers *(P0)*
These ship in the image, so they merge before any (re)deploy.
- **T-24-001** — De-hardcode portal Keycloak base URL + client secret → config (`Authentication:KeycloakBaseUrl`, `Authentication:ClientSecret`), dev defaults preserved. Both portals use the `flowershop-api` client via password grant.
- **T-24-002** — Replace prod CORS `["*"]` with the three real origins (apex + admin + vendors.). This is also where REQ-DEP-10 lands.
- **T-24-003** — Scrub docker-hostname placeholders in `appsettings.Production.json` (DB/Redis/MQTT/JWT); confirm each is Fly-secret-overridden or genuinely unused.

### S-24-02 — Keycloak + API on real auth *(P0, critical blocker)*
- **T-24-004** — `CREATE DATABASE keycloak` on Fly Postgres (`flower-shop-postgres.flycast`).
- **T-24-005** — Deploy `flowershop-keycloak` Fly app (keycloak:23, ≥1 GB, always-on) importing the committed realm. A `fly.keycloak.toml` already exists — reuse/verify.
- **T-24-006** — Attach `auth.findmyflowers.pl` (grey-cloud CNAME) + set the real Google IdP secret + update Google's OAuth redirect. Read the `flowershop-api` client secret here.
- **T-24-007** — Attach `api.findmyflowers.pl` to the API (parallelizable).
- **T-24-008** — Flip API to `UseKeycloak=true` (+ Authority/ClientId/ClientSecret) **only after** T-24-006 verifies; confirm `dev-token` → 404.

### S-24-03 — Three portals on branded domains *(P0)*
- **T-24-009** — Admin Portal → Render → `admin.findmyflowers.pl`.
- **T-24-010** — Vendor Portal → Render → **`vendors.`** (plural); audit workflow/docs for `vendor.` singular refs.
- **T-24-011** — Customer App → **bare apex** `findmyflowers.pl` (Cloudflare CNAME-flattening) + OIDC env + Keycloak redirect URIs. The apex is the one divergence with real technical cost.

### S-24-04 — Public docs *(P1, independent)*
- **T-24-012** — Wire `docs.findmyflowers.pl` to the existing GitHub Pages docs publish (durable `docs/CNAME` + Cloudflare CNAME + Pages custom domain/HTTPS). New work, not in the runbook.

### S-24-05 — Backing services *(P1/P2)*
- **T-24-013** — Point MQTT at HiveMQ Cloud for the vase demo (**required** — physical vase in scope). Flash the vase firmware with the HiveMQ credential.
- **T-24-014** — Wire Stripe test-mode + webhook (**optional** — only if checkout is demoed).

### S-24-06 — Hardening & go-live *(P1)*
- **T-24-015** — cron-job.org keep-warm pings on Render portals + weekly `pg_dump` → R2 (Fly PG has no auto-backups).
- **T-24-016** — Scripted end-to-end smoke test across every branded domain; all green = go-live. Consider driving it via Playwright (MCP browser tools available).

---

## Cost summary (pilot showcase)

| Component | Host | Monthly |
|---|---|---|
| API (always-on, hosts MQTT + background services) | Fly | ~$5.70 |
| Keycloak (always-on) | Fly | ~$5.70 |
| Customer App | Fly or Render free | ~$0–3.89 |
| Admin + Vendor portals | Render free (cold-start OK) | $0 |
| Postgres | Fly | (existing) |
| R2 photos + img. | Cloudflare | $0 |
| MQTT | HiveMQ Cloud free | $0 |
| DNS | Cloudflare | $0 |
| Docs | GitHub Pages | $0 |
| Domain | (already registered) | ~€15/yr |

**≈ $10–15/mo cash** (offset by Fly's $5 Hobby credit), accepting Render cold starts on the two
internal portals.

---

## Notes & risks

- **Apex CNAME wrinkle** (T-24-011): a bare apex can't be a plain CNAME to Render/Fly — use
  Cloudflare CNAME-flattening (proxy ON, Cloudflare terminates TLS) or A/AAAA records to the host
  IPs. This is the single item the requested URL layout adds versus the written runbook.
- **Domain string divergence**: every DNS record, CORS entry, OIDC redirect URI, and workflow
  health-check URL must use the apex (not `app.`) and `vendors.` (not `vendor.`). Audit
  `deploy-vendor-portal.yml` specifically.
- **Ordering guard** (T-24-008): the API must not flip to real auth before Keycloak's JWKS is
  reachable, or everything 401s.
- **Realm has no admin/vendor client** — both portals authenticate through `flowershop-api` via
  password grant, which is why T-24-001 (config-driven secret) is a hard prerequisite for portal
  login in prod.
- Inherited risks (from DEPLOYMENT-SPEC.md): Fly free-VM eviction, Render cold-start chain,
  HiveMQ 100-connection cap, no automated PG backups (mitigated by T-24-015).
