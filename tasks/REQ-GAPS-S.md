# Requirement Gaps — Option S (Flower atlas, item crops and image matching)

> Companion to REQ-GAPS-N.md (EP-35 bouquet analysis, EP-36 cost assessor), REQ-GAPS-K.md (EP-31 stock
> photos) and REQ-GAPS-R.md (EP-40 recognition confidence from human feedback).
> Requirement refs point at the tasks in `tasks-ep41-flower-atlas-image-matching.json`.
> Analysis date: 2026-10-23
> Status: proposed (EP-41, see `EPICS-S.md`). Open decisions are in section H.

## Context

The bouquet analysis review (`_AnalysisReview.cshtml`) shows each detected item as text: a
pictogram, a name, colour swatches and a count. The florist can't see **which part of the photo** the AI
means, or **what that flower should look like**. Checking a line means scanning the whole photo and
remembering what astrantia looks like. Vendors asked for two images per line:

1. **A crop of the item** from their own photo.
2. **A reference image** of the flower the AI named.

They also asked whether a **flower atlas compared with real images** could replace parts of the LLM.

### What the code does today

| Area | Behaviour | Where |
|---|---|---|
| Detected items | Kind, name, colours, variety, quantity (+ range), visibility, raw confidence. **No position** in the photo, and no record of which photo it came from. | `DetectedItem` in `DesignAnalysis.cs`, `OpenAiBouquetAnalyzer` |
| Analysis photos | 1–5 private photos per session, processed by `SkiaDesignImageProcessor` (re-encoded, EXIF stripped), stored through `IDesignMediaStore` (private R2). | `DesignJobProcessor`, `DesignMediaStores.cs` |
| Media serving | `GET /vendor/design-sessions/{id}/media/...` by `DesignMediaKind`; the Vendor Portal proxies it same-origin (`Pages/Design/Media`). | `BouquetDesignController.cs:107`, `Media.cshtml.cs` |
| Stock photos | Up to 5 private photos per stock item with a main photo (EP-31). These are labelled, vendor-owned examples of exactly the flowers the shop buys. | `StockPhoto`, `StockAppearanceService` |
| Matching | `StockMatcher` compares **text** (canonical name, colour words → ΔE, variety). It never looks at pixels. | `StockMatcher.cs` |
| Reference images | None. No flower vocabulary beyond the `StockMatcher` synonym table. | — |

### Approach: combine, don't replace

The LLM stays as the item lister. Images are added in three phases. Each phase is useful on its own
and decides whether the next one is worth doing.

```
 analysis photo(s) ──▶ LLM (or detector) lists items + boxes {photoIndex, x, y, w, h}
                            │
                            ├─▶ crop (Skia) ─▶ private crop per item ──────────────┐
                            │                                                       ▼
                            │            image fingerprint (CLIP/SigLIP/DINOv2) ─▶ nearest neighbours
                            │                                     │                 │
                            ▼                                     ▼                 ▼
                     flower atlas entry (by type key)     global atlas photos   this shop's stock photos
                            │                                     │                 │
                            └──────────── review line: [crop] [atlas photo] [my stock photo] ───────┘
                                          + "AI says Astilbe, the photo looks like Amaranthus" flag
```

| Phase | Delivers | Effort |
|---|---|---|
| 1 | Crop + atlas reference image per line (boxes from the vision model, curated atlas) | ~1.5–2 weeks |
| 2 | Image fingerprints; crop ↔ atlas and crop ↔ own stock photo matching; disagreement flags | ~2–3 weeks |
| 3 | Dedicated detector service + hand-labelled evaluation set; switch only if it wins | ~2–3 weeks, only if needed |

**What it won't do**: replace counting. Stems overlap in crowded bouquets, and nearest-neighbour
matching gives a name, not a count. Counts stay with the LLM plus florist feedback (EP-40). Image
fingerprints are also weak on fine colour ("cappuccino" against "pale pink"), so colour stays judged
in code (ΔE), as today.

---

## A. Phase 1 — Flower atlas

### REQ-ATL-01 — Atlas data model — P0
A platform-level (not vendor-scoped) reference catalogue:
- **`AtlasFlower`**: `typeKey` (EP-40 `FlowerTypeKey`, e.g. `flower:carnation`), kind
  (Flower/Greenery/Material), scientific name, English and Polish names, aliases in both languages
  (fed to the `FlowerTypeKey` synonyms), a short description of how to recognise it, and typical
  colours.
- **`AtlasVariety`**: the parent flower, a name (e.g. "Seeded", "Cappuccino", "Spray"), aliases,
  colours and notes.
- **`AtlasImage`**: the flower or variety, a storage key (public R2 bucket/prefix, served from
  `img.findmyflowers.pl/atlas/...`), width and height, `source` (WikimediaCommons | iNaturalist |
  Own | Licensed), source URL, author, licence (SPDX id, e.g. `CC-BY-SA-4.0`), `attribution` text,
  `isPrimary`, and `reviewedBy`/`reviewedAt`.

Only reviewed images are served. Licences allowed by config: `Atlas:AllowedLicences`, default CC0,
CC-BY, CC-BY-SA; no NC/ND, because the platform is commercial.
**Accept**: domain tests (one primary per flower/variety, licence allowed, attribution required unless
the source is Own).

### REQ-ATL-02 — Seed the atlas with ~150 cut flowers — P0
A curated seed list (JSON in the repo, `docs/atlas/atlas-seed.json`) of about 150 common cut flowers,
greenery and fillers, with PL/EN names, aliases and the main varieties florists distinguish. It must
include the types in recent analyses: carnation/spray carnation (incl. "Cappuccino"), pincushion
protea (Leucospermum), astrantia, astilbe, amaranthus, eucalyptus with the **seeded** variety,
lisianthus, rose and spray rose, gypsophila, chrysanthemum, alstroemeria. An import tool
(`AtlasImportCommand`, Admin only) fetches 2–4 candidate images per entry. It uses the Wikimedia
Commons API (licence and author from `extmetadata`) and the iNaturalist API (research-grade
observations with CC licences only), and stores them as **unreviewed** with full provenance. **Never
Google Images** or any other unlicensed scraping.
**Accept**: the import is idempotent (same source URL → same row); NC/ND images are skipped and
logged.

### REQ-ATL-03 — Admin Portal atlas curation — P0
`Pages/Atlas/*` in the Admin Portal:
- a list of flowers with coverage (has primary image / unreviewed count);
- edit names, aliases, varieties and description;
- review the candidate images (approve, reject, set primary) with the licence and attribution shown;
- upload own photos (source `Own`).

API under `/api/v1/admin/atlas/*` (`PlatformAdmin`).
**Accept**: portal tests; an unreviewed image is never returned by vendor endpoints.

### REQ-ATL-04 — Link analysis lines to the atlas — P0
After analysis (and on Photos-mode recognition and stock count lines), each detected item is linked to
an atlas entry by `FlowerTypeKey` and then the variety (aliases included). This is a pure code lookup
with no model call. `DesignAnalysisItemDto` gains
`atlas: { flowerId, varietyId?, name (in the vendor's language), imageUrl, attribution, sourceUrl } | null`.
When the atlas knows a variety the AI didn't name (e.g. eucalyptus has a common "Seeded" variety), the
review shows "Could be: Seeded eucalyptus" as a one-tap rename. It is never applied automatically.
**Accept**: unit tests for the lookup (synonym, plural, variety alias, unknown → null).

### REQ-ATL-05 — Atlas as the vocabulary for feedback — P1
EP-40's "What is it?" picker (REQ-CNF-02) suggests atlas names (PL/EN) first, then the vendor's stock
names. The `StockMatcher` synonym table is generated from the atlas aliases, so the two cannot drift.
**Accept**: `StockMatcher` tests pass with the generated table.

---

## B. Phase 1 — Item crops

### REQ-ATL-06 — Boxes from the vision model — P0
The bouquet analysis schema (`OpenAiBouquetAnalyzer`, new prompt version `bouquet-analysis-v2` in
`PromptCatalog`) gains per item `boxes: [{ photoIndex, x, y, w, h }]`, normalised 0–1, up to 3 per
item: the clearest instances, not every stem. `DetectedItem` gains `Boxes`.
`DesignAnalysis.Clean` validates them in code:
- the photo index is within range;
- coordinates are clamped to 0–1;
- boxes smaller than 2 % of the photo's area are dropped;
- boxes covering more than 80 % of it are dropped (it's "the whole bouquet").

The free `Palette` analyzer returns no boxes. v2 is rolled out through the EP-33 candidate mechanism
(starting at 0 %, then a pilot).
**Accept**: Clean tests for each rule; v1 output still parses (no boxes).

### REQ-ATL-07 — Crop and store — P0
At the end of the analysis job, for each item with a box, `IDesignImageProcessor` gains
`Crop(bytes, box, padding 8 %, maxLongEdge 384)`, implemented in `SkiaDesignImageProcessor`. The crop
is stored privately through `IDesignMediaStore` under the session's prefix (purged with the session,
like the photos), and served as a new `DesignMediaKind.Crop`
(`GET /vendor/design-sessions/{id}/media/crops/{itemId}`, also proxied by the portal `Media` page).
Crops are made only from the session's own processed photos and never leave the vendor's private
storage.
**Accept**: crop unit tests (padding, clamping, EXIF already stripped); deleting a session purges its
crops.

### REQ-ATL-08 — Review shows the images — P0
In `_AnalysisReview`, the Cost Assessor review and the vendor app (EP-M21), each line shows up to
three thumbnails in place of the pictogram:
- **"In your photo"**: the crop, or the pictogram when there is no box;
- **"Looks like"**: the atlas image, with an attribution tooltip and a link to the source;
- **"Your stock"**: the main photo of the matched stock item (already available as `mainPhotoId`).

Clicking the crop highlights the box on the full photo. Alt text names the flower.
**Accept**: portal tests for all combinations of missing images; the attribution is rendered for CC-BY
and CC-BY-SA images.

---

## C. Phase 2 — Image fingerprints and matching

### REQ-ATL-09 — Embedding provider port — P1
`IImageEmbedder` in Application: `EmbedAsync(images) → float[][]`, plus `Model` and `Dimensions`.
Providers behind `BouquetDesign:Embedding:Provider`:
- `None` (default: Phase 2 off);
- `Onnx` (CLIP ViT-B/32 or SigLIP-base in-process through ONNX Runtime; CPU is fine for crops);
- `Http` (an external embedding service; Phase 3's service can serve this too).

Each vector is stored with its model id. Vectors from different models are never compared.
**Accept**: contract test with a fake embedder; `None` leaves Phase 1 behaviour unchanged.

### REQ-ATL-10 — Vector storage — P1
Add pgvector to PostgreSQL 15 (Fly Postgres supports the extension). Three tables of vectors:
- `AtlasImageEmbeddings` (platform);
- `StockPhotoEmbeddings` (vendor-scoped; computed when an EP-31 photo is added, backfilled for
  existing photos);
- `AnalysisCropEmbeddings` (vendor-scoped, purged with the session).

HNSW index, cosine distance. Every query on vendor tables is filtered by VendorId + VendorType first.
**Accept**: migration applies on a copy of prod; a tenant-isolation test (vendor A's crop never
returns vendor B's stock photo).

### REQ-ATL-11 — Image-based match signals — P1
For each crop:
- the top-3 **atlas** neighbours, grouped by flower, with a similarity score;
- the top-3 **own stock** neighbours, with a similarity score.

The signals are combined in code:
- `StockMatcher.Score` gains an image factor ("photo looks like your <stock item>"). Its weight is
  `Embedding:StockWeight` (default 0.3), and the text factors are rescaled. When there's no
  embedding, today's behaviour is unchanged.
- An **atlas disagreement** flag is set when the atlas top flower ≠ the AI's flower type and the
  similarity margin is above `Embedding:DisagreeMargin`. The review shows "AI says Astilbe, the photo
  looks like Amaranthus" with one-tap rename options for both. It is never applied automatically.
- A crop whose nearest own-stock photo clearly matches a stock item the text match missed is offered
  as "Your stock may have this: <item>".

**Accept**: unit tests with fixed vectors; determinism (same inputs → same score).

### REQ-ATL-12 — Feedback links — P1
EP-40 verdicts on lines with a crop store the crop's embedding id. A confirmed verdict (✓, or ✗ with
the actual name) **on the vendor's own data** can then add the crop as a labelled example to that
vendor's private stock or atlas neighbourhood (`Embedding:LearnFromVerdicts`, default off). Crops are
**never** added to the shared atlas (see H.2).

---

## D. Phase 3 — Detector service and evaluation (only if needed)

### REQ-ATL-13 — Evaluation set and harness — P0 for Phase 3
Before any switch, build a hand-labelled set of 50–100 bench and customer photos: boxes, flower type,
variety and count per item. Store it in a private bucket, not the repo; only the manifest goes in the
repo. Build a harness (`tests/FlowerShop.Tests.Evaluation`, run manually or nightly, not in CI) that
scores each pipeline (LLM v1, LLM v2 boxes, detector, + atlas re-ranking) on:
- identity precision and recall per type;
- box IoU;
- count error;
- cost per photo.

Phase 2 and 3 changes ship only when they beat the current pipeline on this set. EP-40 verdicts add
real-world evidence after release.
**Accept**: the harness runs end to end on 5 sample photos; a report is written as markdown.

### REQ-ATL-14 — Dedicated detector service — P2
An optional `IItemDetector` port: input photo + the item names the LLM found → boxes per name. It is
an open-vocabulary detector (Grounding DINO, OWLv2 or Florence-2) in a small Python service, chosen by
`BouquetDesign:Detector:Provider` = `None` (default) | `Http` (own Fly GPU machine, scale-to-zero)
| `Replicate` | `Roboflow`. When on, its boxes replace the LLM's for cropping. It is metered like other
providers (quota, $ cap, `DesignUsageService`). The service also exposes `/embed` for REQ-ATL-09
`Http`.
**Accept**: adapter contract tests with a fake HTTP server; timeout and failure fall back to the
LLM's boxes; ops notes in `docs/deployment/AI-BOUQUET-ASSISTANT.md`.

---

## E. Quality

### REQ-ATL-15 — Tests, contract and docs — P1
Domain, handler, crop, embedding-math and portal tests for REQ-ATL-01…12. `docs/API-CONTRACTS.md`
§21.3 gets `atlas`, `boxes`, `cropUrl` and `imageMatches` on analysis items, plus the crop media
route. A new §25.x "Flower atlas (admin)" section (or the next free number when EP-39 has taken §25)
covers the atlas API. The vendor manual page "What the pictures mean" (EP-38 vendor audience) and an
`ATTRIBUTIONS` page listing atlas image credits (public) are added, plus a changelog entry.

### REQ-ATL-16 — Vendor mobile app — P1
EP-M21 in the vendor app repo: the three thumbnails per line, tap-to-highlight on the photo, the
disagreement flag and the "Could be" suggestions.

---

## H. Open decisions

1. **Commercial licensed library.** Wikimedia/iNaturalist cover common species well but varieties
   poorly (e.g. carnation "Cappuccino"). Should we buy a stock-image licence, or ask partner growers
   and wholesalers for their catalogue photos? *Proposed:* start with the open sources plus our own
   photos, and fill variety gaps with photos from pilot florists, with their written permission
   (source `Own`).
2. **Shared atlas from vendor crops.** Confirmed crops from many shops would make the best atlas, but
   they are vendor photos. This interacts with EP-40's sharing decision. *Proposed:* not in this epic.
   It needs an explicit opt-in per photo and an updated privacy notice.
3. **Where embeddings run.** In-process ONNX (simple, adds ~150 MB model and CPU load to the API
   machine, which already runs the design worker) or the Phase 3 service. *Proposed:* ONNX in-process
   for Phase 2, and move it when Phase 3 exists.
4. **Language.** Atlas names in PL and EN from the start. The review shows the vendor's portal
   language, and other languages fall back to EN.
