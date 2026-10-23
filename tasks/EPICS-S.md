# Epics, Stories & Tasks — Option S (Flower atlas, item crops and image matching)

> Companion to EPICS-N.md (EP-35 bouquet analysis, EP-36 cost assessor), EPICS-K.md (EP-31 stock
> photos) and EPICS-R.md (EP-40 recognition confidence from human feedback).
> Epic is numbered EP-41 to continue the sequence.
> Task tracker (ground truth): `docs/tasks/tasks-ep41-flower-atlas-image-matching.json`.
> Requirements: `docs/tasks/REQ-GAPS-S.md` (REQ-ATL-01 … REQ-ATL-16).
> Mobile counterpart: vendor app EP-M21 (T-41-018).
> Analysis date: 2026-10-23

---

## EP-41: Flower atlas, item crops and image matching

**Priority**: P1-high
**Dependencies**: EP-35/36 (analysis review), EP-31 (stock photos), EP-40 (`FlowerTypeKey`, verdicts)
**Requirement refs**: REQ-ATL-01 through REQ-ATL-16
**State**: proposed (design only); open decisions in REQ-GAPS-S §H.

> Each line in the bouquet review shows three pictures: what the AI saw in your photo, what that
> flower looks like, and the stock item it was matched to. Later, the pictures themselves are
> compared, so the platform can say "the AI says astilbe, but this looks like amaranthus".

### Concept

```
 photo(s) ─▶ LLM lists items + boxes {photoIndex, x, y, w, h} (bouquet-analysis-v2)
            │                     └─ Phase 3, only if needed: open-vocabulary detector gives better boxes
            ▼
 Skia crop (private, purged with the session)
            ├─ Phase 1 ─▶ atlas lookup by FlowerTypeKey + variety ─▶ licensed reference image + credit
            └─ Phase 2 ─▶ image fingerprint (CLIP/SigLIP, pgvector)
                           ├─▶ nearest atlas photos     → "photo looks like …" disagreement flag
                           └─▶ nearest own stock photos → image factor in StockMatcher
 review line: [In your photo] [Looks like] [Your stock]   + "Could be: Seeded eucalyptus"
```

### Design decisions
- **Combine, don't replace.** The LLM keeps listing and counting. Images confirm the identity and show
  the florist what the AI meant. Image fingerprints can't count overlapping stems and are weak on fine
  colour, so counts stay with the LLM and EP-40 feedback, and colour with the code's ΔE.
- **Licensed atlas only.** No Google Images. Images come from Wikimedia Commons and iNaturalist
  (CC0/CC-BY/CC-BY-SA, with credit) and our own photos. Each one is reviewed in the Admin Portal
  before any vendor sees it. Credits are shown on hover and on a public attributions page.
- **Two atlases.** The global atlas tells the florist what the flower is. The shop's own EP-31 stock
  photos act as a private atlas that says "you have this". The second one is where most of the value
  is, because it matches the bouquet to the shop's stock from the images, not from the AI's wording.
- **Private by default.** Crops and vendor image vectors are vendor-scoped and purged with the
  session. Vendor crops never enter the shared atlas (open decision 2).
- **Prove it before switching.** Phase 2 and 3 changes ship only when they beat the current pipeline
  on a hand-labelled set of 50–100 photos. Labelling starts during Phase 1.
- **Boxes per photo.** An analysis can use up to 5 photos, so every box records which one it is on.

### Phases and estimates
| Phase | Stories | Delivers | Effort |
|---|---|---|---|
| 1 | S-41-02, S-41-03 | Atlas (~150 flowers) + crops from the vision model's boxes + three thumbnails per line | ~1.5–2 weeks |
| 2 | S-41-04 | Image fingerprints, matching against the atlas and own stock photos, disagreement flags | ~2–3 weeks |
| 3 | S-41-05 | Evaluation harness (needed before Phase 2 switches) + optional detector service | ~2–3 weeks, only if needed |

### Stories & tasks
See `tasks-ep41-flower-atlas-image-matching.json` (S-41-01 … S-41-06; T-41-001 … T-41-018).
1. **S-41-01 — Trackers**: T-41-001.
2. **S-41-02 — Phase 1: flower atlas**: T-41-002 atlas model + migration · T-41-003 seed list + licensed import · T-41-004 Admin Portal curation · T-41-005 line → atlas lookup · T-41-006 atlas as the shared vocabulary.
3. **S-41-03 — Phase 1: item crops and review images**: T-41-007 boxes (`bouquet-analysis-v2`) · T-41-008 crop and store · T-41-009 three thumbnails per line.
4. **S-41-04 — Phase 2: image fingerprints and matching**: T-41-010 embedding port · T-41-011 pgvector storage · T-41-012 match signals + disagreement flags · T-41-013 EP-40 verdict links.
5. **S-41-05 — Phase 3: evaluation and detector**: T-41-014 evaluation set + harness · T-41-015 open-vocabulary detector service.
6. **S-41-06 — Tests, contract, docs and mobile**: T-41-016 tests · T-41-017 contract, manual, attributions, changelog · T-41-018 vendor app EP-M21.

**Suggested order**: S-41-02 → S-41-03 (start labelling T-41-014 in parallel) → S-41-04 → S-41-05 →
S-41-06. T-41-002 needs EP-40's `FlowerTypeKey` (T-40-003), so build that first, or carry a local copy.

### Rollout
1. Import and curate the atlas in the Admin Portal. Nothing shows to vendors until images are reviewed.
2. Turn on reference images for all vendors. This is a code lookup, with no model cost.
3. Roll out `bouquet-analysis-v2` (boxes) as an EP-33 candidate, 0 % → pilot shops, and check crop
   quality on the evaluation set.
4. Phase 2 behind `BouquetDesign:Embedding:Provider=None`. Backfill the stock photo vectors, turn it on
   for the pilot, and compare on the evaluation set and with EP-40 verdicts.
5. Phase 3 only if the vision model's boxes or labels aren't good enough.
