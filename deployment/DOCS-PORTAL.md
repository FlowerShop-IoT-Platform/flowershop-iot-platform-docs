# Docs portal — `docs.findmyflowers.pl` behind sign-in (EP-38)

`FlowerShop.DocsPortal` (ASP.NET Core 8, Fly app `flowershop-docs-portal`) serves the Jekyll build of `docs/` to signed-in
readers only, per audience:

| Docs role | Reads | Granted by (Keycloak) |
|---|---|---|
| `developer` | everything | realm role `PlatformAdmin`, group `developers` |
| `vendor` | Vendor Portal + vendor mobile app manuals | realm role `VendorAdmin` |
| `customer` | Customer app manual | realm role `User` (every registered customer) |

Pages are **developer-only unless labelled** (`audience:` front matter or `defaults` in `docs/_config.yml`). The build
emits `access-manifest.json` (path → audiences) and `search-{developer,vendor,customer}.json`; the portal never serves
those files, removes navigation the reader can't open (`data-audience` in `_layouts/default.html` / `_data/nav.yml`)
and answers `/search.json` with the union of the reader's indexes.

## How sign-in works

- **Direct**: a visitor without a session goes to `/signin` → Keycloak (code + PKCE, client `flowershop-docs`). Google
  sign-in works for customers through the EP-23 identity provider; an existing Keycloak session (Customer App) signs in
  silently. `/signout` ends both sessions (RP-initiated logout).
- **Help links from the apps** (Vendor Portal `Help`, Customer App `Help`, vendor app EP-M18): the app calls
  `POST /api/v1/docs/handoff` with its current access token and opens the returned URL. The docs portal redeems the
  60 s single-use code back-channel with its own client credentials (`POST /api/v1/docs/handoff/redeem`), creates its
  session and continues to the page. Any failure falls back to the direct sign-in, then the same page.
- **Session**: cookie `__Host-docs` (Secure, HttpOnly, SameSite=Lax), sliding 8 h, absolute 24 h (handoff sessions 8 h).
  Roles are re-read every 15 minutes (`POST /api/v1/docs/roles` → Keycloak admin API), so removing `VendorAdmin` or the
  `developers` group takes effect within 15 minutes. If the API is unreachable the session keeps its roles and retries
  a minute later.

## 1. Keycloak (once per environment)

Dev: the realm export `docker/keycloak/flowershop-realm.json` already contains the client (secret
`dev-docs-portal-secret-change-me`), the client roles, the composites and the `developers` group. An existing dev
database needs a re-import (see `docker/keycloak/README.md`) or the script below against `http://localhost:8090`.

Production (no re-import):

```bash
fly ssh console -a flowershop-keycloak
# inside the machine
KCADM=/opt/keycloak/bin/kcadm.sh KC_URL=http://localhost:8080 KC_ADMIN=admin KC_ADMIN_PASSWORD=... \
  DOCS_URL=https://docs.findmyflowers.pl bash /path/to/setup-docs-client.sh
```

(`scripts/keycloak/setup-docs-client.sh` is idempotent; copy it in with `fly ssh sftp` or paste it.) It prints the client
secret at the end. Developers are added by a platform admin: Keycloak console → Users → *user* → Groups → `developers`.

## 2. DocsPortal on Fly

```bash
fly apps create flowershop-docs-portal
fly volumes create flowershop_docs_dp_keys --app flowershop-docs-portal --region fra --size 1
fly secrets set --app flowershop-docs-portal Docs__Keycloak__ClientSecret=<secret from step 1>
fly deploy --config fly.docs-portal.toml --dockerfile src/portals/FlowerShop.DocsPortal/Dockerfile
```

The image builds the site with `docs/Gemfile` (same plugins as GitHub Pages) and fails if `access-manifest.json` is
missing. Health: `GET /health` → `{ status, manifestEntries, handoff: { ok, failed, windowMinutes } }`.

The API needs no new secret: it validates the docs client's token through the `flowershop-api` audience mapper, and its
`Docs:PublicUrl` defaults to `https://docs.findmyflowers.pl`.

## 3. Cut-over

1. `fly certs add docs.findmyflowers.pl --app flowershop-docs-portal`.
2. Cloudflare DNS: point `docs` (CNAME) at `flowershop-docs-portal.fly.dev` instead of GitHub Pages.
3. Cloudflare → Caching → Cache Rules: **Bypass cache** for hostname `docs.findmyflowers.pl` (protected HTML must never
   be shared-cached; the portal also sends `Cache-Control: private, no-store`).
4. GitHub → repository variables: `DOCS_PORTAL_DEPLOY = true`. From now on `deploy-docs-portal.yml` deploys on docs
   changes and `docs.yml` stops publishing to GitHub Pages.
5. Check: `curl -I https://docs.findmyflowers.pl/architecture/` → `302` to `/signin`; sign in as a vendor → the vendor
   manual opens, `/architecture/` returns 403; Help in the Vendor Portal opens the manual without a second login.

## 4. Retire the public copy (REQ-DOCS-02) — checklist

- [ ] Delete the `gh-pages` branch of `FlowerShop-IoT-Platform/flowershop-iot-platform-docs`, then archive the repo or make it private.
- [ ] GitHub Pages settings of that repo: remove the custom domain.
- [ ] Remove the `DOCS_DEPLOY_KEY` secret from this repo (and the deploy key from the docs repo).
- [ ] Cloudflare: **Purge everything** for `docs.findmyflowers.pl`.
- [ ] Google Search Console: request removal of `https://docs.findmyflowers.pl/` URLs (the portal sends `X-Robots-Tag: noindex`).
- [ ] Verify an old public URL (e.g. `https://flowershop-iot-platform.github.io/flowershop-iot-platform-docs/`) no longer serves content.

## 5. Writing docs

- New page: it is developer-only by default. For a vendor or customer page add `audience: [vendor]` (or `[customer]`,
  or several) in its front matter; for a static HTML manual add a `defaults` entry in `docs/_config.yml`.
- Sidebar items live in `docs/_data/nav.yml` with an `audience`.
- `python3 scripts/docs/check_audiences.py --site docs/_site` (CI job *Docs audiences*) fails when a manual has no
  explicit audience or an unknown audience value is used.
- Local preview: `cd docs && bundle install && bundle exec jekyll serve` (no access control), or the portal with
  `Docs:AuthDisabled=true` (Development only — ignored and logged anywhere else):
  `ASPNETCORE_ENVIRONMENT=Development Docs__AuthDisabled=true dotnet run --project src/portals/FlowerShop.DocsPortal`
  after `bundle exec jekyll build -s docs -d docs/_site`. For the real sign-in flow leave `AuthDisabled` off; the portal
  then uses the local Keycloak (`localhost:8090`) and the API (`localhost:8080`). The compose service
  `docs-portal-dev` (profile `docs`) runs the image with `AuthDisabled` on.

## 6. Observability (REQ-DOCS-15)

Structured logs (no tokens or codes): `Docs sign-in (OIDC|handoff) for {Subject} with roles {Roles}`, `Docs handoff
failed`, `Docs 403 for {Subject} with roles {Roles} on {Path}`, `Docs access revoked`. Alert on the warning
`Docs handoff failure rate above 10 %` (logged when more than 10 % of at least 10 redemptions in 15 minutes failed);
the same counts are on `/health`. The API logs `Docs handoff issued/redeemed` and, nightly at 03:00 UTC, `Docs role
reconciliation: N customer(s) lack the User realm role` (report only — fix by assigning the role).
