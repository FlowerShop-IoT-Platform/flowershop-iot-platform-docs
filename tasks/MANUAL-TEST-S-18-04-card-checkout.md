# Manual Test — S-18-04: Card checkout + SCA / 3-D Secure

> **Feature under test:** the Customer App web checkout (`Views/Checkout/Payment.cshtml`) no longer
> hardcodes `stripe.confirmBlikPayment()`. It now mounts the **Stripe Payment Element** from the
> order's `client_secret` and calls `stripe.confirmPayment({ elements, confirmParams: { return_url },
> redirect: 'if_required' })`. Card payments that require **3-D Secure** redirect to the bank
> challenge and return to a new **payment-return page** (`/Checkout/PaymentReturn`). The webhook
> (`payment_intent.succeeded` / `.processing`, already wired in EP-17 S-17-05) remains the source of
> truth for order state. The payment **method** used is now captured on the order (S-18-01).
>
> **No backend payment-engine change was required for card:** `StripeService` already sets
> `PaymentMethodTypes = { blik, card }` for PLN (`{ card }` otherwise), and
> `StripeWebhookProcessorService` + `ConfirmPaymentCommandHandler` are method-agnostic. The remaining
> work was UI + the SCA return page + capturing the method.

## Environment prerequisites

| Component | URL / value |
|---|---|
| API + Swagger | http://localhost:8080/ |
| Customer App | http://localhost:5003 |
| Admin Portal (Payments page) | http://localhost:5001 → **Payments** nav |
| Vendor Portal (Orders payment column) | http://localhost:5002 → **Orders** |
| DB | `localhost:5432 / flowershop_iot_dev / postgres / dev123` |
| Migration applied | `20260611163854_AddOrderPaymentMethod` (adds `Orders.PaymentMethod varchar(50)`) |
| Stripe | test keys in `appsettings.Development.json` (`sk_test_…`, `pk_test_…`) **and a non-empty `Stripe:WebhookSecret`** so the webhook processor confirms the order |
| Stripe webhook | `stripe listen --forward-to http://localhost:8080/api/v1/stripe/webhook` (or a dashboard test endpoint). Required: the order flips to Paid via the webhook, not the client. |

> ⚠️ The `Stripe:WebhookSecret` must be set for the end-to-end order-state change. Without it the UI
> still completes the 3-D Secure flow and shows success, but the order will only flip to Paid via the
> best-effort `/Checkout/ConfirmPayment` nudge (or stay Created until reconciliation).

---

## Scenario 1 — Card payment requiring 3-D Secure (the headline path)

Use the Stripe **3DS-required** test card: **`4000 0027 6000 3184`**, any future expiry, any CVC,
any postal code.

1. Log into the Customer App (http://localhost:5003) as a customer with a real Keycloak session.
2. Add an **Available** bouquet to the cart → **Checkout** → choose pickup (or delivery with a full
   address) → **Place order**.
3. On the payment page (`/Checkout/Payment`):
   - ✅ The **Stripe Payment Element** renders with method tabs — **Card** and **BLIK** both offered
     (PLN order). Title reads "Complete payment", not "Pay with BLIK".
   - Choose **Card**, enter `4000 0027 6000 3184`, submit.
4. ✅ Stripe redirects to the **3-D Secure challenge** page. Click **Complete authentication**.
5. ✅ The browser returns to **`/Checkout/PaymentReturn?orderId=…&payment_intent=…&payment_intent_client_secret=…&redirect_status=succeeded`**:
   - The page retrieves the intent via `stripe.retrievePaymentIntent` and shows **"Payment successful"**,
     then redirects to `/Checkout/Confirmation?orderId=…` after ~1.5 s.
6. ✅ Within a few seconds the **`payment_intent.succeeded`** webhook fires and the order flips to
   **Paid**. Verify:
   ```sql
   SELECT "Id","Status","PaidAt","PaymentMethod" FROM "Orders" WHERE "Id" = '<orderId>';
   -- Status=Paid, PaidAt set, PaymentMethod='card'
   ```
7. ✅ **Admin Portal → Payments**: order does NOT appear in the attention buckets (it's healthy), but
   the **method-mix** metric card shows a `card` entry, and the **Paid orders** counter incremented.
8. ✅ **Vendor Portal → Orders**: the order row shows a green **Paid** payment badge with `card`
   underneath; the detail page shows **Payment: Paid via card** and **Paid at**.

## Scenario 2 — Card NOT requiring 3-D Secure (no redirect)

Use **`4242 4242 4242 4242`** (Visa, no SCA).

1. Repeat Scenario 1 steps 1–3, choosing **Card** with `4242 4242 4242 4242`.
2. ✅ `confirmPayment({ redirect: 'if_required' })` resolves **inline** (no redirect). The page shows
   "Payment successful!" and redirects straight to `/Checkout/Confirmation`.
3. ✅ Order flips to Paid via webhook; `PaymentMethod='card'`.

## Scenario 3 — BLIK regression (must still work via the same Payment Element)

1. Repeat Scenario 1 steps 1–3, choosing the **BLIK** tab in the Payment Element.
2. Enter a test BLIK code (`stripe listen` + Stripe's BLIK test flow). BLIK is asynchronous:
   `confirmPayment` returns `processing`.
3. ✅ The page shows "Payment received!" and redirects to confirmation; the order flips to Paid when
   the `payment_intent.succeeded` webhook arrives (the `payment_intent.processing` webhook first
   flags `PaymentProcessing` so the reservation isn't expired mid-flight — unchanged EP-17 behaviour).
4. ✅ `PaymentMethod='blik'` after success.

## Scenario 4 — Failed / declined card

Use **`4000 0000 0000 9995`** (insufficient funds) or **`4000 0000 0000 0002`** (generic decline).

1. Submit the card on the payment page.
2. ✅ The Payment Element shows the decline inline (no redirect); the page surfaces the error and
   re-enables the Pay button. The order stays **Created** and its reservation continues to count down.

---

## Confirm: SCA needs no server change

`requires_action` (the 3-D Secure challenge) is handled entirely client-side by the Payment Element +
`confirmPayment`. The server only needs:
- the existing `client_secret` returned in `OrderCreatedDto.PaymentClientSecret` (unchanged), and
- the existing webhook handlers for `payment_intent.succeeded` and `payment_intent.processing`
  (EP-17 S-17-05, unchanged).

The new `/Checkout/PaymentReturn` page is a **client-only** return target — it issues no new
server endpoint beyond the MVC action that renders it and the pre-existing `/Checkout/ConfirmPayment`
best-effort nudge.

---

## Automated coverage (green)

- `tests/FlowerShop.Tests.Unit/Infrastructure/Payments/StripeWebhookProcessorServiceTests.cs`
  → `SucceededEvent_ThreadsPaymentMethod_IntoConfirmPaymentCommand` proves the method extracted from
  the Stripe event reaches `ConfirmPaymentCommand` (and thus `Order.PaymentMethod`).
- `tests/FlowerShop.Tests.Unit/Infrastructure/Repositories/OrderRepositoryPaymentOperationsTests.cs`
  → the admin payment-ops filters.

```bash
dotnet test tests/FlowerShop.Tests.Unit/FlowerShop.Tests.Unit.csproj \
  --filter "FullyQualifiedName~StripeWebhookProcessorServiceTests|FullyQualifiedName~OrderRepositoryPaymentOperations"
```

---

## Manual verification run — PENDING

> ⏳ **Not yet executed against live Stripe test mode.** Scenarios 1–4 require a running stack with a
> configured `Stripe:WebhookSecret` and `stripe listen` forwarding, plus a real Keycloak customer
> session — which must be driven by a human operator. Record the run results in this section
> (matching the format of `MANUAL-TEST-S-04-06-refund-status.md`) once executed.
