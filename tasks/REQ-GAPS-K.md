# Requirement Gaps — Option K (Stock photos & AI appearance)

> Companion to REQ-GAPS-J.md (Stock management, EP-30) and REQ-GAPS-H.md (AI Bouquet Assistant, EP-28).
> Requirement refs point at the tasks in `tasks-ep31-stock-photos-ai-appearance.json`; the mobile side
> is tracked in the vendor app repo (`docs/tasks/tasks-ep-m12-stock-photos.json`).
> Analysis date: 2026-10-18
> Status: REQ-STKPH-01 … 13 implemented (EP-31, see `EPICS-K.md`; vendor app EP-M12); production deploy pending.

## Context

EP-30 identifies a stock item by **name + colour** ("Rose / Red"). Florists testing it on the vendor
site found that too coarse: "red" covers dozens of shades, many roses are **multi-coloured** (red
outer petals with a white centre, picotee edges, gradients) and the market sells many **varieties**
of the same flower ('Freedom', 'Explorer', 'Vendela'…). A word cannot carry that; a photo can.

Goal: every stock item can hold photos, and an AI option reads a photo into a precise,
confirmable **appearance** — variety, short colour label, a colour palette with swatches, a pattern
and a plain-language description. The AI Bouquet Assistant then uses that appearance and those
photos. Product decisions (2026-10-18): add a **variety** to the item identity; the assistant uses
stock photos for previews, colour descriptions for recipes and matches bench photos to stock; AI
recognition has its **own monthly quota** under the assistant's kill switch and global $ cap.

---

## REQ-STKPH-01 — Variety in the item identity — P0
`StockItem` gains an optional **variety** (≤ 60 chars). The unique catalogue key becomes
name + colour + variety (trimmed, case-insensitive); items without a variety keep their existing key,
so no data migration of keys is needed.
**Accept**: "Rose / Red / Freedom" and "Rose / Red / Explorer" can coexist; a duplicate including the
variety → 409 `duplicate-item`; existing items are unaffected.

## REQ-STKPH-02 — Structured appearance — P0
An item may carry an **appearance**: description (≤ 500), pattern (`Solid`, `Bicolour`, `Edged`,
`Gradient`, `Speckled`, `Multicolour`), a palette of 1–5 colours (name ≤ 30, `#RRGGBB`, share %
summing to ≤ 100) and its source (`Manual` or `Ai` — AI only after the florist confirmed it).
**Accept**: invalid hex / share / pattern → 400 `invalid-request`; updates that omit `variety` or
`appearance` leave them unchanged (older clients cannot wipe them).

## REQ-STKPH-03 — Item photos in private storage — P0
Up to 5 photos per item, uploaded as multipart field `photo`, decoded server-side (declared MIME
ignored), orientation applied, downsized to 1600 px, re-encoded JPEG ≤ 1 MiB (EXIF/GPS stripped) and
stored in the **private** design media store (never the public CDN). One photo is the primary.
Bytes only via authenticated `…/content` endpoints with `Cache-Control: private, no-store`.
**Accept**: 6th photo → 400 `photo-limit`; non-image → 400 `invalid-photo`; deleting the primary
promotes the next photo; another vendor's photo → 404.

## REQ-STKPH-04 — AI "describe this photo" — P0
`POST …/photos/{photoId}/describe` returns a **suggestion** (kind, name, variety, colour label,
description, pattern, palette, confidence). It never changes the item; the florist applies it.
**Accept**: suggestion contains 1–5 palette colours with valid hex; replay with the same
`Idempotency-Key` returns the stored suggestion and is billed once.

## REQ-STKPH-05 — Add an item from a photo — P1
`POST /vendor/stock/photo-suggestions` uploads a photo as a **pending** photo and returns it with a
suggestion (or `null` with a notice when AI is off/out of quota). Creating an item with
`pendingPhotoId` attaches the photo as its primary. Unattached pending photos are purged after 24 h.
**Accept**: the photo is usable after create; a pending photo of another vendor cannot be attached.

## REQ-STKPH-06 — Quota, kill switch and spend cap — P0
Stock recognition has its own monthly per-vendor limit (`BouquetDesign:Limits:StockRecognitionsPerMonth`,
default 100), is gated by `BouquetDesign:Enabled` + `PilotVendorIds`, and its cost counts towards
`GlobalMonthlyUsdCap`. Every call is a ledger row (reservation before the provider call). Provider is
config-switchable (`BouquetDesign:StockRecognition:Provider` = `Palette` free default | `OpenAI`).
**Accept**: quota used up → 429 `quota-exhausted`; disabled → 403 `feature-disabled`; photos still
work when AI is off; `GET /vendor/stock/ai-usage` reports enabled/used/limit/remaining/resetsAt.

## REQ-STKPH-07 — Free palette provider — P1
The default provider reads the real dominant colours from the photo (no AI spend): up to 3 colours
with shares, nearest florist colour names, pattern Solid/Bicolour/Multicolour and a templated
description. Name/variety come from hints only.
**Accept**: a photo that is mostly red with a white centre yields red + white with plausible shares.

## REQ-STKPH-08 — Assistant lines carry the stock item — P0
Assistant stock lines and recipe lines gain optional `stockItemId` and `variety`. Lines imported from
stock (vendor app) keep them; deducting a recipe uses `stockItemId` when present; prices fall back to
that item's unit cost first.
**Accept**: two varieties of "Rose / Red" stay distinct through import → recipe → deduction.

## REQ-STKPH-09 — Previews reference stock photos — P1
When rendering a preview, the primary photos of the recipe's stock items are sent as reference
images first (within the provider's reference limit), then bench photos.

## REQ-STKPH-10 — Recipes see real colours — P1
The composer receives each line's appearance description and palette, and the deterministic composer
uses palette names for the recipe palette.

## REQ-STKPH-11 — Bench photos match stock items — P1
Recognition receives the vendor's stock catalogue (name, variety, colour, description) and may link
each recognised line to a stock item; links are validated server-side and linked lines take the
item's name, variety, colour and unit cost. Recognition still only produces estimates.

## REQ-STKPH-12 — Portal and app UI — P0
Vendor Portal and vendor app: photo gallery on the item (add, delete, make primary), "Describe with
AI" with swatches and Apply, "Add from photo" on the add form, variety + swatches in lists, AI usage
shown; vendor app assistant rows show variety.

## REQ-STKPH-13 — Contracts and docs — P1
`API-CONTRACTS.md` §22 extended, `outbound.md`, change log, both `CLAUDE.md`.

## Explicitly out of scope
- Photo-based visual search/similarity across vendors; a shared variety catalogue.
- Automatic re-recognition when photos change; bulk "describe all".
