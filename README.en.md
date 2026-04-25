# RENOVAX Payments — DHRU Fusion Gateway

Plug-and-play integration that lets any **DHRU Fusion** panel accept payments through
**RENOVAX Payments**: crypto (USDT, USDC, EURC, DAI and more), PayPal, Stripe (cards),
and any other method RENOVAX adds over time — all from a single checkout.

When a payment is confirmed, RENOVAX Payments sends a signed webhook to your DHRU and the
invoice is credited automatically. No database schema changes required.

---

## Table of Contents

1. [Files in this package](#1-files-in-this-package)
2. [Requirements](#2-requirements)
3. [Configuring payment methods in RENOVAX Payments](#3-configuring-payment-methods-in-renovax-payments)
4. [Installation — DHRU side](#4-installation--dhru-side)
5. [Configuration reference](#5-configuration-reference)
6. [How it works](#6-how-it-works)
7. [Webhook event reference](#7-webhook-event-reference)
8. [Supported events and actions](#8-supported-events-and-actions)
9. [Firewall and WAF allowlist](#9-firewall-and-waf-allowlist)
10. [Security](#10-security)
11. [Connection-drop resilience](#11-connection-drop-resilience)
12. [Troubleshooting](#12-troubleshooting)
13. [Support](#13-support)

---

## 1. Files in this package

| File | Copy it to (on your DHRU server) |
| --- | --- |
| `modules/gateways/renovaxpayments.php` | `/modules/gateways/renovaxpayments.php` |
| `renovaxpaymentscallback.php` | `/renovaxpaymentscallback.php` (public web root) |

> Only **2 files** are needed. There is no extra admin panel file to install.

---

## 2. Requirements

| Requirement | Details |
| --- | --- |
| DHRU Fusion | Any version with filesystem access and the `modules/gateways/` path |
| RENOVAX Payments account | Active merchant at [payments.renovax.net](https://payments.renovax.net) |
| PHP | 7.4 or higher, with `curl` and `hash` extensions enabled |
| HTTPS | Mandatory — RENOVAX Payments only delivers webhooks to `https://` URLs |

---

## 3. Configuring payment methods in RENOVAX Payments

Before installing the DHRU files, your merchant account in RENOVAX Payments needs at least
one **active payment method**. This is what your customers will see at the checkout
(crypto wallets, Stripe card form, PayPal button, etc.).

Go to **Merchants → (your merchant) → Payment Methods → Add**.

---

### Cryptocurrency (USDT, USDC, EURC, DAI, and more)

There is **one Crypto method per merchant** that covers all supported tokens and networks.
Go to **Payment Methods → Add → Crypto**.

**Step 1 — Wallet addresses.** Enter the receiving address for each blockchain family
you want to accept (EVM, Tron, Solana, TON, Stellar, Sui, Aptos).
Only fill in the families you will actually use.

> These are deposit-only addresses. RENOVAX Payments monitors incoming transactions and
> notifies you — it never initiates transfers.

**Step 2 — Token × network pairs.** A matrix of all supported stablecoins (USDT, USDC,
EURC, DAI, FDUSD, PYUSD, TUSD, USDE, EURT…) and networks appears automatically based
on the addresses you entered. Enable only the combinations you want to offer.

You can toggle individual pairs on/off at any time from the method list without
disabling the entire crypto method.

**Step 3 — Payment tolerance (optional).**

| Setting | Default |
| --- | --- |
| Absolute tolerance (USD shortfall accepted) | `0.01` |
| Percent tolerance (% of invoice amount) | `0.5 %` |
| Accept overpayments | enabled |

---

### Stripe (credit and debit cards)

You can add **multiple Stripe accounts** to the same merchant (e.g. one per currency or region).
Go to **Payment Methods → Add → Stripe** and fill in:

| Field | Notes |
| --- | --- |
| **Label** | Internal name, e.g. `Stripe USD` |
| **Secret key** | `sk_live_...` |
| **Publishable key** | `pk_live_...` |
| **Webhook secret** | `whsec_...` — **required** |
| **Settlement currency** | Only if your Stripe account settles in a different currency |

**Country policy (optional):** restrict which countries can use this account.

- **Block mode** — accept all countries *except* the listed ones.
- **Allow mode** — accept *only* the listed countries.

---

### PayPal

Go to **Payment Methods → Add → PayPal** and fill in:

| Field | Notes |
| --- | --- |
| **Label** | Internal name, e.g. `PayPal USD` |
| **Client ID** | From your PayPal app |
| **Client Secret** | From your PayPal app |
| **Webhook ID** | **Required** |

---

### PIX (Brazil — BRL only)

Go to **Payment Methods → Add → PIX** and fill in:

| Field | Notes |
| --- | --- |
| **Label** | Internal name, e.g. `PIX BRL` |
| **Provider** | `efi` or `gerencianet` |
| **Client ID** | From your EFI Bank account |
| **Client Secret** | From your EFI Bank account |
| **PIX key** | Your registered PIX key |

The customer sees a QR code inline — no redirect needed.

---

### Mercado Pago

Go to **Payment Methods → Add → Mercado Pago** and fill in:

| Field | Notes |
| --- | --- |
| **Label** | Internal name, e.g. `Mercado Pago ARS` |
| **Access token** | From your Mercado Pago account |
| **Public key** | From your Mercado Pago account |

---

### Enabling and disabling methods

From **Merchants → Payment Methods**:

- Click the **pause / play** icon next to any method to toggle it on or off instantly.
- For Crypto, you can also toggle **individual token × network pairs** without
  disabling the entire crypto method.
- Disabled methods are hidden from the checkout — customers cannot select them.

---

### How customers see the checkout

When a customer opens the payment link (`pay_url`) they see a grid of all active methods:

1. **Crypto** — leads to a token/network selector. The customer picks (e.g. USDT on Tron)
   and gets the exact wallet address + amount to send, with a QR code and wallet
   deep-links (MetaMask, Phantom, TronLink, etc.).
2. **Stripe** — redirects to Stripe's hosted card form.
3. **PayPal** — redirects to PayPal's hosted flow.
4. **PIX** — shows a QR code inline, no redirect.
5. **Mercado Pago** — redirects to Mercado Pago checkout.

If a merchant has **multiple accounts of the same type**, the customer sees an
intermediate screen to choose between them before proceeding.

---

## 4. Installation — DHRU side

### Step 1 — Get your API credentials from RENOVAX Payments

You will need two secret values. Ask your RENOVAX Payments operator to:

1. Open **Merchants → (your merchant) → Edit → API Tokens**.
2. Click **Create** and immediately copy the token value — **it is shown only once**.
3. On the same page, copy the **Webhook Secret**.

Keep these two values handy for the next steps.

---

### Step 2 — Register the webhook URL

Still in the RENOVAX Payments merchant settings, set the webhook URL to:

```text
https://YOUR-DHRU-DOMAIN.com/renovaxpaymentscallback.php
```

Replace `YOUR-DHRU-DOMAIN.com` with your actual domain.

---

### Step 3 — Upload the files

Copy the two PHP files to your DHRU server at the paths shown in the table above.
Set file permissions so the web server can read them:

```bash
chmod 644 modules/gateways/renovaxpayments.php
chmod 644 renovaxpaymentscallback.php
```

---

### Step 4 — Activate the gateway in DHRU admin

1. Log in to your DHRU admin panel.
2. Go to **Configuration → Payment Gateways**.
3. Find **RENOVAX Payments** in the list and click **Edit**.
4. Fill in the fields (see [Configuration reference](#5-configuration-reference) below).
5. Set **Status** to **Active** and save.

The gateway is now live for your clients at checkout.

---

## 5. Configuration reference

| Field | Description | Default |
| --- | --- | --- |
| **API Base URL** | RENOVAX Payments API endpoint. Do not change unless instructed. | `https://payments.renovax.net` |
| **Bearer Token** | API token from Step 1. Paste it exactly — no spaces. | *(required)* |
| **Webhook Secret** | HMAC secret from Step 1. Used to verify webhook authenticity. | *(required)* |
| **Invoice TTL (min)** | How many minutes a payment link stays valid before it expires. | `15` |

> **Currency** is inherited from your merchant profile in RENOVAX Payments.
> You do not need to set it here.

---

## 6. How it works

```text
Customer clicks "Pay"
        │
        ▼
renovaxpayments.php  ──POST /api/merchant/invoices──▶  RENOVAX Payments API
        │                                                       │
        │◀──────────────────── pay_url ─────────────────────────┘
        │
        ▼
Customer is redirected to RENOVAX Payments checkout
(picks Crypto / PayPal / Stripe / etc.)
        │
        ▼
Payment confirmed (on-chain or processor)
        │
        ▼
RENOVAX Payments sends HMAC-signed webhook
        │
        ▼
renovaxpaymentscallback.php
  1. Verifies HMAC signature
  2. Sends 200 OK to RENOVAX Payments immediately (connection closed)
  3. Credits the DHRU invoice in the background
```

### Behind the scenes

- `renovaxpayments.php` is loaded by DHRU when the customer clicks the payment button.
  It calls the RENOVAX Payments API with the invoice amount, stores `dhru_invoiceid` in
  the metadata, and returns a redirect button pointing to `pay_url`.

- `renovaxpaymentscallback.php` receives the webhook, verifies its HMAC signature,
  rewrites the DHRU invoice amounts (gross / net / fees) and calls DHRU's `addpayment()`
  to credit the funds.

- The `dhru_invoiceid` travels inside the invoice `metadata`, so no extra database columns
  or tables are needed.

---

## 7. Webhook event reference

RENOVAX Payments POSTs a JSON body like this to your callback URL:

```json
{
  "event_type": "invoice.paid",
  "invoice_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "confirmed",
  "invoice_amount": "100.00",
  "invoice_currency": "USD",
  "amount_received_fiat": "100.00",
  "amount_net_fiat": "97.00",
  "fee": "3.00",
  "tx_hash": "0xabc123...",
  "network": "tron",
  "paid_at": "2026-04-25T10:05:00Z",
  "metadata": {
    "dhru_invoiceid": "42",
    "dhru_email": "customer@example.com",
    "dhru_systemurl": "https://your-dhru.com/"
  }
}
```

### Request headers

| Header | Value |
| --- | --- |
| `X-Renovax-Signature` | `sha256={hmac}` — HMAC-SHA256 of the raw body using your `webhook_secret` |
| `X-Renovax-Event-Type` | Event type string (same as `event_type` in the body) |
| `X-Renovax-Event-Id` | Unique delivery UUID — use this for idempotency if needed |

---

## 8. Supported events and actions

| Event | Condition | Action |
| --- | --- | --- |
| `invoice.paid` | `status = confirmed` | Rewrites invoice amounts and credits the DHRU invoice |
| `invoice.overpaid` | `status = confirmed` | Same as above — customer overpaid, full amount credited |
| `invoice.partial` | `status = confirmed` | Same as above — partial payment credited as received |
| `invoice.expired` | any | Acknowledged (200) and ignored — no credits |
| Any other event | any | Acknowledged (200) and ignored |

### How amounts are calculated

The callback rewrites the DHRU invoice to correctly reflect what the customer actually
paid, including RENOVAX Payments fees:

```text
tax      = (invoice.taxrate% × amount_net_fiat) + invoice.fixedcharge + renovax_fee
subtotal = amount_received_fiat − tax
total    = subtotal   ← makes the invoice close naturally in DHRU
```

The `AddFunds` line item is updated to `subtotal` so DHRU credits the wallet with
the net amount. A full audit note is appended to the invoice.

---

## 9. Firewall and WAF allowlist

Two hosts must be able to communicate through any firewall, reverse proxy, or WAF
sitting in front of your DHRU server.

### Outbound — your DHRU server → RENOVAX Payments

`renovaxpayments.php` makes an outbound HTTPS call every time a customer clicks Pay.
Allow TCP 443 to:

```text
payments.renovax.net
```

If your firewall only accepts IPs (not hostnames), contact
[payments.renovax.net/support](https://payments.renovax.net/support) to get the
current IP range for `payments.renovax.net`. IPs can change; use the hostname wherever
possible.

### Inbound — RENOVAX Payments → your DHRU server

RENOVAX Payments POSTs webhook events to `renovaxpaymentscallback.php` from its own
servers. Your WAF or firewall must:

1. **Allow `POST /renovaxpaymentscallback.php`** from RENOVAX Payments IPs.
   Request the current egress IP list from
   [payments.renovax.net/support](https://payments.renovax.net/support).

2. **Pass these headers through unmodified** — stripping or rewriting them will cause
   every webhook to fail with `401 invalid_signature`:

   | Header | Purpose |
   | --- | --- |
   | `X-Renovax-Signature` | HMAC-SHA256 signature of the raw body |
   | `X-Renovax-Event-Type` | Event type (e.g. `invoice.paid`) |
   | `X-Renovax-Event-Id` | Unique delivery UUID |
   | `Content-Type` | Must arrive as `application/json` |

3. **Do not buffer or modify the request body.** The HMAC is computed over the exact
   raw bytes received. Any transformation (pretty-printing, re-encoding, gzip) will
   invalidate the signature.

### Common WAF rules to review

| Rule type | What to check |
| --- | --- |
| Rate limiting | Whitelist RENOVAX Payments IPs on `renovaxpaymentscallback.php` — retry storms can trigger rate limits |
| Body size limit | Webhook payloads are typically under 4 KB; a 1 MB limit is more than enough |
| JSON validation / normalization | Disable for this endpoint — body must reach PHP unmodified |
| Bot / crawler protection | Exclude `renovaxpaymentscallback.php` from JS-challenge or CAPTCHA flows |
| Geo-blocking | RENOVAX Payments infrastructure may egress from multiple regions — use IP allowlist instead of country rules |

---

## 10. Security

- **Signature verification** uses `hash_equals()` (constant-time comparison — safe against timing attacks).
- **Idempotency** — if the DHRU invoice is already `Paid` or the transaction ID was already recorded, the webhook is acknowledged and discarded without double-crediting.
- **Bearer Token** and **Webhook Secret** are stored as `password`-type fields in DHRU — they are not shown in the admin UI after saving.
- Never log or expose `webhook_secret` in frontend code or error outputs.
- Always serve `renovaxpaymentscallback.php` over **HTTPS**.

---

## 11. Connection-drop resilience

The callback is designed to complete even if the network connection drops mid-request:

1. `ignore_user_abort(true)` is set at the top — the script keeps running even if the
   remote side disconnects.
2. `set_time_limit(120)` gives the DB operations enough time to finish.
3. After HMAC verification and input validation, the callback **immediately flushes a
   `200 OK`** response and closes the HTTP connection to RENOVAX Payments.
4. All database writes (idempotency check, invoice rewrite, `addpayment`) happen
   **after** the response is sent, in the background.

This means RENOVAX Payments always gets its `200` quickly and will never retry due to
a timeout on your server's DB operations.

> On PHP-FPM (the most common setup) the flush uses `fastcgi_finish_request()`.
> On mod_php / CGI it falls back to output-buffering + `Connection: close`.

---

## 12. Troubleshooting

### 401 invalid_signature

The `webhook_secret` in DHRU does not match the one in RENOVAX Payments.
Go to the merchant settings in RENOVAX Payments, copy the secret again, and paste it
into the DHRU gateway config **without leading or trailing spaces**.

### 400 missing_dhru_invoiceid

The callback received a webhook but the `metadata.dhru_invoiceid` field was empty.
This usually means `modules/gateways/renovaxpayments.php` was not uploaded or is
an older version. Re-upload the file from this package.

### Payment button not appearing at checkout

- Confirm `modules/gateways/renovaxpayments.php` is in the correct path.
- Check that the gateway is set to **Active** in DHRU admin.
- Verify the Bearer Token is set (no spaces).

### Invoice not credited after payment

1. Check your PHP error log for lines starting with `[renovaxpayments]`.
2. Verify the webhook URL in RENOVAX Payments points to `https://YOUR-DOMAIN/renovaxpaymentscallback.php`.
3. Query the invoice status directly:

```bash
curl https://payments.renovax.net/api/merchant/invoices/{invoice_id} \
     -H "Authorization: Bearer {your_token}"
```

### No payment methods shown at checkout

The customer reaches the `pay_url` but sees no payment options. This means the merchant
has no active payment methods. Go back to [section 3](#3-configuring-payment-methods-in-renovax-payments)
and add at least one method.

### Automatic retries

If your server responds with `5xx` (or times out), RENOVAX Payments retries with
exponential backoff: **30s → 2min → 10min → 1h → 6h → 24h** (6 attempts total).
A `200` response stops retries immediately.

### Where to find logs

Errors are written via PHP `error_log()`. Location depends on your server setup:

- **Apache**: `error_log` directive in `httpd.conf` or the virtual host config.
- **Nginx + PHP-FPM**: the pool's `error_log` setting (usually `/var/log/php-fpm/www-error.log`).

---

## 13. Support

[payments.renovax.net/support](https://payments.renovax.net/support)
