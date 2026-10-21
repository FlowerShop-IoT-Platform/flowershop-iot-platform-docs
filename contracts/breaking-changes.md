# Contract change log

Cross-repo contract changes, newest first. Breaking changes state the safe deployment order.

## 2026-10-21 — EP-33 AI Florist feedback loop (additive, non-breaking)

- **What changed**: new vendor endpoints for ratings, refine, copy signal, house style and "forget my
  ratings"; `POST …/preview` accepts an optional `{ corrections, note }` body (re-render); `POST …/recipes`
  accepts `occasion`, `colourPreference`, `mustInclude`, `leaveOut`, `briefNote` and sizes `S`/`L`.
  `DesignRecipeDto` gains `why`, `size`, `parentRecipeId`, `generation`, `feedback`, `previewFeedback`;
  inventory items gain `hintApplied`; settings gain the brief fields; usage gains `sizeRules`, `occasions`,
  `refineChips`, `recipeFeedbackReasons`, `previewFeedbackReasons` and `sizes` is now `["S","M","L"]`;
  jobs gain `refineOfRecipeId`. New 422 codes `invalid-brief`, `must-include-missing`, `refine-limit`.
  Admin endpoints `/api/v1/admin/analytics/ai-florist[/feedback-export]`. Config
  `BouquetDesign:{Step}:PromptVersion|CandidatePromptVersion|CandidatePercent`, `Composition:RepairEnabled`.
  Migration `AddDesignFeedbackLoop` (4 new tables, job/session columns).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M13 for the new UI); Vendor and Admin Portals in this repo.
- **Deployment order**: API first (migration applies on start), then the portals, then the mobile build.
  Old clients ignore the new fields; an old client that renders `sizes` as a picker now also sees S and L,
  which the API accepts.

## 2026-10-18 — EP-31 Stock photos & AI appearance (additive, non-breaking)

- **What changed**: new photo / AI endpoints under `/api/v1/vendor/stock` (`API-CONTRACTS.md` §22.1,
  `outbound.md` §2c); `StockItemDto` gains `variety`, `appearance`, `primaryPhoto`; create/update accept
  `variety`, `appearance` (update: omitted = unchanged), create accepts `pendingPhotoId`; consume lines
  accept `variety`. Assistant inventory items and recipe lines gain optional `stockItemId` + `variety`.
  New config `BouquetDesign:StockRecognition:*` and `Limits:StockRecognitionsPerMonth`. Migration
  `AddStockPhotosAndAppearance` (column `Variety`, jsonb `Appearance`, 2 new tables; catalogue key
  becomes `name|colour[|variety]`, unchanged for items without a variety).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M12); Vendor Portal in this repo.
- **Deployment order**: API first (migration applies on start), then the Vendor Portal, then the mobile
  build. Old clients ignore the new fields; an old client's PUT omits `appearance`/`variety` and leaves them as they are.

## 2026-10-18 — EP-30 Vendor stock (additive, non-breaking)

- **What changed**: new vendor endpoints under `/api/v1/vendor/stock` (see `outbound.md` §2c and
  `API-CONTRACTS.md` §22); migration `AddStockManagement` (2 new tables). Assistant sessions now fill a
  missing unit price from the vendor's stock unit cost before the remembered assistant price — same
  payload shape, only the filled value can differ.
- **Affected repos**: `flower-shop-vendor-mobile-app` (new consumer, EP-M11); Vendor Portal in this repo.
- **Deployment order**: backend first (migration applies on start), then the Vendor Portal, then the
  mobile build. An older app never calls the new endpoints.

## 2026-10-14 — EP-28 AI Florist (additive, non-breaking)

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
