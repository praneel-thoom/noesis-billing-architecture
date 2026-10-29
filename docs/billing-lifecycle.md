# Billing Lifecycle

End-to-end flow from initial signup through subscription management, covering checkout, subscription state transitions, dunning, and cancellation.

## Checkout: Initiating a Session

```mermaid
sequenceDiagram
    participant U as User
    participant BE as Backend
    participant S as Stripe

    U->>BE: POST /subscriptions/checkout
    Note over BE: Look up or create Stripe customer
    BE->>S: Create Checkout Session
    S-->>BE: session.url
    BE-->>U: Redirect to Stripe Checkout
    U->>S: Completes payment
    S-->>U: Redirect to success_url
```

## Checkout: Post-Payment Processing

```mermaid
sequenceDiagram
    participant S as Stripe
    participant BE as Backend
    participant DB as Supabase

    S->>BE: POST /webhooks/stripe
    Note over BE: checkout.session.completed
    BE->>DB: UPSERT subscriptions
    Note over DB: status=active, plan_tier, stripe IDs
    BE->>DB: sync_user_quota_plan()
    Note over DB: Update limits without resetting counters
```

---

## Subscription State Machine

```mermaid
stateDiagram-v2
    [*] --> Free: signs up

    Free --> CheckoutInitiated: POST /subscriptions/checkout
    CheckoutInitiated --> Active: payment succeeds
    CheckoutInitiated --> Free: checkout abandoned

    Active --> Trialing: trial started
    Trialing --> Active: trial ends

    Active --> PastDue: invoice.payment_failed
    PastDue --> Active: invoice.payment_succeeded
    PastDue --> Canceled: subscription.deleted

    Active --> CancelScheduled: cancel at period end
    CancelScheduled --> Canceled: period ends
    Active --> Canceled: cancel immediately

    Canceled --> Free: quota downgraded
```

---

## Dunning Policy

When a payment fails, Stripe retries automatically over several days. Noesis retains paid access throughout this window:

| Status | Access Level | `last_payment_failed_at` |
|--------|-------------|--------------------------|
| `active` | Full paid access | `NULL` |
| `past_due` | Full paid access (grace) | Set to failure timestamp |
| `canceled` / `incomplete` | Free tier only | Preserved (audit trail) |

On payment recovery (`invoice.payment_succeeded`), `last_payment_failed_at` is cleared and status returns to `active`. The frontend reads `last_payment_failed_at` to show a payment failure banner without hitting the Stripe API.

---

## Cancellation Flows

```mermaid
flowchart TD
    A["POST /subscriptions/cancel"] --> B{"cancel_immediately?"}
    B -- false --> C["Modify: cancel_at_period_end=true"]
    B -- true --> D["Delete subscription now"]
    C --> E["Access until period end"]
    D --> F["Access ends immediately"]
    E --> G["subscription.deleted webhook\ndowngrades quota to free"]
    F --> G
```

---

## Billing Portal

Users manage payment methods, invoices, and plan changes via the Stripe Customer Portal:

```
POST /subscriptions/portal-session
→ stripe.billing_portal.Session.create(customer=cus_xxx, return_url=...)
→ { url: "https://billing.stripe.com/session/..." }
```

The portal is hosted by Stripe — no custom invoice or payment-method UI is needed on the Noesis side.

---

## Team Plan Seat Handling

The Team plan supports 2–3 paid seats. Seat count is adjustable at checkout and tracked in `subscriptions.seats`:

- **Checkout:** `adjustable_quantity` enabled with `minimum=2`, `maximum=3`
- **Webhook sync:** `items.data[0].quantity` from `customer.subscription.updated` is written to `seats`
- **Pricing:** $20 per seat per month
- **Quota:** all seats share a single user's quota row (team collaborator model not yet implemented)

---

## API Reference

| Endpoint | Auth | Purpose |
|----------|------|---------|
| `GET /subscriptions/plans` | None | Returns plan features and pricing |
| `POST /subscriptions/checkout` | Bearer | Creates Stripe Checkout session |
| `POST /subscriptions/cancel` | Bearer | Cancels subscription (immediate or at period end) |
| `GET /subscriptions/usage` | Bearer | Returns current usage counters and limits |
| `POST /subscriptions/portal-session` | Bearer | Creates Stripe Customer Portal session |
| `POST /webhooks/stripe` | Stripe signature | Receives and processes Stripe events |
