# Spec: Request detail and review

Module id: `review`
Status: Approved
Implements: MVP.md 3.1 (Reopen), 4.4, 4.5 (requester side), 5.3 (request detail and review), 9.1 (approve, change requests), 12 (`review_opened`, `item_approved`, `change_requests_sent`)
Designs: `3 Document review - 01 Review.png`, `3 Document review - 02 Complete.png`; design system DS 07 (seal, buttons), DS 05 (260 / fluid / 380 review split)
Reading doc: [docs/specs/review.md](../docs/specs/review.md)
Last updated: 2026-10-20

## Objective

The page where requesters spend most of their time. They:
- see each file next to what they asked for
- approve it with a seal, or ask for a fix with a note
- send every fix in one email
- finish with a completed request and a ZIP

MVP.md 2: "approves it in two minutes, downloads a tidy ZIP."

Success means:
- a 3-item request can be fully reviewed by keyboard alone
- rejecting 3 items sends exactly one email
- no unscanned or infected file is ever displayed

## Tech Stack

Foundation conventions, plus:
- `requests` (transitions, header actions, activity), `files` (file URL route, `removeFile`), `email`, `ui` (Seal, StatusBadge, FolioNumber, Button with `kbd`)
- **PDF rendering:** `pdfjs-dist` (worker build), with `isEvalSupported: false`, scripting off and annotations rendered without forms (MVP.md 10)
- **Images:** `<img>`, with CSS rotate and zoom for rotation

## Commands

```
Test:         pnpm vitest run src/server/review src/components/review
Integration:  pnpm vitest run --project integration src/server/review
E2E:          pnpm test:e2e e2e/review
```

## Project Structure

```
src/app/(app)/app/requests/[id]/page.tsx   → Detail and review page (server component loads; client island for the viewer)
src/server/review/index.ts, rules.ts, service.ts, queries.ts, actions.ts, schemas.ts
src/components/review/                      → ItemRail, Viewer (pdf/image), ViewerToolbar, AskPanel, ChangeNoteForm, ReviewBar, ActivityTimeline, CompleteView, ShortcutsPopover
```

## Interfaces

### Page data (`queries.getReviewPage(ctx, requestId)`)

- **Header:** No., title, client name and email, status badge (with Invite bounced), submitted time, "{x} of {y} approved", due and expiry, reminders state ("Next reminder Oct 21" or "Reminders finished").
- **Items** (position order): label, instructions, reason, required, status, the drafted or sent note, and files. Each file: id, display name, size, type, `scan_status`, `confirmed_at`, and "From" (mobile or desktop, from the event meta `ua_class`).
- **Activity:** the latest 50 events, with "Load more".
- **Allowed actions,** from `requests.allowedActions` and the review rules.

Tracks `review_opened` once per request per day.

### Layout (design `3 … 01`)

**Left rail (260 px): items.**
- Each item shows its folio number, label and a status line: "Approved", "To review · 2 pages", "Change drafted", "Change requested", "Not provided (optional)", "Scanning...", "Blocked".
- The keyboard legend sits at the bottom.

**Center (fluid): viewer.**

| File | How it's shown |
|---|---|
| PDF | Pages rendered by pdf.js in Spread or Single mode, with zoom (−, %, +), Rotate and Download |
| Image | Shown inline |
| HEIC, or a file that fails to render | "Preview isn't available for this file. Download to view." |
| Scanning | A spinner and "Scanning... This usually takes a few seconds." Polls every 5 s for up to 2 minutes |
| Blocked | "Blocked: failed the security scan." with actions Remove file and Request changes |

**Right panel (380 px): the ask.**
- "{nn} · You asked for", the label in display type, and the instructions in italic.
- Metadata (mono labels): File, Size and pages, Added, From.
- A note field ("Note to {first name}, if you request a change"), with quick-pick chips above it: "Blurry or unreadable", "Pages missing", "Wrong document", "Expired document", "Other".
- **Request change** (secondary, kbd R) and **Approve** (primary, kbd A).

**Bottom review bar** (sticky):
- "{a} approved, {r} to review, {d} change(s) drafted", with **Send change requests ({d})**. It is primary only when d > 0; otherwise Approve stays the single primary (DS 01).
- **Download ZIP** or **Prepare ZIP**, from `exports`.

**Activity:** a "History" disclosure below the rail on desktop, and a tab on narrow screens.

**Before submission** (pending or in_progress): files appear read-only as they arrive, with the note "You can review once {first name} submits." No approve or change controls.

**Completed** (design `3 … 02`):
- The "Complete." headline, the Complete seal and a summary line.
- Items with Approved badges, file counts and sizes.
- **Download all · ZIP** and **Reopen** (below). Reopen is hidden once the files are deleted.
- The retention line "Files will be deleted on {date}, {n} days from today."

**Reopen** (MVP.md 3.1, 4.4 step 5):
1. **Reopen** switches the completed view back to the review layout, with every item approved and the bar reading "Reopening. Draft a note on each item that needs another look."
2. The requester drafts notes on approved items (R, or Request change). Items stay approved while drafted; nothing changes for the client yet.
3. **Send change requests ({d})** sends them as usual. **Cancel** undoes every draft and returns to the completed view.
4. While drafts exist, the request stays `completed`, and nightly retention skips it (SPEC-automation).

**Header menu** (calls `requests` actions): Send reminder now, Extend deadline, Fix email and resend, Regenerate link, Copy link, Revoke link, Delete files. Each appears only when allowed.

**Bounced banner** (MVP.md 5.3): "The invite to {email} bounced. Check the address and resend." with **Fix email**, which opens an inline field and **Resend invite**.

### Actions (Server Actions; `requireWorkspace`; file-touching ones `requireTwoFactor`)

| Action | Allowed when | Effects |
|---|---|---|
| `approveItem(itemId)` | Request submitted or changes_requested; item `uploaded`; every active file on the item is `clean` | Item `APPROVE`, `reviewed_at`, clears any drafted note; logs `item_approved`. Then, if the request is submitted and complete (rule below): `ALL_APPROVED`, `completed_at`, `stopReminders`, logs `request_completed`. Tracks `item_approved` |
| `unapproveItem(itemId)` | Request submitted or changes_requested (not completed); item `approved` | Item `UNAPPROVE` (→ uploaded). Logs `item_change_drafted` with `meta.unapproved` (MVP.md 9.1) |
| `draftChangeRequest(itemId, note)` | Request submitted, with the item `uploaded` or `approved`; or request completed (Reopen), with the item `approved` and `files_deleted_at` null | Sets `rejection_note` (1–500 chars, trimmed), `rejection_sent_at = null`. On a submitted request an approved item takes `UNAPPROVE` first; on a completed request it stays approved until sent. Logs `item_change_drafted`. No email |
| `undoChangeRequest(itemId)` | Item has a drafted note | Clears the note. Not logged (drafts are private) |
| `sendChangeRequests(requestId)` | Request submitted, or completed with files not deleted; at least one drafted note | One `withTx`: drafted items `CHANGES_SENT` (→ rejected, `rejection_sent_at = now`; approved items take `UNAPPROVE` first); request `SEND_CHANGES` from submitted, or `REOPEN` from completed, which also clears `completed_at`, sets `expires_at = max(expires_at, now + 14 days)` and logs `request_reopened`; `resumeReminders(now)`; logs `change_requests_sent` (meta: item ids). After commit: one `change_request` email listing every item and note, deep-linking to the first item. Tracks `change_requests_sent` |
| `removeBlockedFile(fileId)` | File `infected` or `failed` | `files.removeFile` (actor requester). The item goes back to `pending` only if the request is not submitted; otherwise the item keeps its status, and the requester requests changes |

Completion rule (`rules.isComplete`):

> Every item that has at least one active file is `approved`, no item has a drafted note, and no item is `rejected`.

Optional items with no files don't block completion, and show "Not provided" (MVP.md 3.1).

Optimistic UI (MVP.md 5.8): approve, unapprove and draft update instantly with rollback, plus an error toast on failure. The seal animation plays when the approve call succeeds, or immediately with rollback.

### Keyboard (MVP.md 5.3)

| Keys | Action |
|---|---|
| J / K | Next / previous item |
| ← / → | Previous / next file within the item |
| A | Approve |
| R | Focus the note, or draft the change if the note is filled |
| Shift+Enter in the note | Draft the change |
| ? | Shortcuts popover |
| Esc | Leave the note |

Shortcuts are off while typing in an input, and announced in the popover. Every action is also reachable with Tab and Enter.

## Data

No owned tables. Writes `checklist_items` (status, `rejection_note`, `rejection_sent_at`, `reviewed_at`) and `document_requests` status through `requests` functions, plus `activity_events`. Reads `uploaded_files`.

## Code Style

```ts
// src/server/review/rules.ts
export function isComplete(items: ReviewItem[]): boolean {
  return items.every((i) =>
    i.activeFileCount === 0 ? i.status === "pending" : i.status === "approved" && i.draftNote === null,
  );
}

export function canApprove(item: ReviewItem, requestStatus: RequestStatus): Check {
  if (!["submitted", "changes_requested"].includes(requestStatus)) return no("You can review once the client submits.");
  if (item.status !== "uploaded") return no("This item isn't waiting for review.");
  if (item.files.some((f) => f.scanStatus === "pending")) return no("Finishing security scan...");
  if (item.files.some((f) => f.scanStatus !== "clean")) return no("Remove the blocked file or request changes.");
  return yes();
}
```

## Testing Strategy

- **Unit:** `isComplete` (all combinations of approved, drafted, rejected and optional-empty), `canApprove`, `canDraft`, the status-line text for each item state, and the keyboard map reducer.
- **Component:**
  - viewer states (PDF fixture rendered, image, HEIC fallback, scanning, blocked)
  - the ask panel
  - the review bar counts
  - keyboard navigation and the shortcut guard while typing
  - axe checks
- **Integration (Postgres, outbox):**
  - approve: the last approve completes the request and stops reminders
  - approve is blocked while scanning or blocked
  - draft, undo, unapprove
  - `sendChangeRequests` with 3 drafts → 3 items rejected, status changes_requested, exactly one email listing 3 notes, reminders resumed
  - a double send → the second is a `CONFLICT` no-op, no second email
  - `removeBlockedFile`
  - Reopen: drafts on a completed request keep the items approved; sending → items rejected, request changes_requested, `completed_at` cleared, `expires_at` at least 14 days out, reminders resumed, one email, `request_reopened` logged; Cancel clears the drafts; Reopen is refused after Delete files
  - cross-workspace 404s
  - `TWO_FACTOR_REQUIRED` on preview
- **E2E (Playwright, staging):**
  - client submits 3 items → requester reviews by keyboard only (J, A, J, R + note, J, R + note) → Send change requests → the client email lists 2 notes
  - client fixes → resubmit → approve all → Complete view with the seal → Reopen → draft one note → send → the client email lists it and the portal shows the fix screen

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| Rejecting 3 items sends exactly one email to the client, listing all 3 notes | Integration plus E2E above |
| Every status and badge is understandable without color (shared with `ui`) | Component test: each item status line has text and an icon |

## Edge Cases

- **New files arrive while reviewing** (status changes_requested, client re-uploading) → the page polls every 30 s while visible and shows "{n} new file(s)" with Refresh. Actions use conditional updates, so stale views get `CONFLICT` "This request changed. Refresh to see the latest."
- **The requester approves the last item by mistake** → the request completes immediately. Reopen, draft a note on that item and send it.
- **Reopen drafts are left unsent** → the request stays completed and its files are kept until the drafts are sent or cancelled. The completed view shows "{d} change(s) drafted, not sent."
- **A file is infected after the request was submitted** → shown as Blocked and can't be approved. The requester removes it or requests changes. MVP.md 8.4 keeps it listed.
- **A PDF that fails to render** (encrypted, malformed) → falls back to "Download to view". The download is still scanned clean and uses attachment disposition.
- **A huge PDF (500 pages)** → pdf.js renders lazily, visible pages only, with the thumbnail strip off by default.
- **The note contains a URL** → shown as plain text in the client email and portal.
- **Two requesters?** Not possible in the MVP (one member per workspace). With Teams, conditional updates already handle races.

## Boundaries

- **Always:**
  - require two-factor for any file view or download
  - check `scan_status`
  - send exactly one email per Send change requests
  - keep drafts private until sent
- **Ask first:**
  - approving before submission (MVP.md 15 Q1, currently no)
  - bulk "Approve all"
- **Never:**
  - render a PDF with scripting enabled
  - show unscanned bytes
  - email the client per item

## Success Criteria

- [ ] The review page matches design `3 … 01` and the complete view matches `3 … 02`, at 1280 px and on a tablet.
- [ ] All actions follow the state rules, with integration tests including completion and the one-email rule.
- [ ] The keyboard-only review flow passes E2E.
- [ ] Scan states are shown correctly, and blocked files can't be approved or previewed.
- [ ] PostHog events `review_opened`, `item_approved` and `change_requests_sent` fire with no file names.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 39 and 45)

1. **MVP.md 10, Storage:** PDF previews are rendered with pdf.js (scripting off, `isEvalSupported: false`) instead of a sandboxed iframe. The design's Spread, Single, zoom and rotate viewer needs page-level rendering, and pdf.js never executes PDF JavaScript.
2. **MVP.md 9.1:** `unapproveItem` and `removeBlockedFile`.
3. **MVP.md 3.1:** completion happens on the approve that makes `isComplete` true (`ALL_APPROVED`); Reopen adds `completed → changes_requested`.

## Decisions (Oct 20, 2026)

1. **Reopen:** yes, option (b). The requester drafts notes on a completed request and sends them as change requests; one email to the client.
2. **Activity:** a collapsible "History" section below the item rail.
