# Contract change log

Cross-repo contract changes, newest first. Breaking changes state the safe deployment order.

## 2026-10-25 — Production bug triage fixes (additive; some texts change)

- **What changed**: `VendorMarketplaceSettingsDto` gains `latitude` / `longitude` (M17-1); `StockCountLineDto` gains
  `checkNote` and `suggestedStockItemId` (M14-2) (`API-CONTRACTS.md` §22, §23).
- **Behaviour changes**: `StockMatchDto.factors` no longer contains `"AI NN % sure"` (M20-1); identical colour text
  scores "colour close" and colour phrases are read by their main clause (M16-1); stock counts merge duplicate lines
  (larger quantity, flagged) and no longer link a line to an item of a clearly different colour (M14-2); marketplace
  alert texts (dates, distances, prices) are formatted in the invariant culture — "Sat 24 Oct 14:00", "0.2 km",
  "45.00 PLN" — instead of the request culture (M17-2b; already stored alerts keep their text). A preview rejected by
  the provider no longer uses the preview allowance or the $ cap (M13-1). The API refuses to start with a mismatched
  preview provider/model or a missing provider key while previews are enabled (config, not contract).
- **Affected repos**: `flower-shop-vendor-mobile-app` (requests map marker/circle, optional `checkNote` display, the
  "sure" factor guard); Vendor Portal in this repo.
- **Deployment order**: set the preview secrets first (`BouquetDesign__Preview__Provider` / `__Model` must agree), then
  the API, then the portal and mobile build. Old clients ignore the new fields.

## 2026-10-24 — EP-41 Flower atlas, item crops and image matching (additive)

- **What changed**: analysis items gain `aiName`, `aiVariety`, `atlas`, `boxes`, `cropUrl`, `imageMatches`; recognised
  inventory lines and stock count lines gain `atlas` (`API-CONTRACTS.md` §21.6). `PUT …/analysis/items/{itemId}` accepts
  `rename` / `clearRename`. New routes: `GET /api/v1/vendor/design-sessions/{id}/media/crops/{itemId}`,
  `GET /api/v1/vendor/cost-assessments/{id}/media/crops/{itemId}`, `GET /api/v1/vendor/atlas/vocabulary`, anonymous
  `GET /api/v1/atlas/images/{imageId}/content` and `GET /api/v1/atlas/attributions`, and the admin atlas API
  `/api/v1/admin/atlas/*` (§26). Prompt `bouquet-analysis-v2` (boxes) as a 0 % candidate. Config
  `BouquetDesign:Embedding:*` (Provider `None` by default) and `Atlas:*`. Migrations `AddFlowerAtlas` and
  `AddImageEmbeddings` (installs pgvector when the server offers it; never fails without it).
- **Behaviour changes**: none until an admin approves atlas images (then `atlas.imageUrl` appears), the v2 candidate
  is raised above 0 % (then crops appear), or the embedding provider is switched on (then `imageMatches` and the image
  factor in match scores). A renamed line reports the florist's name in `name` (the AI's in `aiName`).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M21: three thumbnails, tap-to-highlight, disagreement flag,
  "Could be"); Vendor Portal and Admin Portal in this repo.
- **Deployment order**: API first (migrations apply on start), then the portals, then the mobile build. Old clients
  ignore the new fields.

## 2026-10-23 — EP-40 Recognition confidence from human feedback (additive; `needsCheck` meaning changes)

- **What changed**: new verdict endpoints `PUT /api/v1/vendor/design-sessions/{id}/analysis/items/{itemId}/verdict`,
  `PUT …/inventory/items/{itemId}/verdict` and `PUT /api/v1/vendor/stock/counts/{countId}/lines/{lineId}/verdict`;
  recognised line DTOs gain `confidenceScore` and `verdict`; `DesignStyleProfileDto` gains `shareRecognitionFeedback`
  (default true, omitted on PUT = unchanged); `AiFloristQualityDto` gains `calibration`; the feedback CSV gains verdict
  columns; admin `POST …/ai-florist/recognition-verdicts/backfill`; config `BouquetDesign:Confidence:*`; migration
  `AddRecognitionConfidence` (`API-CONTRACTS.md` §21.5).
- **Behaviour changes** (once `BouquetDesign:Confidence:ScoresEnabled` is switched on — it ships off, so at first only
  the verdict endpoints and `verdict` fields are live): `confidence` (raw) is still returned but deprecated for display. `StockCountLineDto.needsCheck`
  is now true when the calibrated count score is Low **or there are no checks yet** — at launch every new count line
  is flagged; clients that untick flagged lines by default should base that on `confidenceScore.count` instead (the
  Vendor Portal does). Bouquet analysis lines that florists' checks show are often wrong are no longer auto-linked, so
  some lines move from Matched/Short to Missing until the florist confirms them. Match scores use the calibrated score
  (about 5 points lower at cold start for 90 %-sure lines).
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M20: verdict control, display states, opt-out); Vendor
  Portal and Admin Portal in this repo.
- **Deployment order**: API first (migration applies on start), then the portals, then the mobile build. Old clients
  ignore the new fields; an old client's house-style PUT leaves `shareRecognitionFeedback` as it is.

## 2026-10-23 — EP-39 Online-only vendors (additive, with nullable vase ids and stricter status codes)

- **What changed**: new `POST /api/v1/vendor/online-listings`, `GET/PUT /api/v1/vendor/settings/sales-mode` and
  `PUT /api/v1/admin/vendors/{id}/sales-mode`; `register-with-admin` accepts optional `salesMode`
  (`API-CONTRACTS.md` §25). Bouquet read models (nearby, detail, `/bouquets/available`, vendor bouquets) gain
  `listingChannel` (`"vase"`/`"online"`) and `deliveryOnly` (vendor bouquets also `expiresAt`); vendor DTOs gain
  `salesMode`; the order preview gains `pickupAvailable` and `lines[].deliveryOnly`. **`vaseId` on cart items, order
  lines and vendor order lines can now be `null`** (online listings). The map no longer returns vase-less catalog
  bouquets (they could never be bought). New error codes: `bouquet-not-listed` (409, replaces "not placed in a vase"),
  `pickup-not-offered` / `mixed-listing-channels` (422), `vendor-not-online` / `vendor-has-vases` (409). A lost race
  for a bouquet is now **409** (`bouquet-reserved` on cart, `bouquet-unavailable` on checkout) instead of 400.
  `PUT /vendor/orders/{id}/status` refuses `ReadyForPickup` for online orders. Domain events `BouquetBecameAvailable` /
  `BouquetSold` carry a nullable `VaseId`. Config `OnlineListings:*`. Migration `AddOnlineOnlyListings` (int
  `SalesMode` on `FlowerShops`/`FreelanceFlorists`, int `ListingChannel` on `Bouquets`, all default 0; `CartItems.VaseId`
  and `OrderLines.VaseId` nullable). No MQTT or Keycloak change.
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M19: sales mode, "Add online bouquet"; order lines must
  tolerate a null `vaseId`), customer mobile app (Delivery-only badge, hide pickup when `pickupAvailable` is false, the
  new codes; cart/order models must accept a null `vaseId` and treat 409 on add-to-cart/checkout as "taken").
- **Deployment order**: API (migration applies on start) → portals. Mobile apps only see `vaseId: null` once an
  online-only vendor lists a bouquet, so ship the null-tolerant mobile builds before enabling the first online-only
  vendor.

## 2026-10-21 — EP-38 Docs portal access control (additive; docs hosting moves)

- **What changed**: new `POST /api/v1/docs/handoff` (any signed-in role), `POST /api/v1/docs/handoff/redeem` and
  `POST /api/v1/docs/roles` (docs portal client credentials only, policy `DocsPortalClient`) — `API-CONTRACTS.md` §24.
  New Keycloak client `flowershop-docs` with client roles and the `docs_roles` claim; realm roles `User`, `VendorAdmin`,
  `PlatformAdmin` become composites; new group `developers`. New app `FlowerShop.DocsPortal` (`fly.docs-portal.toml`)
  serves `docs.findmyflowers.pl` behind sign-in; GitHub Pages publishing stops once `DOCS_PORTAL_DEPLOY` is set.
  Config `Docs:*`. No database migration.
- **Affected repos**: `flower-shop-vendor-mobile-app` (EP-M18 Help), customer mobile app (Help follow-up); docs links
  everywhere now need a signed-in user.
- **Deployment order**: Keycloak client (`scripts/keycloak/setup-docs-client.sh`) → API → DocsPortal (Fly secret
  `Docs__Keycloak__ClientSecret`) → DNS cut-over → portals (Help) → retire the public docs repo
  (`docs/deployment/DOCS-PORTAL.md`).

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
