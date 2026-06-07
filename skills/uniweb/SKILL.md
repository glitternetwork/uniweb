---
name: uniweb
description: Help developers safely add uniweb payments to an app. Use when the user asks to accept card, WeChat Pay, Alipay, or PayNow payments; create payment links; add checkout, subscriptions, billing, or invoice-style payment collection; handle webhooks; or use the uniweb CLI or TypeScript SDK.
argument-hint: "[what you want to do]"
---

# uniweb Payment Integrations

You are helping a developer integrate uniweb into an existing app. Your goal is to choose the lightest correct payment path, generate server-safe code, and make fulfillment depend on verified webhooks instead of browser redirects.

**User request:** $ARGUMENTS

Before editing an app, inspect its framework, routing style, package manager, and environment variable conventions. Keep secrets in server-side environment variables and match the project's existing patterns.

## Choose The Integration Path

| User needs | Use | Why |
| --- | --- | --- |
| One fixed amount, no backend | Payment Link | Permanent `/p/plink_xxx` URL; safe to put in HTML, email, invoices, or social posts. |
| Fixed product or subscription price | Product + Price URL | Permanent `/buy/price_xxx` URL; good for pricing pages and recurring plans. |
| Cart, quantity, order, POS, dynamic amount, or per-request metadata | TypeScript SDK + Checkout Session | Create a one-time session on the server, then redirect to `session.url`. |

When the user says "invoice", use a Payment Link for simple invoice-style collection unless they need a real per-order checkout session. uniweb does not expose a separate invoice API for merchants.

## Security Defaults

1. Never put `sk_live_` in deployed code or browser code. It is a full key for trusted local/admin use.
2. For backends, create and use `sk_server_` keys with the smallest scopes needed.
3. Prefer `@uniweb/sdk` over hand-written `fetch()` for server integrations.
4. The SDK must run only on the server: Node.js, Cloudflare Workers, Deno, or Bun. Do not import it in browser bundles.
5. Fulfill orders and grant access only after a verified webhook. Treat `successUrl` as UX only.
6. Amounts are integer cents/minor units. Default currency is `SGD`.

Server keys are scope-based. Default server keys can create products, prices, checkout sessions, customers, and payment links, and can read payments/subscriptions/refunds. Extra scopes such as `payments.write`, `refunds.create`, `payouts.create`, `kyc.manage`, or `bank_accounts.manage` should be granted only when the backend truly needs those money-moving or account-management actions.

## Path A: Payment Link

Use this for a fixed amount with no backend.

```bash
npm install -g uniweb
uniweb wallet create
uniweb link create 1000 -c SGD -n "My Product" --methods card,wechat,alipay,paynow
# returns https://vibecash.dev/p/plink_xxx
```

```html
<a href="https://vibecash.dev/p/plink_xxx">Pay $10.00</a>
```

Optional fulfillment notification:

```bash
uniweb webhook set https://yoursite.com/webhook
```

## Path B: Product + Price URL

Use this for a known product, pricing page, or subscription plan.

```bash
uniweb product create "Pro Plan"
uniweb price create prod_xxx -a 999 -t recurring -i month -c SGD
# returns https://vibecash.dev/buy/price_xxx
```

```html
<a href="https://vibecash.dev/buy/price_xxx">Subscribe - $9.99/month</a>
```

Notes:

- One-time prices use `-t one_time` and do not need `-i`.
- Subscriptions support card payments only. Ignore WeChat Pay, Alipay, and PayNow for subscription checkout.
- Customer email is collected at checkout; uniweb creates or reuses the Customer record.
- Subscription statuses: `trialing`, `active`, `past_due`, `unpaid`, `canceled`.
- Renewal retries follow 1 hour, 1 day, 3 days, 7 days, then final unpaid handling.

Subscription management from a trusted machine:

```bash
uniweb subscription list
uniweb subscription pause <id>       # cancel at period end
uniweb subscription resume <id>      # undo pause
uniweb subscription cancel <id>      # cancel immediately
```

## Path C: TypeScript SDK + Checkout Session

Use this when the amount, quantity, customer, order, or metadata is dynamic.

Create a restricted server key:

```bash
uniweb key create -t server -n "Checkout Backend"
```

Install and use the SDK on the backend:

```bash
npm install @uniweb/sdk
```

```typescript
import Uniweb from '@uniweb/sdk';

const vc = new Uniweb(process.env.VIBECASH_SERVER_KEY!);

const product = await vc.products.create({
  name: 'Nasi Lemak',
  description: 'Spicy coconut rice',
});

const price = await vc.prices.create({
  productId: product.id,
  amount: 500,
  currency: 'SGD',
  type: 'one_time',
});

// For fixed product pages, this permanent URL is enough.
res.redirect(price.paymentUrl);
```

For carts or per-order checkout, create a fresh session server-side and redirect immediately:

```typescript
const session = await vc.checkout.create({
  mode: 'payment',
  lineItems: [{ priceId: price.id, quantity: 2 }],
  successUrl: 'https://yoursite.com/success',
  cancelUrl: 'https://yoursite.com/cancel',
  customerEmail: 'customer@example.com',
  paymentMethodTypes: ['card', 'wechat', 'alipay', 'paynow'],
  metadata: { orderId: 'ord_123' },
});

res.redirect(session.url!);
```

Checkout Session URLs expire after 24 hours and are one-time checkout URLs. That is correct for production dynamic orders; create them at checkout time. Do not store them as permanent product links.

For subscriptions through the SDK:

```typescript
const session = await vc.checkout.create({
  mode: 'subscription',
  lineItems: [{ priceId: recurringPrice.id, quantity: 1 }],
  successUrl: 'https://yoursite.com/billing/success',
  cancelUrl: 'https://yoursite.com/billing/cancel',
  trialPeriodDays: 14,
  paymentMethodTypes: ['card'],
});
```

## Webhook Fulfillment

Set a webhook URL and save the `whsec_xxx` secret. Webhook URLs must be HTTPS.

```bash
uniweb webhook set https://yoursite.com/webhook
```

Use the SDK verifier with the raw request body:

```typescript
import express from 'express';
import { verifyWebhook } from '@uniweb/sdk';

app.post('/webhook', express.raw({ type: 'application/json' }), async (req, res) => {
  try {
    const event = await verifyWebhook(
      req.body.toString('utf8'),
      req.header('uniweb-signature') ?? '',
      process.env.UNIWEB_WEBHOOK_SECRET!,
    );

    switch (event.type) {
      case 'payment.succeeded':
        // Mark the order paid after checking amount/currency/metadata.
        break;
      case 'payment.failed':
        // Keep the order unpaid or notify the customer.
        break;
      case 'checkout.session.completed':
        // Optional: record checkout completion.
        break;
      case 'subscription.created':
      case 'subscription.renewed':
        // Grant or extend access.
        break;
      case 'subscription.past_due':
      case 'subscription.unpaid':
      case 'subscription.canceled':
        // Update billing/access state.
        break;
      case 'refund.succeeded':
      case 'refund.failed':
        // Update refund state.
        break;
    }

    res.status(200).send('ok');
  } catch {
    res.status(400).send('Invalid signature');
  }
});
```

Webhook rules:

- Do not use JSON body parsing before verification; verification needs the exact raw body.
- Enforce idempotency using `event.id`, payment id, or your `metadata.orderId`.
- Before fulfillment, check expected amount, currency, customer/order metadata, and current order state.
- Return a 2xx only after durable processing. uniweb retries failed deliveries.
- Per-link webhook overrides per-product webhook, which overrides the wallet webhook. The signing secret is wallet-level.

Events:

```text
payment.succeeded, payment.failed
refund.succeeded, refund.failed
checkout.session.completed, checkout.session.expired
subscription.created, subscription.renewed, subscription.past_due,
subscription.unpaid, subscription.canceled, subscription.trial_ending
payout.requested, payout.approved, payout.settled, payout.rejected, payout.failed
account.kyc.approved, account.kyc.rejected
```

## SDK Surface

```typescript
await vc.products.create({ name, description?, webhookUrl?, metadata? });
await vc.products.list({ limit?, startingAfter? });
await vc.products.get(id);
await vc.products.update(id, { name?, description?, webhookUrl?, active?, metadata? });
await vc.products.del(id);
for await (const p of vc.products.listAll()) {}

await vc.prices.create({ productId, amount, currency, type, interval?, intervalCount?, trialPeriodDays?, metadata? });
await vc.prices.list({ productId?, limit?, startingAfter? });
await vc.prices.get(id);
await vc.prices.deactivate(id);
for await (const p of vc.prices.listAll({ productId? })) {}

await vc.checkout.create({ mode, lineItems, successUrl?, cancelUrl?, customerEmail?, customerId?, trialPeriodDays?, paymentMethodTypes?, metadata? });
await vc.checkout.list({ limit?, startingAfter? });
await vc.checkout.get(id);

await vc.payments.create({ checkoutSessionId, paymentMethod, ... });   // requires payments.write
await vc.payments.list({ status?, customerId?, limit?, startingAfter? });
await vc.payments.get(id);
await vc.payments.sync(id);                                            // requires payments.write
await vc.payments.void(id);                                            // requires refunds.create
for await (const p of vc.payments.listAll({ status?, customerId? })) {}

await vc.refunds.create({ paymentId, amount?, reason?, offlineRefundFlag?, metadata? }); // requires refunds.create
await vc.refunds.get(id, { gateway? });                                                  // gateway:true → gateway enquiry

await vc.customers.create({ email, name?, metadata? });
await vc.customers.list({ email?, limit?, startingAfter? });
await vc.customers.get(id);
await vc.customers.update(id, { email?, name?, metadata? });
await vc.customers.del(id);
for await (const c of vc.customers.listAll({ email? })) {}

await vc.subscriptions.create({ customerId, priceId, ... });
await vc.subscriptions.list({ customerId?, status?, limit?, startingAfter? });
await vc.subscriptions.get(id);
await vc.subscriptions.update(id, { cancelAtPeriodEnd? });
await vc.subscriptions.cancel(id);                                     // cancel immediately
await vc.subscriptions.resume(id);                                     // undo "cancel at period end"
for await (const s of vc.subscriptions.listAll({ customerId?, status? })) {}

await vc.links.create({ amount, currency, name?, description?, successUrl?, cancelUrl?, webhookUrl?, paymentMethodTypes?, metadata? });
await vc.links.list({ limit?, startingAfter? });
await vc.links.get(id);
await vc.links.update(id, { name?, description?, successUrl?, cancelUrl?, webhookUrl?, active? });
await vc.links.deactivate(id);
for await (const l of vc.links.listAll()) {}

// Wallet-level webhook configuration. The signing secret returned by set/rollSecret
// is the only place you can read whsec_xxx — store it before the call returns.
await vc.webhooks.set(url);          // returns { url, secret }
await vc.webhooks.info();            // current wallet, url, hasSecret
await vc.webhooks.remove();
await vc.webhooks.rollSecret();      // returns { url, secret } — invalidates the previous secret
```

The SDK constructor reads from `https://apiskill.uniwebpay.com` (production API edge) and `https://vibecash.dev` (production pay/checkout host) by default. Override only when the deployment instructs it (staging, local dev) — wrong overrides silently route real keys to the wrong environment:

```typescript
// Production: no overrides needed.
const vc = new Uniweb(process.env.VIBECASH_SERVER_KEY!);

// Staging or local dev only:
const vc = new Uniweb(process.env.VIBECASH_SERVER_KEY!, {
  baseUrl: process.env.VIBECASH_API_URL,   // e.g. https://api.staging.vibecash.dev
  payUrl: process.env.VIBECASH_PAY_URL,    // e.g. https://staging.vibecash.dev
});
```

## CLI Reference

```bash
npm install -g uniweb

# Wallet and dashboard claim
uniweb wallet create
uniweb wallet info
uniweb wallet claim

# Products and permanent /buy/ URLs
uniweb product create <name> [-d desc] [-w webhook-url]
uniweb product list
uniweb product get <id>
uniweb product update <id> [-n name] [-d desc] [-w webhook-url]
uniweb product delete <id>
uniweb price create <productId> -a <cents> -t one_time|recurring [-i day|week|month|year] [-c currency]
uniweb price list
uniweb create

# Payment links and permanent /p/ URLs
uniweb link create <cents> [-c currency] [-n name] [-d desc] [--methods card,wechat,alipay,paynow] [--success-url url] [--cancel-url url] [-w webhook-url]
uniweb link list
uniweb link get <id>
uniweb link update <id> [-n name] [-d desc] [--success-url url] [--cancel-url url] [-w webhook-url]
uniweb link activate <id>
uniweb link deactivate <id>

# Checkout sessions: one-time, 24-hour URLs for dynamic orders
uniweb checkout create -p <priceId> --success-url <url> --cancel-url <url> [-m payment|subscription] [-q quantity] [--methods card,wechat,alipay,paynow] [--customer-id <id>] [--trial-days days]
uniweb checkout get <id> [-w] [-i seconds]

# Payments and refunds from trusted/server contexts with appropriate scopes
uniweb payment list [-s status] [-c customerId]
uniweb payment get <id> [--gateway]
uniweb payment void <id>
uniweb payment refund <id> [-a cents] [--offline] [-r reason]
uniweb refund get <id> [--gateway]

# Customers and subscriptions
uniweb customer create -e <email> [-n name]
uniweb customer list
uniweb customer get <id>
uniweb customer update <id> [-e email] [-n name]
uniweb customer delete <id>
uniweb subscription list
uniweb subscription get <id>
uniweb subscription pause <id>
uniweb subscription resume <id>
uniweb subscription cancel <id>

# Keys and webhooks
uniweb key create -t server|full [-n name] [-s scopes]
uniweb key list
uniweb key revoke <id>
uniweb key roll <id>
uniweb webhook set <url>
uniweb webhook info
uniweb webhook remove
uniweb webhook roll-secret
uniweb webhook events

# KYC, bank accounts, payouts
uniweb kyc submit
uniweb kyc status
uniweb bank add
uniweb bank list
uniweb bank remove <id>
uniweb bank set-default <id>
uniweb payout create -a <cents> -b <bankAccountId>
uniweb payout list
uniweb payout status
uniweb payout cancel <id>
```

## Constraints To Remember

- Supported payment methods: `card`, `wechat`, `alipay`, `paynow`.
- Subscriptions: card only.
- Currency support by method:
  - `card` — broad multi-currency, decided by gateway acquirer (default `SGD`).
  - `paynow` — `SGD` only (DBS PayNow rail).
  - `wechat` — `CNY` and a small set of cross-border currencies; check before using anything other than `CNY`.
  - `alipay` — primarily `CNY` plus Alipay Cross-border supported currencies; same caveat as wechat.
- Card minimum: `10` minor units (e.g. `0.10 SGD`, `0.10 USD`). For zero-decimal currencies (e.g. `JPY`, `IDR`) the minor unit IS the major unit, so `10 JPY` minimum applies — confirm with the gateway when listing such currencies.
- Payment Links and Product/Price URLs are reusable and permanent.
- Checkout Sessions are one-time and expire after 24 hours.
- Prices are effectively immutable for amount/currency; create a new price instead of editing the old one.
- KYC is not required to accept payments, but is required before payouts.

### Subscription status machine

```
trialing → active                   (trial ends with successful charge)
active   → past_due                 (renewal failed; retry schedule below)
past_due → active                   (a retry succeeded)
past_due → unpaid                   (all retries exhausted; access usually paused)
unpaid   → canceled                 (grace window over; subscription terminated)
*        → canceled                 (`subscriptions.cancel` ends immediately)
*        → cancelAtPeriodEnd=true   (`pause` schedules cancel at current period end; `resume` undoes)
```

Renewal retry attempts: `1h`, `1d`, `3d`, `7d`. After the 4th failure the subscription transitions to `unpaid`; if no recovery occurs in the grace window it transitions to `canceled`. Listen on `subscription.past_due` / `subscription.unpaid` / `subscription.canceled` for access changes.
