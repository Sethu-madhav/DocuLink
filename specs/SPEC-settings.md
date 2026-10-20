# Spec: Settings, onboarding steps 2–3 and account deletion

Module id: `settings`
Status: Approved
Implements: MVP.md 5.3 (Settings, onboarding steps 2 and 3), 7 (workspace settings columns), 10 (account deletion), 12 (`onboarding_completed`)
Designs: `5 Settings - 01 Workspace.png` (`5 Settings - 02 Team.png` is post-MVP and hidden); design system DS 07 (fields, segmented control, unsaved-changes pattern)
Reading doc: [docs/specs/settings.md](../docs/specs/settings.md)
Last updated: 2026-10-20

## Objective

Let the requester set up how DocuLink represents them to clients, decide how long files are kept, and leave cleanly:
- business name, logo, reply-to, time zone
- retention, default reminders
- security and billing, through panels owned by `auth` and `billing`
- deleting the account and every document

Success means:
- a new requester finishes onboarding (steps 1–3) and sends a first request in one sitting
- account deletion leaves no S3 object, no database row and no Stripe customer behind

## Tech Stack

Foundation conventions, plus:
- `auth`: guards, the Security panel, the user deletion API
- `files`: logo upload, workspace object deletion
- `billing`: the Billing panel, `deleteCustomer`
- `requests`: starter templates for onboarding step 3
- `ui`: NumberedNav, UnsavedChangesBar, SegmentedControl, Select

## Commands

```
Test:         pnpm vitest run src/server/settings
Integration:  pnpm vitest run --project integration src/server/settings
E2E:          pnpm test:e2e e2e/settings
```

## Project Structure

```
src/server/settings/index.ts, schemas.ts, service.ts, actions.ts, deletion.ts
src/app/(app)/app/settings/layout.tsx          → NumberedNav: 01 Workspace, 02 Security, 03 Billing, 04 Account (Team hidden)
src/app/(app)/app/settings/workspace/page.tsx
src/app/(app)/app/settings/security/page.tsx   → mounts auth's SecurityPanel
src/app/(app)/app/settings/billing/page.tsx    → mounts billing's BillingPanel
src/app/(app)/app/settings/account/page.tsx
src/app/(app)/app/onboarding/[step]/page.tsx   → steps 2 and 3 (step 1 is auth)
src/app/api/jobs/account-deletion/route.ts
```

## Interfaces

### Workspace tab (design `5 … 01`)

| Field | Control | Validation | Note |
|---|---|---|---|
| Name | Input | 1–80 characters, trimmed | "Shown on every request" |
| Logo | Thumbnail, **Replace**, **Remove**, plus a "Clients see" preview of the portal header | PNG, JPEG or WEBP, ≤ 1 MB, at least 128 × 128, square-ish (aspect 0.8–1.25, checked in the browser) | Upload via `files.presignLogoPut` → PUT → `confirmLogo` |
| Reply-to email | Input | Valid email or empty | "Where client replies go". Empty → the sign-in email is used |
| Time zone | Select (US zones first) | IANA name in the supported list | Reminder times and dates |
| Delete files | SegmentedControl: **30 days**, **90 days**, **Custom**. Custom shows a number field ("days") | A whole number from 1 to 365. "Enter a number of days from 1 to 365." | "After a request is complete". Helper text: "Applies to completed requests too." Default 90 (MVP.md 4.7) |
| Default reminders | Select: Off, Every 2, 3, 5, 7 days | — | "9 to 11 am client time, at most 5" |

How the tab saves:
- Changes are staged in the form. `UnsavedChangesBar` shows "{n} unsaved changes" with **Discard** and **Save changes**.
- Navigating away with unsaved changes asks for confirmation.
- `updateWorkspaceSettings` saves every field in one action. Logo changes are immediate (separate action) and aren't counted as unsaved.
- **Retention warning.** If the new value is shorter, Save first counts completed requests whose files would now be past their date (`countRetentionImpact(days)`). When the count is above 0, a dialog says "Files for {n} completed requests will be deleted tonight. This can't be undone." with **Delete them tonight** (danger) and **Cancel**. Otherwise it saves directly.

The design system's helper text "Requesters can shorten this per request, never extend it" describes per-request retention, which isn't in the MVP. The MVP drops that line.

### Account tab

- **Sign-in methods:** the email (read-only) and Google (Connect or Disconnect, via Better Auth `linkSocial`/`unlinkAccount`). You can't disconnect the last method; magic link is always available.
- **Delete account:**
  - Copy: "This deletes your workspace, every request, every client file and your billing account. It can't be undone."
  - A typed confirmation of the business name, two-factor verified (`requireTwoFactor`).
  - The button is **Delete account** (danger).

### Actions

| Action | Guard | Effects |
|---|---|---|
| `updateWorkspaceSettings(input)` | `requireWorkspace` | Updates `business_name`, `reply_to_email`, `time_zone`, `retention_days`, `default_reminder_days` |
| `removeLogo()` | `requireWorkspace` | Clears `logo_key`, deletes the object |
| `completeOnboarding({ step })` | `requireWorkspace` | Step 2 saves the logo and reply-to (skippable); step 3 redirects to the builder with the chosen starter template. Tracks `onboarding_completed` when step 3 is reached or skipped |
| `requestAccountDeletion(confirmName)` | `requireTwoFactor`; `confirmName` equals `business_name`, case-insensitive and trimmed | Sets `workspaces.deleting_at`; revokes all sessions; publishes the `account-deletion` job; signs out. The page shows "Your account is being deleted. You'll get a confirmation email." |

### Account deletion job (`POST /api/jobs/account-deletion`, body `{ workspaceId }`)

MVP.md 10: delete S3 first, then the database, then Stripe.

1. Load the workspace. If it's already gone, return (idempotent). If `deleting_at` is null, return.
2. `files.deleteWorkspaceObjects(workspaceId)`: every version of `requests/`, `exports/` and the workspace's logos. Resumable.
3. `withTx`:
   - `SET LOCAL app.allow_event_delete = 'on'`
   - delete the workspace (cascading to requests, items, files, events, schedules, exports, templates, email messages, members)
   - delete the users who have no other memberships, through Better Auth's internal adapter (sessions, accounts, two-factor)
4. `billing.deleteCustomer(stripeCustomerId)`, if one exists. A failure here doesn't roll back the database: it retries, and alerts after the final attempt.
5. Send a confirmation email to the former user's address (captured before deletion), and log to Sentry as info, with no personal data.

Open client links for this workspace's requests resolve to "invalid link" afterwards, because the rows are gone.

### Onboarding steps 2 and 3

- **Step 2: "Make it yours."** Logo and reply-to. **Continue** or **Skip**.
- **Step 3: "Send your first request."** Group chips with icons (Tax files, Accounting, HR, Legal, Universities), the starter templates of the selected group as cards, plus "Start from scratch". Choosing one opens the builder with that template.
- Progress dots: 1 is done (auth), then 2 and 3.

## Data

Owns the settings columns on `workspaces`: `business_name` (after onboarding), `logo_key` (through `files`), `time_zone`, `reply_to_email`, `default_reminder_days`, `retention_days`.

Column (in MVP.md 7 since v1.2): `workspaces.deleting_at timestamptz null`. It blocks sign-in and all actions while deletion runs: `requireWorkspace` throws `UNAUTHENTICATED` when it's set.

## Code Style

```ts
// src/server/settings/schemas.ts
export const workspaceSettingsSchema = z.object({
  businessName: z.string().trim().min(1, "Enter your business name.").max(80),
  replyToEmail: z.union([z.literal(""), z.email("Enter an email address, like name@example.com")]).transform((v) => v || null),
  timeZone: z.enum(SUPPORTED_TIME_ZONES),
  retentionDays: z.number().int("Enter a whole number of days.").min(1, "Enter a number of days from 1 to 365.").max(365, "Enter a number of days from 1 to 365."),
  defaultReminderDays: z.union([z.null(), z.literal(2), z.literal(3), z.literal(5), z.literal(7)]),
});
```

## Testing Strategy

- **Unit:** the settings schema (each field; retention 0, 1, 365, 366 and 1.5), the logo client-side checks (size, dimensions, aspect), the unsaved-changes counting.
- **Integration:**
  - update settings
  - `countRetentionImpact` counts only completed requests with files that the new value puts past their date, skipping reopen drafts
  - logo replace deletes the old object
  - deletion: typed confirmation required; two-factor required; the job deletes S3 objects (all versions, verified with the mock), the rows (including `activity_events`, through the trigger bypass) and the Stripe customer; rerunning after a partial failure completes; open portal links resolve to invalid
  - `deleting_at` blocks actions
- **E2E:**
  - onboarding steps 1 to 3 → builder prefilled from "Individual tax return"
  - edit settings → unsaved bar → save
  - delete account on staging → S3 prefix empty, database rows gone, Stripe test customer deleted (verified by script)

## Edge Cases

- **Retention changed while completed requests exist** → MVP rule: deletion date = `completed_at` + the current `retention_days`. Shortening it can delete older files sooner (at the next maintenance run), so the save dialog warns: "Files for {n} completed requests will be deleted tonight." Lengthening it extends all of them. This is simpler and more honest than per-request snapshots (decided Oct 20, 2026).
- **Logo upload fails halfway** → the old logo stays; nothing changes until `confirmLogo`.
- **Time zone changed** → future reminders use the new zone. Due dates already stored are dates, not times, and don't shift.
- **Deletion while client uploads are in progress** → presign fails with "invalid link" once the rows are gone. Uploads already in S3 are deleted by step 2, or by the orphan cleanup if they land after it.

## Boundaries

- **Always:**
  - typed confirmation plus two-factor for destructive actions
  - delete S3 before the database
  - warn before a retention change deletes files
- **Ask first:**
  - showing the Team tab
  - per-request retention
  - data export before deletion
- **Never:**
  - delete without verifying all versions are gone
  - keep personal data after deletion, except the Stripe invoices Stripe must keep and backups within the Neon point-in-time window (stated in the privacy policy)

## Success Criteria

- [ ] The Workspace, Security, Billing and Account tabs work, with the design's layout and the unsaved-changes pattern.
- [ ] Onboarding steps 2–3 lead into the builder. `onboarding_completed` is tracked.
- [ ] Account deletion removes S3 objects, database rows and the Stripe customer, verified on staging.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 40, 44 and 45)

1. **MVP.md 5.3 Settings:** the tabs are Workspace (which includes the Defaults fields, per the design), Security, Billing and Account. Templates move to the top-nav Templates page (owned by `requests`).
2. **MVP.md 7:** `workspaces.deleting_at`.
3. **MVP.md 10:** account deletion runs as a background job, with sessions revoked first.
4. **MVP.md 4.7:** retention is 30 days, 90 days (default) or a custom 1 to 365 days, and a change applies to completed requests after a warning.

## Decisions (Oct 20, 2026)

1. **Retention options:** 30 days, 90 days, or a custom number of days the requester chooses (1 to 365).
2. **Retention changes** apply to requests already completed, with a warning before anything is deleted.
