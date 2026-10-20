# Spec: Client portal

Module id: `portal`
Status: Approved
Implements: MVP.md 4.2, 4.3, 4.5, 5.6, 9.2, 10 (portal hardening, rate limits), 12 (recipient events)
Designs: `1 Client portal - 01 Checklist.png`, `1 Client portal - 02 Success.png`; design system DS 07 (request items, fields, buttons, seal), DS 09 (voice)
Reading doc: [docs/specs/portal.md](../docs/specs/portal.md)
Last updated: 2026-10-20

## Objective

Let a client who has no account and no loyalty to the product hand over sensitive documents from a phone on a poor connection, in under 2 minutes, with confidence that they went to the right person. MVP.md 2: "Design for the recipient first."

User stories:
- **Open the link.** I tap the email link and see my accountant's name and logo, what they need, why, and by when.
- **Add files.** I photograph a document (several pages if needed) or choose files. I watch them upload, and can retry or remove one.
- **Come back later.** I can leave and come back on any device; my files are still there.
- **Submit.** Once the required items are in, I submit, see a "Received" seal and get a receipt email.
- **Fix items.** If my accountant asks for a fix, I see exactly which items and why, change only those, and resubmit.

Success means:
- portal viewed → submitted is 70% or more
- the upload failure rate is under 2% (MVP.md 12)
- the MVP.md 14 iOS Safari criterion passes

## Tech Stack

Foundation conventions, plus:
- `requests` (token lookup, transitions, digest trigger)
- `files` (limits, presign, confirm, remove)
- `email` (submitted, resubmitted and receipt emails)
- `ui` (PortalShell, RequestItemFrame, DropZone, Sheet, FileChip, StickyActionBar, Seal)

Client-side:
- **Turnstile:** `@marsidev/react-turnstile`, invisible mode.
- **HEIC conversion:** `heic2any`, lazy-loaded only for HEIC/HEIF.
- **Hashing:** Web Crypto `subtle.digest("SHA-256")`, in a Web Worker for files over 5 MB.
- **Uploads:** `XMLHttpRequest` (for upload progress events).
- **QR code:** `qrcode` (already used by `auth`), lazy-loaded on desktop for the phone handoff.

## Commands

```
Test:         pnpm vitest run src/server/portal src/components/portal
Integration:  pnpm vitest run --project integration src/server/portal
E2E:          pnpm test:e2e e2e/portal --project=webkit-mobile --project=chromium-mobile --project=chromium
```

## Project Structure

```
src/app/c/[token]/page.tsx                  → Server component: lookup, state routing, render
src/app/c/[token]/not-found.tsx             → Invalid link screen
src/app/api/portal/session/route.ts         → Turnstile verify, set cookie
src/app/api/portal/opened/route.ts          → Beacon
src/app/api/portal/uploads/presign/route.ts
src/app/api/portal/uploads/confirm/route.ts
src/app/api/portal/files/[fileId]/route.ts  → DELETE
src/app/api/portal/files/[fileId]/thumb/route.ts
src/app/api/portal/submit/route.ts
src/server/portal/index.ts, session.ts, service.ts, rules.ts, schemas.ts, queries.ts
src/components/portal/                      → Checklist, ItemCard, AddFileSheet, PhotoReview, UploadQueue, SubmitSheet, Success, ChangesBanner, TerminalScreen, PhoneHandoff
src/lib/upload/                             → prepare.ts (HEIC/resize), hash.worker.ts, put.ts (XHR + retry), queue.ts (3 parallel per item)
```

## Interfaces

### Page `GET /c/{token}`

1. Rate limit `portalPageLoad` by IP. Malformed tokens skip the database but still count toward `invalidToken`.
2. `requests.findByToken(token)`. Not found → the `invalidToken` limit → the invalid-link screen. Unknown and malformed tokens take the same path, so the timing is the same (MVP.md 5.6).
3. Choose the screen by the effective status:
   - If `expires_at` has passed and the status is active, treat it as `expired`, even before maintenance runs.
   - **Active** (pending, in_progress, changes_requested) → the checklist.
   - **submitted** → the locked screen.
   - **completed** → the completed summary.
   - **expired**, **cancelled** → their terminal screens.
4. Response headers: `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, `X-Robots-Tag: noindex, nofollow`.
5. The page data includes, and only includes:
   - the branding (business name, logo URL, requester display name, reply-to)
   - the request (number, title, recipient first name, due date, message)
   - the items (id, position, label, instructions, reason, required, client-facing status, the rejection note when sent)
   - for active screens only, the client's own active files (id, display name, size, MIME type, scan state reduced to `ok` or `processing`)

   It never includes requester emails other than the reply-to, internal ids of other workspaces, or other items' notes.

### Portal session (`POST /api/portal/session`)

| | |
|---|---|
| Body | `{ token: string, turnstileToken: string }` (≤ 4 KB) |
| Checks | Turnstile `siteverify` (secret, `remoteip`, `idempotency_key`), with an expected hostname and action `portal`; the token resolves to an active, submitted or completed request |
| Cookie | `__Secure-dlp_{rid8}` = `base64url(payload).hmac`, payload `{ rid, th: tokenHash[0..8], exp: now + 12h }`. Flags: HttpOnly, Secure, SameSite=Lax, Path=`/api/portal`, Max-Age 43200 |
| Response | `{ requestId }` |

`requirePortalSession(requestId)` is the context for every other portal route. It:
- reads the cookie named for that request
- verifies the HMAC and `exp`
- loads the request and checks that `token_hash` still starts with `th`, so regenerate and revoke kill existing sessions
- returns `{ request }`, or 401 "Your session ended. Reload the page."

Every portal API body carries `requestId`. The token never appears in API calls after the session is created (MVP.md 9.2).

### Beacon (`POST /api/portal/opened`)

- Throttled with Redis `SET po:{rid} NX EX 1800`.
- When it fires: sets `first_opened_at` if null, and logs `link_opened` (recipient).
- Always returns 204.
- It runs after hydration, so link scanners don't count as opens (MVP.md 0 row 11).

### Presign (`POST /api/portal/uploads/presign`)

Body:

```ts
{ requestId: uuid, itemId: uuid, name: string(1..255), type: string, size: int, sha256: base64(44) }
```

| Check | Error |
|---|---|
| Portal session | 401 |
| `portalPresign` rate limit (60 per 10 min per request) | 429 "Too many uploads at once. Wait a minute and try again." |
| The request is uploadable: pending or in_progress (any item), or changes_requested (rejected items only) | 409 "This request is locked." / "This item was already accepted." |
| The item belongs to the request | 404 |
| `files.checkUploadAllowed`, inside `files.createPendingFile` | 400 with MVP.md 5.6 copy |

Returns `{ fileId, url, headers, expiresAt }`.

### Confirm (`POST /api/portal/uploads/confirm`)

Body: `{ requestId, fileId }`.

The `withTx` transaction:
1. `files.confirmUpload`.
2. If the item is `pending`: `FILE_ACTIVE` (→ uploaded). If it's `rejected`, it stays rejected.
3. If the request is `pending`: `FIRST_UPLOAD` (→ in_progress).
4. Logs `file_uploaded` (recipient; meta: size and type, no name).
5. After commit: `requests.noteActivity` and the PostHog `upload_succeeded` event (client-side).

Returns `{ file, item, requestStatus }`.

### Remove (`DELETE /api/portal/files/{fileId}`)

- Body: `{ requestId }`.
- Allowed only while the file's item is editable: request pending or in_progress, or the item is `rejected` in changes_requested.
- Effects: `files.removeFile` (actor recipient). If the item has no active files left and it was `uploaded`: `LAST_FILE_REMOVED`.
- After commit: `noteActivity`.

### Thumbnail (`GET /api/portal/files/{fileId}/thumb?requestId=`)

- Allowed only for the client's own active image files, while the item is editable, and only if `scan_status = clean`.
- Returns a 302 to a presigned inline GET valid for 10 minutes.
- Otherwise 404. The UI shows a file icon.
- During the session the UI never calls this route for a file it uploaded itself: it shows the local preview (an object URL of the prepared bytes). Server thumbnails are for return visits.

### Submit (`POST /api/portal/submit`)

Body: `{ requestId, mode: "submit" | "resubmit" }`.

| Mode | Gate | Transaction |
|---|---|---|
| submit | Request in_progress; every required item has at least one active file; no file of the request is `pending` (still uploading); at least one active file overall | `SUBMIT`; `submitted_at`; `advanceDigestCursor`; logs `request_submitted` |
| resubmit | Request changes_requested; every rejected item has changed since its `rejection_sent_at` (a file confirmed or removed after it); no `pending` files | Rejected items `RESUBMITTED` (→ uploaded; `rejection_note` and `rejection_sent_at` cleared, with the note kept in the `request_resubmitted` event meta); `RESUBMIT`; `advanceDigestCursor`; logs `request_resubmitted` |

After commit:
- email `submitted` or `resubmitted` to the requester, and `receipt` to the client (idempotency `submit:{rid}:{submittedAt}`)
- the client-side PostHog `submitted` event

Gate failures return 409 with the reason, for example "1 required item left: W-2 form". The same text appears in the sticky bar.

Reminders need no explicit pause, because the reminder query only selects active statuses. That is how MVP.md 4.3's "reminders paused" is implemented.

### Client-side upload pipeline (`src/lib/upload`)

1. **Pick.** Take photo uses `accept="image/*" capture="environment"`; Choose file uses `accept="application/pdf,image/*" multiple`. Check type and size before anything else (MVP.md 5.6 messages, and the upload never starts).
2. **Prepare.**
   - Camera captures and HEIC/HEIF files are converted to JPEG: longest edge 2,400 px, quality 0.85. `heic2any` is lazy-loaded.
   - If conversion fails, the original HEIC is uploaded (MVP.md 15).
   - Row state: "Preparing photo...".
3. **Photo review.** For camera captures: Use photo or Retake, with the tip "Make sure all four corners are visible and the text is sharp." Then "Add another page" or "Done".
4. **Hash.** SHA-256 of the final bytes, in a worker for files over 5 MB.
5. **Presign**, then **PUT** with XHR to the URL with the signed headers, showing a progress bar and percent.
   - Up to 3 uploads in parallel per item; the rest queue.
   - On a network error or 5xx: retry 3 times (1 s, 3 s, 9 s), getting a fresh presign if the URL expired. Then the Failed state with **Retry**.
   - A 400 checksum or signature error → re-hash once and retry once. Then Failed.
6. **Confirm.** On success, Done (green check, name, size, local preview, Remove).
7. A `beforeunload` prompt is set while anything is uploading or preparing.
8. PostHog events: `add_file_opened`, `photo_taken`, `file_chosen`, `upload_succeeded`, and `upload_failed` with a reason enum (`network`, `checksum`, `too_large`, `type`, `server`). Never file names.

### Screens (MVP.md 5.6 copy unless noted)

**Checklist:**
- **Layout.** Mobile: a single column. Desktop (design `1 … 01`): two columns, with the title, message and progress on the left and items on the right.
- **Header.** "Requested by {name} · {date}". Title in display type. "Prepared for {first name}. Due {date}." The message as a quote with the requester's avatar.
- **Progress.** "1 of 2 required items added".
- **Items** use `RequestItemFrame`:
  - Required or Optional is written in text, and the "Why:" reason is shown.
  - Added items show their files and a "Change" text button that reopens the item.
  - On desktop, an empty item shows a drop zone ("Drop your W-2 here").
- **Trust line**, plus footer links "How is my data protected?" and "Why did I get this?" (`site` pages).
- **Sticky bar.** Submit, with the reason it's disabled.
- **Phone handoff** (MVP.md 5.6, design `1 … 01`, desktop only):
  - A "Paper copy? Use your phone" panel in the left column, with a QR code and "Scan with your phone's camera to take photos of your documents."
  - The QR encodes the page's own URL without the hash (`/c/{token}`). It's generated in the browser with `qrcode` as an SVG; the token is never sent anywhere to make it.
  - Shown only when `(pointer: fine)` and the viewport is at least 1024 px wide.
  - The panel has `ph-no-capture` and is excluded from PostHog replay, because the QR contains the link.
  - The phone opens the same link and starts its own session. Uploads from either device appear on the other within 30 s (the existing poll).

**Add file sheet:** item label and instructions; **Take photo** (hidden without a camera) and **Choose file**; "PDF, JPG, PNG or HEIC. Up to 25 MB each."

**Submit sheet:**
- "Send {n} files to {business}? You won't be able to change them unless {business} asks for a fix."
- A per-item count list, with empty optional items shown as "Not included".
- **Submit documents** and **Keep editing**.

**Success** (design `1 … 02`):
- The "Received" seal animates once (static with reduced motion).
- "Thank you, *{first name}*."
- "{business} has your {n} files. {requester} will review them and email you if anything needs fixing."
- The request label card listing items with file counts and sizes.
- The retention line "Files are deleted {retention} days after the request is complete."
- No further actions; the receipt arrives by email (MVP.md 5.6). The design's "Download receipt" and "Add something else" are not built.

**Changes requested:**
- A banner: "{business} asked you to fix {n} item(s)."
- Rejected items come first, with an attention-colored left border and the note quoted. Other items show a lock and "Accepted" or "Under review".
- The bar shows **Resubmit**, with "Update {n} item(s) to resubmit".
- The page scrolls to `#item-{id}` from the email deep link.

**Infected file notice:** an item with an infected file removed in the last 24 hours, and no newer active file, shows "We couldn't accept this file. Please upload it again." (MVP.md 4.2 step 8). The data comes from the `file_scan_infected` events.

**Terminal screens:** submitted (locked), completed, expired, revoked, invalid link. Each has exact copy from MVP.md 5.6 and the requester's name and reply-to as the way forward.

**Accessibility:**
- 17 px base, 44 px targets
- upload progress announced with `aria-live="polite"`, errors with `role="alert"`
- 200% zoom with no horizontal scroll
- the sheet traps focus

## Data

No owned tables. Writes through `files` (uploaded_files) and `requests` (status, items, events). Reads branding (business name, logo, reply-to) from `workspaces`.

## Code Style

```ts
// src/server/portal/rules.ts — pure
export function submitGate(request: PortalRequest, items: PortalItem[], files: PortalFile[]): GateResult {
  if (files.some((f) => f.status === "pending")) return blocked("Wait for uploads to finish.");
  if (request.status === "in_progress") {
    const missing = items.filter((i) => i.required && !files.some((f) => f.itemId === i.id && f.status === "active"));
    if (missing.length) return blocked(`${missing.length} required item${missing.length > 1 ? "s" : ""} left: ${missing.map((i) => i.label).join(", ")}`);
    return files.some((f) => f.status === "active") ? open("submit") : blocked("Add at least one file.");
  }
  if (request.status === "changes_requested") {
    const unchanged = items.filter((i) => i.status === "rejected" && !changedSince(i, files));
    return unchanged.length ? blocked(`Update ${unchanged.length} item${unchanged.length > 1 ? "s" : ""} to resubmit`) : open("resubmit");
  }
  return blocked("This request is locked.");
}
```

## Testing Strategy

- **Unit:**
  - `submitGate` for every status and combination
  - `isItemEditable`
  - cookie sign and verify (tampered, expired, wrong `th`)
  - the effective-status calculation with `expires_at`
  - `prepare.ts`: size and type pre-checks
  - the retry and backoff schedule with fake timers
  - the queue's 3-parallel limit
- **Component:** checklist states, the sheet, the submit sheet, the changes banner, terminal screens. Each gets a render test, an axe check and keyboard checks. The phone handoff renders only for a fine pointer at ≥ 1024 px, encodes the URL without the hash, and carries `ph-no-capture`.
- **Integration (Postgres, mocked S3 and Turnstile):**
  - session: Turnstile failure, a wrong token, cookie scoping per request, regenerate invalidating the session
  - presign: every rejection path, including approved or locked items during changes_requested and expired-by-time
  - confirm transitions (`pending → in_progress`; item `pending → uploaded`; rejected stays rejected)
  - remove re-pending an item
  - submit and resubmit gates, plus the emails in the outbox and the digest cursor advanced
  - beacon throttling
  - the page loader returns only the allowed fields (a snapshot of the serialized props)
- **E2E (Playwright, staging S3):**
  - mobile WebKit: open the invite → add a photo (fixture JPEG via `setInputFiles`) → add a PDF → submit → success → receipt email in the outbox
  - close mid-way and reopen in a new context → the confirmed uploads are shown
  - checksum mismatch (a test hook alters bytes) → Failed, then Retry
  - revoked and expired links → their screens, and presign returns 409
  - changes requested → fix → resubmit
- **Manual device matrix (M5):** iOS Safari latest and previous, Android Chrome, Samsung Internet, desktop Chrome, Safari, Firefox and Edge. Real camera capture; HEIC from an iPhone opened on Windows; VoiceOver and TalkBack.

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| A client on iOS Safari over 4G can open the invite, photograph a two-page document, and submit in under 2 minutes | Manual timed run on a device over 4G (M5); E2E with network throttling as a regression guard |
| HEIC photos arrive as JPEGs that open on Windows without extra software | Manual (iPhone → staging → download on Windows); unit test that converted output is `image/jpeg` |
| Closing the tab mid-way and reopening the link on desktop shows all confirmed uploads | E2E above |
| Approved items cannot be changed by the client while other items are being fixed | Integration: presign and remove on approved or uploaded items during changes_requested return 409; E2E: no Add or Remove controls on locked items |
| A revoked or expired link shows the correct screen and cannot presign uploads | E2E plus integration above |
| An upload whose bytes don't match the signed checksum is rejected by S3 and shown as a failed upload | E2E above (with `files`) |

## Edge Cases

- **The link is opened by an email scanner** → the page renders, but no beacon fires (no hydration) and no session is created (no Turnstile token).
- **Turnstile can't load** (blocked network, privacy extension) → after 10 seconds: "We couldn't verify your browser. Turn off content blockers for this page or try another browser." Plus a Retry. No upload without a session.
- **The client keeps the page open past the 12-hour cookie** → the next API call returns 401. The page silently re-runs Turnstile and the session call, then retries the request once.
- **Regenerated while the client is mid-upload** → the next call returns 401, and re-running the session with the old token fails. "This link was replaced. Check your email for the new one."
- **Two devices at once** → both see their own uploads appear after the next refresh (polled every 30 s while visible). Submit is atomic, so the second device's submit gets 409 "Already sent".
- **The client removes the last file of a required item after submitting is enabled** → the gate re-evaluates and the bar shows the reason again.
- **A pending upload whose presign URL expired** → the client re-presigns. The old pending row is left for orphan cleanup (2 hours).
- **Very slow connection, 25 MB file** → there's no timeout on the PUT, only progress. The 15-minute URL is long enough for about 220 kbps. Below that the PUT fails with 403 and the client gets a fresh URL and retries.
- **Item instructions contain a URL** → rendered as plain text, not a link (phishing safety).
- **The request expires while the client is uploading** → presign returns 409 "This request has expired. Contact {name} at {reply-to}."

## Boundaries

- **Always:**
  - authenticate portal APIs by cookie only
  - send the portal headers on `/c/*`
  - show the reason Submit is disabled
  - follow DS 09 copy
  - never show file contents from other items or requests
- **Ask first:**
  - adding fields to the client view
  - adding third-party scripts to the portal
  - changing upload limits or retry behavior
- **Never:**
  - tokens in query strings or API bodies after the session
  - marketing, sign-up prompts or chat widgets on the portal (MVP.md 5.6)
  - logging file names to analytics
  - showing server thumbnails of unscanned files
  - sending the link to a third-party QR service, or letting replay record the QR

## Success Criteria

- [ ] All portal routes in MVP.md 9.2 exist with the checks above and integration tests.
- [ ] The checklist, sheet, submit, success, changes-requested and terminal screens match the designs and the MVP.md 5.6 copy, with accessibility checks passing.
- [ ] The upload pipeline handles HEIC, resizing, checksums, parallelism, retries and `beforeunload`.
- [ ] Every MVP.md 14 criterion in the table above passes on staging and the device matrix.
- [ ] The PostHog recipient funnel events arrive without file names.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 41 and 45)

1. **MVP.md 4.3:** submitting also requires at least one active file overall (an all-optional request can't be submitted empty) and no uploads in progress.
2. **MVP.md 9.2:** the portal cookie is per request (`__Secure-dlp_{rid8}`) and bound to the current token. Regenerating the link ends existing sessions.
3. **MVP.md 9.2:** thumbnails are served only for clean files; during the session the browser's local preview is used.
4. **MVP.md 5.6:** the desktop QR phone handoff is a must-have.

## Decisions (Oct 20, 2026)

1. **Success screen:** follow MVP.md. No "Download receipt" or "Add something else".
2. **QR phone handoff:** must-have, encoding the same link.
3. **"Why did I get this?":** written for launch by `site` and linked from the portal footer.
4. **Thumbnails:** the browser's local preview during the session; server thumbnails only for clean files.
