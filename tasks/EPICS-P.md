# Epics, Stories & Tasks — Option P (Documentation portal authentication & role-based access)

> Companion to EPICS-B.md (portals), EP-23 (customer Google SSO) and EP-24 (showcase deployment).
> Epic is numbered EP-38 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep38-docs-portal-access-control.json`.
> Requirements: `docs/tasks/REQ-GAPS-P.md` (REQ-DOCS-01 … REQ-DOCS-16).
> Mobile counterparts: vendor app EP-M18 (T-38-016); customer mobile app Help (T-38-021).
> Analysis date: 2026-10-20

---

## EP-38: Documentation portal authentication & role-based access

**Priority**: P1-high
**Dependencies**: EP-23 (Keycloak Google IdP, JIT customers), EP-24 (Fly.io + Cloudflare)
**Requirement refs**: REQ-DOCS-01 through REQ-DOCS-16
**State**: in progress — implemented and tested (`FlowerShop.DocsPortal`, `/api/v1/docs/handoff` §24, Keycloak client `flowershop-docs` in the realm export + `scripts/keycloak/setup-docs-client.sh`, audience manifest/search build, Vendor Portal + Customer App Help, `fly.docs-portal.toml`, `deploy-docs-portal.yml`). Open: production cut-over and retiring the public copy (T-38-011, `docs/deployment/DOCS-PORTAL.md`), vendor app EP-M18 (T-38-016), customer mobile app Help (T-38-021).

> `docs.findmyflowers.pl` stops being public. Developers see everything, vendors see the vendor
> site and vendor app docs, and registered customers see the customer app docs. Help in each app
> opens the right page without a second login while the app's token is still valid.

### Concept

```
 Vendor Portal / Customer App / vendor app
   Help ─▶ POST /api/v1/docs/handoff (Bearer: current access token) ─▶ { url ?code=… (60 s, single use) }
        ─▶ docs.findmyflowers.pl/auth/handoff?code=…
                 │  back-channel redeem (flowershop-docs client credentials)
                 ▼
          FlowerShop.DocsPortal (Fly.io) ── cookie session (docs_roles) ── re-check every 15 min
                 │  token expired / code invalid ─▶ Keycloak sign-in (Google OK for customers)
                 ▼
          access-manifest.json (path → audiences, deny by default)
          developer: all · vendor: vendor-portal + vendor-mobile manuals · customer: customer manual
          nav + search filtered per role · 403 page otherwise · private, no-store, noindex
```

Roles: `User` → `docs-customer` · `VendorAdmin` → `docs-vendor` · `PlatformAdmin` and group
`developers` → `docs-developer` (Keycloak composites on client `flowershop-docs`).

### Design decisions
- **The hosting must move first.** GitHub Pages can't authenticate, and the current copy sits in
  a public repo. Docs are served by a small ASP.NET Core host (the same stack and Fly.io setup as
  the other portals). The public copy is retired only after cut-over is verified.
- **Deny by default.** Every page is developer-only unless its front matter says otherwise, and CI
  fails a manual with no explicit audience.
- **No leaks through navigation.** Search indexes are built per audience, and nav items are removed
  server-side, so a customer never sees vendor or developer titles.
- **Composites, not per-user grants.** Every registered customer gets the customer docs role
  through realm role `User`, including Google JIT users. There is nothing to backfill.
- **Token reuse without tokens in URLs.** The Vendor and Admin Portals use the password grant, so
  there is no Keycloak SSO cookie to rely on. A single-use, 60 s handoff code, redeemed
  back-channel by the docs client, reuses the app's valid token. Anything else falls back to a
  normal Keycloak sign-in.
- **Revocation works.** Roles are re-validated every 15 min, and sessions are capped at 24 h.

### Stories & tasks
See `tasks-ep38-docs-portal-access-control.json` (S-38-01 … S-38-06; T-38-001 … T-38-021).
1. **S-38-01 — Trackers** — T-38-001.
2. **S-38-02 — Identity and roles** — T-38-002 Keycloak client + composites · T-38-003 customer provisioning.
3. **S-38-03 — Audience-labelled docs build** — T-38-004 front matter + manifest · T-38-005 per-audience search/nav · T-38-006 content audit.
4. **S-38-04 — DocsPortal host** — T-38-007 project · T-38-008 nav/search filtering · T-38-009 session + re-validation · T-38-010 Fly + CI · T-38-011 retire GitHub Pages.
5. **S-38-05 — Token handoff and Help entry points** — T-38-012 handoff API · T-38-013 portal handoff + fallback · T-38-014 Vendor Portal Help · T-38-015 Customer App Help · T-38-016 EP-M18 · T-38-021 customer mobile app Help.
6. **S-38-06 — Operations, tests and rollout** — T-38-017 audit/health · T-38-018 local dev · T-38-019 tests · T-38-020 contract, runbook, changelog.

**Suggested order**: S-38-02 → S-38-03 → S-38-04 (deploy to a staging host name first) →
S-38-05 → DNS cut-over → T-38-011 retirement → S-38-06.

### Rollout
1. Create the Keycloak client/roles in prod (idempotent script) and verify `docs_roles` in tokens
   for one user of each role.
2. Deploy DocsPortal on a temporary host name (e.g. `docs-next.findmyflowers.pl`) and run the
   authorization matrix against it.
3. Ship the Help entries pointing at the configured docs URL (`Docs:BaseUrl`).
4. Switch the `docs.findmyflowers.pl` DNS to Fly and purge the Cloudflare cache.
5. Retire the public GitHub Pages copy (T-38-011) and request search de-indexing.
