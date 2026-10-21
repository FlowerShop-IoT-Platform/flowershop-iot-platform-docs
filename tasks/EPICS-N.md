# Epics, Stories & Tasks — Option N (AI Florist: two wizard modes, stock count from photos, cost assessor)

> Companion to EPICS-H.md (EP-28 AI Florist), EPICS-K.md (EP-31 Stock photos & AI appearance),
> EPICS-L.md (EP-32 portal wizard) and EPICS-M.md (EP-33 feedback loop).
> Epics are numbered EP-34 … EP-36 to continue the sequence.
> Task trackers (ground truth): `docs/tasks/tasks-ep34-design-visual-stock-start.json`,
> `docs/tasks/tasks-ep35-design-from-bouquet-photos.json`, `docs/tasks/tasks-ep36-ai-bouquet-cost-assessor.json`.
> Requirements: `docs/tasks/REQ-GAPS-N.md` (REQ-VSS-01 … 10, REQ-INSP-01 … 11, REQ-ASSESS-01 … 10).
>
> **Flow split (decided 2026-10-20)**: stock updates live on the Stock page, including the bench
> photo counting moved out of the wizard. The AI Florist wizard has exactly two modes:
> (1) start from my stock by drag-and-drop, (2) reconstruct a bouquet from photos.
> Mobile counterparts: vendor app EP-M14, EP-M15, EP-M16 (follow-ups).
> Analysis date: 2026-10-20

---

## EP-34: AI Florist mode 1 — start from my stock (visual, drag-and-drop) + stock count from photos

**Priority**: P1-high
**Dependencies**: EP-30 (stock), EP-31 (stock photos, appearance palette), EP-32 (portal wizard)
**Requirement refs**: REQ-VSS-01 through REQ-VSS-10
**State**: todo — trackers done (T-34-001).

> Step 2 stops being a spreadsheet. Each line shows the flower's photo, a pictogram for its kind and
> the colours as swatches. The vendor builds a starting set by dragging stock cards into a drop area.

```
 My stock  [🌸][🍃][🎀]  colour ●●●  search…        Starting set
 ┌──────────────────────────────┐                  ┌───────────────────────────┐
 │ [img] 🌸 Peach rose  ▮▮▯ 24  │  ── drag/Add ──▶ │ [img] 🌸 Peach rose  × 5  │
 │ [img] 🍃 Eucalyptus  ▮   40  │                  │ [img] 🍃 Eucalyptus  × 3  │
 │ [img] 🎀 Kraft paper ▮   12  │                  │   Drag flowers here…      │
 └──────────────────────────────┘                  └──────── Use as my stock ──┘
```

### Design decisions
- **UI only.** Posted values stay `Kind=Flower|Greenery|Material` and `color` = string, so
  `DesignWizard` validation and the §21 inventory PUT don't change. The only API change is
  `mainPhotoId` on the stock and inventory-line DTOs, so there are no per-row calls.
- **Swatches have one source.** Stock-linked rows reuse `Stock/_Swatches` (palette with shares).
  Other rows parse the colour words of the descriptive phrase ("Creamy white with light green")
  through a shared term table and `StockLabels.SafeHex`.
- **No hidden prices.** Empty price cells show the stock-cost or remembered price that will be
  used. Same-name rows (three "Rose" lines) show variety or the main colour term.
- **The step bar follows the mode.** Mode 1 has 3 steps, mode 2 has 4. A "from stock" session no
  longer shows "1. Photos".
- **Drag is optional.** Add button, keyboard and multi-select work everywhere (WCAG 2.5.7). Without
  JS the picker is a checkbox list.
- **Two wizard modes only.** The hub offers "Start from my stock" and "Reconstruct a bouquet from
  photos". Bench recognition leaves the wizard. The `Photos` input mode stays readable for old
  sessions and is no longer offered once the vendor app has migrated.
- **Counting is a stock operation.** "Count stock from photos" on the Stock page reuses the EP-28
  recognizer. It shows current → counted per item and writes `Counted` movements only on **Apply
  counts** (one idempotent batch). It is billed to the EP-31 stock AI quota.
- **The starting set is the inventory draft.** It is saved as an unconfirmed inventory, so a reload
  keeps it. A quantity above stock warns but doesn't block. "Use stock" still enforces stock levels.

### Stories & tasks
See `tasks-ep34-design-visual-stock-start.json` (S-34-01 … S-34-06; T-34-001 … T-34-019).
1. **S-34-01 — Trackers** — T-34-001.
2. **S-34-02 — Data for a visual stock table** — T-34-002 mainPhotoId · T-34-003 stock photo proxy.
3. **S-34-03 — Visual stock table** — T-34-004 kind pictograms · T-34-005 thumbnails · T-34-006 swatches + picker · T-34-019 price hint + same-name labels.
4. **S-34-04 — Drag-and-drop starting set** — T-34-007 picker · T-34-008 drag/non-drag · T-34-009 draft persistence · T-34-010 quantity guard.
5. **S-34-05 — Accessibility, tests and parity** — T-34-011 tests · T-34-012 manual · T-34-013 EP-M14.
6. **S-34-06 — Separate the flows** — T-34-014 wizard hub with two modes · T-34-015 stock count job + API · T-34-016 Stock page "Count stock from photos" · T-34-017 retire `Photos` mode for new sessions · T-34-018 tests.

---

## EP-35: AI Florist mode 2 — reconstruct a bouquet from photos

**Priority**: P1-high
**Dependencies**: EP-28, EP-31, EP-32, EP-34 (stock cards); optional EP-33 hooks
**Requirement refs**: REQ-INSP-01 through REQ-INSP-11
**State**: todo — trackers done (T-35-001).

> A reverse-engineering assistant: the vendor gives the wizard 1–5 photos of a bouquet (typically
> the picture a customer sent with a request or order). The wizard lists what is in it, matches each
> item to the vendor's stock with a score, and puts what's missing at the bottom, ready to order
> once supplier integrations exist. It never changes stock levels.

```
 1–5 photos ──▶ BouquetAnalysis job (bouquet-analysis-v1) ──▶ detected items
                                                                   │
                       StockMatcher (code): kind · name/synonym · palette distance · variety · confidence
                                                                   ▼
        Matched 92 % ●High   ·   Short (need 9, have 6)   ·   Missing ──▶ Copy / CSV / [Order — soon]
                                                                   │
                 Use matched items ──▶ confirmed inventory + target ──▶ recipes (close + best substitute)
```

### Design decisions
- **The model describes, code matches.** Detection is the only model step. Code computes the match
  score so it is deterministic and explainable (factor tooltip). An optional model-assisted pick
  may only choose from a code-ranked candidate list.
- **"Match", not "probability".** Bands are High ≥ 80 %, Medium 50–79 % and Low < 50 %. Nothing
  below 50 % is linked automatically.
- **One engine, two purposes.** `IBouquetAnalyzer`, the `BouquetAnalysis` job kind and
  `StockMatcher` are purpose-neutral (`Inspiration` | `Assessment`), so EP-36 reuses them.
- **Not a stock update.** Bench counting moved to the Stock page (EP-34 REQ-VSS-08). This mode
  only reads stock.
- **Built for customer pictures.** "Reconstruct with AI Florist" on an EP-37 request or custom
  order pre-loads its images. The review also shows the estimated cost (EP-36 calculator) and a
  draft offer note with in-stock alternatives for missing items.
- **Procurement is a port.** `IFlowerSupplierCatalog` + `NullFlowerSupplierCatalog` now, real
  supplier adapters in a future epic. Copy/CSV makes the missing list useful today.

### Stories & tasks
See `tasks-ep35-design-from-bouquet-photos.json` (S-35-01 … S-35-06; T-35-001 … T-35-017).
1. **S-35-01 — Trackers** — T-35-001.
2. **S-35-02 — Bouquet analysis engine** — T-35-002 port + model · T-35-003 providers · T-35-004 job kind + quota.
3. **S-35-03 — Stock matching with scores** — T-35-005 StockMatcher · T-35-006 model-assisted pick.
4. **S-35-04 — BouquetPhotos session mode and API** — T-35-007 mode + snapshot · T-35-008 endpoints · T-35-009 use matched items + target.
5. **S-35-05 — Portal review screen and missing items** — T-35-010 hub entry · T-35-011 review groups · T-35-012 export + procurement port · T-35-013 preview reference · T-35-017 reconstruct from a request/order.
6. **S-35-06 — Contract, tests and rollout** — T-35-014 contract · T-35-015 tests · T-35-016 guide/manual/EP-M15.

---

## EP-36: AI Bouquet Items & Cost Assessor

**Priority**: P1-high
**Dependencies**: EP-35 S-35-02/S-35-03 (shared engine), EP-28 (cost breakdown), EP-30 (stock costs, consume)
**Requirement refs**: REQ-ASSESS-01 through REQ-ASSESS-10
**State**: todo — trackers done (T-36-001).

> Photograph a finished bouquet and see what it's made of and what it costs at your stock prices,
> as a range, with margin and a suggested price.

```
 photos + note ──▶ BouquetAnalysis (purpose Assessment: hidden fillers, qty min/likely/max)
                      ──▶ StockMatcher ──▶ lines (editable)
 unit cost: StockItem cost → remembered price → override → unknown
 total = flowers + materials + labour + overhead   (range 118–141 PLN; null while a price is missing)
 margin @ selling price · suggested price @ target margin
 actions: Deduct from stock · Open in AI Florist · Prefill listing · Print
```

### Design decisions
- **Separate aggregate.** `CostAssessment` has its own lifecycle, quota
  (`BouquetDesign:Assessment:MonthlyLimit`) and history. It sits under the same kill switch, pilot
  list and $ cap.
- **Prices never come from the model.** Cost precedence is stock unit cost, then remembered price,
  then the vendor's override. Missing prices make the total incomplete, as in EP-28.
- **Ranges are honest.** Hidden stems make counts uncertain, so the assessor shows min/likely/max
  quantities and a cost range.
- **Nothing automatic.** Stock deduction and listing prefill are explicit actions. Deduction
  reuses the EP-30 consume rules (`cost-assessment:{id}` reference).

### Stories & tasks
See `tasks-ep36-ai-bouquet-cost-assessor.json` (S-36-01 … S-36-05; T-36-001 … T-36-012).
1. **S-36-01 — Trackers** — T-36-001.
2. **S-36-02 — CostAssessment domain and API** — T-36-002 aggregate · T-36-003 endpoints · T-36-004 shared engine · T-36-005 quota.
3. **S-36-03 — Cost calculation** — T-36-006 cost resolution + totals · T-36-007 margin + suggested price.
4. **S-36-04 — Vendor Portal cost assessor** — T-36-008 pages · T-36-009 actions.
5. **S-36-05 — Contract, tests and rollout** — T-36-010 contract · T-36-011 tests · T-36-012 guide/manual/EP-M16.

---

**Suggested order across Option N**: EP-34 S-34-02..04 (stock cards, mode 1) → EP-35 S-35-02/03
(engine + matcher) → EP-36 and the rest of EP-35 in parallel → EP-34 S-34-06 (flow split; `Photos`
mode is retired only after the vendor app migration EP-M14) → T-35-017 once EP-37 requests exist.
