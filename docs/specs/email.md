# Email: every message DocuLink sends

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-email.md](../../specs/SPEC-email.md) · Product spec: MVP.md 4.8, 5.7, 8.4

## In one paragraph
All emails go through one place: invites, reminders, change requests, receipts, digests, sign-in links and security alerts. It makes sure client emails come from the firm ("Acme Accounting via DocuLink"), with replies going to the firm. That matters because an unknown sender asking for a driver's license looks like phishing. It also tracks whether each email arrived. If an invite bounces because of a typo in the address, the request is flagged "Invite bounced" within minutes so the accountant can fix it.

## Who it's for
- **Clients**, who receive invites, reminders and receipts.
- **Accountants**, who receive digests, alerts and sign-in links.

## How it works
1. **Sending.** A feature asks for an email of a given kind, such as an invite or a reminder.
2. **Building it.** The email is built from a template: one column, one button, the full link written out underneath, and a plain-text version.
3. **Delivery.** It goes out through Amazon's email service, and we keep a record of it.
4. **Tracking.** Amazon tells us when the email was delivered, bounced, or marked as spam. A bounced invite, reminder or change request flags the request and emails the accountant.
5. **No duplicates.** If a background job retries, the email isn't sent twice.

## Key decisions
| Decision | Why |
|---|---|
| Firm name in the sender, firm address for replies | Trust; clients reply to the people they know |
| Never attach documents or name files | Emails can be forwarded and show up on lock screens |
| No open or click tracking | It would confuse "link opened" with email scanners, and it isn't needed |
| A switchable provider | If Amazon's approval is slow, a backup provider can be dropped in |

## Not included
- Marketing email or newsletters.
- The "files expiring soon" and "retention warning" emails, which are should-haves after launch.

## Decided
- Emails come from requests@mail.doculinkapp.com, shown as "Acme Accounting via DocuLink" on client emails.
- Reminder emails include a one-click "Stop reminders for this request" option. It stops reminders for that one request only; the accountant sees that the client turned them off, and change requests still arrive.
