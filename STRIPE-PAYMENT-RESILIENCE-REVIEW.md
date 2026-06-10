# Stripe Payment Integration — Resilience Review

> **Date**: 2026-06-10
> **Scope**: Payment processing flow (order creation → PaymentIntent → confirmation → cancellation/refund)
> **Reference**: [Design a Payment System — systemdesign.one](https://newsletter.systemdesign.one/p/design-a-payment-system)
> **Goal**: Make the payment flow as resilient as possible. This is the most critical part of the platform — a single error can cause money to vanish from a customer's account.

---

## Files reviewed

| Concern | File |
|---------|------|
| Stripe SDK calls | `src/backend/FlowerShop.Infrastructure/Payments/StripeService.cs` |
| Stripe abstraction | `src/backend/FlowerShop.Application/Services/IStripeService.cs` |
| Webhook receiver | `src/backend/FlowerShop.API/Controllers/StripeWebhookController.cs` |
| Order command handlers | `src/backend/FlowerShop.Application/Handlers/Order/OrderCommandHandlers.cs` |
| Order aggregate | `src/backend/FlowerShop.Domain/Aggregates/Order/Order.cs` |
| Orders API | `src/backend/FlowerShop.API/Controllers/OrdersController.cs` |
| Outbox | `src/backend/FlowerShop.Infrastructure/Outbox/{OutboxDispatcherService,EfOutboxWriter,OutboxMessage}.cs` |
| Idempotency middleware | `src/backend/FlowerShop.API/Middleware/IdempotencyMiddleware.cs` |
| Reservation expiry | `src/backend/FlowerShop.Infrastructure/BackgroundServices/OrderReservationExpiryService.cs` |
| Order repository | `src/backend/FlowerShop.Infrastructure/Persistence/Repositories/OrderRepository.cs` |
| DI registration | `src/backend/FlowerShop.Infrastructure/Extensions/ServiceCollectionExtensions.cs` |

---

## How the current flow works (as built)

```
POST /orders                → CreateOrderCommandHandler
  reserve bouquets (SaveChanges per bouquet)
  add OrderCreatedEvent to outbox + AddAsync(order)  [SaveChanges — order+outbox atomic]
  Stripe.CreatePaymentIntent  → SetPaymentIntent + UpdateAsync   [separate SaveChanges]
  return client_secret

client confirms card with Stripe.js (PAN never hits our server ✓)

Two confirmation paths, both → ConfirmPaymentCommandHandler → order.ConfirmPayment():
  (a) POST /orders/{id}/confirm-payment   (client-driven)
  (b) Stripe webhook payment_intent.succeeded → StripeWebhookController

Cancel → CancelOrderCommandHandler → refund if wasPaid
Background: OrderReservationExpiryService cancels stale Created orders after 15 min
```

---

## What is already correct ✓

- **PCI surface is correct** — `client_secret` + Stripe.js/Elements; card data never touches our servers.
- **Webhook signature verification** — `EventUtility.ConstructEvent` with HMAC secret. Present and correct.
- **Server-authoritative amounts** — the PaymentIntent amount is derived from `order.TotalAmount`, and `Order.Create` validates `ExpectedTotalAmount` against the recomputed total. Clients cannot tamper with the charge amount.
- **State-machine guards** — `ConfirmPayment`/`Cancel` enforce valid transitions (`Status == Created`), giving *some* webhook idempotency for free.
- **Transactional outbox exists** — `OrderCreatedEvent`/`OrderPaidEvent` are written in the same `SaveChanges` as the aggregate, dispatched by a polling worker with `FOR UPDATE SKIP LOCKED`. This is the right backbone.
- **Reservation-expiry worker** — the article's "stuck payment detector," at least for the unpaid case.

---

## 🔴 CRITICAL — money-loss paths (fix before any real traffic)

### C1. Charged customer can have their order silently cancelled with **no refund**

This is the single worst path and it is fully reachable today:

1. Customer pays → Stripe charges the card → `payment_intent.succeeded` fires.
2. Webhook processing throws (DB blip, MQTT, deserialization, deploy) — the controller **catches the exception, logs, and returns `200 OK`** (`StripeWebhookController.cs:77-83`). Stripe sees success and **never retries**.
3. The order stays in `Created`. 15 minutes later `OrderReservationExpiryService` calls `order.CancelDueToReservationExpiry()` — which **does not refund** (`Order.cs:210`). The refund logic lives only in `CancelOrderCommandHandler`, which the expiry worker bypasses entirely.

**Result: card charged, order cancelled, bouquet released, no refund, no alert.** Money vanishes exactly as the article warns.

**Fixes (all three):**
- Webhook must **not** swallow processing errors — return non-2xx so Stripe retries (see C2).
- The expiry/auto-cancel path must verify with Stripe before cancelling, and refund if a charge succeeded. Never auto-cancel a `Created` order without confirming the PaymentIntent status at the PSP.
- Reconciliation job (see H1) as the safety net.

### C2. Webhook returns `200` on failure → at-least-once delivery is defeated

`catch (Exception) { log; } return Ok();` means a transient failure is reported to Stripe as handled. The article's core mechanism — at-least-once delivery + idempotent processing — only works if you return an error on failure so Stripe re-delivers. Today, **one failed webhook = permanently lost state transition**.

**Fix:** persist the raw event first, then process. Return `200` only after the event is durably stored (or successfully processed). On processing failure return `500`. Distinguish "already processed" (→200) from "transiently failed" (→500).

### C3. No idempotency key on Stripe `PaymentIntent.Create` or `Refund`

`StripeService` attaches **no `IdempotencyKey`** to either call (`StripeService.cs:48-49`, `105-106`). The article requires an idempotency key on *every* Stripe POST.

- `CreateOrderCommandHandler` makes the Stripe call **after** several `SaveChanges`. If the request is retried after a mid-flight timeout, you create a **second PaymentIntent** for the same order. The `CreateOrderCommand.IdempotencyKey` field exists but **is never read in the handler** — it's dead code. The only protection is `IdempotencyMiddleware`, which caches the response *only after a 2xx completes* — so it does nothing for the exact timeout/crash window that idempotency keys are designed for.
- `RefundPaymentAsync` with no key → a retried `CancelOrderCommand` can issue **two refunds** (merchant loses money).

**Fix:** pass `RequestOptions { IdempotencyKey = ... }` to both calls. Use a deterministic key — e.g. `order_id` for the intent, `order_id:refund` for the refund (or the inbound webhook/event id). Highest-leverage, lowest-effort fix here.

### C4. Refund result is never checked

`CancelOrderCommandHandler` logs `"Refund issued"` regardless of `refund.Status` (`OrderCommandHandlers.cs:499-502`). Stripe refunds can be `pending` or `failed`. You will report success on a failed refund and the customer never gets their money.

**Fix:** branch on `refund.Status`; on `failed`/unexpected, do not mark the order refunded — flag for manual/automated retry and alert.

---

## 🟠 HIGH — correctness & resilience gaps

### H1. No reconciliation worker

The article's daily internal-vs-Stripe reconciliation is absent. This is the last line of defense against C1/C2 and against any drift. Without it, a single dropped webhook is invisible forever.

**Fix:** a scheduled job that, for every order in a non-terminal paid-ambiguous state (`Created` with a PaymentIntentId, recently cancelled, refund-pending), queries Stripe and converges local state — confirming paid orders, refunding orphaned charges, retrying failed refunds.

### H2. Webhook does heavy synchronous business logic

The receiver runs the full `ConfirmPaymentCommand` inline — DB writes per bouquet, vase updates, fire-and-forget MQTT — under Stripe's ~20s timeout. The article is explicit: webhook receiver should **verify → store raw event → 200**, and process asynchronously. Heavy inline work risks timeouts (→ duplicate deliveries) and partial application.

**Fix:** receiver persists the event and enqueues; a worker applies the transition idempotently. The outbox/worker pattern already exists — reuse it for *inbound* events.

### H3. No persisted record of processed webhook events

Idempotency rests entirely on the `Status == Created` guard. That breaks for any event that should be processed when the order is *not* `Created` (refunds, disputes, async BLIK state changes) and gives no audit trail. The article calls for an append-only event log keyed by Stripe event id.

**Fix:** a `processed_stripe_events(event_id PK, type, received_at, payload)` table. Insert-or-skip on event id = true idempotency independent of aggregate state, plus the audit log the article wants.

### H4. BLIK is asynchronous — the flow assumes synchronous card semantics

`blik` is enabled for PLN (`StripeService.cs:38-40`). BLIK confirmation is async: `payment_intent.processing` arrives first, then `succeeded`/`failed` possibly seconds later. The client `confirm-payment` call can return before funds are confirmed, and only `succeeded`/`failed` are handled — not `processing`. Combined with the 15-min expiry, a slow-but-valid BLIK payment can be cancelled mid-flight.

**Fix:** treat the webhook as the source of truth for BLIK (don't rely on the client confirm path), handle `payment_intent.processing`, and don't auto-cancel orders with an in-flight intent.

### H5. Concurrency race between the two confirmation paths

Client `confirm-payment` and the webhook can run concurrently. Both read `Status == Created`, both proceed (TOCTOU), both sell bouquets / emit `OrderPaidEvent`. `AggregateRoot.IncrementVersion()` is called, but no mapped EF concurrency token on `Order` was found — **verify** a `[ConcurrencyCheck]`/`xmin` rowversion is configured. Without it, the guard is not race-safe.

**Fix:** map an optimistic-concurrency token on `Order` so the second writer fails and is retried/ignored.

### H6. ConfirmPayment is not atomic

The handler issues many independent `SaveChanges` (each `bouquet.UpdateAsync`, then outbox, then `order.UpdateAsync`). A crash mid-loop leaves some bouquets `Sold` but the order **not** `Paid` and no `OrderPaidEvent`. The article's "atomic phases" principle is violated.

**Fix:** wrap the state transition + bouquet updates + outbox write in a single explicit transaction (`BeginTransactionAsync`/one `SaveChanges`), so the paid transition is all-or-nothing. The fire-and-forget MQTT should be driven off the `OrderPaidEvent` consumer, not done inline.

---

## 🟡 MEDIUM — hardening

- **M1. Money rounding:** `(long)(amount * 100)` truncates (`StripeService.cs:35,102`). Use `(long)Math.Round(amount * 100m, MidpointRounding.AwayFromZero)`. Off-by-one grosze is still a money bug.
- **M2. No amount/currency check on confirm:** the webhook trusts `succeeded` without asserting `paymentIntent.Amount`/`Currency` match the order. Add a defensive equality check before confirming.
- **M3. No timeout / retry / circuit breaker on Stripe calls** — the article explicitly recommends a circuit breaker (it protects *you*, not Stripe). Wrap `StripeService` in Polly with sane timeouts and bounded retries (retries safe *only* once idempotency keys from C3 are in place).
- **M4. Stripe API version not pinned** — pin `StripeConfiguration.ApiVersion` / `appInfo` so SDK upgrades don't silently change event/object shapes.
- **M5. Unhandled event types** — no `charge.refunded`, `charge.dispute.created`, `payment_intent.canceled`. Disputes/chargebacks are money events currently ignored.
- **M6. No alerting** — every "Manual refund required" / webhook failure is a silent log line. These must page someone.

---

## Recommended priority order

1. **C3** — idempotency keys on PaymentIntent + Refund (cheapest, stops double-charge/double-refund).
2. **C2 + C1** — stop returning 200 on webhook failure; never auto-cancel/expire a paid intent without refunding.
3. **C4** — check refund status.
4. **H3 + H2** — persist raw events, dedup by event id, process async.
5. **H1** — reconciliation job (the catch-all safety net).
6. **H5/H6** — concurrency token + atomic confirm transaction.
7. Medium items as hardening.

---

## Summary

The architecture is fundamentally sound — clean PSP abstraction, signature verification, transactional outbox, an expiry worker. The danger is concentrated in four reachable money-loss paths (C1–C4), and **C1 is live today**: a paid customer can be auto-cancelled with no refund. C1–C4 should be treated as ship-blockers.
