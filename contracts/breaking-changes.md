# Contract change log

Cross-repo contract changes, newest first. Breaking changes state the safe deployment order.

## 2026-10-14 — EP-28 AI Bouquet Assistant (additive, non-breaking)

- **What changed**: new vendor endpoints under `/api/v1/vendor/design-sessions`, `/design-recipes`,
  `/design-cost-profile`, `/design-assistant/usage` (see `outbound.md` §2c and `API-CONTRACTS.md` §21);
  new config section `BouquetDesign:*` (§6a); migration `AddBouquetDesignAssistant` (3 new tables).
- **Behaviour change (server-internal)**: `IdempotencyMiddleware` now scopes its response cache by
  caller + path + key (`idempotency:{sub}:{path}:{key}`) instead of the bare key, so two users can no
  longer replay each other's cached response. Clients are unaffected; cached entries written before
  the deploy simply stop matching (24 h TTL).
- **Affected repos**: `flower-shop-vendor-mobile-app` (new consumer, EP-M10). No change for portals,
  firmware or the customer app.
- **Deployment order**: backend first (migration applies on start), then the mobile build. An old
  mobile build never calls the new endpoints; a new mobile build against an old backend still shows the
  assistant entry points (the app hides them only on an explicit `enabled: false`), and the
  assistant screens report the 404 as an error.
