# Requirement Gaps — Option P (Documentation portal authentication & role-based access)

> Companion to REQ-GAPS-B.md (portals) and EP-23 (customer Google SSO) / EP-24 (showcase
> deployment) trackers.
> Requirement refs point at the tasks in `tasks-ep38-docs-portal-access-control.json`.
> Analysis date: 2026-10-20
> Status: REQ-DOCS-01 … 16 implemented in code (EP-38, see `EPICS-P.md`); REQ-DOCS-02 (retiring the public copy) and the production cut-over are operational steps in `docs/deployment/DOCS-PORTAL.md`; REQ-DOCS-13 is vendor app EP-M18.

## Context

`docs.findmyflowers.pl` is a Jekyll site built from `docs/` by `.github/workflows/docs.yml` and
pushed to the `gh-pages` branch of the **public** repository
`FlowerShop-IoT-Platform/flowershop-iot-platform-docs`. Anyone can read everything: architecture,
deployment guides and all manuals (`docs/manuals/admin-portal-manual.html`,
`vendor-portal-manual.html`, `vendor-mobile-app-manual.html`, `customer-app-manual.html`). GitHub
Pages cannot enforce authentication, so access control means **moving the hosting**.

Identity today (`docker/keycloak/flowershop-realm.json`, realm `flowershop`): realm roles `User`
(customers, including Google SSO JIT provisioning from EP-23), `VendorAdmin`, `PlatformAdmin`;
clients `flowershop-api`, `flowershop-customer-app` (OIDC code flow), device clients. The Vendor
Portal and Admin Portal sign in with the **password grant** against Keycloak (no Keycloak browser
SSO session). The Customer App uses the OIDC code flow (it has one). The vendor mobile app holds
its own tokens.

**Target**

| Docs role | Can read | Who gets it |
|---|---|---|
| `developer` | everything | members of Keycloak group `developers`; `PlatformAdmin` (composite) |
| `vendor` | Vendor Portal manual, vendor mobile app manual, vendor FAQ/changelog | every `VendorAdmin` (composite) |
| `customer` | Customer app manual, customer FAQ | every registered customer, realm role `User` (composite) |

A user with several roles sees the union. Unlabelled pages are **developer-only**
(deny by default).

---

## A. Hosting and enforcement

### REQ-DOCS-01 — Serve docs from an authenticating host — P0
A new `FlowerShop.DocsPortal` (ASP.NET Core 8, minimal) serves the Jekyll-built `_site` as static
files behind cookie authentication and per-path authorization. It is deployed on Fly.io
(`fly.docs-portal.toml`, small machine, scale-to-zero allowed) at `docs.findmyflowers.pl`.
The CI workflow builds Jekyll, then the DocsPortal image with `_site` embedded, and deploys it.
The GitHub Pages deployment is removed.
**Accept**: every request except `/signin`, `/signout`, `/auth/*`, `/health` and static
theme assets (CSS/JS/fonts, no content) needs an authenticated session.

### REQ-DOCS-02 — Retire the public copy — P0
After cut-over: delete the `gh-pages` branch / archive (or make private) the public docs repo,
remove the Cloudflare/GitHub Pages DNS target, purge the Cloudflare cache, add
`X-Robots-Tag: noindex` on the new host and request removal of indexed URLs (Google Search
Console).
**Accept**: none of the old public URLs serve content; a checklist is recorded in the deployment
guide.

### REQ-DOCS-03 — Audience labels and access manifest — P0
Each page declares `audience:` in front matter (`developer`, `vendor`, `customer`; list allowed).
`_config.yml` defaults set `audience: [developer]` for the whole site and explicit overrides for
`manuals/vendor-*`, `manuals/customer-*` and the vendor/customer FAQ pages. The Jekyll build emits
`access-manifest.json` (path → audiences). A CI check fails the build if a manual has no explicit
audience. The manifest is not served.
**Accept**: unit test of the DocsPortal resolver (exact path, directory index, trailing slash,
`.html` and pretty URLs, assets referenced only by a protected page).

### REQ-DOCS-04 — Per-request authorization — P0
DocsPortal middleware resolves the path's audiences from the manifest. If the user's docs roles
intersect them, it serves the file with `Cache-Control: private, no-store`. Otherwise it returns
403 with a friendly page ("This page is for vendors. Signed in as … · Switch account"). An
unknown path returns 404.
**Accept**: Cloudflare is set to bypass cache for the host (no shared caching of protected HTML).

### REQ-DOCS-05 — Role-filtered navigation and search — P0
The sidebar and home page only list pages the user can open. The build emits one search index per
audience (`search-developer.json`, `search-vendor.json`, `search-customer.json`). The portal
serves the union for the user's roles, so search never leaks titles or snippets of forbidden
pages. The layout (`_layouts/default.html`) marks nav items with `data-audience`. The portal
removes forbidden items server-side (HTML rewrite at serve time, or per-audience nav partials at
build time).
**Accept**: a customer's page source contains no link or search entry to vendor or developer pages.

---

## B. Identity and roles

### REQ-DOCS-06 — Keycloak client and docs roles — P0
New confidential client `flowershop-docs` (authorization code + PKCE, redirect
`https://docs.findmyflowers.pl/signin-oidc`, post-logout `https://docs.findmyflowers.pl/`). It has
client roles `docs-developer`, `docs-vendor` and `docs-customer`, mapped into the token as claim
`docs_roles`.
Composite mappings: realm `User` → `docs-customer`; `VendorAdmin` → `docs-vendor`;
`PlatformAdmin` → `docs-developer`; group `developers` → `docs-developer`.
**Accept**: added to the dev realm export (`docker/keycloak/flowershop-realm.json`) and to the prod
realm by an idempotent setup script/runbook. All existing customers get `docs-customer` through
the composite without a per-user migration.

### REQ-DOCS-07 — Every registered customer is a docs customer — P0
Customers created by self-registration, by the API, or JIT through Google SSO (EP-23) get realm
role `User`, and through it `docs-customer`. `KeycloakAdminClient` verifies the role on
provisioning. A nightly reconciliation job reports (does not fix silently) customers without it.
**Accept**: integration test creates a customer via each path and asserts `docs_roles` contains
`customer`.

### REQ-DOCS-08 — Developer accounts — P1
Developers are Keycloak users in group `developers` (created by a platform admin in the Admin
Portal or the Keycloak console). A GitHub identity provider is not planned (decided
2026-10-20).
**Accept**: removing a user from the group removes access at their next request (session
re-validation, REQ-DOCS-11).

---

## C. Sign-in, token reuse and Help entry points

### REQ-DOCS-09 — Direct sign-in — P0
A visitor without a session is redirected to Keycloak (OIDC code + PKCE). Google login works for
customers (EP-23 IdP). If the browser already has a Keycloak SSO session (e.g. the Customer App),
sign-in is silent.
**Accept**: `/signout` ends the docs session and the Keycloak session (RP-initiated logout).

### REQ-DOCS-10 — Reuse the app's token: one-time handoff — P0
Help links from apps must not put tokens in URLs. Handoff flow:
1. The app (server side for portals, the mobile app directly) calls
   `POST /api/v1/docs/handoff { returnPath }` with its **current, valid access token**.
2. The API checks the token, derives docs roles from the user's realm roles/claims, and stores a
   single-use code (Redis, 60 s TTL, bound to `sub`, roles, `returnPath` and token expiry). It
   returns `{ url: "https://docs.findmyflowers.pl/auth/handoff?code=…" }`.
3. The app opens the URL (portal: 302; mobile: in-app browser tab, Custom Tabs /
   SFSafariViewController).
4. DocsPortal redeems the code back-channel (`POST /api/v1/docs/handoff/redeem`, authenticated
   with the `flowershop-docs` client credentials), creates its cookie session and redirects to
   `returnPath`.

If the app's token is expired or the code is invalid, used or expired, DocsPortal falls back to
REQ-DOCS-09 (fresh sign-in) and then continues to `returnPath`.
**Accept**: a code works once; replay → sign-in page; `returnPath` must be a relative docs path
(open-redirect test); the code is never logged.

### REQ-DOCS-11 — Session lifetime and re-validation — P0
The docs session cookie is `Secure`, `HttpOnly`, `SameSite=Lax`, sliding 8 h, absolute 24 h, and
never longer than the source token's refresh window for handoff sessions. Roles are re-read every
15 min (userinfo/introspection, or a refresh of the docs client's token), so revoked roles stop
working quickly.
**Accept**: removing `VendorAdmin` from a user blocks vendor pages within 15 min.

### REQ-DOCS-12 — Help in the Vendor Portal — P0
The main menu gets **Help** (icon `bi-question-circle`) and a per-page "?" link with deep links into
the Vendor Portal manual (e.g. `Stock/*` → `manuals/vendor-portal-manual#stock`, `Design/*` →
`#ai-florist`). A portal endpoint `GET /Help?topic=…` performs the handoff (REQ-DOCS-10) with
the session's access token and redirects. If the token can't be used, it redirects to the docs
sign-in.
**Accept**: opens in a new tab; works for every signed-in vendor role.

### REQ-DOCS-13 — Help in the vendor mobile app — P1
Vendor app (EP-M18): Help item in the main menu/drawer and contextual "?" on key screens. It calls
the handoff endpoint and opens the URL in an in-app browser tab. If the token is expired it first
refreshes; if that fails, it opens the docs sign-in.
**Accept**: no token in URLs, logs or analytics events.

### REQ-DOCS-14 — Help in the Customer App — P0
`FlowerShop.CustomerApp` menu shows **Help** only to signed-in customers. Anonymous visitors see
"Sign in to view help", which starts the normal sign-in and then the handoff. Deep links go into
the customer manual (map, orders, requests from EP-37). It uses the same handoff (the Customer App
holds an access token from its OIDC code flow).
**Accept**: anonymous users can't reach customer docs; a signed-in customer lands on the right
section without a second login. The **customer mobile app** gets the same Help item, using the
mobile flow of REQ-DOCS-13 (follow-up in its repo).

---

## D. Operations

### REQ-DOCS-15 — Audit and observability — P1
DocsPortal logs sign-in, handoff redemptions and 403s (user id, roles, path; no tokens/codes).
`/health` endpoint, Fly checks, and an alert on handoff redemption failures > 10 % in 15 min.

### REQ-DOCS-16 — Local development — P1
`docker-compose.dev.yml` gains the DocsPortal service against the local Keycloak. A
`Docs:AuthDisabled=true` switch is honoured **only** in Development, for writing docs locally.
`bundle exec jekyll serve` still works for content authors.
**Accept**: the switch is ignored (and logged) in any other environment.

---

## Cross-cutting
- **Contracts**: new API endpoints `POST /api/v1/docs/handoff` and `/handoff/redeem`. There is a new
  Keycloak client and the `docs_roles` claim. Update `docs/contracts/outbound.md` (Keycloak
  claim/scope) through `contracts-check`.
- **Content audit**: before go-live, review which pages are customer/vendor-safe. Admin Portal
  manual, architecture, deployment and API docs stay developer-only.
- **Changelog**: bump `docs_version` (`_config.yml`) and add a `changelog.md` entry for the access
  change.

## Explicitly out of scope
- Per-vendor (tenant-specific) documentation.
- Editing docs in the portal (content stays in Git).
- Making the source repository private. The `docs/` folder in this repo is unaffected; repository
  visibility is a separate decision.
