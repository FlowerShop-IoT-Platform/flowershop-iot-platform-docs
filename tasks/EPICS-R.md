# Epics, Stories & Tasks — Option R (Recognition confidence from human feedback)

> Companion to EPICS-H.md (EP-28 recognition), EPICS-M.md (EP-33 feedback loop) and EPICS-N.md
> (EP-34 stock count, EP-35 bouquet analysis, EP-36 cost assessor).
> Epic is numbered EP-40 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep40-recognition-confidence-feedback.json`.
> Requirements: `docs/tasks/REQ-GAPS-R.md` (REQ-CNF-01 … REQ-CNF-16).
> Mobile counterpart: vendor app EP-M20 (T-40-016).
> Related: EP-41 (`EPICS-S.md`) adds the flower atlas, item crops and image matching; it shares `FlowerTypeKey` (T-40-003).
> Analysis date: 2026-10-23

---

## EP-40: Recognition confidence from human feedback

**Priority**: P1-high
**Dependencies**: EP-33 (corrections, quality dashboard), EP-34/35/36 (the three reviews)
**Requirement refs**: REQ-CNF-01 through REQ-CNF-16
**State**: proposed (design only); open decisions in REQ-GAPS-R §H.

> "90 % sure" today is the model rating itself, and it says that about almost everything. After this
> epic, the number means "florists checked answers like this N times and it was right X % of the
> time". It is computed in code from florists' verdicts, and it tells the florist which lines to check.

### Concept

```
 review line ── ✓ Right · ✗ Wrong (it's actually …/not in the bouquet) · ? ── count kept/edited
      │ explicit verdict (weight 1)            confirm without touching → implicit verdict (0.25)
      ▼
 RecognitionVerdicts (vendor-scoped) ──hourly──▶ platform cells (model, prompt, task, flower type, raw bucket;
      │                                          anonymous, used only with ≥ 3 vendors, per-vendor cap)
      └──on write──▶ shop cells (vendor, model, task, flower type)
                                 ▼
 score = shop rate shrunk toward platform type rate shrunk toward model bucket prior (Beta-Binomial, decay)
      ▼
 { identity, count } × { score, band High/Medium/Low, basis shop|platform|model|none, checks }
      ▼
 portal: "Usually right · 41 checks" / "Check this" / "Often wrong" / "Not enough checks yet"
 code:   StockMatcher multiplier · no auto-link when Low · stock count NeedsCheck · wider cost range
```

### Design decisions
- **Platform counts plus a shop adjustment** (decided 2026-10-23). One shop's verdicts alone are too
  few per flower type to be reliable. Anonymous counts are pooled across vendors, with no vendor id,
  text, photos or prices, and a cell is used only when at least 3 vendors contributed to it. Each
  shop's own verdicts then pull the score toward what that shop sees. This relaxes EP-33's "no
  cross-vendor learning" for counts only. Known-correction hints stay per shop.
- **Verdicts, not just corrections.** `RecognitionCorrection` logs only what changed. A rate also
  needs the lines that were checked and found right, so a new verdict ledger records both.
- **"Wrong flower" is not "not in my stock".** The analysis review mixes the two today. The identity
  verdict is a separate control, so a correct recognition of a flower the shop lacks counts as right.
- **Silent confirmation is weak evidence.** Florists often confirm without looking, so implicit
  verdicts count at 0.25. Lines nobody touched and nobody used count for nothing.
- **Code, not the model.** The score is a deterministic Beta-Binomial estimate. The model's raw value
  is used only to choose the calibration bucket.
- **Identity and count are scored separately.** A florist needs to know "is it astrantia?" and "are
  there really 5?" as two different questions.

### Stories & tasks
See `tasks-ep40-recognition-confidence-feedback.json` (S-40-01 … S-40-07; T-40-001 … T-40-018).
1. **S-40-01 — Trackers**: T-40-001.
2. **S-40-02 — Feedback capture**: T-40-002 verdict entity · T-40-003 `FlowerTypeKey` · T-40-004 explicit verdict endpoints · T-40-005 implicit verdicts on confirm · T-40-006 migration `AddRecognitionConfidence` + backfill.
3. **S-40-03 — Confidence model**: T-40-007 scoring service · T-40-008 aggregator + cell cache · T-40-009 abuse/noise limits · T-40-010 new-version fallback.
4. **S-40-04 — Using the score**: T-40-011 `confidenceScore` on line DTOs · T-40-012 matcher, auto-link, `NeedsCheck` and cost range.
5. **S-40-05 — Vendor Portal and Admin Portal**: T-40-013 verdict control + score display · T-40-014 sharing opt-out · T-40-015 calibration dashboard.
6. **S-40-06 — Mobile follow-up**: T-40-016 vendor app EP-M20.
7. **S-40-07 — Tests, contract and docs**: T-40-017 tests · T-40-018 contract, manual, changelog.

**Suggested order**: T-40-003 → rest of S-40-02 → S-40-03 → S-40-04 → S-40-05 → S-40-07. S-40-06
follows the API.

### Rollout
1. Ship verdict capture (S-40-02) and the backfill. Nothing changes for florists except the new
   ✓/✗/? control, and evidence starts to build.
2. Ship the scoring service and aggregator with the display behind
   `BouquetDesign:Confidence:ShowScores=false`. Check the Admin calibration table for the pilot
   shops.
3. When the platform cells for the common flower types reach the minimum, switch the display on.
   Then switch the score-driven decisions on (T-40-012).
4. Tune the thresholds (REQ-GAPS-R §H.4) from the dashboard after one month.
