# Spec: Requests, state machines, builder and dashboard

Module id: `requests`
Status: Approved
Implements: MVP.md 3 (state machines), 4.1, 4.7, 4.8, 5.3 (dashboard, builder, templates), 5.7 (reminder opt-out), 7 (`document_requests`, `checklist_items`, `templates`, `reminder_schedules`, request-scoped `activity_events` reads), 9.1 (request actions), 10 (tokens)
Designs: `2 Requester dashboard - 01 Requests.png`, `2 Requester dashboard - 02 New request.png`; design system DS 07 (request items, request label, progress and rows)
Reading doc: [docs/specs/requests.md](../docs/specs/requests.md)
Last updated: 2026-10-20

## Objective

The requester's core loop up to the point the client takes over:
- build a checklist, from scratch, a starter template or a saved template
- send it as one secure link
- watch it on a dashboard grouped by who has the next move
- manage the request's life: remind, fix a bounced email, regenerate the link, extend the deadline, revoke, delete files

`requests` also owns the business rules everyone else relies on:
- the request and item state machines (MVP.md 3)
- the link token
- the reminder schedule state
- the digest trigger

Success means:
- a requester can send their first request from a template in under 3 minutes after onboarding
- every status change in the product goes through one tested transition function

## Tech Stack

Foundation conventions (zod, `defineAction`, `withTx`, `logEvent`, `crypto`, `jobs`, `rateLimit`), plus:
- `auth` guards
- `email.send`
- `billing.getEntitlements`
- `ui` components
- `@dnd-kit/core` and `@dnd-kit/sortable` for drag-to-reorder, with keyboard sensors (MVP.md 5.9)
- `@date-fns/tz` for local-morning and expiry math

## Commands

```
Test:         pnpm vitest run src/server/requests
Integration:  pnpm vitest run --project integration src/server/requests
E2E:          pnpm test:e2e e2e/requests
```

## Project Structure

```
src/server/requests/index.ts        → Public API (rules, services other modules call, queries)
src/server/requests/schemas.ts      → zod: draftSchema, sendSchema, itemSchema, templateSchema
src/server/requests/rules.ts        → transition(), itemTransition(), canEdit(), expiryFor(), nextLocalMorning(), progress()
src/server/requests/token.ts        → issueToken, findByToken, buildPortalUrl
src/server/requests/service.ts      → create/update/send/duplicate/delete/remind/regenerate/fixEmail/extend/revoke/deleteFiles
src/server/requests/reminders.ts    → createSchedule, resumeReminders, stopReminders, setInterval
src/server/requests/digest.ts       → noteActivity, advanceDigestCursor
src/server/requests/bounces.ts      → markInviteBounced
src/server/requests/infected.ts     → handleInfectedFile
src/server/requests/templates.ts    → template CRUD, STARTER_TEMPLATES, TEMPLATE_CATEGORIES (order and labels)
src/server/requests/queries.ts      → dashboard, request header, activity timeline
src/server/requests/actions.ts      → Server Actions
src/app/(app)/app/page.tsx                    → Dashboard
src/app/(app)/app/requests/new/page.tsx       → Builder (new, from template)
src/app/(app)/app/requests/[id]/edit/page.tsx → Builder (draft or sent edit)
src/app/(app)/app/templates/                  → Templates list and editor
src/components/requests/template-categories.ts → category → Lucide icon map (client side)
```

## Interfaces

### State machines (pure, `rules.ts`)

```ts
type RequestEvent =
  | "SEND" | "FIRST_UPLOAD" | "SUBMIT" | "SEND_CHANGES" | "RESUBMIT" | "ALL_APPROVED"
  | "REOPEN" | "EXPIRE" | "EXTEND_WITH_FILES" | "EXTEND_NO_FILES" | "REVOKE";
transition(from: RequestStatus, event: RequestEvent): RequestStatus   // throws AppError("CONFLICT") if not allowed
```

| From → event | To |
|---|---|
| draft → SEND | pending |
| pending → FIRST_UPLOAD | in_progress |
| in_progress → SUBMIT | submitted |
| submitted → SEND_CHANGES | changes_requested |
| changes_requested → RESUBMIT | submitted |
| submitted → ALL_APPROVED | completed |
| completed → REOPEN | changes_requested |
| pending, in_progress, changes_requested → EXPIRE | expired |
| expired → EXTEND_WITH_FILES | in_progress |
| expired → EXTEND_NO_FILES | pending |
| pending, in_progress, submitted, changes_requested, expired → REVOKE | cancelled |

Submitting needs at least one active file (SPEC-portal), and the first confirmed upload already moved the request to `in_progress`, so `pending` never submits directly. This matches MVP.md 3.1.

`REOPEN` is applied only by `review.sendChangeRequests` on a completed request whose files aren't deleted (MVP.md 3.1, 0 row 39). The same transaction clears `completed_at`, sets `expires_at = max(expires_at, now + 14 days)` and resumes reminders.

Item transitions (`itemTransition`):

| From → event | To |
|---|---|
| pending → FILE_ACTIVE | uploaded |
| uploaded → LAST_FILE_REMOVED | pending |
| uploaded → APPROVE | approved |
| approved → UNAPPROVE | uploaded |
| uploaded → CHANGES_SENT | rejected |
| rejected → RESUBMITTED | uploaded |

How the item states behave:
- **Approve and unapprove** are allowed while the request is `submitted` or `changes_requested`, and on a `completed` request being reopened (SPEC-review).
- **Rejected items** keep `rejected` while the client adds or removes files.
- **Drafted changes** are not a status. An item with `rejection_note` set and `rejection_sent_at` null is "Change drafted" on the requester side and still `uploaded`.

Other pure rules:

```ts
expiryFor({ dueDate, sentAt, tz }): Date   // dueDate → end of (dueDate + 14 days) in tz; else sentAt + 60 days
nextLocalMorning(after: Date, tz: string, rand = Math.random): Date   // random minute 09:00–10:59 local on the first day ≥ after's local date; DST-safe
progress(items, files): { required: { done, total }, optional: { done, total }, label }  // "1 of 2 required added"
canEdit(status): { title, message, dueDate, addItems, items, recipient, reminders }   // the edit matrix below
allowedActions(request): Set<ActionName>   // drives menus; same rules the services enforce; includes "reopen" for completed requests with files
```

### Token (`token.ts`)

- **`issueToken(tx, requestId)`:**
  1. `token = randomToken(32)`.
  2. Stores `token_hash = sha256Hex(token)` and `token_ciphertext = encryptToken(token)`.
  3. Returns the token for immediate use. It is never logged.
- **`findByToken(token)`:**
  - Validates the token's shape first: 43 base64url characters. Anything else gets the same "invalid" result without a database call.
  - Then looks up by `token_hash`.
  - Returns the request with its workspace branding, or null.
- **`buildPortalUrl(request, anchor?)`:** decrypts the ciphertext and returns `${APP_URL}/c/${token}${anchor ? "#item-" + id : ""}`. Used for reminders, change requests, resends and Copy link.
- **Regenerate** overwrites both columns, so the old token immediately resolves to "invalid".
- **Revoke** clears both columns.

### Input validation (`schemas.ts`)

| Field | Draft save | Send / sent edit |
|---|---|---|
| `recipientName` | trimmed, ≤ 100, may be empty | 1–100, required |
| `recipientEmail` | trimmed, lowercased, ≤ 254, may be empty | valid email (zod `email()`), required |
| `title` | trimmed, ≤ 120, may be empty | 1–120, required |
| `message` | ≤ 2,000 | ≤ 2,000 |
| `dueDate` | `YYYY-MM-DD` or null | null or ≥ today in the workspace time zone and ≤ today + 2 years |
| `reminderDays` | null, 2, 3, 5 or 7 | same |
| `items` | 0–50 | 1–50 |
| `items[].label` | trimmed, ≤ 80 | 1–80 |
| `items[].instructions` | ≤ 500 | ≤ 500 |
| `items[].reason` ("Why:") | ≤ 200 | ≤ 200 |
| `items[].required` | boolean | boolean |
| template `category` | `tax`, `accounting`, `hr`, `legal`, `university` or `custom` | same |

All strings are NFC normalized, with control characters stripped (newlines are allowed in `message` and `instructions`). Field error copy follows DS 09, for example "Enter an email address, like name@example.com" and "Add at least one item."

### Edit matrix (`canEdit`)

| Status | Recipient | Title, message | Due date | Items (edit, remove, reorder) | Add items | Reminders |
|---|---|---|---|---|---|---|
| draft | yes | yes | yes | yes | yes | yes |
| pending, in_progress | via Fix email only | yes | yes (recomputes `expires_at`) | no | yes (appended) | yes |
| changes_requested | via Fix email only | yes | yes | no | no | yes |
| submitted | no | yes | no | no | no | no |
| completed, expired, cancelled | no | no | no (expired: use Extend) | no | no | no |

### Actions (Server Actions, all with `requireWorkspace` and `requireOnboarded`)

| Action | Allowed when | Effects (one `withTx`; side effects after commit) |
|---|---|---|
| `createRequest(draft, { templateId? })` | entitlements `canCreate` | Assigns `number` (see Data); inserts the request (draft), items (positions 1..n), and a schedule placeholder holding `reminderDays` (default: the workspace's `default_reminder_days`); logs `request_created`; tracks `request_created`. Returns the id |
| `updateRequest(id, patch)` | edit matrix | Updates the allowed fields. Item reorder rewrites positions in one statement. Sent-request due-date change → recompute `expires_at`. Interval change → `setInterval` |
| `sendRequest(id)` | draft; `canSend`; `sendRequest` rate limit; send schema valid | `issueToken`; `status = pending`, `sent_at`, `expires_at = expiryFor`; `createSchedule` (if reminders on: `next_send_at = nextLocalMorning(now + interval)`); logs `request_sent`; after commit: email `invite` (idempotency `invite:{id}:{tokenHash8}`); tracks `request_sent`. Toast "Request sent to {name}." |
| `duplicateRequest(id)` | any status | A new draft with a new number: same recipient, title "{title} (copy)", message, items (statuses reset, no notes), reminders. No files |
| `deleteDraft(id)` | draft | Hard delete of the request, its items and schedule, in a transaction that runs `SET LOCAL app.allow_event_delete = 'on'` (its `request_created` event cascades). MVP.md 3.1 |
| `sendReminderNow(id)` | pending, in_progress, changes_requested; not bounced; client hasn't stopped reminders; `reminderNow` limit (1 per 12 h) | After commit: email `reminder` (manual). Logs `reminder_sent` with `meta.manual = true`. Doesn't change `sent_count` or `next_send_at` |
| `regenerateLink(id, { resend })` | pending, in_progress, submitted, changes_requested, expired | `issueToken`; logs `link_regenerated`; if `resend`, email `invite` |
| `fixRecipientEmail(id, email)` | pending, in_progress, changes_requested | Updates `recipient_email`; `issueToken`; clears `invite_bounced_at`; `resumeReminders(now)`; logs `invite_resent`; email `invite` (MVP.md 4.8) |
| `extendDeadline(id, dueDate)` | pending, in_progress, changes_requested, expired | Sets `due_date` and `expires_at = expiryFor`. If expired: transition `EXTEND_WITH_FILES` or `EXTEND_NO_FILES` (any active file on the request) and `resumeReminders(now)`; logs `deadline_extended` |
| `revokeRequest(id)` | pending, in_progress, submitted, changes_requested, expired | `cancelled`, `cancelled_at`; clears the token columns; `stopReminders`; cancels the digest; logs `request_revoked` |
| `deleteFiles(id, confirmName)` | completed, cancelled, expired; `requireTwoFactor`; `confirmName` equals `recipient_name` exactly, case-insensitive | Marks every file `removed`; sets `files_deleted_at`; logs `files_deleted`; after commit: `files.deleteRequestObjects(id)` (all versions, including exports) |
| `saveTemplate({ name, category, items })`, `updateTemplate`, `deleteTemplate` | any | Template CRUD. `name` 1–80, unique per workspace (case-insensitive). `category` defaults to `custom`. Items follow the same schema, 1–50. Saving a starter creates a workspace copy that keeps the starter's category |

Every action returns `NOT_FOUND` for ids outside the workspace and `CONFLICT` with a plain message for disallowed states. For example: "This request was already submitted, so the checklist can't change."

### Functions other modules call (`index.ts`)

```ts
// state (all take tx; all use conditional updates WHERE status = expected)
applyTransition(tx, requestId, event, patch?)                 // validates with transition(), updates status + timestamps
applyItemTransition(tx, itemId, event, patch?)
getRequestForUpdate(tx, workspaceId | null, requestId)        // null workspace only for system/portal callers
// reminders
createSchedule(tx, requestId, intervalDays | null, from, tz)
resumeReminders(tx, requestId, from)                          // next_send_at = nextLocalMorning(from + interval, tz) if sent_count < max and not stopped by the recipient
stopReminders(tx, requestId)
stopRemindersByRecipient(requestId)                           // unsubscribe link: is_active = false, stopped_by_recipient_at = now, logs reminders_stopped (recipient); idempotent
// digest (MVP.md 8.1)
noteActivity(requestId)                                       // Redis dg:last/dg:first/dg:job keys + publishJob("/api/jobs/digest", { rid }, { delaySeconds: 600 })
advanceDigestCursor(tx, requestId)                            // last_digest_event_id = max(id) and clears the Redis keys after commit
// hooks wired in the AWS events route
markInviteBounced({ requestId, kind, type, to })
handleInfectedFile({ requestId, itemId, fileId })
// portal
findByToken(token), buildPortalUrl(request, anchor?)
// queries
getRequestHeader(ctx, id), listActivity(ctx, id, { cursor })
```

**`markInviteBounced`:**
- Ignores the event unless `to` equals the current `recipient_email` (a late bounce for an address that was already fixed), the request is in pending, in_progress or changes_requested, and `invite_bounced_at` is null.
- Otherwise: sets `invite_bounced_at`, and logs `email_bounced` or `email_complained`.
- After commit: email `invite_bounced` to the request's creator (idempotency `bounce:{id}:{to}`).
- Reminders pause because the reminder query excludes bounced requests (MVP.md 8.2).

**`handleInfectedFile`:**
- If the request is pending, in_progress or changes_requested: the file becomes `removed`, and the item takes `LAST_FILE_REMOVED` if it was `uploaded` and has no other active files. Rejected items stay rejected.
- Otherwise: no state change (the requester sees "Blocked").

### Queries and pages

**Dashboard** (`/app`), following the design (decided Oct 20, 2026; MVP.md 5.3):

- **Header** (mono date) and a display headline:
  - "{n} to review." when there are submitted requests
  - "Nothing to review." otherwise
  - "Next due: {client}, {title}, {relative}." from the soonest due date among waiting requests
- **Count list.** These act as the tabs:

  | Tab | Statuses | Notes |
  |---|---|---|
  | Needs review | submitted | Default when non-empty |
  | Waiting on client | pending, in_progress, changes_requested, expired | |
  | Completed | completed, cancelled | Revoked rows show the "Revoked" badge |
  | Drafts | draft | |

- **Needs review cards** (at most 4, then "Show all"): No., client, title, "{n} to review", "Submitted {relative}", **Review** (one primary per view; the others use the secondary style).
- **Waiting table:** client, request, progress ("2 of 4"), status badge, due ("Tomorrow" or "Overdue" in attention or danger), and a row action:
  - "Send reminder" for active requests
  - "Reopen" for expired (opens Extend deadline)
  - "Fix email" for bounced

  Sorting:
  - bounced first
  - then by last activity, newest first; last activity is the latest `activity_events` row of a meaningful type, shown as "Opened 3h ago" or "Uploaded 2 files 20m ago"
- **Row menu:** Open, Copy link, Send reminder now, Fix email (only when bounced), Duplicate, Revoke (MVP.md 5.3).
- **Search:** "/" focuses it. Matches client name, email and title with `ILIKE`, with `%` and `_` escaped. At least 2 characters, debounced 250 ms. Searches all tabs.
- **Pagination:** keyset on `(updated_at, id)`, 50 per page.
- **Empty state (first run):** "Send your first request. Pick a template or start from scratch."
- **Mobile:** stacked cards; the tabs become a segmented control.

**Builder** (`/app/requests/new`, `/app/requests/{id}/edit`), following design `2 … 02`:
- **Title** in display type, editable in place.
- **Fields:** Client (name, email), Due date, Reminders (Every 3 days; "9 to 11 am their time, at most 5"), then **Items**.
- **Items:** folio numbers, label, instructions, "Why:" reason, a Required switch, a drag handle with keyboard up and down, "Add item", and Enter in the last label adds an item.
- **Template picker:** "Template: Individual tax return". It opens a menu grouped by category (icon and label per group), with the workspace's own templates under Custom.
- **Header actions:** **Save draft** (secondary) and **Send request** (primary).
- **Phone and desktop preview** of what the client sees. It's built from `ui` patterns with the draft data and the workspace logo; it is not the live portal.
- **Validation:** inline on blur with `sendSchema`. Save draft uses `draftSchema`.
- **Send confirmation dialog** (MVP.md 5.3 copy): "Send to {email}? They'll get an email from {business} via DocuLink."
- **Lapsed workspace:** **Send request** opens `billing`'s upgrade dialog (`PAYMENT_REQUIRED`).
- **Save as template:** in an overflow menu.

**Templates** (`/app/templates`):
- Sections in this order: Tax files, Accounting, HR, Legal, Universities, Custom. Each section header shows its icon and label; a category filter (segmented control) sits above them.
- Each row: name, item count, updated, and a "Starter" tag for built-in templates.
- The editor uses the same item editor as the builder, plus a Category select. Editing a starter creates a workspace copy; the starter stays available.
- Custom shows "Templates you save appear here." when empty.

Categories (`TEMPLATE_CATEGORIES`, icons from `lucide-react`, always next to the text label per MVP.md 5.1):

| Category | Label | Icon |
|---|---|---|
| `tax` | Tax files | `Receipt` |
| `accounting` | Accounting | `Calculator` |
| `hr` | HR | `Users` |
| `legal` | Legal | `Scale` |
| `university` | Universities | `GraduationCap` |
| `custom` | Custom | `PencilRuler` |

### Starter templates (US)

Fifteen starters in five groups. Custom has no starters; it holds the requester's own templates. "Why" is the line clients see under each item.

**Tax files** (`tax`, `Receipt`)

*Individual tax return*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Photo ID | Yes | To confirm your identity when we file. |
| 2 | W-2 forms | Yes | They show your wages and the tax already withheld. |
| 3 | 1099 forms (INT, DIV, B, NEC, R) | No | They report other income, like interest or investments. |
| 4 | Form 1098 mortgage interest | No | It can lower your tax. |
| 5 | Form 1098-T tuition statement | No | For education credits. |
| 6 | Form 1095-A | No | Only if you had Marketplace health insurance. |
| 7 | Last year's tax return | No | So we can carry figures forward. |

*Self-employed and small business*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Photo ID | Yes | To confirm your identity when we file. |
| 2 | Income and expense summary or profit and loss | Yes | It's the basis of your Schedule C. |
| 3 | 1099-NEC and 1099-K forms | No | If clients or payment apps sent you one. |
| 4 | Business bank and credit card statements, all 12 months | No | To check income and expenses. |
| 5 | Vehicle mileage log | No | For the vehicle deduction. |
| 6 | Home office size in square feet | No | For the home office deduction. |
| 7 | Estimated tax payment records | No | So payments you already made are counted. |

*Rental property income*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Rent received for the year | Yes | It's reported on Schedule E. |
| 2 | Form 1098 mortgage interest | No | It's deductible. |
| 3 | Property tax bill | No | It's deductible, unless your lender paid it from escrow. |
| 4 | Repair, insurance and management invoices | No | They're deductible expenses. |
| 5 | Closing statement | No | Only if you bought or sold a property this year. |
| 6 | 1099-MISC forms | No | If a property manager or tenant sent one. |

**Accounting** (`accounting`, `Calculator`)

*New client onboarding*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Photo ID | Yes | To confirm who we're working with. |
| 2 | Signed engagement letter | Yes | It sets out what we'll do for you. |
| 3 | Last year's tax return | Yes | So we start from the right figures. |
| 4 | IRS or state notices | No | So nothing is missed. |
| 5 | EIN confirmation letter (CP 575) | No | For business clients. |
| 6 | Business formation documents | No | Articles, operating agreement or bylaws. |

*Monthly bookkeeping*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Bank statements for the month, every account | Yes | To reconcile your books. |
| 2 | Credit card statements for the month | Yes | To record expenses. |
| 3 | Receipts for business purchases | No | To back up each expense. |
| 4 | Sales or invoicing report | No | To record income. |
| 5 | Payroll reports | No | If you ran payroll this month. |
| 6 | Loan statements | No | To split principal and interest. |

*Year-end close*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | December bank statements, every account | Yes | To confirm year-end balances. |
| 2 | December credit card and loan statements | Yes | To confirm what you owe. |
| 3 | Invoices for large purchases | No | Equipment and vehicles are depreciated. |
| 4 | Inventory count at year end | No | If you hold stock. |
| 5 | Unpaid customer invoices and unpaid bills | No | For receivables and payables. |
| 6 | Contractor list with W-9 forms | No | So we can file 1099s by January 31. |

**HR** (`hr`, `Users`)

*New hire paperwork*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Form I-9 identity and work authorization documents | Yes | We must check these within 3 business days of your start date. Bring the originals on day one. |
| 2 | Form W-4 | Yes | It sets your federal tax withholding. |
| 3 | State withholding form | No | If your state has one. |
| 4 | Signed offer letter | Yes | For your employee file. |
| 5 | Direct deposit form or voided check | No | So we can pay you by direct deposit. |
| 6 | Emergency contact form | No | Who to call if something happens at work. |

*Contractor onboarding*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Form W-9 | Yes | We need your taxpayer details to issue a 1099. |
| 2 | Signed contractor agreement | Yes | It sets out the work and payment terms. |
| 3 | Certificate of insurance | No | If the work requires it. |
| 4 | Business license | No | If your work requires one. |

*Benefits enrollment*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Marriage certificate | No | To add a spouse to your coverage. |
| 2 | Birth or adoption certificates | No | To add children. |
| 3 | Proof that other coverage ended | No | To enroll outside open enrollment. |

**Legal** (`legal`, `Scale`)

*New client intake*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Photo ID | Yes | To confirm who we're representing. |
| 2 | Signed engagement letter | Yes | It sets out our fees and the work we'll do. |
| 3 | Documents about your matter | Yes | Letters, contracts, notices or emails that explain what happened. |
| 4 | Court papers you've received | No | So we don't miss a deadline. |
| 5 | Insurance policies | No | A policy may cover your costs or your claim. |

*Estate planning*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Photo ID | Yes | To confirm your identity. |
| 2 | Current will or trust | No | So we can update it rather than start over. |
| 3 | Deeds for real estate you own | No | To plan how the property passes. |
| 4 | Recent bank, brokerage and retirement statements | No | To list your assets. |
| 5 | Life insurance policies | No | To check who the beneficiaries are. |
| 6 | Existing powers of attorney and health care directives | No | So the new ones replace them cleanly. |

*Family law financial disclosure*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Tax returns, last 2 years | Yes | Courts use them to work out support. |
| 2 | Pay stubs, last 3 months | Yes | To show your current income. |
| 3 | Bank statements, last 6 months, every account | Yes | To list your assets. |
| 4 | Retirement and brokerage statements | No | These accounts are often divided. |
| 5 | Mortgage and loan statements | No | To list your debts. |
| 6 | Credit card statements, last 3 months | No | To list your debts and spending. |

**Universities** (`university`, `GraduationCap`)

*Admissions documents*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Transcripts from every school attended | Yes | To review your academic record. |
| 2 | Personal statement | Yes | It's part of your application. |
| 3 | Test scores | No | Only if you choose to submit them. |
| 4 | Resume | No | To show your work and activities. |
| 5 | Photo ID or passport | No | To confirm your identity. |

*Financial aid verification*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Verification worksheet | Yes | We're required to confirm your FAFSA information. |
| 2 | IRS tax return transcript (student and parents) | Yes | To confirm reported income. |
| 3 | W-2 forms | Yes | To confirm wages. |
| 4 | Proof of untaxed income | No | For example child support or veterans' benefits. |
| 5 | Photo ID | No | Only if you were selected for identity verification. |

*International student (I-20)*

| # | Item | Required | Why |
|---|---|---|---|
| 1 | Passport identity page | Yes | To issue your I-20. |
| 2 | Academic transcripts | Yes | To confirm your previous study. |
| 3 | Bank statements or sponsor letter | Yes | To show you can cover the first year of costs. |
| 4 | English test scores (TOEFL, IELTS or Duolingo) | No | If English isn't your first language. |
| 5 | Previous I-20s and visas | No | If you've studied in the US before. |

Content rules:
- Labels ≤ 80 characters and "Why" lines ≤ 200, as in the item schema. At most 8 items per template, so the portal fits on one phone scroll.
- Wording follows DS 09: plain, second person, no jargon without the form name.
- Never ask for Social Security cards or full account numbers in a label; the documents themselves carry what's needed.
- Starters are content, not law or tax advice. The template page says "Check these against your own requirements." under the Starter tag.
- Before launch, someone who works in each field reviews their group (tracked in the M5 checklist).

## Data

Owns `document_requests`, `checklist_items`, `templates` and `reminder_schedules`, and writes the request-level `activity_events`.

Schema additions (in MVP.md 7 since v1.2):

| Change | Why |
|---|---|
| `document_requests.number int not null`, `unique (workspace_id, number)` | "No. 0142" appears on every screen in the designs and the design system's request label |
| `workspaces.next_request_number int not null default 1` | Assigned with `UPDATE workspaces SET next_request_number = next_request_number + 1 … RETURNING` inside the create transaction, so there are no gaps or races |
| `checklist_items.reason text null`; template items JSON gains `reason` | The designs and DS principle "Show the ask" put "Why:" under every item |
| `templates.category template_category not null default 'custom'` | Groups templates as Tax files, Accounting, HR, Legal, Universities and Custom (MVP.md 0 row 37) |
| `reminder_schedules.stopped_by_recipient_at timestamptz null`; `next_send_at` nullable | The client's unsubscribe must survive `resumeReminders`; drafts have no send time yet |

The reminder schedule row is created at create time with `is_active = false` and `interval_days` from the builder, and activated at send. That way the builder's choice persists in drafts, which MVP.md 7 has no other column for.

## Code Style

```ts
// src/server/requests/rules.ts
const REQUEST_TRANSITIONS: Record<RequestStatus, Partial<Record<RequestEvent, RequestStatus>>> = {
  draft: { SEND: "pending" },
  pending: { FIRST_UPLOAD: "in_progress", EXPIRE: "expired", REVOKE: "cancelled" },
  in_progress: { SUBMIT: "submitted", EXPIRE: "expired", REVOKE: "cancelled" },
  submitted: { SEND_CHANGES: "changes_requested", ALL_APPROVED: "completed", REVOKE: "cancelled" },
  changes_requested: { RESUBMIT: "submitted", EXPIRE: "expired", REVOKE: "cancelled" },
  expired: { EXTEND_WITH_FILES: "in_progress", EXTEND_NO_FILES: "pending", REVOKE: "cancelled" },
  completed: { REOPEN: "changes_requested" },
  cancelled: {},
};

export function transition(from: RequestStatus, event: RequestEvent): RequestStatus {
  const to = REQUEST_TRANSITIONS[from][event];
  if (!to) throw new AppError("CONFLICT", conflictMessage(from, event));
  return to;
}
```

## Testing Strategy

- **Unit:**
  - Every (status, event) pair. Allowed pairs produce the table's result and all others throw. This is an exhaustive, table-driven test of 8 statuses × 11 events, plus the item table.
  - `expiryFor`: due-date edge cases, the DST change, no due date.
  - `nextLocalMorning`: always 09:00–10:59 local across a year of dates in 4 US zones plus `Pacific/Honolulu`, including DST transitions, with a seeded RNG.
  - `canEdit` and `allowedActions` for every status.
  - `STARTER_TEMPLATES`: every item passes `itemSchema`, every category except `custom` has at least one starter, names are unique.
  - Schemas: each field boundary.
  - `progress` labels: required only, and with optional items.
- **Integration (Postgres):**
  - `createRequest` assigns sequential numbers under concurrency (10 parallel creates give 10 distinct numbers)
  - send: token stored hashed and encrypted, schedule active, `invite` email in the outbox with a working link
  - the edit matrix enforced server-side
  - regenerate invalidates the old token
  - fix email: clears the bounce, resumes reminders, resends
  - extend from expired restores `pending` or `in_progress` correctly
  - revoke clears the token and stops reminders
  - deleteFiles: typed confirmation, requires two-factor, calls deletion
  - `markInviteBounced` ignores mismatched addresses and repeats
  - `handleInfectedFile` before and after submit
  - templates: unique names; category defaults to `custom`; saving a starter keeps its category
  - `stopRemindersByRecipient`: stops scheduled reminders, blocks `sendReminderNow`, survives `resumeReminders` after a change request
  - `deleteDraft` removes the draft and its events (trigger bypass), and the bypass doesn't leak to other transactions
  - every action returns 404 across workspaces (added to auth's table-driven test)
- **E2E (Playwright):**
  - the builder: add, reorder by keyboard, required toggle, validation, Save draft, Send, then the dashboard shows "Sent"
  - start from a starter template in each group; the Templates page shows the five sections with their icons
  - duplicate
  - revoke, then the portal shows the revoked screen
  - the bounce flow on staging (shared with `email`)

MVP.md 14 acceptance criteria owned or shared:

| Criterion | Test |
|---|---|
| An invite to the SES bounce simulator flags "Invite bounced" within 5 minutes, pauses reminders and emails the requester; fixing the email sends a new invite with a new link | Staging E2E (with `email`); integration for the pause and the new token |
| A revoked or expired link shows the correct screen and cannot presign uploads | Integration: revoke clears the token; `portal` owns the screen and presign test |
| A lapsed subscription blocks sending new requests but not client uploads, review or download | Integration: `createRequest` and `sendRequest` return `PAYMENT_REQUIRED` for a lapsed workspace (with `billing`) |

## Edge Cases

- **Two tabs send the same draft** → the second `sendRequest` fails the `WHERE status = 'draft'` update and returns `CONFLICT` "This request was already sent." One email only (idempotency key).
- **Due date set to today, sent late in the evening** → valid. `expires_at` is 14 days after the end of today, local time.
- **The workspace time zone changes after sending** → existing `expires_at` values stay. Future reminder times use the new zone.
- **Recipient email equals the requester's own email** (testing) → allowed.
- **The bounce arrives after the request was submitted** → ignored (not an active status). The requester already has the documents.
- **Item added to a sent request while the client has the page open** → the portal refreshes its checklist on the next API call (it returns the current item list).
- **A template with 50 items is applied to a draft that already has items** → replace or append is asked in a dialog. The 50-item cap is enforced.
- **Search input `%` or `_`** → escaped, so it matches literally.
- **Request number reaches 9999** → continues to 10000; the display width is fluid.
- **The client stopped reminders, then the requester sends change requests** → the change-request email still goes out (it's not a reminder), and reminders stay off.
- **A starter template changes in a later release** → workspace copies are unaffected; requests already created keep their own items.

## Boundaries

- **Always:**
  - change status only through `transition` and `applyTransition` with a conditional update
  - log an event for every state change
  - check entitlements on create and send
  - check workspace scope everywhere
- **Ask first:**
  - new statuses or transitions
  - changing expiry defaults or reminder limits
  - editing the starter template content after beta review
- **Never:**
  - log or return a token except to its intended use
  - let a client-facing route read request data without going through the token or portal session
  - edit `activity_events`

## Success Criteria

- [ ] Transition tables match MVP.md 3 and are exhaustively tested.
- [ ] Builder, send, dashboard and templates match the designs, with the fifteen starter templates in five groups plus Custom, each group with its icon.
- [ ] Every 9.1 request action works with the status rules above and has integration tests, including cross-workspace 404s.
- [ ] Tokens are stored hashed and encrypted. Regenerate and revoke invalidate old links immediately.
- [ ] Bounced invites are flagged, sorted first, and fixable, with the reminder pause verified.
- [ ] Request numbers are unique and sequential per workspace.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 37, 39, 43, 44, 45)

1. **MVP.md 7:** `document_requests.number`, `workspaces.next_request_number`, `checklist_items.reason` (plus `reason` in template items), `templates.category`, `reminder_schedules.stopped_by_recipient_at`, nullable `next_send_at`.
2. **MVP.md 3.1:** "Delete draft" hard-deletes the draft; `expired → cancelled` (revoke an expired request); `completed → changes_requested` (Reopen).
3. **MVP.md 9.1:** `sendReminderNow` doesn't count toward the 5 scheduled reminders, and is unavailable once the client stops reminders.
4. **MVP.md 5.3:** the dashboard follows the design's layout; the "Completed" tab also lists revoked requests; Templates are grouped by category.

## Decisions (Oct 20, 2026)

1. **Dashboard layout:** follow the design. The count list is the tab switcher.
2. **Starter templates:** grouped as Tax files, Accounting, HR, Legal, Universities and Custom, each with its own icon. Content chosen and corrected by us (you delegated it); a reviewer from each field checks it before launch (MVP.md 13 M5).
3. **Edit after send:** no. Sent requests allow title, due date, message and adding items only.
4. **Legal group:** added, with client intake, estate planning and family law financial disclosure.

## Open Questions

None.
