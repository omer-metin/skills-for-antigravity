---
name: stripe-integration
description: "Implement production-grade Stripe payment integrations including payment intents, checkout sessions, webhook handlers, subscription lifecycle management, dunning flows, metered billing, and Stripe Connect. Covers idempotency, signature verification, error handling, and multi-currency support with concrete patterns and anti-pattern detection. Use when stripe, payments, subscription, billing, checkout, pricing, metered billing, stripe connect, webhooks, payment intent, customer portal, dunning, failed payment, refund, monetization, or SaaS billing mentioned."
---

# Stripe Integration

## Principles

- **Webhooks are the source of truth** — API responses are optimistic snapshots; grant access and update state only from webhook events.
- **Idempotency keys on every mutation** — attach keys to payment intents, subscriptions, refunds, charges, and transfers to prevent duplicates on retry.
- **Verify every webhook signature** — use `stripe.webhooks.constructEvent()` with the raw request body; never parse JSON before verification.
- **Never store card details** — use Stripe Elements or Checkout; raw card data triggers PCI scope.
- **Sync before acting** — always retrieve the latest object from Stripe before making financial decisions; local state drifts.
- **Log everything** — payment debugging requires full audit trails of API calls, webhook events, and state transitions.

## Workflow

### 1. Setting Up a Payment (Checkout Session)

1. Create a Stripe customer (with idempotency key) or retrieve an existing one by email.
2. Create a Checkout Session with `metadata` containing `userId` and `orderId` so the webhook can identify the buyer.
3. Also pass `subscription_data.metadata` or `payment_intent_data.metadata` so downstream objects carry context.
4. Redirect the user to the Checkout URL. **Do not grant access from the success URL** — wait for the webhook.

### 2. Handling Webhooks

The webhook handler must follow this sequence:

1. **Read the raw body** — use `req.text()` (App Router) or `express.raw()` (Express). Never use `req.json()` before verification.
2. **Verify the signature** with `constructEvent()`.
3. **Store the event** in a durable queue or database row.
4. **Return 200 immediately** — Stripe times out at 5 seconds and retries, risking duplicate processing.
5. **Process asynchronously** — a background worker handles the event with retry logic.

#### Webhook handler example (Next.js App Router)

```typescript
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const rawBody = await req.text(); // Must be raw, not parsed JSON
  const sig = req.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      rawBody,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return new Response("Invalid signature", { status: 400 });
  }

  // Queue for async processing, return 200 fast
  await webhookQueue.add("stripe-event", { eventId: event.id, type: event.type, data: event.data });

  return new Response(null, { status: 200 });
}
```

#### Key webhook events to handle

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Provision access, link customer to user via metadata |
| `customer.subscription.created` | Record subscription, set status to active |
| `customer.subscription.updated` | Sync plan, status, and `current_period_end` |
| `customer.subscription.deleted` | Revoke access, update local status |
| `invoice.paid` | Confirm renewal, extend access period |
| `invoice.payment_failed` | Start dunning flow (see below) |

### 3. Subscription Management

1. **Create subscriptions** with idempotency keys and metadata.
2. **Sync state from webhooks** — upsert the local subscription record on every `customer.subscription.*` event:

```typescript
async function handleSubscriptionUpdate(subscription: Stripe.Subscription) {
  await db.subscriptions.upsert({
    stripeId: subscription.id,
    status: subscription.status,
    currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    priceId: subscription.items.data[0].price.id,
  });
}
```

3. **Upgrades and downgrades** — always retrieve the subscription from Stripe before modifying; never rely on cached status.
4. **Cancellations** — prefer `cancel_at_period_end: true` so the customer retains access until the billing cycle ends.

### 4. Dunning (Failed Payment Recovery)

Handle `invoice.payment_failed` with a grace-period workflow:

| Day | Action |
|-----|--------|
| 0 | Payment fails — send email with update-card link (Customer Portal) |
| 3 | Reminder email |
| 7 | Final warning — service will be paused |
| 10 | Downgrade or suspend; send recovery email |

Stripe Smart Retries handle automatic re-attempts. Complement them with proactive emails. Expose the Stripe Customer Portal so users can self-serve payment method updates.

### 5. Stripe Connect (Marketplace / Platform)

Understand the three parties: **platform** (you), **connected account** (seller), **customer** (buyer).

- Use `stripeAccount` header for operations on a connected account.
- For destination charges, specify `transfer_data.destination` and the amount the connected account receives.
- Test from **both** the platform dashboard and the connected account dashboard.

### 6. Refunds

Before processing any refund:

- Verify the requesting user has authorization.
- Validate the amount does not exceed the original charge minus prior refunds.
- Require a `reason` string for audit logging.
- Use an idempotency key to prevent duplicate refunds.
- Log who triggered the refund, when, and why.

## Common Pitfalls

- **Parsing JSON before signature verification** — breaks `constructEvent()`; the signature is computed over the raw body.
- **Trusting API responses for access control** — payments are asynchronous; only webhooks confirm final state.
- **Missing metadata on Checkout Sessions** — makes it impossible to link a payment to a user in the webhook.
- **Hard-coding currency as USD** — store and display currency from the Price object; use `Intl.NumberFormat` for formatting.
- **Hard-coding price IDs** — store them in config or database so pricing changes do not require redeployment.
- **Synchronous webhook processing** — heavy work before returning 200 causes Stripe timeouts and retries.
- **Stale local subscription state** — always sync from webhooks or re-fetch from Stripe before decisions.

## Test Mode Checklist

- Use `sk_test_` / `pk_test_` keys from environment variables — never hard-code.
- Set up separate webhook endpoints for test and production with distinct `whsec_` secrets.
- Exercise failure paths with Stripe test card numbers:
  - `4242 4242 4242 4242` — success
  - `4000 0000 0000 0002` — generic decline
  - `4000 0000 0000 9995` — insufficient funds
- Use the Stripe CLI (`stripe listen --forward-to`) for local webhook testing.

## Reference System Usage

Ground all responses in the provided reference files and treat them as the authoritative source for this domain:

* **For creation tasks:** Consult **`references/patterns.md`** — it defines how Stripe objects and flows should be built, including idempotency, webhook state machines, and dunning.
* **For diagnosis and debugging:** Consult **`references/sharp_edges.md`** — it catalogues critical failure modes (signature bypass, raw-body issues, state mismatch) with symptoms and solutions.
* **For code review and validation:** Consult **`references/validations.md`** — it contains regex-based rules and severity ratings for catching unsafe Stripe code patterns.

If a user request conflicts with guidance in these files, explain the risk using the reference material and recommend the safe approach.
