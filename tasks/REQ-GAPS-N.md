# Requirement Gaps — Option N (AI Florist: two wizard modes, stock count from photos, cost assessor)

> Companion to REQ-GAPS-H.md (AI Florist, EP-28), REQ-GAPS-K.md (Stock photos & AI appearance, EP-31),
> REQ-GAPS-L.md (portal wizard, EP-32) and REQ-GAPS-M.md (feedback loop, EP-33).
> Requirement refs point at the tasks in `tasks-ep34-design-visual-stock-start.json`,
> `tasks-ep35-design-from-bouquet-photos.json` and `tasks-ep36-ai-bouquet-cost-assessor.json`.
> Analysis date: 2026-10-20
> Status: REQ-VSS-01 … 10 (EP-34), REQ-INSP-01 … 11 (EP-35), REQ-ASSESS-01 … 10 (EP-36) planned
> (see `EPICS-N.md`).

## Context

The wizard's step 2 ("Check your stock", `Pages/Design/Session.cshtml`) is a plain form table:
Kind is a `<select>` with the words Flower / Greenery / Material, Colour is a free-text input, and
"Fill from my stock" (`OnPostFillStockAsync`) dumps every stock item into the table at once. EP-31
already stores what the UI needs to make this visual: a main photo per stock item (private,
`IDesignMediaStore`), an appearance palette with shares (`PaletteColourDto`, rendered by
`Pages/Stock/_Swatches.cshtml`) and a variety.

"Start from photos" (EP-28) means "photograph your **bench**": the recognizer counts the stock
laid out in front of the vendor, so the wizard mixes two jobs, **updating stock** and **designing
a bouquet**. The Stock page's AI (EP-31) only *describes* one item per photo (kind, name, variety,
palette) and creates one item from a photo. It never counts quantities or updates stock levels.

**Decision (2026-10-20): separate the flows.**
- **Stock updates live on the Stock page.** Bench counting moves out of the wizard into
  "Count stock from photos" on the Stock page. It writes reviewed `Counted` movements to the EP-30
  ledger (REQ-VSS-08).
- **The AI Florist wizard has exactly two modes:**
  1. **Start from my stock**: drag-and-drop stock items into a working area for the initial set
     (EP-34).
  2. **Reconstruct a bouquet from photos**: "reverse engineering" of a bouquet in 1–5 images, for
     example when a customer placed an order or request with an image. The wizard lists what is in
     it, maps it to the vendor's stock and shows what is missing (EP-35).

A closely related need is pricing a finished bouquet: "what is it made of and what does it cost me
at my stock prices?" (EP-36).

EP-35 and EP-36 share one new capability, **bouquet photo analysis**: detect the items in 1–5
photos of one bouquet, then map each detected item to the vendor's stock with a match score. Code
does the mapping and the arithmetic. The model only describes what it sees.

**Principles kept from EP-28 / EP-31 / EP-33**
1. Nothing the model says is trusted: stock links, quantities and prices are decided and
   validated in code; detected items are suggestions until the vendor confirms them.
2. Vendor data sent to the model (stock catalogue candidates, notes) is labelled JSON **data**,
   never part of the system prompt.
3. Strict tenant isolation: only the calling vendor's stock is ever a match candidate.
4. Every paid call goes through quota, $ cap, kill switch, pilot list and ledger-before-call.
5. Photos are private (`IDesignMediaStore`), re-encoded ≤ 1600 px JPEG with EXIF stripped, and
   follow the session retention rules.

---

## EP-34 — Wizard mode 1: start from my stock (visual, drag-and-drop) + stock count on the Stock page

### REQ-VSS-01 — Thumbnail per stock line — P0
Every row in step 2 shows a square thumbnail (48 px):
- row linked to a stock item (`stockItemId`) → the item's main photo (EP-31) through the portal's
  same-origin media proxy (`Pages/Design/Media`, extended to stock photos);
- unlinked row, or stock item without a photo → the kind pictogram (REQ-VSS-02) on a tinted tile.

The thumbnail id comes with the data already loaded (`mainPhotoId` on the stock list DTO and on
the inventory line DTO). There are no per-row API calls.
**Accept**: a 100-line inventory renders with one stock list call; thumbnails are `loading="lazy"`
and `Cache-Control: private`.

### REQ-VSS-02 — Kind as pictograms — P0
The Kind column shows three pictograms instead of words: **flower** (Flower), **leaf** (Greenery),
**ribbon/wrap** (Material). Shipped as local SVGs (`wwwroot/img/kinds/*.svg`), not a CDN.
- Editable rows use a three-button segmented control (radio group) with the pictograms; the
  posted value stays `Flower|Greenery|Material`, so the API is unchanged.
- Each pictogram has `aria-label`/visually-hidden text and a tooltip; selection is not conveyed by
  colour alone.

**Accept**: keyboard (arrow keys) changes the kind; works without JS (styled radio inputs).

### REQ-VSS-03 — Colours as swatches — P0
The Colour column shows colour **swatches like the stock appearance** (EP-31, `_Swatches`):
- stock-linked row with an appearance → the item's palette, widths proportional to share, tooltip
  "Peach 60 %";
- otherwise → swatches from the **colour words in the phrase**. Stock colours are descriptive
  ("Creamy white with light green", "Peachy pink with yellow highlights"), so a whole-phrase
  lookup would fail. A shared colour-term table splits the phrase into known terms. The first term
  is dominant (≈ 70 %) and the rest share the remainder. Modifiers (light/pale/deep/dark/creamy)
  adjust the base colour, and every hex goes through `StockLabels.SafeHex`. If no term is
  recognised, the swatch is neutral and hatched. The full phrase stays as a tooltip / secondary
  line.

Editing: a click on the swatch opens a small picker of the known colour names (chips with
swatches) plus free text; the posted `color` stays a string (≤ 40 chars).
**Accept**: the swatch updates live as the vendor types or picks; no hex value ever comes from the
model or the user without `SafeHex`.

### REQ-VSS-10 — Show the price that will be used; tell same-name rows apart — P1
Today every price cell shows "—" even when the stock item has a unit cost, although the page says
missing prices are filled from stock costs. An empty price input shows the fill-in value as its
placeholder ("4.00 · stock cost", "3.50 · remembered"), or "no price yet" with a link to the stock
item. Rows with the same name (e.g. three "Rose" lines) show the variety or the main colour term
next to the name ("Rose · peachy pink").
**Accept**: the inventory line DTO carries `fallbackUnitPrice` + `fallbackSource`; no extra calls.

### REQ-VSS-04 — Drag-and-drop starting set — P0
"Start from my stock" (hub) and "Fill from my stock" (step 2) open a **two-pane picker** instead of
dumping all stock into the table:
- **My stock** (left): active, non-archived items with quantity > 0. Each card shows the thumbnail,
  kind pictogram, swatches, name/variety, available quantity, unit cost and low-stock badge.
  Filters: kind pictograms, colour chips, text search. Sort: name, quantity, recently used.
- **Starting set** (right): a drop area ("Drag flowers here to start"). Dropping a card adds a line
  linked to that stock item with a default quantity of `min(available, 5)` for stems and 1 for
  items. The quantity is editable in place. Dropping the same item again increases the quantity.
- Non-drag alternatives (WCAG 2.5.7): an **Add** button on each card, keyboard (Enter adds), and
  multi-select + "Add selected".
- "Add all" keeps today's behaviour as an explicit action.
- **Use as my stock** writes the starting set as the session inventory (`PUT …/inventory` with
  `stockItemId` per line) and goes on to the table (REQ-VSS-01..03) for final checks.

**Accept**: works with mouse, touch (portal on tablets) and keyboard; without JS the picker
degrades to a checkbox list with quantity inputs; the starting set survives a reload (kept in the
session inventory draft, not local storage).

### REQ-VSS-05 — Quantity guard against stock — P1
A starting-set or table quantity above the stock item's available quantity shows a warning ("only
8 in stock") but is allowed. Recipes still use the confirmed inventory, and the vendor may hold
stock they haven't recorded. "Use stock" (EP-30 consume) keeps its all-or-nothing rule.
**Accept**: the warning uses the quantity read when the picker opened; a stale read is resolved by
the consume step, not by the wizard.

### REQ-VSS-06 — Design tokens and accessibility — P1
Pictograms, swatches and the drop area use the portal's theme variables (light/dark), 4.5:1
contrast for text, and visible focus. The drop area announces additions through an `aria-live`
region ("Added 5 × Peach rose").
**Accept**: axe check of the step-2 page has no serious violations.

### REQ-VSS-07 — Vendor app parity — P2
Vendor app (EP-M14): thumbnail, pictograms and swatches in the stock check list; the starting set
via long-press drag **and** tap-to-add.
**Accept**: same API calls as the portal; no new endpoints beyond REQ-VSS-01's DTO fields.

### REQ-VSS-08 — Count stock from photos (Stock page) — P1
Bench recognition moves out of the wizard to the Stock page: **Count stock from photos** (1–5
photos of the bench or cold room).
- The existing EP-28 recognizer (`inventory` prompt, with REQ-STKPH-11 catalogue linking) runs as
  a stock job and returns estimated lines linked to stock items where possible.
- Review list: thumbnail, pictogram, swatches (EP-34 components), **current quantity → counted
  quantity** with the delta, confidence, "please check" below 0.6. Unlinked lines offer "Link to
  an item" or "Create item" (prefilled; the photo becomes a pending photo as in REQ-STKPH-05).
- **Apply counts** writes one `Counted` movement per confirmed line to the EP-30 ledger in one
  all-or-nothing batch with an `Idempotency-Key`. Lines the vendor didn't confirm are not written.

Quota: the EP-31 stock AI monthly quota (not the AI Florist recognition quota), under the same kill
switch, pilot list and $ cap.
**Accept**: nothing changes stock until **Apply counts**; replaying the batch key is a no-op;
another vendor's item can't be linked.

### REQ-VSS-09 — Wizard modes only — P0
The wizard hub offers two starts: **Start from my stock** (this epic) and **Reconstruct a bouquet
from photos** (EP-35). "Photograph your stock" and "Enter stock by hand instead" leave the wizard
entry. The working area table (REQ-VSS-01..03) stays for adjusting quantities and adding a line
by hand. `inputMode: "Photos"` is no longer offered for new sessions once the vendor app has
migrated (EP-M14). Existing `Photos` sessions stay readable and usable.
The step bar follows the mode: mode 1 shows **1. Your stock → 2. Choose design → 3. Recipes**;
mode 2 shows **1. Bouquet photos → 2. Match to stock → 3. Choose design → 4. Recipes**. Today a
"from stock" session still shows "1. Photos".
**Accept**: `GET usage` lists `inputModes: ["Stock", "BouquetPhotos"]` (plus `Photos` until the
app migration flag is off); the `recognize` endpoint keeps working for old sessions.

---

## EP-35 — Wizard mode 2: reconstruct a bouquet from photos

### REQ-INSP-01 — New input mode "BouquetPhotos" — P0
A design session can start with `inputMode: "BouquetPhotos"`. In the portal hub it is
**Reconstruct a bouquet from photos**: "Add 1–5 photos of the bouquet you want to make, e.g. the
picture a customer sent". This is a reverse-engineering assistant, not a stock update. It never
changes stock levels. Bench counting is on the Stock page (REQ-VSS-08), and the wizard has only
the two modes of REQ-VSS-09.
**Accept**: existing `Photos` sessions keep working unchanged; `GET usage` reports the new mode in
`inputModes`.

### REQ-INSP-02 — Detect the bouquet's items — P0
`POST /design-sessions/{id}/analyze` (required `Idempotency-Key`, counts as one recognition)
queues a `BouquetAnalysis` job (shared with EP-36). The analyzer (`bouquet-analysis-v1` prompt)
returns per detected item: kind, common name, colour(s), variety guess (optional), estimated
quantity, visibility note (`partly-hidden`), and a model confidence 0–1. It returns one list for
the bouquet across all photos, not one per photo. Photos of different bouquets → the analyzer
says so (`mixed-bouquets`) and the job asks the vendor to keep one bouquet.
**Accept**: ≤ 40 detected items; names ≤ 80 chars; quantity 1–200; invalid output fails the job
with a safe message; the free deterministic provider (`Palette`-style) exists for dev/tests.

### REQ-INSP-03 — Map detected items to stock with a match score — P0
For each detected item, code ranks the vendor's stock items (active + archived-but-in-stock
excluded) and keeps the best match plus up to 2 alternatives with a **match score 0–100 %**:
- name/synonym similarity (normalised catalogue key `name|colour[|variety]`, `InventoryLine.NormalizeKey`),
- kind must agree (hard filter),
- colour: palette distance between the detected colours and the stock appearance palette,
- variety bonus when both are known,
- the model's confidence as a multiplier.

Bands: **High ≥ 80 %**, **Medium 50–79 %**, **Low < 50 %** (no automatic link below 50 %). The
score is labelled "match", not "probability", and the factors are shown on hover ("name ✓, colour
close, variety unknown").
Optional second pass (config `BouquetDesign:Analysis:ModelAssistedMatching`): send the top 10
code-ranked candidates per item to the model as JSON data and let it pick one by id. The pick is
accepted only if it is in the candidate list.
**Accept**: another vendor's stock is never a candidate; matching runs in code in < 200 ms for
500 stock items; deterministic for the same input.

### REQ-INSP-04 — Review screen: matched, short, missing — P0
Step 2 for this mode shows the inspiration photos and three groups:
1. **Matched**: detected item → stock card (thumbnail, pictogram, swatches from EP-34), score
   badge, needed vs available quantity. The vendor can switch to an alternative, search any stock
   item, or mark "not in my stock".
2. **Short**: matched but available < needed, showing the shortfall.
3. **Missing** (bottom of the page): detected items with no match ≥ 50 % or marked "not in my
   stock". Shows kind, name, colour swatch, needed quantity and a reference crop if available.

**Accept**: every change is recomputed in code without a model call; the vendor's link choices are
stored as `RecognitionCorrection` rows when EP-33 is present (REQ-FBK-10).

### REQ-INSP-05 — Missing items → procurement hook — P1
The missing (and short) list has **Order missing items** with a "coming soon" state and two
actions that work today: **Copy list** and **Download CSV** (name, colour, variety, quantity,
unit). Server side, an `IFlowerSupplierCatalog` port (no-op implementation
`NullFlowerSupplierCatalog`) defines `SearchAsync(items) → offers` and `CreateOrderAsync`, so a
future supplier-integration epic only adds adapters.
**Accept**: the DTO has `procurement: { available: false, providers: [] }`; no external call is
made in this epic.

### REQ-INSP-06 — From inspiration to recipes — P0
**Use matched items** writes the inventory (matched lines, quantities = detected quantity capped
at the vendor's edit, linked by `stockItemId`) and confirms it. Composition (step 3) receives the
detected bouquet as a `target` data field (lines + palette). The composer is asked for recipes
that stay as close to the target as the stock allows, plus one "best substitute" variant when
items are missing. Recipes still pass `RecipeValidator` and the cost ceiling.
**Accept**: a recipe never uses a missing item; the "why" line (EP-33 REQ-FBK-14, if present)
names the substitutions.

### REQ-INSP-07 — Inspiration photo as preview reference — P2
Preview rendering may use the first inspiration photo as a style reference (composition, wrap,
colours) next to stock main photos.
**Accept**: off by default (`BouquetDesign:Analysis:UseInspirationAsPreviewReference`); the
vendor sees a note that reference photos are sent to the image provider.

### REQ-INSP-08 — Quota, cost and limits — P0
An analysis counts as one recognition against the monthly quota and is reserved against the $ cap
before the call. Max 5 photos, 8 MB raw each. One active analysis per session; 429
`too-many-active` otherwise.
**Accept**: replaying the same `Idempotency-Key` returns the same job; the kill switch stops new
analyses.

### REQ-INSP-09 — API contract — P0
§21 gets: input mode `BouquetPhotos`, `POST …/analyze`, `analysis` on `DesignSessionDto`
(`detectedItems[] { id, kind, name, colors[], variety, quantity, confidence, visibility, match {
stockItemId, score, band, factors[] }, alternatives[], status: Matched|Short|Missing }`,
`missing[]`, `procurement`), `PUT …/analysis/items/{itemId}` (choose match / not in stock /
quantity), and session states `Analyzing → AnalysisReady`.
**Accept**: `contracts-check` run; vendor app contract copy updated.

### REQ-INSP-10 — Vendor app parity — P2
Vendor app (EP-M15): camera/gallery 1–5 photos, review groups, missing list share sheet.

### REQ-INSP-11 — Reconstruct from a customer's request or order — P1
The main use case: a customer sent a picture (EP-37 bouquet request, or a custom order). On a
request the vendor can see (and on their own accepted custom order), **Reconstruct with AI
Florist** creates a `BouquetPhotos` session pre-loaded with the request's images (server-side
copy into the private design media, no re-upload) and linked to the request (`sourceRequestId`).
The review screen then also shows:
- an **estimated cost** of the reconstruction from stock prices (EP-36 calculator), with missing
  items marked "price unknown";
- a **draft offer note**: for each missing item, the best in-stock alternative ("no peonies — I'd
  use garden roses"), which the vendor can copy into the EP-37 offer.

**Accept**: only images the vendor is allowed to see (REQ-BRQ-06/14) can be copied; the link to
the request never exposes customer PII in the session; deleting the session doesn't touch the
request.

---

## EP-36 — AI Bouquet Items & Cost Assessor

### REQ-ASSESS-01 — Cost assessment entry point — P0
A new vendor feature **Cost assessor**, separate from design sessions: Vendor Portal
`Pages/Design/Assess/*` (hub tile "What does this bouquet cost me?") and a vendor app screen.
The vendor adds 1–5 photos of one **finished** bouquet and an optional note (≤ 200 chars, e.g.
"hidden: 3 eucalyptus inside").
**Accept**: same photo rules as design sessions (private, re-encoded, EXIF stripped).

### REQ-ASSESS-02 — CostAssessment aggregate — P0
Vendor-scoped `CostAssessment`: photos, state (`Draft → Analyzing → Ready | Failed`), detected
items (same shape as REQ-INSP-02), confirmed lines (stock link, quantity, unit cost + its source),
optional selling price, timestamps, `Provider`/`Model`/`PromptVersion` of the analysis job.
Photos follow the design-session retention. Lines and totals are kept until the vendor deletes
the assessment.
**Accept**: another vendor's assessment → 404; delete purges photos.

### REQ-ASSESS-03 — Analysis reuses the bouquet-analysis engine — P0
`POST /vendor/cost-assessments/{id}/analyze` (required `Idempotency-Key`) runs the same
`BouquetAnalysis` job and stock mapping as EP-35 (REQ-INSP-02/03). It uses the assessor prompt
variant (`bouquet-analysis-v1`, purpose `Assessment`), which asks for countable stems and hidden
fillers ("likely 2–3 hidden greenery stems") and returns a quantity range (min/likely/max) per
item.
**Accept**: one engine, one prompt registry entry with a purpose field; no duplicated matching code.

### REQ-ASSESS-04 — Cost from stock prices — P0
Per confirmed line, the unit cost comes from (first hit wins):
1. the linked `StockItem` unit cost (EP-30);
2. the vendor's remembered price for the catalogue key (`DesignCostProfile.prices`);
3. unknown → the line is marked "price missing" and the total is incomplete.

Totals (shared `CostBreakdown` code from EP-28): flowers subtotal, materials subtotal, labour
(cost profile minutes × hourly rate), overhead %, **known cost** and **total** (null while any
price is missing), plus a **range** from the min/max quantities ("118–141 PLN").
**Accept**: the arithmetic is in code and unit-tested; the model never returns prices.

### REQ-ASSESS-05 — Edit and recalculate — P0
The vendor can change any line's match (same picker as REQ-INSP-04), quantity, or unit cost
override (one-off, or "remember this price"), add a missed item from stock, or remove a false
detection. Totals recalculate instantly without a model call.
**Accept**: `PUT /vendor/cost-assessments/{id}/lines` with `expectedVersion` (409 on conflict).

### REQ-ASSESS-06 — Selling price and margin — P1
Optional selling price → margin amount and gross margin % (same formula as `RecipeCostDto`).
**Suggested price** at a target margin from the cost profile (new field `targetMarginPercent`,
default 60 %), rounded to whole PLN.
**Accept**: no margin and no suggestion while the cost is incomplete; the reason is shown.

### REQ-ASSESS-07 — Act on the result — P2
- **Deduct from stock**: `POST /vendor/stock/consume` with reference `cost-assessment:{id}`
  (all-or-nothing, idempotent), for bouquets the vendor has already made.
- **Open in AI Florist**: start a design session (`BouquetPhotos` mode) pre-filled with the
  assessment's lines.
- **Create listing**: prefill the bouquet create form with the first photo and the suggested price.
- **Print / PDF** summary.

**Accept**: each action is explicit; nothing happens automatically.

### REQ-ASSESS-08 — History — P1
List of past assessments (thumbnail, date, total, margin), newest first, paged; delete.

### REQ-ASSESS-09 — Quota and kill switch — P0
Separate monthly quota `BouquetDesign:Assessment:MonthlyLimit` (default 30). It sits under the
assistant's kill switch, pilot list and global $ cap, with ledger-before-call. It shows in
`DesignUsageDto` (`assessmentsUsed/Limit/Remaining`).
**Accept**: 429 `quota-exhausted` with `Retry-After` at the reset date.

### REQ-ASSESS-10 — API contract and vendor app — P1
New §21.2 "Cost assessor" in `API-CONTRACTS.md` (routes under `/api/v1/vendor/cost-assessments`).
Vendor app EP-M16.

---

## Cross-cutting

- **Migrations**: `DesignSession.InputMode` gains `BouquetPhotos`; `DesignAnalysis` (detected
  items + matches, JSON snapshot on the session); `CostAssessments` + `CostAssessmentLines`;
  `DesignCostProfile.TargetMarginPercent`; `StockItem` list DTO `mainPhotoId` (no schema change).
- **Prompt registry**: `bouquet-analysis-v1` (purposes `Inspiration`, `Assessment`) is added to the
  EP-33 registry if it has landed, otherwise as a `const` with the same id so it moves cleanly.
- **Admin quality dashboard** (EP-33 REQ-FBK-12): analysis jobs per purpose, match-band
  distribution, vendor-changed-match rate (the main accuracy signal).
- **Privacy**: inspiration photos may show third-party work or people. They stay private,
  are never shown to customers, and are deleted with the session.

## Explicitly out of scope
- Real supplier integrations and ordering (only the port + null adapter; future epic).
- Pricing advice from market data or competitors.
- Recognising a bouquet from a URL or social-media link.
- Automatic stock deduction or listing creation.
