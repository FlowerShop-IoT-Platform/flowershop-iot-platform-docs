# Requirement Gaps — Option R (Recognition confidence from human feedback)

> Companion to REQ-GAPS-H.md (EP-28 recognition), REQ-GAPS-M.md (EP-33 feedback loop, known corrections)
> and REQ-GAPS-N.md (EP-34 stock count, EP-35 bouquet analysis, EP-36 cost assessor).
> Requirement refs point at the tasks in `tasks-ep40-recognition-confidence-feedback.json`.
> Analysis date: 2026-10-23
> Status: proposed (EP-40, see `EPICS-R.md`). Open decisions are in section H.

## Context

Every recognised line carries a "confidence", shown as "90 % sure" in the Vendor Portal
(`_AnalysisReview.cshtml`, the stock count review and the Photos-mode inventory). Today that number is
**the model's own guess about itself**. OpenAI returns it as a field in the structured output
(`OpenAiBouquetAnalyzer`, schema `confidence: number`). Vision models are badly calibrated at this:
almost every line comes back at 0.90–0.95, whether it is right or not. A real bouquet analysis showed
spray carnation 90 %, pincushion protea 95 %, astrantia 90 %, astilbe 90 % and eucalyptus 90 %. The
"astilbe" was probably partly amaranthus, and the "eucalyptus" was seeded eucalyptus. The number told
the florist nothing about which line to check.

The goal is a **confidence score that says how often this kind of answer turned out to be right
when florists checked it**. It is computed from human feedback, in code, never by the model.

### What the code does today

| Area | Behaviour | Where |
|---|---|---|
| Raw confidence | The model's 0–1 self-rating, clamped and stored on `DetectedItem.Confidence` / `InventoryLine.Confidence` / `StockCountLine.Confidence`. | `BouquetAnalyzers.cs`, `HostedAiProviders.cs`, `DesignAnalysis.Clean` |
| Uses of it | `StockMatcher.Score` multiplies the match by `0.7 + 0.3 × confidence` (default 0.75). The stock count flags `NeedsCheck` below 0.6 (`StockCount.CheckBelowConfidence`). | `StockMatcher.cs`, `StockDtos.cs` |
| Feedback: Photos mode | When the florist confirms the inventory, `TakeRecognitionDiff` stores Renamed / Recoloured / Recounted / Removed / Added rows (`RecognitionCorrection`). **Lines confirmed unchanged are not stored**, so there is no denominator. | `DesignSessionCommands.cs:296`, `RecognitionCorrection.cs` |
| Feedback: bouquet analysis | A row is stored **only** when the florist picks a different stock item. "Not in my stock" and a changed quantity leave no trace. The florist **cannot say "this is not astrantia, it's scabiosa"** on a line with no stock match, and that is exactly the Missing case. | `EditDesignAnalysisItemCommandHandler` |
| Feedback: stock count | Reviewed `Counted` movements are written, but recognised-versus-confirmed is not compared. | `StockCountCommands.cs` |
| Learning | `RecognitionCorrections.Build` turns renames seen ≥ 2 times into hints for the same vendor's next recognition (EP-33), with no cross-vendor learning. | `RecognitionCorrection.cs`, `DesignJobProcessor.cs:161` |
| Quality dashboard | `CorrectedLineRate` per provider/model/prompt version. | `AiFloristQualityQuery.cs` |

So there is a correction log but **no verdict log**: we know some answers were wrong, but not how many
were checked and found right. A confidence score needs both.

### Decision: platform counts + shop adjustment (decided 2026-10-23)

One shop rarely checks enough lines of one flower type to give a stable rate. Ten astrantia lines a
month gives an error bar of about ±25 points. EP-33 ruled out cross-vendor learning. This epic
relaxes that rule **for anonymous counts only**:

- **Pooled**: per (recognizer provider, model, prompt version, task, flower type, raw-confidence
  bucket), the number of lines checked and the number found right. No vendor id, names typed by
  florists, photos, notes, prices or stock data enter the pool. A cell is used only when at least
  `MinVendorsPerCell` (default 3) different vendors contributed to it.
- **Per shop**: the same counts for the vendor's own verdicts, used to adjust the pooled rate toward
  what this shop sees (its lighting, suppliers and varieties).
- Known corrections (EP-33 hints) **stay vendor-scoped**. "In this shop, X is usually Y" is not shared.

```
 line recognised ──▶ raw model confidence (0–1)  ─┐
                                                  ▼
          platform cell (model, prompt, task, flower type, raw bucket)  ← anonymous counts, ≥ 3 vendors
                                                  ▼  shrink toward
          shop cell (vendor, model, task, flower type)                  ← this shop's verdicts
                                                  ▼
          calibrated score 0–1 + band (High/Medium/Low) + basis + number of checks
                                                  ▲
 florist verdict on the line: ✓ right · ✗ wrong (it's actually …) · ? not sure · count right / actual count
```

---

## A. Capturing feedback

### REQ-CNF-01 — Recognition verdict ledger — P0
A new vendor-scoped table `RecognitionVerdicts`, one row per checked line:
- **Source**: `BouquetAnalysis` | `InventoryRecognition` (Photos mode) | `StockCount`.
- **Ids**: session or count id, line id, job id.
- **Stamp**: provider, model, prompt version (from the job, as `RecognitionCorrection` does).
- **Prediction**: `kind`, `typeKey` (the canonical flower type, REQ-CNF-04), detected name, colours,
  variety, raw confidence and quantity.
- **Verdict**: identity `Correct | Wrong | Unsure`; `actualName/actualColor/actualVariety` when
  wrong; `countCorrect` and `actualQuantity`.
- **Weight**: `Explicit` | `Implicit` (REQ-CNF-03); also `createdAt` and `updatedAt`.

One row per (source, line). Changing a verdict updates the row, never adds a second one. The table
complements `RecognitionCorrection`: corrections stay the hint source, and verdicts are the evidence
for scores.
**Accept**: domain tests for creation, update and invariants (wrong ⇒ an actual name or "not in the
bouquet"; count verdict optional).

### REQ-CNF-02 — Explicit verdicts in the review — P0
Each recognised line in the three reviews gets a compact verdict control next to "Change":
**✓ Right · ✗ Wrong · ?**. Choosing ✗ opens "What is it?". It suggests names from the flower
vocabulary (`StockMatcher` synonyms, the vendor's stock names and the EP-41 flower atlas once it exists), and
offers a **"Not in the bouquet"** option (false positive). The count is confirmed by keeping or editing
the number, as today.
The identity verdict is **separate from stock**. "Not in my stock" means the florist doesn't have it.
It says nothing about whether the AI recognised it correctly, and the UI must not mix the two. In the
analysis example all five lines were Missing, yet the recognition was mostly right.
A florist can also add a line the AI missed ("Added", a false negative).
API: `PUT /vendor/design-sessions/{id}/analysis/items/{itemId}/verdict`,
`PUT /vendor/design-sessions/{id}/inventory/items/{itemId}/verdict` and
`PUT /vendor/stock/counts/{id}/lines/{lineId}/verdict`, each with body
`{ identity: "correct|wrong|unsure", actualName?, actualColor?, actualVariety?, notInBouquet?, actualQuantity? }`.
They are idempotent and tenant-scoped, and make no model call. A wrong identity with an actual name
also writes the existing EP-33 `Renamed` correction, so hints keep working.
**Accept**: handler tests; another vendor's session gives 404; repeating a verdict leaves one row.

### REQ-CNF-03 — Implicit verdicts, weighted down — P1
When the florist confirms (Photos mode `confirmed: true`, analysis "Use matched items" or Cost Assessor
"Use", stock count "Apply"), every line **without** an explicit verdict becomes an `Implicit` verdict:
- unchanged name, linked to a High match or kept as-is → `Correct`;
- renamed → `Wrong` with the new name;
- removed → `Wrong` + not in bouquet;
- the count is compared with the confirmed quantity.

Implicit rows count with `Confidence:ImplicitWeight` (default 0.25). People often confirm without
looking, so silent acceptance is weak evidence. Lines left on the screen that the florist did not use
("Missing" and never touched) produce **no** implicit verdict.
**Accept**: the confirm handlers write implicit rows only for untouched lines; an explicit verdict is
never overwritten by an implicit one.

### REQ-CNF-04 — Canonical flower type key — P0
Make the `StockMatcher` name canonicaliser public as `FlowerTypeKey.For(kind, name)`. It handles
synonyms, plurals and leading colour words ("pale pink spray carnation" → `flower:carnation`).
Verdicts and scores are keyed by it, so "Dianthus", "Carnations" and "Spray carnation" share evidence.
Spray and standard forms share the type key, and the variety stays a separate field. Extend the
synonym table with the types in recent analyses (astrantia/masterwort, protea/leucospermum
"pincushion", seeded eucalyptus as a eucalyptus variety, amaranthus).
**Accept**: unit tests for each mapping; `StockMatcher` behaviour unchanged (same tests pass).

---

## B. Computing the score

### REQ-CNF-05 — Confidence model — P0
A pure `RecognitionConfidence` service in Application computes, for one predicted line, an
**identity score** and a **count score**. Each is a Beta-Binomial estimate with hierarchical
shrinkage:

1. **Model prior** `b`: the pooled accuracy of this (provider, model, prompt version, task) in the
   line's raw-confidence bucket (5 buckets: <0.5, 0.5–0.7, 0.7–0.85, 0.85–0.95, ≥0.95). Histogram
   binning calibrates the model's self-rating. With no data, `b` = the raw confidence × 0.8.
2. **Type rate** `g = (correct_t + m·b) / (n_t + m)` from the pooled type cell (`m` =
   `Confidence:PriorStrength`, default 10).
3. **Shop rate** `p = (correct_v + k·g) / (n_v + k)` from the vendor's own cell (`k` =
   `Confidence:ShopStrength`, default 10).

The counts are weighted: an explicit verdict counts 1, an implicit one counts `ImplicitWeight`.
Verdicts decay with a half-life `Confidence:HalfLifeDays` (default 120), so the score follows model
and supplier changes. `Unsure` verdicts are excluded.
"Correct" for **count** means |actual − predicted| ≤ max(1, 15 % of actual).
The output is `{ score (0–1), band (High ≥ 0.85, Medium 0.6–0.85, Low < 0.6), basis ("shop" | "platform" | "model" | "none"),
checks (weighted n behind it), raw }`. The basis is the deepest level with ≥ `MinChecks` (default 5)
weighted verdicts. `none` means there is no feedback at all, and clients must show "not enough
checks yet" rather than a number.
**Accept**: table-driven unit tests: cold start, shop-only data, platform-only data, a shop that
disagrees with the platform, decay, implicit weight and buckets.

### REQ-CNF-06 — Aggregates and refresh — P0
Two count tables are kept up to date by a background service (`RecognitionConfidenceAggregator`,
hourly, plus an incremental update on each verdict write for the shop cell):
- `RecognitionConfidenceCells` (platform, no vendor id): the keys above plus `vendors` (distinct
  contributors), weighted `n`, weighted `correct` and `asOf`.
- `RecognitionConfidenceShopCells` (vendor-scoped): (vendor, provider, model, task, typeKey) with the
  same counts.

A platform cell with `vendors < MinVendorsPerCell` (default 3) is **not used**. Scores read from an
in-process cache of the cells, refreshed with the aggregator, and are never computed per request
from raw verdicts.
**Accept**: aggregator test over seeded verdicts; a cell with 2 vendors is ignored; the shop cell
updates within the request.

### REQ-CNF-07 — Abuse and noise limits — P1
- A single vendor contributes at most `Confidence:MaxPerVendorPerCell` (default 30 weighted
  verdicts per 30 days) to a platform cell. One busy or careless shop can't dominate the shared
  numbers.
- Verdicts from sessions that used the free `Palette` analyzer (warning `colours-only`) are
  **excluded**, because their names are placeholders.
- Verdicts given within 2 s of the review opening for ≥ 5 lines at once are stored as `Implicit`.
  "Mark all right" clicking is weak evidence.

**Accept**: unit tests for each rule.

### REQ-CNF-08 — New model or prompt versions — P1
A new (model, prompt version) has no cells. Its score falls back to the bucket prior of the version
it replaces (`Confidence:InheritFrom` map in config, at half strength), shown with basis `"model"`,
until it has its own evidence. EP-33 candidate prompts (`*-v2` at a rollout %) build their own cells
from their share of traffic. The Admin dashboard compares calibrated accuracy per version
(REQ-CNF-12).

---

## C. Using the score

### REQ-CNF-09 — Score on every recognised line — P0
`DesignAnalysisItemDto`, `InventoryItemDto` (recognised lines) and the stock count line DTO gain:

```
confidenceScore: {
  identity: { score, band, basis, checks },
  count:    { score, band, basis, checks } | null
}
```

and keep `confidence` (raw) for compatibility, documented as deprecated for display. The score is
computed at the end of the job (stored on the line) and recomputed when the review is read if the
cells changed since (cheap, cached).
**Accept**: DTO tests; contract §21.3 / §22 updated.

### REQ-CNF-10 — Portal display — P0
Replace "90 % sure" in `_AnalysisReview`, the inventory step and the stock count review with:
- **High**: "Usually right · 41 checks" (green, quiet).
- **Medium**: "Check this · right 7 of 10 times" (amber).
- **Low**: "Often wrong · check it" (red). Low lines are listed first.
- **None**: "New: not enough checks yet" (grey).

A tooltip explains the basis ("from florists' checks across the platform" or "from your own checks")
and that the number is how often this kind of answer was right, not a guarantee. The count shows "AI
saw 11–13 · counts like this are usually within ±1" when the count score is High. Hide the raw
percentage.
**Accept**: portal tests for the four states; `DesignWizard` rules unit-tested in Domain.Tests.

### REQ-CNF-11 — Decisions driven by the score — P1
- `StockMatcher.Score` uses the calibrated identity score in place of the raw confidence. Bands,
  caps and determinism stay the same.
- A line with identity band Low is **never auto-linked**, even to a High stock match. The florist
  confirms it.
- The stock count `NeedsCheck` uses the count band (Low or None) in place of `< 0.6` raw.
- The Cost Assessor (EP-36) widens the quantity range of lines with a Low count score
  (min −25 %, max +25 %) before computing the cost range.

**Accept**: existing matcher tests updated with explicit scores; new tests for the auto-link rule and
the range widening.

---

## D. Admin and quality

### REQ-CNF-12 — AI quality dashboard: calibration — P1
The Admin Portal AI quality page (EP-33) gains, per provider/model/prompt version and task:
- a **reliability table**: raw-confidence bucket → lines checked → observed accuracy;
- **ECE**, the expected calibration error of raw versus calibrated;
- **accuracy per flower type** (top 20 by volume, worst first);
- **count accuracy** (share within ±1/15 %);
- **verdict coverage** (explicit / implicit / none).

These are platform aggregates only, with the cell minimum applied, so no vendor is identifiable. The
export (`ExportAiFloristFeedbackQuery`) adds the verdict columns, scrubbed by `FeedbackScrubber`.
**Accept**: calculator unit tests; page renders with empty data.

### REQ-CNF-13 — Vendor transparency and opt-out — P1
The Vendor Portal AI Florist house-style page (`Pages/Design/HouseStyle`) gains "Help improve recognition for everyone" (default **on**,
see H.1). Off means the vendor's verdicts feed only its own shop cells; it still receives platform
scores. Turning it off excludes the vendor's verdicts from the next aggregation. The docs page (EP-38
vendor audience) explains what is shared: counts per flower type only.

---

## E. Data and migration

### REQ-CNF-14 — Migration and backfill — P0
Migration `AddRecognitionConfidence`: the `RecognitionVerdicts`, `RecognitionConfidenceCells` and
`RecognitionConfidenceShopCells` tables; `ShareRecognitionFeedback` (bool, default true) on the
vendor style profile (`DesignStyleProfile`); the score columns on stored lines are JSON inside the existing snapshots, with
no schema change.
A one-off backfill command turns existing `RecognitionCorrection` rows into `Wrong` verdicts
(Renamed/Recoloured/Removed) and count verdicts (Recounted). It turns confirmed Photos-mode sessions'
untouched lines into `Implicit` `Correct` verdicts, using the session snapshots. It is idempotent.
**Accept**: the migration applies on a copy of prod; running the backfill twice gives the same counts.

---

## F. Mobile follow-ups

### REQ-CNF-15 — Vendor mobile app — P1
EP-M20 in the vendor app repo: the verdict control on the review screens (EP-M14/M15/M16), the
score display states of REQ-CNF-10, and the opt-out in settings.

---

## G. Quality

### REQ-CNF-16 — Tests, contract and docs — P1
Domain, handler, aggregator and portal tests for REQ-CNF-01…14. `docs/API-CONTRACTS.md` §21.3 /
§21 inventory / §22 stock counts gain `confidenceScore` and the verdict endpoints. The vendor manual
gets a "How sure is the AI?" page (EP-38 vendor audience), and the changelog an entry.

---

## H. Open decisions

1. **Default for sharing.** Opt-out (on by default, proposed) or opt-in. Opt-in keeps EP-33's promise
   strictly, but the platform cells would fill slowly.
2. **What counts as "right" for identity.** v1: the flower type matches (canonical key). Should a
   wrong colour or variety also make a line "wrong"? *Proposed:* track colour and variety as separate,
   later scores; they need the EP-41 flower atlas to give the florist a vocabulary.
3. **Image-based evidence.** Once per-item crops exist (EP-41, `REQ-GAPS-S.md`), a
   verdict could carry the crop for an atlas of confirmed examples. That would mean storing vendor
   photo crops in a shared set: a separate privacy decision, not part of this epic.
4. **Thresholds.** The bands (0.85 / 0.6), `MinChecks` 5, `MinVendorsPerCell` 3, `ImplicitWeight`
   0.25 and the half-life of 120 days are starting values, to be tuned after the pilot from the
   dashboard.
