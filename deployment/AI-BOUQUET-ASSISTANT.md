# AI Florist — deployment, costs, testing & validation (EP-28 / EP-M10)

> Scope: what has to change in production to run the AI Florist as an **in-app pilot**,
> how to switch on (and later self-host) the image model, what it costs, and how to test and
> validate it. Design: `docs/superpowers/specs/2026-10-13-ai-bouquet-assistant-design.md`.
> Cost research: `docs/superpowers/plans/2026-10-13-ai-bouquet-assistant-costs.md`.
> Date: 2026-10-14. The vendor app is not live; this is a testing prototype.

## 1. What ships

| Piece | Where | Notes |
|---|---|---|
| API endpoints | `BouquetDesignController` — `/api/v1/vendor/design-*` | `VendorAccess`; contract in `docs/API-CONTRACTS.md` §21 |
| Migration | `20261014090556_AddBouquetDesignAssistant` | 3 new tables: `DesignSessions`, `DesignJobs`, `DesignCostProfiles`. Applied automatically on API start (as all migrations are) |
| Job worker | `DesignJobWorkerService`, **inside the API process** | Claims jobs from PostgreSQL, ≤ 2 concurrent provider calls, lease recovery every 15 s, retention sweep hourly |
| Private media | `IDesignMediaStore` → private R2 bucket in prod | Stock photos + previews; **never** on `img.findmyflowers.pl` |
| Providers | config `BouquetDesign:{Recognition,Composition,Preview}:Provider` | Defaults are **free**: `Fake` / `Deterministic` / `Placeholder` |
| Mobile app | vendor app EP-M10 | Home quick action + More → "AI Florist" |

```
Vendor app ──HTTPS──▶ Fly: flower-shop-backend-core (API + in-process design worker, always on)
                         │            │                     │
                         │            ▼                     ▼
                         │   Fly Postgres (DesignJobs   Cloudflare R2 private bucket
                         │   = queue + reservations)    flowershop-design-private
                         ▼
          OpenAI (recognition, recipes) · BFL FLUX.2 [klein] or OpenAI GPT Image (previews)
```

## 2. Fly.io resources

### 2.1 Required for the pilot: keep one API machine awake

The worker runs inside the API process. Today `fly.toml` uses `auto_stop_machines = 'suspend'` and
`min_machines_running = 0`, so the machine sleeps when there is no HTTP traffic — **a queued job would
then stall until the next request wakes the machine** (the app polls, so in practice it usually wakes,
but a vendor who closes the app mid-job leaves it frozen, and lease/retention sweeps stop).

Change `fly.toml` `[http_service]`:

```toml
  auto_stop_machines = 'off'      # was 'suspend' — the design-job worker lives in this process
  auto_start_machines = true
  min_machines_running = 1        # was 0
```

Side effect (positive): MQTT ingest, outbox dispatch, reservation expiry and payment reconciliation
also stop pausing.

| Resource | Change | Approx. monthly cost* |
|---|---|---|
| API machine `shared-cpu-1x`, 1 GB, `fra` | from scale-to-zero to always-on | **≈ $5–7** (full-month price of the machine you already have; check *Billing → Usage* for the real delta) |
| Memory | keep 1 GB. Photo processing is bounded (≤ 20 MP decode, one photo at a time per request, ≤ 2 concurrent jobs). Raise to 2 GB only if `fly logs` shows OOM kills | +≈ $5 if needed |
| Fly Postgres | no change; 3 small tables | $0 |
| GPU | **none** — Fly has deprecated GPUs; no local inference | $0 |

\* Fly prices change by region; treat as an estimate and read the invoice after the first month.

### 2.2 Later (not needed for the pilot): separate worker

When the API should scale to zero again, run the worker as its own always-on machine and disable it in
the API: the queue already supports several workers (`SKIP LOCKED` + leases).
Plan: a `FlowerShop.DesignWorker` host (follow-up T-28-022) deployed as a second Fly app or process
group with `BouquetDesign__Worker__Enabled=true`, while the API sets `BouquetDesign__Worker__Enabled=false`.
Cost: one more `shared-cpu-1x` 512 MB–1 GB machine (≈ $3–6/month).

### 2.3 Cloudflare R2 — private bucket

**Manual prerequisite:** Fly deployments and database migrations do not create this bucket or its
credentials. Create it in Cloudflare before testing stock-photo uploads. Stock photos and AI Florist
media use this private store; deploying the application alone is not enough.

1. Cloudflare → R2 → **Create bucket** `flowershop-design-private`, location EU (same account as
   `flowershop-bouquets`). **Do not** connect a custom domain and leave *Public access* disabled.
2. R2 → *Manage API tokens* → **Create token**: permission *Object Read & Write*, scoped to **only**
   `flowershop-design-private`. Copy the Access Key ID / Secret.
3. `appsettings.Production.json` already points `BouquetDesign:Storage` at this bucket and the account
   endpoint; only the keys are secrets.

Set `BouquetDesign__Storage__AccessKeyId` and `BouquetDesign__Storage__SecretAccessKey` on the Fly app
`flower-shop-backend-core` as shown below. Keep this bucket private; do not substitute the public
bouquet bucket or attach a public domain.

Verify setup through **Stock → item → Photos & appearance → Add photo**. Reload the item and check
that the photo still displays. Uploads do not require an OpenAI key; OpenAI recognition and generation
require separate provider configuration. On 2026-10-18, stock uploads returned HTTP 500 before the
bucket and credentials were provisioned, then persisted and displayed successfully after setup.

Cost: R2 free tier covers 10 GB storage + 1 M writes + 10 M reads/month. A pilot stores ~5 × 300 KB
photos per session (purged after 7 days) plus ~150 KB per preview — well inside the free tier.

### 2.4 Secrets and config

```bash
APP=flower-shop-backend-core

# Required — private media
fly secrets set --app $APP \
  BouquetDesign__Storage__AccessKeyId='<r2 key id>' \
  BouquetDesign__Storage__SecretAccessKey='<r2 secret>'

# Optional — paid providers (leave unset to stay on the free providers)
fly secrets set --app $APP \
  BouquetDesign__OpenAI__ApiKey='sk-…' \
  BouquetDesign__Recognition__Provider=OpenAI \
  BouquetDesign__Analysis__Provider=OpenAI \
  BouquetDesign__Composition__Provider=OpenAI

fly secrets set --app $APP \
  BouquetDesign__Bfl__ApiKey='<bfl key>' \
  BouquetDesign__Preview__Provider=Bfl

# Pilot controls (non-secret; may also live in [env] of fly.toml)
fly secrets set --app $APP \
  BouquetDesign__PilotVendorIds__0='<vendor guid>' \
  BouquetDesign__Limits__GlobalMonthlyUsdCap=100
```

Every `fly secrets set` restarts the machine; batch them. Add the new keys to
`SECRETS-AND-ENV-VARS.md` when you set them.

| Key | Purpose | Default |
|---|---|---|
| `BouquetDesign__Enabled` | Kill switch for new sessions/paid work | `true` |
| `BouquetDesign__PreviewsEnabled` | Previews only | `true` |
| `BouquetDesign__PilotVendorIds__N` | Allow-list (empty = everyone) | empty |
| `BouquetDesign__Recognition__Provider` | `Fake` \| `OpenAI` | `Fake` |
| `BouquetDesign__Composition__Provider` | `Deterministic` \| `OpenAI` | `Deterministic` |
| `BouquetDesign__Preview__Provider` | `Placeholder` \| `Bfl` \| `OpenAI` | `Placeholder` |
| `BouquetDesign__{Recognition,Composition,Preview}__Model` | Model id | `gpt-4.1-mini`, `gpt-4.1-mini`, `flux-2-klein-4b` |
| `BouquetDesign__{…}__EstimatedUsd` | Max $ reserved per call | 0.010 / 0.005 / 0.020 |
| `BouquetDesign__Limits__*` | `RecognitionsPerMonth` 20, `CompositionsPerMonth` 20, `PreviewsPerMonth` 40, `MaxActivePreviewsPerVendor` 2, `GlobalMonthlyUsdCap` 100 | |
| `BouquetDesign__Bfl__BaseUrl` | `https://api.eu.bfl.ai/v1` (EU data residency) | |
| `BouquetDesign__Worker__Enabled` | Run the worker in this process | `true` |
| `BouquetDesign__{Recognition,Composition,Preview}__PromptVersion` | Active prompt version (EP-33): `inventory-v1\|v2`, `recipe-v1\|v2`, `preview-v1\|v2`; empty = newest | `*-v1` (appsettings.json) |
| `BouquetDesign__{…}__CandidatePromptVersion` | Candidate version for a share of vendors | `*-v2` |
| `BouquetDesign__{…}__CandidatePercent` | 0–100 % of vendors (sticky hash of the vendor id) on the candidate; 0 = everyone on the active version | `0` |
| `BouquetDesign__Analysis__Provider` | "Reconstruct a bouquet from photos" (EP-35): `Palette` (free, colours only — always the same placeholder items, flagged with the `colours-only` warning; not for production) \| `OpenAI` (vision, `bouquet-analysis-v1`) | `Palette` |
| `BouquetDesign__Analysis__Model` / `__EstimatedUsd` | Analysis model and $ reserved per call (counts as one recognition of the vendor's quota) | `gpt-4.1-mini` / 0.010 |
| `BouquetDesign__Analysis__ModelAssistedMatching` | Let the model pick among the top 10 code-ranked stock candidates (the pick is re-scored in code; Low is never linked) | `false` |
| `BouquetDesign__Analysis__UseInspirationAsPreviewReference` | Send the bouquet photo to the image provider as a preview reference (third-party image — check rights before switching on) | `false` |
| `BouquetDesign__Assessment__MonthlyLimit` | Cost assessor (EP-36): billable assessments per vendor per month; the analysis uses the `Analysis` provider and counts in the global $ cap | `30` |
| `BouquetDesign__Composition__RepairEnabled` | One validator-guided repair call per composition (free for the vendor, counted in the global $ cap) | `true` |

> **Model ids are configuration, not code.** Before switching a provider on, confirm the id is
> current (`gpt-4.1-mini` was the plan's priced baseline; GPT Image models are named like
> `gpt-image-1`/`gpt-image-2`). If you use GPT Image for previews set
> `BouquetDesign__Preview__Model` and `__EstimatedUsd=0.075`.

### 2.5 Deploy

```bash
# 1. fly.toml change from §2.1 committed
# 2. secrets from §2.4 set
fly deploy --config fly.toml --app flower-shop-backend-core
fly logs --app flower-shop-backend-core | grep -i "design job worker started"
#    → "Design job worker started (concurrency 2, recognition=Fake, composition=Deterministic, preview=Placeholder)"
```

Then run the smoke test in §5.3.

## 3. Image generation model

### 3.1 Recommended now: hosted FLUX.2 [klein] 4B via the BFL API (no deployment)

Nothing to host. The `Bfl` adapter submits `POST {BaseUrl}/flux-2-klein-4b` with the recipe prompt and
up to `MaxReferenceImages` (default 2, klein accepts 4) resized stock photos, persists the returned
`polling_url` as the job's provider task id, polls it until `Ready`, and copies the result (BFL
result URLs expire within minutes) into the private bucket. Because the task id is stored, a worker
crash or redeploy resumes polling instead of paying for a second render.

1. Create an account on bfl.ai, buy credits (1 credit = $0.01; ~$10 is plenty for the pilot) and
   create an API key.
2. `fly secrets set BouquetDesign__Bfl__ApiKey=… BouquetDesign__Preview__Provider=Bfl`.
3. Check the request/response field names once against BFL's current API reference
   (`https://docs.bfl.ai`, OpenAPI at `https://api.bfl.ai/openapi.json`). The adapter was written to the
   documented shape (`prompt`, `input_image`, `input_image_2…`, `width`, `height`, `output_format`;
   response `id`, `polling_url`; poll `status` `Ready|Pending|…`, `result.sample`) and is covered by
   mock-HTTP tests only — this build environment could not reach BFL.
4. Generate one preview from the app on a test vendor and check `fly logs` for `BFL returned …`
   warnings.

Price: ≈ $0.014 per 1 MP text-to-image, slightly more with reference images → budgeted at $0.020.

### 3.2 Alternative: OpenAI GPT Image

`BouquetDesign__Preview__Provider=OpenAI`, model via `BouquetDesign__Preview__Model`, size
`BouquetDesign__OpenAI__ImageSize` (1024x1024), quality `…__ImageQuality` (`medium`). Synchronous —
there is no recovery handle, so a response lost mid-flight is recorded as `OutcomeUnknown` and never
re-sent automatically. Medium quality ≈ $0.053 output + input → budgeted at $0.075.

Set the model explicitly (`BouquetDesign__Preview__Model=gpt-image-1`): the default `flux-2-klein-4b`
is a BFL model and OpenAI rejects it. While previews are enabled, the API **refuses to start** when
`Preview__Provider=OpenAI` has a non-`gpt-image-*` model or no `OpenAI__ApiKey`, or when
`Preview__Provider=Bfl` has a non-`flux-*` model or no `Bfl__ApiKey` (`PreviewConfigValidator`).
A request the provider rejects (4xx, or 429/5xx after the retries) is not billed, so it uses no
preview allowance and books $0 against the global cap.

### 3.3 Optional experiment: self-hosting FLUX.2 [klein] 4B on a serverless GPU

Only worth it once there is measured demand (break-even analysis in the cost report: thousands of
previews/month). Fly has no GPUs, so this lives outside Fly; keep the API on Fly.

1. **Platform**: Runpod Serverless (or Modal/Replicate) with a 24 GB GPU (A5000/4090/L4 class). klein
   4B needs ~13 GB VRAM. Use the **4B** weights — Apache-2.0; the 9B variant has a different licence.
2. **Container**: Python 3.11, `torch` + CUDA, `diffusers` (version with FLUX.2 klein support — pin it),
   weights `black-forest-labs/FLUX.2-klein-4B` baked into the image or on a network volume (avoids a
   multi-GB download per cold start). Handler: accept `{prompt, images[base64], width, height, seed}`,
   run the pipeline (≈ 4 steps for klein), return a JPEG base64.
3. **Endpoint security**: Runpod endpoint API key as a Fly secret; never expose the endpoint publicly.
4. **App integration**: add a `SelfHosted` `IPreviewRenderer` (≈ 1–2 days; follow-up) that calls the
   endpoint's async `run` + `status` API and stores the job id as `ProviderTaskId`, exactly like the BFL
   adapter, so crash recovery keeps working. No mobile change.
5. **Measure before switching**: cold-start seconds, warm seconds per image, billed seconds per
   *accepted* preview. At $0.69/h serverless: 10 billed s ≈ $0.002/image, 60 s ≈ $0.012/image.
   Idle-scaling to zero is mandatory — an always-on A5000 is ≈ $197/month.

## 4. Expected costs

AI usage is bounded by the in-app limits (per vendor per month: 20 recognitions, 20 recipe sets,
40 previews; global cap $100). Planning allowances (from the cost report): recognition + composition
≈ $0.015/session, economy preview (klein) $0.020, quality preview (GPT Image medium) $0.075, +20 %
buffer.

| Phase | Fixed infra delta | AI usage | Monthly total |
|---|---|---|---|
| **Prototype testing now** (free providers) | ≈ $5–7 (API always on) + R2 $0 | **$0** | **≈ $5–7** |
| Provider evaluation (one-off, 2–4 days) | — | $25–50 one-off (≈ 300 klein + 300 GPT previews + 50 sessions) | — |
| Pilot, 10 shops, 20 sessions × 2 previews each | ≈ $5–7 | $13 (klein) – $40 (GPT medium) | **≈ $20–50** |
| 50 shops | ≈ $10–15 (consider separate worker) | $66 – $198 | ≈ $80–215 |
| 100 shops | ≈ $15–25 | $132 – $396 | ≈ $150–420 |

Hard ceiling: the worst a single vendor can cost per month is 20 × $0.010 + 20 × $0.005 + 40 × $0.020
= **$1.10 (klein)** or 40 × $0.075 + $0.30 = **$3.30 (GPT)** before the 20 % buffer; the global cap
($100) stops all new paid work platform-wide. Set a provider-side budget alert too (OpenAI usage
limit, BFL credit balance) — the app's cap is the primary gate, the dashboards are a backstop.

## 5. Testing and validation flow

### 5.1 Automated tests (every change)

```bash
# Backend
dotnet test tests/FlowerShop.Domain.Tests --filter FullyQualifiedName~BouquetDesign   # 31 domain tests
dotnet test tests/FlowerShop.Tests.Unit   --filter FullyQualifiedName~BouquetDesign   # 32 flow + provider tests
# Mobile (sibling repo)
dotnet test tests/FlowerShop.VendorEmployee.Tests/ -p:TestsOnly=true --filter FullyQualifiedName~BouquetDesign
```

Covered: stock validation (duplicate lines, unknown ids, stem bounds), cost maths (64.00 PLN /
46.67 %, incomplete costs, labor+overhead 92.40, rounding), state machine, idempotent replays and
409 on changed payloads, cross-tenant 404s, quotas / $ cap / kill switch, preview caching and
double-tap, crash recovery (unsubmitted → retried; submitted → `OutcomeUnknown`, never resent),
stale stock discard, deletion, image processing, BFL/OpenAI adapters on mock HTTP.

### 5.2 Local end-to-end (free providers, no keys)

```bash
docker-compose -f docker/docker-compose.dev.yml up -d postgres      # or any local PostgreSQL 15+
cd src/backend/FlowerShop.API
ASPNETCORE_ENVIRONMENT=Development Authentication__UseKeycloak=false ConnectionStrings__Redis= \
  dotnet run --no-launch-profile --urls http://localhost:8080
```

```bash
TOKEN=$(curl -s -X POST localhost:8080/api/v1/auth/dev-token -H 'Content-Type: application/json' \
  -d '{"username":"v@x.pl","vendorId":"11111111-1111-1111-1111-111111111111","vendorType":"TraditionalShop","role":"VendorAdmin"}' | jq -r .token)
H="Authorization: Bearer $TOKEN"
S=$(curl -s -X POST localhost:8080/api/v1/vendor/design-sessions -H "$H" -H 'Content-Type: application/json' -d '{"inputMode":"Photos"}')
SID=$(jq -r .id <<<"$S")
curl -s -X POST localhost:8080/api/v1/vendor/design-sessions/$SID/photos -H "$H" -F photo=@stock.jpg -F clientPhotoId=p1 | jq '.version,.photos'
curl -s -X POST localhost:8080/api/v1/vendor/design-sessions/$SID/recognize -H "$H" -H 'Idempotency-Key: r1' \
  -H 'Content-Type: application/json' -d '{"expectedVersion":2}' | jq .state            # Recognizing
sleep 3; curl -s localhost:8080/api/v1/vendor/design-sessions/$SID -H "$H" | jq '.state,.inventory'
# … PUT /inventory (confirmed:true, add unitPrice), POST /recipes (Idempotency-Key), poll,
#     POST /recipes/{rid}/preview, poll, GET preview/content > preview.jpg
```

The mobile app's Debug build talks to `http://10.0.2.2:8080` from the Android emulator, so the same
local API serves the app.

What was verified this way on 2026-10-14 against PostgreSQL 16: migration applied; create (201) →
photo upload re-encoded to 1600 px → recognize (202, 400 without `Idempotency-Key`) → estimates →
stale `expectedVersion` 409 → confirm → cost ceiling with a missing price 422 → compose (202) →
3 recipes with incomplete cost and no margin → preview (202) → Ready, JPEG bytes → save → saved list
→ cost profile GET/PUT → usage counters → delete (204) → 404; another vendor got 404 for session and
photo bytes.

### 5.3 Production smoke (after deploy, free providers)

1. `GET https://api.findmyflowers.pl/health/ready` → 200; logs show *Design job worker started*.
2. Log into a **Release** build of the vendor app (it targets `api.findmyflowers.pl`) as a test vendor.
3. Home → *AI Florist* → take 1–2 photos → *Review stock*: estimates appear with the
   "Counts are estimates" banner (Fake recogniser = roses / lisianthus / eucalyptus / wrap / ribbon).
4. Edit a count, add prices for some lines, *Confirm stock* → *Create 3 recipes* → three recipes
   appear within ~5 s. One missing price ⇒ "Known cost … · 1 price missing", no margin.
5. *Generate preview* on one recipe → placeholder image labelled "AI inspiration" within ~5 s.
6. Save, open *Saved recipes*, Share (text). Kill and reopen the app mid-composition — it resumes.
7. Check the database:

```sql
SELECT "Kind", "State", count(*), sum(coalesce("ActualUsd","ReservedUsd")) AS usd
FROM "DesignJobs" WHERE "CreatedAt" >= date_trunc('month', now()) GROUP BY 1,2;
-- stuck work (should be empty)
SELECT "Id","Kind","State","LeaseExpiresAt" FROM "DesignJobs"
WHERE ("State"='Queued' AND "CreatedAt" < now() - interval '5 minutes')
   OR ("State"='Running' AND "LeaseExpiresAt" < now() - interval '2 minutes');
```

8. Confirm the R2 objects exist under `design/{vendorId}/…` in `flowershop-design-private` and that
   the bucket has no public URL.

### 5.4 Provider and prompt evaluation — Admin Portal "AI quality" (EP-33)

The manual evaluation CSV is replaced by the Admin Portal dashboard **AI quality**
(`/Analytics/AiFlorist`, API `GET /api/v1/admin/analytics/ai-florist`). It shows, per step × provider ×
model × prompt version and date range: jobs, failure rate, p50/p95 latency and $; for recipes the 👍 rate,
saved / used-stock / refine rates, validator rejection and repair success; for recognition the
**corrected-line rate**; for previews "looks right" and re-render rates; plus a daily trend. Numbers come
from vendors' own ratings and actions in the wizard (`DesignFeedback`, `DesignSignals`,
`RecognitionCorrections`) — aggregates only, no shop names or comments.

1. Restrict to test vendors with `BouquetDesign__PilotVendorIds__N` while comparing providers.
2. Switch the provider under test on (`Recognition=OpenAI`, `Composition=OpenAI`, `Preview=Bfl|OpenAI`)
   and have florists use the wizard normally: rate recipes 👍/👎, mark previews "Looks right / Not right",
   confirm or correct recognised stock.
3. Read the dashboard for the test period. **Gates**: 100 % of persisted recipes stock-valid (guaranteed
   by code); recognition corrected lines **< 30 %** (shown green/red); median session-to-recipes < 45 s;
   preview p95 < 90 s; 👍 rate ≥ 70 %; economy session ≤ $0.10 with two previews. Choose the renderer
   with the lowest cost per "looks right" preview. If none passes, ship recipes with
   `BouquetDesign__PreviewsEnabled=false`.
4. For prompt-design work, export consented feedback (**Export consented feedback (CSV)** on the same
   page): only vendors who switched on "Help improve the AI Florist" on the House style page are
   included; comments are scrubbed of contact data and shop names; every export is logged
   (`AUDIT ai-florist-feedback-export`).

### 5.4.1 Prompt versions and candidate rollout (EP-33)

Prompts live in `Infrastructure/BouquetDesign/Providers/PromptCatalog.cs`, keyed `{step}-v{n}`; v1 stays
reproducible. `recipe-v2` adds house style, brief, shop history, refine and the "why" sentence;
`inventory-v2` adds the shop's known corrections and count bias; `preview-v2` adds corrections and the
house colour mood. Only providers that read the catalogue (OpenAI, BFL) use it; the free providers keep
their own fixed versions.

1. **Migrate**: deploy the API (migration `AddDesignFeedbackLoop`). Ratings start accumulating on v1.
2. **Candidate at 0 %**: `appsettings.json` ships `PromptVersion=*-v1`, `CandidatePromptVersion=*-v2`,
   `CandidatePercent=0` — behaviour equals v1.
3. **Pilot**: raise the share for one step at a time, e.g.
   `fly secrets set BouquetDesign__Composition__CandidatePercent=50` (combine with `PilotVendorIds` to
   limit who runs paid providers). A vendor always sees the same version.
4. **Promote** when the dashboard shows the candidate's 👍 rate higher and its validator rejection
   (recipes) / corrected-line rate (recognition) / re-render rate (previews) lower than v1:
   `BouquetDesign__Composition__PromptVersion=recipe-v2` and `CandidatePercent=0`. Promotion is always an
   operator change — never automatic.
5. **Roll back** the same way (set `PromptVersion` back to v1). Repair can be switched off on its own with
   `BouquetDesign__Composition__RepairEnabled=false`; the kill switch still stops all paid work.

### 5.4.2 Flower atlas, item crops and image matching (EP-41)

Nothing changes for vendors until each step below is done; every step can be undone with config alone.

1. **Migrate**: deploy the API (migrations `AddFlowerAtlas`, `AddImageEmbeddings`). The second one runs
   `CREATE EXTENSION IF NOT EXISTS vector` and an HNSW index inside a guarded block: on a server without pgvector it
   only logs a notice and the code ranks vectors itself, so it never blocks the deploy. Check with
   `SELECT extversion FROM pg_extension WHERE extname = 'vector';` (Fly Postgres images ship pgvector; the database user
   needs the right to create extensions — if it lacks it, run the statement once as the superuser and redeploy).
2. **Atlas**: Admin Portal → **Flower atlas** → *Add seed list entries* (≈150 entries from `docs/atlas/atlas-seed.json`),
   then *Import candidate images* repeatedly (10 entries per run, 2 candidates per source from Wikimedia Commons and
   iNaturalist, CC0/CC-BY/CC-BY-SA only; NC/ND are skipped and logged). Images go to the **public** bucket under
   `atlas/` (`img.findmyflowers.pl/atlas/...`) as *Unreviewed* — open each entry, check the photo, licence and credit,
   and *Approve* (the first approved image becomes the main one). Fill variety gaps (e.g. carnation "Cappuccino") with
   our own photos or pilot florists' photos given with written permission (*Upload a photo*, source Own/Licensed).
   Approved images appear as "Looks like" at once (the read model refreshes within 10 minutes on other machines).
3. **Crops**: raise the `bouquet-analysis-v2` candidate for the pilot — it is the v1 prompt plus item boxes:
   `fly secrets set BouquetDesign__Analysis__CandidatePercent=100` together with `PilotVendorIds`, or a share. Only the
   OpenAI analyzer uses it (the free Palette analyzer has no boxes). Check crop quality on real bouquets before
   promoting with `BouquetDesign__Analysis__PromptVersion=bouquet-analysis-v2`.
4. **Image fingerprints (Phase 2, off by default)**. In-process ONNX (REQ-GAPS-S H.3):
   - get the **image encoder** of CLIP ViT-B/32 as ONNX (e.g. `Xenova/clip-vit-base-patch32` → `onnx/vision_model.onnx`,
     ≈ 350 MB fp32 / ≈ 90 MB quantised) and put it on a Fly volume mounted on the API machine, e.g. `/data/models`;
   - `fly secrets set BouquetDesign__Embedding__Provider=Onnx BouquetDesign__Embedding__Onnx__ModelPath=/data/models/clip-vit-b32-vision.onnx
     BouquetDesign__Embedding__Model=clip-vit-b32 BouquetDesign__Embedding__Dimensions=512`
     (SigLIP-base: `Dimensions=768`, `Onnx__Mean__0`…`__2=0.5`, `Onnx__Std__0`…`__2=0.5`, a new `Model` id — vectors of
     different models are never compared, and the HNSW index covers 512 dimensions; other sizes use exact search);
   - expect ≈ 50–150 ms CPU per crop and ≈ 300 MB more memory on the API machine that also runs the design worker —
     size it up one step;
   - Admin Portal → **Flower atlas** → *Backfill image fingerprints* until it reports 0 (atlas images and existing stock
     photos; new stock photos and approved images are fingerprinted as they come);
   - alternatively `Provider=Http` with `Http__Endpoint` (a service answering `POST /embed`, §26 of the API contract), e.g.
     the Phase 3 detector service.
   Switch on for the pilot and compare match corrections and verdicts before and after. `StockWeight`, `DisagreeMargin`,
   `MinSimilarity` and `StockSuggestSimilarity` tune the signals; `LearnFromVerdicts=true` turns checked crops into the
   shop's own examples (vectors only, never shared).
5. **Roll back**: `BouquetDesign__Embedding__Provider=None` (signals stop at once; stored vectors are ignored),
   `BouquetDesign__Analysis__CandidatePercent=0` (no new crops). Rejecting an atlas image removes it from the review
   immediately.

### 5.5 Failure-mode checks (once per environment)

| Scenario | How | Expected |
|---|---|---|
| Lost response / double tap | Re-send a POST with the same `Idempotency-Key` | Same job, no second charge |
| Changed payload, same key | Different body, same key | 409 `idempotency-conflict` (or the cached original response) |
| Worker restart mid-job | `fly machine restart` while a preview is Running | BFL: polling resumes; GPT Image: `OutcomeUnknown`, no resend |
| Quota | Lower `Limits__PreviewsPerMonth` to 1 | 2nd preview → 429 with `Retry-After`; app shows reset date |
| Kill switch | `BouquetDesign__Enabled=false` | New work 403; saved recipes still open |
| Cross-tenant | Second vendor token, first vendor's session id | 404 on session, photo and preview bytes |
| Retention | Set `Retention__SourcePhotoDays=0`, wait for the hourly sweep (or restart) | Photos show "Expired"; recipes untouched |
| Crop isolation (EP-41) | Second vendor token, first vendor's `media/crops/{itemId}` | 404 |
| Unreviewed atlas image | `GET /api/v1/atlas/images/{id}/content` of an unreviewed image, no token | 404 |

### 5.6 Monitoring during the pilot

- `fly logs | grep -E "Design job|BFL returned|OpenAI returned"` — failures, provider status codes.
- Weekly: the spend query in §5.3 vs. the provider invoices; replace the `EstimatedUsd` allowances with
  measured p95 costs.
- Alert (manual for now, follow-up T-28-021): any `OutcomeUnknown`, jobs queued > 5 min, monthly spend
  > 80 % of the cap.

## 6. Rollback

- **Stop paid work**: `fly secrets set BouquetDesign__Enabled=false` (or `PreviewsEnabled=false`).
  Existing sessions and saved recipes stay readable.
- **Revert to free providers**: unset the `*__Provider` secrets.
- **Code rollback**: redeploy the previous image. The three new tables are unused by older code and can
  stay; do **not** drop them (saved recipes live there). The EP-33 tables (`DesignFeedback`,
  `DesignSignals`, `DesignStyleProfiles`, `RecognitionCorrections`) and new job/session columns are
  likewise ignored by older code.
- **Prompt rollback**: set `BouquetDesign__{Step}__PromptVersion` to the previous version (§5.4.1).
- **Return to scale-to-zero**: revert §2.1 only after the worker moved to its own machine (§2.2).
