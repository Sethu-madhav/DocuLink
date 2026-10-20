# Settings: workspace, onboarding and leaving

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-settings.md](../../specs/SPEC-settings.md) · Product spec: MVP.md 5.3, 10

## In one paragraph
Settings is where the accountant decides how DocuLink represents them to clients (name, logo, reply-to email, time zone) and how long client files are kept. Security and billing appear here too. It also handles the end of the relationship: deleting an account removes every document, every record and the billing account.

## Who it's for
Accountants.

## How it works
1. **Workspace tab:**
   - business name, logo (with a preview of what clients see), reply-to email and time zone
   - how many days after completion files are deleted (30, 90, or any number you choose up to 365), and default reminders
   - changes are saved together with "Save changes", or thrown away with "Discard"
2. **Security tab:** two-factor, devices and sessions (from the auth module).
3. **Billing tab:** plan and "Manage billing" (from the billing module).
4. **Account tab:**
   - connect or disconnect Google
   - delete the account, which means typing the business name and passing two-factor
5. **Onboarding** after sign-up:
   1. business name and time zone
   2. logo and reply-to, which can be skipped
   3. "Send your first request": pick a template group (Tax files, Accounting, HR, Legal, Universities), then a template, and the request builder opens ready to go
6. **Deleting an account:**
   1. Sign everyone out.
   2. Delete every stored file and every copy.
   3. Delete all records.
   4. Cancel and remove the Stripe customer.
   5. Send a confirmation email.

## Key decisions
| Decision | Why |
|---|---|
| Defaults live on the Workspace tab | Matches the design; fewer tabs |
| Templates live on their own Templates page | They're used day to day, not set once |
| Changing retention warns before anything gets deleted | Shortening it can delete older files that night |
| Deletion runs in the background, files first | Large accounts take time; documents must go first |

## Not included
- The Team tab (hidden until the Team plan).
- Downloading all data before deleting (later).

## Decided
- Files are kept 30 days, 90 days (the default), or a custom number of days from 1 to 365.
- A change applies to requests already completed. If it would delete files, a warning says how many requests first.
