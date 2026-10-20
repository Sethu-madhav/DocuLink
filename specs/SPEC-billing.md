# Spec: Billing and entitlements

Module id: `billing`
Status: Approved
Implements: MVP.md 1.1 (Stripe subscription), 5.3 (billing states, Billing tab), 9.1 (`/api/billing/*`), 9.3 (Stripe webhook), 11, 12 (`trial_converted`)
Designs: none specific (Settings shell from `5 Settings - 01`); design system DS 07 (banner, buttons)
Reading doc: [docs/specs/billing.md](../docs/specs/billing.md)
Last updated: 2026-10-20

## Objective

Charge for the product without ever getting in the client's way:
- A 14-day trial needs no card.
- The Solo plan is $29 a month or $290 a year, with US sales tax handled by Stripe Tax.
- A lapsed subscription blocks only creating and sending requests. Review, download and every client link keep working (MVP.md 0 row 19).

Success means:
- the right tax is charged for a taxable test address, monthly and annual
- entitlements are correct for every subscription state
- webhooks are idempotent and handle events arriving out of order

## Tech Stack

- `stripe` (Node SDK, pinned API version).
- Foundation (`defineRoute`, Redis for event dedupe, `withTx`, `track`), `auth` (`requireWorkspace`), `ui` (Banner, Dialog).

## Commands

```
Test:         pnpm vitest run src/server/billing
Integration:  pnpm vitest run --project integration src/server/billing
Webhooks dev: stripe listen --forward-to localhost:3000/api/webhooks/stripe
Fixtures:     stripe trigger customer.subscription.updated
```

## Project Structure

```
src/server/billing/index.ts          → getEntitlements, BillingBanner data, panel queries
src/server/billing/entitlements.ts   → pure rules
src/server/billing/stripe.ts         → client, checkout, portal, syncSubscription
src/server/billing/webhook.ts        → handleStripeEvent
src/app/api/billing/checkout/route.ts, portal/route.ts
src/app/api/webhooks/stripe/route.ts
src/components/billing/              → BillingPanel (mounted in settings), TrialBanner, UpgradeDialog
```

## Interfaces

### Entitlements (pure; built first, in week 2, so `requests` can depend on it)

```ts
getEntitlements(w: Pick<Workspace, "subscriptionStatus" | "trialEndsAt" | "currentPeriodEnd" | "pastDueSince">, now: Date): {
  canCreate: boolean; canSend: boolean;              // the only gated capabilities
  state: "trial" | "active" | "past_due_grace" | "lapsed";
  banner: null | { kind: "trial"; daysLeft: number } | { kind: "past_due"; graceDaysLeft: number } | { kind: "lapsed" };
}
```

| `subscription_status` | Condition | State | Can create and send |
|---|---|---|---|
| trialing | `trial_ends_at > now` | trial (banner when 5 days or fewer are left) | yes |
| trialing | `trial_ends_at <= now` | lapsed | no |
| active | — | active | yes |
| past_due | `past_due_since + 7 days > now` | past_due_grace (banner) | yes |
| past_due | otherwise | lapsed | no |
| canceled | `current_period_end > now` | active (Stripe cancel at period end) | yes |
| canceled | otherwise | lapsed | no |

When `canCreate` or `canSend` is false, `requests` throws `PAYMENT_REQUIRED`. The UI shows the `UpgradeDialog`: "Your trial has ended. Choose a plan to keep sending requests. Your clients can still finish their uploads, and you can still review and download." with Monthly or Annual, then Checkout.

### Checkout: `POST /api/billing/checkout`

- **Body:** `{ interval: "month" | "year" }`. Guard: `requireWorkspace`.
- **Customer:** create the Stripe customer on first use (`metadata.workspace_id`, email = user email, name = business name) and store `stripe_customer_id`.
- **Session:**
  - `mode: "subscription"`, with the price for the interval
  - `automatic_tax: { enabled: true }`, `billing_address_collection: "required"`, `customer_update: { address: "auto", name: "auto" }`, `tax_id_collection: { enabled: true }`
  - `allow_promotion_codes: true`, so beta firms can enter the beta code (MVP.md 11; setup below)
  - `subscription_data.metadata.workspace_id`
  - if the app trial is still running, `subscription_data.trial_end = trial_ends_at`, so the card isn't charged before the trial ends
  - `success_url = /app/settings/billing?checkout=success`, `cancel_url = /app/settings/billing`
- **Response:** `{ url }`.
- If the workspace already has an active subscription → 409 "You already have a plan. Use Manage billing to change it."

### Beta promotion code (MVP.md 11)

Set up once per Stripe account (test mode on staging) and documented in the runbook, not created by code:
- a coupon with `duration: "repeating"`, `duration_in_months: 12`, `applies_to` the Solo product. The amount is whatever beta pricing settles on.
- a promotion code `BETA2027` on it, with `max_redemptions: 20`, `restrictions.first_time_transaction: true`, and `expires_at` 30 days after public launch

The app has no code path for it beyond `allow_promotion_codes`. Redemptions show up in the normal subscription sync.

### Customer portal: `POST /api/billing/portal`

Guard: `requireWorkspace`, and a customer must exist. Creates a Billing Portal session with `return_url`. The portal configuration (set once per Stripe account and documented) allows switching between monthly and annual, updating the payment method, cancelling at period end and viewing invoices.

### Webhook: `POST /api/webhooks/stripe`

- Verify the signature with `stripe.webhooks.constructEvent` on the raw body (`await req.text()`).
- Dedupe with `SET stripe:evt:{id} NX EX 604800`; the key is released if the handler throws.
- Handled events (MVP.md 9.3): `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`.
- **Ordering-safe sync:** for every event, read `workspace_id` from the metadata (or via `stripe_customer_id`). Then **re-fetch the subscription from Stripe** and write its current state, instead of trusting the event payload. `syncSubscription(workspaceId, subscription)` sets:
  - `subscription_status` (Stripe `active`/`trialing` → as is; `past_due` → past_due; `canceled`, `unpaid` or `incomplete_expired` → canceled; `incomplete` → unchanged)
  - `plan = "solo"`, `billing_interval`, `current_period_end`, `trial_ends_at` (Stripe's trial end, if any)
  - `past_due_since` (set when entering past_due, cleared when leaving it)
- On the first transition to `active` from `trialing`: `track("trial_converted")`.
- `invoice.payment_failed` just runs a sync. Stripe sends its own dunning emails, and the banner covers the app.
- Unknown events → 200. A missing workspace → 200, plus a Sentry warning.

### UI

**`TrialBanner`**, under the requester header:

| State | Copy |
|---|---|
| Trial, 5 days or fewer left | "Trial ends in {n} days. Add billing to keep sending requests." |
| Past due | "We couldn't charge your card. Update it within {n} days to keep sending requests." |
| Lapsed | "Your plan has ended. Reviewing and downloading still work." |

All three include **Add billing**, **Update card** or **Choose plan** as appropriate. MVP.md 5.3 calls this a "quiet banner": it uses the neutral or review tone, and is never blocking.

**`BillingPanel`** (settings):
- plan "Solo", price and interval, status line ("Renews Nov 30, 2026", "Trial ends Nov 9", "Cancels Dec 1")
- **Manage billing** (portal), or **Choose plan** with a Monthly/Annual segmented control, showing "$29 a month" and "$290 a year, two months free"
- "Prices exclude sales tax where applicable."

## Data

Owns the billing columns on `workspaces`: `stripe_customer_id`, `subscription_status`, `plan`, `billing_interval`, `trial_ends_at`, `current_period_end`.

Column (in MVP.md 7 since v1.2): `workspaces.past_due_since timestamptz null`. MVP.md 11 says past due "keeps full access for 7 days", which needs the start of the 7 days.

## Code Style

```ts
// src/server/billing/entitlements.ts
export function getEntitlements(w: BillingFields, now: Date): Entitlements {
  switch (w.subscriptionStatus) {
    case "trialing": return w.trialEndsAt && w.trialEndsAt > now ? trial(w, now) : lapsed();
    case "active": return active();
    case "past_due": return w.pastDueSince && addDays(w.pastDueSince, 7) > now ? grace(w, now) : lapsed();
    case "canceled": return w.currentPeriodEnd && w.currentPeriodEnd > now ? active() : lapsed();
  }
}
```

## Testing Strategy

- **Unit:** `getEntitlements` for every row of the table, at the boundaries (exactly at `trial_ends_at`, day 7 of past due), and the banner copy.
- **Integration** (Postgres, `stripe-mock` or a recorded SDK, signed payloads via `stripe.webhooks.generateTestHeaderString`):
  - a bad signature → 400
  - a duplicate event → processed once
  - out-of-order `updated` before `created` → the final state is correct (the re-fetch)
  - past due sets and clears `past_due_since`
  - `trial_converted` tracked once
  - checkout during the trial sets `trial_end`
  - checkout with an active subscription → 409
- **E2E (staging, Stripe test mode):** trial → Choose plan (annual) → Checkout with a test card and a taxable address (for example, a New York ZIP) → tax line present → back in the app, status active. Then the same for monthly. A lapsed workspace (fixture) → New request opens the upgrade dialog, while review and ZIP still work.

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| Checkout charges the right sales tax for a test address in a taxable state, for both monthly and annual prices | Staging E2E above (Stripe test mode with Tax enabled and a test registration) |
| A lapsed subscription blocks sending new requests but not client uploads, review or download | Integration in `requests`, plus the E2E above; `portal`, `review` and `exports` never call entitlements (a code search test asserts this) |

## Edge Cases

- **The user completes Checkout but the webhook is delayed** → the success page calls `syncSubscription` directly, using the session id from the URL (verified through the Stripe API), so the app updates immediately. The webhook later is a no-op.
- **The customer was deleted in Stripe** → the next portal call fails. A Sentry alert fires, and the Billing panel shows "Contact support".
- **Interval switch in the portal** → `customer.subscription.updated` → the sync writes `billing_interval`.
- **Refund or dispute** → not handled in the app. Stripe's dashboard covers it, and status changes arrive through subscription events.
- **Account deletion** → `settings` calls `billing.deleteCustomer(workspace)`, which cancels the subscription immediately (no proration) and deletes the customer.

## Boundaries

- **Always:**
  - verify signatures
  - re-fetch before writing
  - gate only create and send
  - Stripe Tax on every checkout
- **Ask first:**
  - prices, a Team plan, coupons beyond the beta code
  - usage limits
  - Stripe API version upgrades
- **Never:**
  - gate client uploads, review or downloads
  - store card data (Stripe only)
  - trust webhook payload ordering

## Success Criteria

- [ ] The entitlements table is implemented and fully unit tested. `requests` uses it from week 2.
- [ ] Checkout (monthly and annual, with tax), the portal and the webhook sync work on staging in test mode.
- [ ] The banners and the upgrade dialog match the copy above. Lapsed gating is verified end to end.
- [ ] `trial_converted` is tracked once per workspace.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 44 and 45)

1. **MVP.md 7:** `workspaces.past_due_since`.
2. **MVP.md 11:** when a workspace subscribes during its trial, Stripe's `trial_end` is set to the app's trial end, so the first charge happens when the trial ends. Beta pricing is a promotion code at Checkout.

## Decisions (Oct 20, 2026)

1. **Beta pricing:** a promotion code (`BETA2027`) entered at Checkout.
