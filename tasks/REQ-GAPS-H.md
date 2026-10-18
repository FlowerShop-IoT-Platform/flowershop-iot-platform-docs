# Requirement Gaps — Option H (AI Florist)

> Companion to REQ-GAPS.md (Mobile API), REQ-GAPS-B.md (Portals/Analytics/IoT),
> REQ-GAPS-C.md (Vendor onboarding/subscription/vase), REQ-GAPS-D.md (Showcase deployment),
> REQ-GAPS-F.md (Customer portal mobile redesign) and REQ-GAPS-G.md (Contact us).
> Requirement refs point at the tasks in `tasks-ep28-ai-bouquet-assistant.json`; the mobile side is
> tracked in the vendor app repo (`docs/tasks/tasks-ep-m10-ai-bouquet-assistant.json`).
> Design: `docs/superpowers/specs/2026-10-13-ai-bouquet-assistant-design.md`;
> plan: `docs/superpowers/plans/2026-10-13-ai-bouquet-assistant-implementation.md`;
> costs: `docs/superpowers/plans/2026-10-13-ai-bouquet-assistant-costs.md`;
> mockup board: `docs/superpowers/mockups/ai-bouquet-assistant/01-feature-flow.png`.
> Analysis date: 2026-10-14

## Context

Shop employees and freelance florists want to turn the flowers and materials they **already have**
into usable bouquet recipes, see what each costs, and — only when they ask — an illustrative AI
preview. The platform had nothing comparable: no design session, no recipe or cost model, no
durable execution for slow paid calls (the RabbitMQ bus is a stub and the outbox only dispatches
domain events), and the only image storage is the **public** bouquet CDN.

Board flow: **Capture → Review stock → Configure → Recipes → Recipe & preview**. Board screen 06
("Additions · later release") is explicitly out of scope for this epic.

The vendor mobile app is not live yet; the first release is a **pilot prototype** that must run
end-to-end in the app with zero AI spend (free deterministic providers) and switch to paid
providers by configuration once the evaluation (REQ-AIB-18) picks one.

---

## REQ-AIB-01 — Vendor-only feature with kill switch and pilot allow-list — P0
All endpoints use the `VendorAccess` policy. `BouquetDesign:Enabled` stops new sessions and paid
work; `PreviewsEnabled` stops previews alone; `PilotVendorIds` restricts new work to listed vendors.
Disabling never hides or deletes existing recipes.
**Accept**: disabled flag → 403 `feature-disabled` on create/start; existing session GETs still 200;
`GET /vendor/design-assistant/usage` reports `enabled`/`previewsEnabled` so the app can hide entry points.

## REQ-AIB-02 — Design session lifecycle with optimistic concurrency — P0
A vendor-owned `DesignSession` aggregate moves Draft → Recognizing → InventoryReady →
InventoryConfirmed → Composing → RecipesReady (manual: Draft → InventoryConfirmed). Every mutation
bumps `version`; writes carry `expectedVersion`. Failed steps keep the last useful state plus a
plain-language `notice`.
**Accept**: stale `expectedVersion` → 409 `version-conflict`; illegal transition → 409 `invalid-state`;
a failed composition returns to InventoryConfirmed (or RecipesReady if recipes exist).

## REQ-AIB-03 — Private stock-photo upload — P0
Up to 5 photos per session via multipart field `photo` (+ optional `clientPhotoId` for replay-safe
retries). The server decodes the bytes (declared MIME ignored), rejects > 20 MP before allocating,
applies EXIF orientation, downsizes to 1600 px, re-encodes JPEG ≤ 1 MiB (strips EXIF/GPS) and stores
it in **private** storage (separate private R2 bucket in prod; never `img.findmyflowers.pl`).
**Accept**: 6th photo → 400 `photo-limit`; non-image → 400 `invalid-photo`; replayed `clientPhotoId`
is a no-op; bytes only via the authenticated `…/content` endpoint with `Cache-Control: private, no-store`.

## REQ-AIB-04 — Photo recognition produces estimates that must be confirmed — P0
A billable recognition job turns photos into stock lines flagged `estimated` with a confidence.
Overlapping photos must not be summed blindly. Recognition never confirms stock.
**Accept**: after recognition the session is InventoryReady with `inventory.confirmed=false` and
every line `estimated=true`; composing before confirmation → 409.

## REQ-AIB-05 — Manual stock entry and editing — P0
`PUT …/inventory` replaces the stock snapshot: kind (Flower/Greenery/Material), name, colour,
quantity 1–9999, unit (flowers/greenery in stems), optional unit price; ≤ 100 lines. Editing confirmed
stock bumps `stockRevision` and drops **unsaved** recipes; saved recipes keep their snapshot.
**Accept**: manual sessions never call a recogniser; invalid lines → 400 `invalid-inventory`.

## REQ-AIB-06 — Stock-valid recipe composition — P0
A billable composition job returns up to 3 independent alternatives for size **M** (11–21 flower +
greenery stems; materials excluded), available-stock-only mode, optional style and cost ceiling.
Every proposal — LLM or rule-based — is validated in code: known line ids only, positive integers,
duplicate references aggregated before the stock check, stem range, distinct alternatives.
**Accept**: invalid proposals are dropped (never "repaired" with another paid call); fewer than
requested → `notice` explains; none → session back to InventoryConfirmed with an explanation;
not enough stock to make size M → 422 `insufficient-stock` before any job is created.

## REQ-AIB-07 — Deterministic cost, margin and cost ceiling — P0
Decimal PLN maths: line extensions rounded to 2 dp away from zero and summed; labor =
minutes/60 × hourly rate; overhead % on flowers + materials + labor. Any unknown price →
`total=null`, `isComplete=false`, `knownSubtotal` shown, no margin. Margin = (price − total) ÷ price
(gross margin). A cost ceiling requires every stock price and applies to the complete total.
A per-vendor cost profile holds one price basis, labor, overhead and remembered unit prices.
**Accept**: mockup recipe = 64.00 PLN, 46.67 % at 120 PLN; 30 min @ 40 + 10 % → 92.40; ceiling with a
missing price → 422 `cost-profile-incomplete`; recipe cost is an immutable snapshot.

## REQ-AIB-08 — Previews only on demand, cached, labelled — P1
A preview is requested per recipe (never automatically), runs as a billable job, and is cached:
asking again for a Ready preview costs nothing; a pending one cannot be double-submitted. Previews
never change the session state; a recipe is fully usable and saveable without one. Generated images
are copied to private storage (provider URLs expire) and labelled "AI inspiration".
**Accept**: preview states NotRequested/Queued/Running/Ready/Failed/OutcomeUnknown on the recipe;
repeat request on Ready → 200 with no new job.

## REQ-AIB-09 — Durable jobs, idempotency and no blind retries — P0
Jobs persist in PostgreSQL and are claimed with `FOR UPDATE SKIP LOCKED` + a renewable lease; no
DB transaction is held across provider calls. Billable starts require `Idempotency-Key`; same key +
payload returns the original job, a changed payload → 409 `idempotency-conflict`. Only failures known
to precede submission are retried (max 3 attempts). An ambiguous post-submission loss becomes
`OutcomeUnknown` with its spend kept reserved; a BFL task id is persisted so a crashed worker resumes
polling instead of paying twice. Completion re-checks lease, stock revision and deletion.
**Accept**: replayed key → same job; worker crash after submission never resubmits; stale result is discarded.

## REQ-AIB-10 — Quotas and global spend cap — P0
Per vendor per calendar month (UTC): 20 recognitions, 20 compositions, 40 previews; ≤ 2 active
previews; global reserved/actual spend ≤ $100. Checks and the job insert (= reservation) run under one
PostgreSQL advisory lock so concurrent requests cannot both take the last unit.
**Accept**: exhausted → 429 `quota-exhausted`/`spend-cap-reached`/`too-many-active` with `Retry-After`;
jobs that failed before reaching a provider don't count.

## REQ-AIB-11 — Tenant isolation — P0
Tenant identity comes only from `ITenantContextService.Current` (JWT claims); headers never override
it. Every read filters by VendorId **and** VendorType. Another vendor's session/photo/recipe/preview
is indistinguishable from a missing one.
**Accept**: cross-tenant GET of session, photo bytes, preview bytes, cost → 404; saved list and usage
contain only the caller's data.

## REQ-AIB-12 — Save, list and share recipes — P1
Recipes can be saved/unsaved; `GET /vendor/design-recipes?saved=true` lists saved recipes across
sessions. Sharing (text, optional preview) is a client action with the caption "AI inspiration;
actual flowers and stem counts may differ." No automatic publishing or saleable bouquet creation.
**Accept**: save works without a preview; saved recipes survive stock edits and session retention.

## REQ-AIB-13 — Retention and deletion — P1
Source photos are purged after 7 days, unsaved sessions after 30; saved recipes and their previews
stay until deleted. Deleting a session tombstones it, cancels queued work, and purges private media;
a running job's late result cannot reappear. Cleanup is retried until it succeeds.
**Accept**: deleted session → 404; queued jobs Cancelled; media keys gone.

## REQ-AIB-14 — Provider abstraction with free defaults — P1
Recognition (`Fake` | `OpenAI`), composition (`Deterministic` | `OpenAI`) and preview (`Placeholder` |
`Bfl` | `OpenAI`) are separate interfaces selected by configuration. Free defaults make the whole flow
work with no keys. Paid adapters use bounded requests, strict JSON schemas, treat text in photos as
data, never surface raw provider errors, and only fetch provider-owned URLs.
**Accept**: pilot runs with zero AI spend; switching provider is a config/secret change, no code change.

## REQ-AIB-15 — Mobile app screens and entry points — P0 (mobile EP-M10)
Capture, stock review, configure, recipe list, recipe detail (cost, margin, preview, save, share),
saved recipes and cost settings in the vendor app; entry from Home quick actions and the More menu;
no sixth tab. Tracked in the mobile repo.

## REQ-AIB-16 — Polling, resume and recoverable drafts — P0 (mobile EP-M10)
The app polls the session (2 s for 60 s, then 5 s; paused in background), shows "Still working" after
120 s, persists the session id and idempotency keys so a lost response or app restart resumes the
same job instead of paying again.

## REQ-AIB-17 — Deployment, cost and test documentation — P1
Document the Fly.io changes (always-on API machine for the in-process worker, private R2 bucket,
secrets), how to deploy/choose the image model (hosted BFL / OpenAI, optional self-hosted FLUX.2
klein on serverless GPU), expected costs, and the testing/validation flow.
**Accept**: `docs/deployment/AI-BOUQUET-ASSISTANT.md` exists and is linked from the deployment README.

## REQ-AIB-18 — Provider evaluation before paid rollout — P1 (open)
30–50 consented stock photos from 2–3 shops; compare recognition and previews (klein vs GPT Image)
with florist ratings; record cost/latency per accepted preview. Gate: all accepted recipes valid,
≥ 70 % rated useful, economy session ≤ $0.10 with two previews.

## REQ-AIB-19 — Pilot rollout — P2 (open)
Backend deployed with free providers → in-app testing by the team → paid providers for one test
vendor → 2–3 pilot shops → 10 shops after gates pass; alerts on old jobs, unknown outcomes and
near-cap spend.
