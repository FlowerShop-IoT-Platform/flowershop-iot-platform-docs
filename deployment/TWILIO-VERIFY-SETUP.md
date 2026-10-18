# Twilio Verify — operator setup (EP-29)

Real SMS codes for customer phone verification (`POST /api/v1/profile/phone/send-code`).
Until this is done, production answers **503 `sms-not-configured`** and the Customer App shows
"Phone verification is temporarily unavailable" — it never pretends a code was sent.

> Twilio's console menus get renamed from time to time. If a step's menu path is not where this
> page says, type the feature name ("Geo permissions", "Verify Services") into the Console search.

## 1. Create the account (free trial)

1. Go to <https://www.twilio.com/try-twilio> and sign up (email or Google). No credit card needed.
2. Verify your email, then verify **your own mobile number** by SMS — it becomes the first
   *verified caller ID*.
3. Onboarding questions: product **Verify**, use case **2FA / phone verification**, language **C#**.

Trial limits: Verify can only send to **verified caller IDs** (up to 5 recipients), messages may
carry a "Sent from a Twilio trial account" prefix. Add test phones under
**Phone Numbers → Manage → Verified Caller IDs**. Upgrade the account before real customers verify.

## 2. Create the Verify service

1. **Verify → Services → Create new**. Friendly name `FindMyFlowers` (it appears in the SMS text).
2. Channels: enable **SMS** only. Code length **6** (the UI expects 6 digits).
3. Copy the **Service SID** (`VA…`).
4. In the service settings make sure **Fraud Guard** is enabled (blocks known SMS-pumping patterns).

## 3. Restrict destinations — Geo Permissions

**Verify → Settings → Geo permissions** (or Messaging → Settings → Geo permissions): allow
**Poland (+48)** only; disable every other country. This mirrors the in-app allow-list
`Sms:Limits:AllowedCountryPrefixes` (default `["+48"]`) — keep both in sync when expanding.

## 4. Credentials → Fly secrets

Copy **Account SID** (`AC…`) and **Auth Token** from Console → Account Info, then:

```bash
fly secrets set -a flower-shop-backend-core \
  Sms__Provider=Twilio \
  Sms__Twilio__AccountSid=AC... \
  Sms__Twilio__AuthToken=... \
  Sms__Twilio__VerifyServiceSid=VA...
```

Set all four in one command (rolling restart). `Provider=Twilio` with any blank credential makes the
API throw when the SMS service is first resolved.

## 5. In-app abuse limits (no secrets — `appsettings.json` → `Sms:Limits`)

| Setting | Default | Meaning |
|---|---|---|
| `AllowedCountryPrefixes` | `["+48"]` | Destination E.164 prefixes allowed |
| `MaxDistinctNumbersPerCustomer` / `DistinctNumbersWindowDays` | 2 / 30 | Numbers one account may try |
| `MaxSendsPerCustomerPerDay` | 6 | Codes per account per 24 h |
| `MaxSendsPerNumberPerDay` | 5 | Codes per destination number per 24 h, all accounts |
| `GlobalDailySendBudget` | 200 | Platform-wide successful sends per 24 h (0 = unlimited) |
| `RequireVerifiedEmail` | `true` in Production | Keycloak email must be verified first |

Override any of them with a Fly env/secret, e.g. `Sms__Limits__GlobalDailySendBudget=500`.
Optional captcha: set `Turnstile__SecretKey` on the API (same Turnstile site as the contact form,
whose public site key the Customer App already has as `ContactForm__TurnstileSiteKey`).

## 6. Smoke test

1. Sign in to <https://app.findmyflowers.pl> with an account whose email is verified.
2. Profile → **Verify phone** → enter a verified caller ID → **Send code**. SMS arrives in seconds.
3. Enter the code → badge turns **Verified**.
4. Twilio Console → **Monitor → Logs → Verify** shows the attempt.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "temporarily unavailable" | `Sms__Provider` not `Twilio` → step 4 |
| "couldn't send an SMS to this number" (API 502 `sms-send-failed: 21608`) | Trial: number is not a verified caller ID |
| `sms-send-failed: 60605` / 60410 | Geo permission off for that country, or Fraud Guard blocked the prefix |
| "Too many attempts" (429) | App limit or Verify rate limit hit — wait, or raise `Sms:Limits` |
| API log `SMS daily budget exhausted` | `GlobalDailySendBudget` reached — check Verify logs for an attack before raising it |

Keycloak self-registration captcha is configured in Keycloak (Authentication → Flows → registration →
reCAPTCHA), outside this codebase; recommended once public sign-ups grow.
