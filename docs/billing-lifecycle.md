# Billing Lifecycle

End-to-end flow from initial signup through subscription management, covering checkout, subscription state transitions, dunning, and cancellation.

## Checkout Flow

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant BE as Backend
    participant SS as stripe_service
    participant S as Stripe
    participant DB as Supabase

    U->>FE: Clicks "Subscribe" on Pricing page
    FE->>BE: POST /subscriptions/checkout<br/>{ plan_tier, success_url, cancel_url, team_seats? }
    BE->>SS: create_checkout_session(user_id, plan_tier, ...)

    SS->>DB: SELECT stripe_customer_id FROM subscriptions
    alt No existing customer
        SS->>S: stripe.Customer.create(email, metadata)
        S-->>SS: cus_xxx
        SS->>DB: UPSERT subscriptions (stripe_customer_id)
    end

    SS->>S: stripe.checkout.Session.create(<br/>  customer=cus_xxx,<br/>  line_items=[{price_id, quantity}],<br/>  mode=subscription,<br/>  metadata={user_id, plan_tier}<br/>)
    S-->>SS: session.url, session.id
    SS-->>BE: { checkout_url, session_id }
    BE-->>FE: { checkout_url, session_id }
    FE->>U: Redirect to Stripe Hosted Checkout

    U->>S: Completes payment
    S->>FE: Redirect to success_url?subscribed=true&plan=pro
    S->>BE: POST /webhooks/stripe (checkout.session.completed)
    BE->>DB: UPSERT subscriptions (status=active, plan_tier, stripe IDs)
    BE->>DB: UPDATE user_quotas (sync to paid tier)
```

---

## Subscription State Machine

```mermaid
stateDiagram-v2
    [*] --> Free: User signs up

    Free --> CheckoutInitiated: POST /subscriptions/checkout
    CheckoutInitiated --> Active: checkout.session.completed\n(payment succeeds)
    CheckoutInitiated --> Free: User abandons checkout

    Active --> Trialing: Trial started\n(if trial period configured)
    Trialing --> Active: Trial ends, first invoice paid

    Active --> PastDue: invoice.payment_failed\n(card declined, expired)
    PastDue --> Active: invoice.payment_succeeded\n(retry succeeds)
    PastDue --> Canceled: Stripe gives up retrying\n(customer.subscription.deleted)

    Active --> CancelScheduled: POST /subscriptions/cancel\n(cancel_immediately=false)
    CancelScheduled --> Canceled: Billing period ends\n(customer.subscription.deleted)
    Active --> Canceled: POST /subscriptions/cancel\n(cancel_immediately=true)

    Canceled --> Free: customer.subscription.deleted\nhandler downgrades quota

    note right of PastDue
        Dunning grace policy:
        past_due retains full
        paid-tier access.
        last_payment_failed_at
        is set for UI banner.
    end note

    note right of CancelScheduled
        cancel_at_period_end=true
        User retains access
        until period end.
    end note
```

---

## Dunning Policy

When a payment fails, Stripe retries automatically over several days. Noesis retains paid access throughout this window:

| Status | Access Level | `last_payment_failed_at` |
|--------|-------------|--------------------------|
| `active` | Full paid access | `NULL` |
| `past_due` | Full paid access (grace) | Set to failure timestamp |
| `canceled` / `incomplete` | Free tier only | Preserved (audit trail) |

On payment recovery (`invoice.payment_succeeded`), `last_payment_failed_at` is cleared and `status` returns to `active`. The frontend can read `last_payment_failed_at` to show a payment failure banner without hitting the Stripe API.

---

## Cancellation Flows

```mermaid
flowchart TD
    A[POST /subscriptions/cancel] --> B{cancel_immediately?}

    B -- false --> C[stripe.Subscription.modify\ncancel_at_period_end=true]
    C --> D[UPDATE subscriptions\ncancel_at_period_end=true]
    D --> E[User retains access\nuntil period end]
    E --> F[Stripe fires\ncustomer.subscription.deleted\nat period end]
    F --> G[Downgrade to free]

    B -- true --> H[stripe.Subscription.delete]
    H --> I[Stripe fires\ncustomer.subscription.deleted\nimmediately]
    I --> G
```

---

## Billing Portal

Users can manage their payment method, view invoices, and update their subscription directly via the Stripe Customer Portal:

```
POST /subscriptions/portal-session
→ stripe.billing_portal.Session.create(customer=cus_xxx, return_url=...)
→ { url: "https://billing.stripe.com/session/..." }
```

The portal is hosted and managed entirely by Stripe — no custom invoice or payment-method UI is needed on the Noesis side.

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
