# Contract change log

Cross-repo contract changes, newest first. Breaking changes state the safe deployment order.

## 2026-10-21 — EP-37 Customer bouquet requests (additive; two behaviour changes on vendor orders)

- **What changed**: new `/api/v1/customer/bouquet-requests`, `/api/v1/vendor/bouquet-requests`, `/vendor/custom-orders`,
  `/vendor/marketplace-settings`, `/vendor/notifications`, `/api/v1/admin/bouquet-requests` endpoints and SignalR hub
  `/hubs/marketplace` (`API-CONTRACTS.md` §23). `NotificationType.BouquetRequest` (5) and `bouquetRequestId` on customer
  notifications. `DesignAnalysisDto.estimatedCost` (set when the design came from a request). Vendor order DTOs gain
  `isCustom`, `customTitle`, `customNotes`, `bouquetRequestId`; for custom orders the customer name and address are
  **empty until paid**, and `PUT /vendor/orders/{id}/status` returns 409 for custom orders (use `…/phase`). Config
  `Marketplace:*`. Migration `AddBouquetRequests` (5 new tables; nullable columns on `Orders`, `SavedAddresses`,
  `Notifications`).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M17; order list must tolerate custom orders without a bouquet
  and must not offer status buttons for them); customer mobile app (follow-up); portals in this repo.
- **Deployment order**: API (migration on start) → portals → mobile builds.

## 2026-10-21 — EP-36 Cost assessor (additive, non-breaking)

- **What changed**: new `/api/v1/vendor/cost-assessments` endpoints (`API-CONTRACTS.md` §21.4); `DesignUsageDto`
  `assessmentsUsed/Limit/Remaining`; `DesignCostProfileDto.targetMarginPercent` (PUT accepts it, optional). Config
  `BouquetDesign:Assessment:MonthlyLimit`. Migration `AddCostAssessments` (1 new table, 1 column on
  `DesignCostProfiles` defaulting to 60).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M16); Vendor Portal in this repo.
- **Deployment order**: API (migration on start) → Vendor Portal → mobile build.

## 2026-10-21 — EP-35 Reconstruct a bouquet from photos (additive, non-breaking)

- **What changed**: `inputMode: "BouquetPhotos"` (listed in `usage.inputModes`); new endpoints
  `POST /design-sessions/{id}/analyze`, `PUT …/analysis/items/{itemId}`, `POST …/analysis/use` (`API-CONTRACTS.md`
  §21.3); `DesignSessionDto.analysis` and `sourceRequestId`; session states `Analyzing`, `AnalysisReady`; job kind
  `BouquetAnalysis` (counts as a recognition). Config `BouquetDesign:Analysis:*`. Migration `AddBouquetAnalysis`
  (2 nullable columns on `DesignSessions`).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M15; clients must tolerate the new states and job kind
  when listing sessions); Vendor Portal in this repo.
- **Deployment order**: API (migration on start) → Vendor Portal → mobile build.

## 2026-10-21 — EP-34 Visual stock start + count stock from photos (additive; one deprecation)

- **What changed**: new `/api/v1/vendor/stock/counts` endpoints (`API-CONTRACTS.md` §22.2); `StockItemDto.mainPhotoId`;
  design `InventoryItemDto` gains `mainPhotoId`, `fallbackUnitPrice`, `fallbackSource`; `DesignUsageDto.inputModes`;
  `POST /design-sessions` accepts `inputMode: "Stock"` (= `Manual`). Config `BouquetDesign:LegacyPhotosMode`.
  Migration `AddStockCounts` (1 new table). Stock counts share the stock AI monthly quota and the global $ cap.
- **Deprecation**: `inputMode: "Photos"` for **new** sessions. Still accepted while `LegacyPhotosMode=true` (default);
  once off, a new `Photos` session is a 400. Existing Photos sessions and `/recognize` keep working.
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M14 must move "photograph your stock" to the stock screens
  before the flag is switched off); Vendor Portal in this repo.
- **Deployment order**: API (migration on start) → Vendor Portal → mobile build (EP-M14) → then set
  `BouquetDesign__LegacyPhotosMode=false`.

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
