# Stripe BLIK Payment Integration — Technical Guide

> **Epic**: EP-04 (Order Lifecycle & Stripe)
> **Last Updated**: 2026-04-12

---

## 1. Current State

The backend already has significant Stripe scaffolding in place:

| Component | Status | Location |
|---|---|---|
| `IStripeService` interface | Done | `src/backend/FlowerShop.Application/Services/IStripeService.cs` |
| `StripeService` implementation | Done | `src/backend/FlowerShop.Infrastructure/Payments/StripeService.cs` |
| `StubStripeService` (no-key fallback) | Done | `src/backend/FlowerShop.Infrastructure/Extensions/ServiceCollectionExtensions.cs` |
| `Stripe.net` NuGet (v43.16.0) | Done | `FlowerShop.Infrastructure.csproj` |
| DI registration (conditional) | Done | Uses real `StripeService` when `Stripe:SecretKey` is set, otherwise `StubStripeService` |
| `StripeWebhookController` | Done | `src/backend/FlowerShop.API/Controllers/StripeWebhookController.cs` |
| `CreateOrderCommandHandler` → Stripe | Done | Creates PaymentIntent, stores ID on order |
| `CancelOrderCommandHandler` → refund | Done | Refunds via Stripe if order was paid |
| Webhook: `payment_intent.succeeded` | Done | Confirms payment via `ConfirmPaymentCommand` |
| Webhook: `payment_intent.payment_failed` | Done | Cancels order via `CancelOrderCommand` |
| **Stripe config in appsettings** | **Missing** | No `Stripe` section in any appsettings file |
| **BLIK payment method type** | **Missing** | `StripeService` uses `AutomaticPaymentMethods` — needs explicit BLIK opt-in |
| **Checkout page payment form** | **Missing** | CustomerApp checkout has no Stripe.js / BLIK code input |
| **Stripe CLI webhook forwarding** | **Not set up** | Needed for local testing |
| **Reservation expiry BackgroundService** | **Missing** | Orders expire after 15 min but no service cleans them up proactively |

---

## 2. Prerequisites

### 2.1 Stripe Account (Required)

**Yes, you need a Stripe account.** Registration is free and instant.

1. Go to **https://dashboard.stripe.com/register**
2. Sign up with email — no business verification needed for test mode
3. After signup you are in **Test Mode** by default (toggle in top-right of dashboard)
4. Get your test API keys from **Developers → API keys**:
   - **Publishable key**: `pk_test_...` (used in browser JavaScript)
   - **Secret key**: `sk_test_...` (used in backend)

> **Important**: Test mode is completely isolated from live mode. No real money is ever charged. No business verification is required to use test mode. You can stay in test mode indefinitely.

### 2.2 Stripe CLI (Required for Webhook Testing)

The Stripe CLI forwards webhook events from Stripe to your local machine.

```bash
# Install (Windows — via scoop)
scoop install stripe

# Or download from https://stripe.com/docs/stripe-cli

# Login (one-time)
stripe login

# Forward webhooks to your local API
stripe listen --forward-to http://localhost:8080/api/stripe/webhook
```

When you run `stripe listen`, it prints a **webhook signing secret** (`whsec_...`). Copy this — it goes into your config.

### 2.3 Configuration

Add to `src/backend/FlowerShop.API/appsettings.Development.json`:

```json
{
  "Stripe": {
    "SecretKey": "sk_test_YOUR_SECRET_KEY_HERE",
    "PublishableKey": "pk_test_YOUR_PUBLISHABLE_KEY_HERE",
    "WebhookSecret": "whsec_YOUR_WEBHOOK_SECRET_HERE"
  }
}
```

> **Never commit real keys.** The `.gitignore` already excludes `appsettings.Development.json` user secrets. Alternatively, use `dotnet user-secrets`:
> ```bash
> cd src/backend/FlowerShop.API
> dotnet user-secrets set "Stripe:SecretKey" "sk_test_..."
> dotnet user-secrets set "Stripe:PublishableKey" "pk_test_..."
> dotnet user-secrets set "Stripe:WebhookSecret" "whsec_..."
> ```

### 2.4 Enable BLIK in Stripe Dashboard

1. Go to **Settings → Payment methods** in the Stripe Dashboard
2. Enable **BLIK** under "Bank redirects" section
3. BLIK is available in test mode for all Stripe accounts

---

## 3. How BLIK Works with Stripe

### 3.1 Customer Experience

1. Customer clicks "Place Order" in the CustomerApp
2. A **6-digit code input** appears (no card form)
3. Customer opens their **banking app** (PKO, mBank, ING, etc.), copies the BLIK code
4. Customer enters the 6-digit code and clicks "Pay"
5. A push notification arrives in the banking app: "Confirm payment of PLN 150.00 to FlowerShop?"
6. Customer confirms → payment succeeds → order moves to `Paid`

### 3.2 Technical Flow

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  CustomerApp  │      │  Backend API │      │    Stripe    │      │  Customer's  │
│   (Browser)   │      │  (.NET 8)    │      │              │      │  Banking App │
└──────┬───────┘      └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
       │                     │                      │                     │
       │  POST /api/v1/orders│                      │                     │
       │ ───────────────────>│                      │                     │
       │                     │  CreatePaymentIntent │                     │
       │                     │  (payment_method_types: ["blik"])          │
       │                     │ ────────────────────>│                     │
       │                     │  { clientSecret }    │                     │
       │                     │ <────────────────────│                     │
       │  { clientSecret }   │                      │                     │
       │ <───────────────────│                      │                     │
       │                     │                      │                     │
       │  [User enters 6-digit BLIK code]           │                     │
       │                     │                      │                     │
       │  stripe.confirmBlikPayment(clientSecret,   │                     │
       │    { blik: { code: "123456" } })           │                     │
       │ ──────────────────────────────────────────>│                     │
       │                     │                      │  Push notification  │
       │                     │                      │ ───────────────────>│
       │                     │                      │  [User confirms]    │
       │                     │                      │ <───────────────────│
       │                     │                      │                     │
       │                     │  Webhook: payment_intent.succeeded         │
       │                     │ <────────────────────│                     │
       │                     │  ConfirmPaymentCommand                     │
       │                     │  Order: Created → Paid                     │
       │  { status: "paid" } │                      │                     │
       │ <───────────────────│                      │                     │
```

### 3.3 Key API Details

**Creating a PaymentIntent with BLIK:**
```csharp
var options = new PaymentIntentCreateOptions
{
    Amount = 15000,  // PLN 150.00 in grosze
    Currency = "pln",
    PaymentMethodTypes = new List<string> { "blik" },
    // Or keep AutomaticPaymentMethods for flexibility:
    // AutomaticPaymentMethods = new() { Enabled = true },
    Metadata = new Dictionary<string, string>
    {
        ["order_id"] = orderId.ToString(),
        ["customer_id"] = customerId.ToString(),
    },
};
```

**Confirming BLIK payment (browser-side JavaScript):**
```javascript
const { error } = await stripe.confirmBlikPayment(clientSecret, {
    payment_method: {
        blik: {},
        billing_details: { email: customerEmail }
    },
    payment_method_options: {
        blik: { code: document.getElementById('blik-code').value }
    }
});
```

---

## 4. Testing BLIK Payments

### 4.1 Stripe Test Mode — BLIK Codes

In test mode, enter **any valid 6-digit code** (e.g., `123456`) — it will succeed.

### 4.2 Simulating Failures

Use specific email addresses on the PaymentIntent to trigger failure scenarios:

| Email pattern | Behavior |
|---|---|
| `*@test.com` (default) | Success |
| `*invalid_code@*` | Immediate failure — invalid BLIK code |
| `*expired_code@*` | Immediate failure — code expired |
| `*insufficient_funds@*` | Fails after 8s — insufficient funds |
| `*customer_timeout@*` | Fails after 60s — customer didn't confirm in banking app |
| `*limit_exceeded@*` | Fails — daily transaction limit exceeded |
| `*bank_declined@*` | Fails — bank declined |
| `*blik_declined@*` | Fails — BLIK-specific decline |

### 4.3 Testing Webhooks Locally

```bash
# Terminal 1: Start the API
cd src/backend/FlowerShop.API && dotnet run

# Terminal 2: Forward Stripe webhooks
stripe listen --forward-to http://localhost:8080/api/stripe/webhook

# Terminal 3: Trigger a test event
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
```

### 4.4 End-to-End Test Flow

1. Configure `Stripe:SecretKey` in appsettings
2. Start the API + CustomerApp
3. Start `stripe listen --forward-to http://localhost:8080/api/stripe/webhook`
4. Log in as customer, add bouquet to cart, proceed to checkout
5. Place order → enter any 6-digit BLIK code → confirm
6. Check `stripe listen` terminal output for webhook delivery
7. Verify order status changed to `Paid` in the Orders page

---

## 5. What Needs to Be Built

### 5.1 Backend Changes (Minimal)

The backend is mostly ready. Remaining work:

| Task | Effort | Description |
|---|---|---|
| Add `Stripe` section to appsettings template | 5 min | Add placeholder keys to `appsettings.json` (not Development) |
| Explicit BLIK in `StripeService` | 10 min | Optionally add `PaymentMethodTypes = ["blik", "card"]` instead of `AutomaticPaymentMethods` for explicit control |
| `Stripe:PublishableKey` config endpoint | 15 min | Add `GET /api/v1/config` returning the publishable key for the frontend |
| Reservation expiry BackgroundService | 1-2 h | EP-04 T-04-013..T-04-016 — auto-cancel orders after 15 min |

### 5.2 Frontend Changes (CustomerApp Checkout)

This is the main work — the checkout page currently has no payment form:

| Task | Effort | Description |
|---|---|---|
| Add Stripe.js to checkout page | 15 min | `<script src="https://js.stripe.com/v3/"></script>` |
| BLIK code input field | 30 min | 6-digit input with validation, "Pay" button |
| `stripe.confirmBlikPayment()` call | 1 h | Wire clientSecret from order creation response, handle success/error |
| Payment status polling/redirect | 1 h | After payment confirmation, redirect to order details or show success |
| Update `CheckoutController.PlaceOrder` | 30 min | Return `clientSecret` to the view instead of redirecting to confirmation |
| Error handling UI | 30 min | Show BLIK-specific errors (expired code, declined, etc.) |

### 5.3 Total Estimated Effort

**~4-6 hours** for a working BLIK checkout flow, assuming the backend stays as-is and we focus on the CustomerApp frontend.

---

## 6. Costs

| Provider | BLIK Fee per Transaction | On a PLN 150 bouquet |
|---|---|---|
| **Stripe** | ~0.25 EUR (~1.10 PLN) + 2.0% | ~4.10 PLN |
| PayU | 0.30 PLN + 1.1% | ~1.95 PLN |
| Tpay | 0.29 PLN + 1.19% | ~2.08 PLN |

Stripe is ~2x more expensive per BLIK transaction than local Polish providers. The tradeoff: Stripe is already integrated in the codebase, has an official .NET SDK, and best-in-class developer experience. Switching to PayU later is possible — the `IStripeService` abstraction can be generalized to an `IPaymentService`.

---

## 7. BLIK Constraints

- **Currency**: PLN only
- **Country**: Poland only
- **Code validity**: 2 minutes from generation
- **Confirmation**: Customer must approve in their banking app within ~90 seconds
- **Recurring**: BLIK does not support recurring/subscription payments (use cards for subscriptions)
- **Refunds**: Full and partial refunds are supported
- **Mobile banking apps**: PKO BP, mBank, ING, Santander, Pekao, Alior, Millennium, BNP Paribas, Credit Agricole, Revolut, and ~10 more
- **Active users**: ~17 million in Poland (2024)

---

## 8. File Reference

| File | Role |
|---|---|
| `src/backend/FlowerShop.Application/Services/IStripeService.cs` | Payment abstraction |
| `src/backend/FlowerShop.Infrastructure/Payments/StripeService.cs` | Stripe SDK integration |
| `src/backend/FlowerShop.Infrastructure/Extensions/ServiceCollectionExtensions.cs` | DI — conditional real/stub registration |
| `src/backend/FlowerShop.API/Controllers/StripeWebhookController.cs` | Webhook handler (`payment_intent.succeeded/failed`) |
| `src/backend/FlowerShop.API/Controllers/OrdersController.cs` | `POST /api/v1/orders` — creates PaymentIntent |
| `src/backend/FlowerShop.Application/Handlers/Order/OrderCommandHandlers.cs` | Order creation, payment confirmation, cancellation |
| `src/portals/FlowerShop.CustomerApp/Controllers/CheckoutController.cs` | Checkout flow — needs payment form |
| `src/portals/FlowerShop.CustomerApp/Views/Checkout/Index.cshtml` | Checkout UI — needs BLIK input |
| `docs/tasks/tasks-ep04-orders-stripe.json` | EP-04 task breakdown (partially stale — many tasks already done) |
