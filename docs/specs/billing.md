# Billing: trial, plans and what a lapsed account can do

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-billing.md](../../specs/SPEC-billing.md) · Product spec: MVP.md 5.3, 11

## In one paragraph
New accounts get 14 days free with no card. After that, the Solo plan costs $29 a month or $290 a year, with US sales tax worked out by Stripe. If payment stops, the accountant can't create or send new requests, but everything already in motion keeps working: clients can still upload, and the accountant can still review and download. A failed card gets 7 days' grace.

## Who it's for
Accountants (paying customers). Clients are never affected by billing.

## How it works
1. **Trial.** It starts at sign-up. Quiet banner in the last 5 days: "Trial ends in 5 days."
2. **Choose a plan.** Monthly or annual, through Stripe's checkout page, which collects the billing address so the right sales tax is charged. Subscribing during the trial doesn't charge until the trial ends.
3. **Manage billing.** Opens Stripe's own page to switch plans, update the card, see invoices or cancel.
4. **Staying in sync.** Stripe tells us about every change, and we always re-check the current state with Stripe before saving it, so messages arriving out of order can't confuse us.
5. **When it lapses.** Only "New request" and "Send request" are blocked. They open a dialog to choose a plan.

## Key decisions
| Decision | Why |
|---|---|
| Block only creating and sending | A lapsed card must never stop a client mid-upload |
| No card needed for the trial | Lower friction for busy firms in tax season |
| Stripe Tax from day one | Software is taxable in many US states |
| A 7-day grace period on failed payments | Cards expire; give people time to fix it |

## Not included
- The Team plan (after launch; pricing still being validated).
- Usage-based pricing.

## Decided
- Beta firms get their price through a promotion code entered at checkout.
