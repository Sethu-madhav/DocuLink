# Spec: Email sending, templates and delivery events

Module id: `email`
Status: Approved
Implements: MVP.md 4.1 (invite with SES message ID), 4.8 (bounce detection), 5.7 (all emails), 7 (`email_messages`), 8.4 (SES events), 10 (privacy)
Designs: design system DS 03 (colors), DS 09 (voice); no email mockups exist, so layout follows MVP.md 5.7
Reading doc: [docs/specs/email.md](../docs/specs/email.md)
Last updated: 2026-10-20

## Objective

Send every DocuLink email through one function that brands it correctly, delivers it reliably through SES, and records it so that delivery, bounce and complaint events can be traced back to a request. Emails are where DocuLink first meets the client. MVP.md 15 lists two risks here, "clients think the invite is phishing" and "invites land in spam", so the sender name, reply-to and copy are part of this spec.

Success means:
- every email in MVP.md 5.7 renders correctly in Gmail and Outlook
- every send has an `email_messages` row with the SES message ID
- a permanent bounce of an invite, reminder or change-request email reaches `requests` within 5 minutes

## Tech Stack

- `@aws-sdk/client-sesv2`, via foundation's `ses()` with OIDC credentials. Sends use the configuration set from `SES_CONFIGURATION_SET`.
- `react-email` (`@react-email/components`, `@react-email/render`) for the templates, rendered to HTML plus a plain-text part.
- Foundation's `withTx`, `logEvent`, the Redis client (idempotency) and `createAwsEventsRoute` (the route itself is composed in `src/app`).

## Commands

```
Preview:      pnpm email:dev                      # react-email preview server on :3001 with fixture props
Test:         pnpm vitest run src/server/email
Integration:  pnpm vitest run --project integration src/server/email
Seed bounce:  pnpm email:simulate -- bounce       # staging only: sends an invite to bounce@simulator.amazonses.com
```

## Project Structure

```
src/server/email/index.ts           → Public API: send, handleSesEvent, types
src/server/email/send.ts            → Render, transport, record, idempotency
src/server/email/transport.ts       → EmailTransport interface; SesTransport, OutboxTransport (dev/test)
src/server/email/ses-events.ts      → handleSesEvent (EventBridge detail → email_messages, onBounce callback)
src/server/email/templates/*.tsx    → One React Email component per kind, plus layout.tsx
src/server/email/fixtures.ts        → Props for previews and snapshot tests
src/server/email/unsubscribe.ts     → signUnsubscribeToken, verifyUnsubscribeToken (reminder opt-out)
src/app/api/email/unsubscribe/route.ts → GET confirm page, POST one-click; wires email + requests
```

## Interfaces

```ts
type EmailKind =
  | "invite" | "reminder" | "change_request" | "receipt"                    // to client
  | "digest" | "submitted" | "resubmitted" | "invite_bounced" | "zip_ready" // to requester
  | "sign_in_link" | "two_factor_changed";                                  // account

send(input: {
  kind: EmailKind;
  to: string;                       // validated, lowercased
  requestId?: string;               // required for client kinds and request-related requester kinds
  workspace?: { businessName: string; replyTo: string | null };  // branding for client kinds
  props: PropsFor<kind>;            // typed per template
  idempotencyKey?: string;          // same key within 24 h → no second send, returns the first messageId
}): Promise<{ messageId: string } | { skipped: "duplicate" }>;

handleSesEvent(event: EventBridgeSesEvent, hooks: {
  onRecipientBounced: (args: { requestId: string; kind: EmailKind; type: "bounce" | "complaint"; to: string }) => Promise<void>;
}): Promise<void>;
```

### `send` behavior

1. **Validate** `to` with zod's email check. Build headers:
   - **Client kinds:** From `"{Business name} via DocuLink" <requests@mail.doculinkapp.com>`; Reply-To is the workspace's `reply_to_email`, falling back to the requesting user's email.
   - **Requester and account kinds:** From `"DocuLink" <requests@mail.doculinkapp.com>`, no Reply-To.
   - The address comes from `SES_FROM_EMAIL`, so staging uses `requests@mail.staging.doculinkapp.com`.
   - **`reminder` only:** `List-Unsubscribe: <{APP_URL}/api/email/unsubscribe?t={t}>` and `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058), passed in SES v2 `Simple.Headers`.
   - Display names are sanitized: CR/LF and `"` stripped, at most 70 characters.
2. **Idempotency.** If `idempotencyKey` is given: `SET email:idem:{key} NX EX 86400`. If the key already exists, return `skipped`.
3. **Render** HTML and plain text. Subject lines are exactly as in MVP.md 5.7, with variables filled in.
4. **Send** via `SendEmail` with `ConfigurationSetName` and the `EmailTags` `kind` and `env`. Then insert `email_messages { ses_message_id, request_id, kind, to_email, status: "sent" }`.
5. **Transient SES errors** (throttling, 5xx) are retried 3 times with backoff inside `send`. Then `UPSTREAM` is thrown, and the idempotency key is deleted so a job retry can send.
6. **Permanent SES errors** (`MessageRejected`, `AccountSuspended`, `MailFromDomainNotVerified`) raise a Sentry alert and are thrown as `UPSTREAM`.

### Templates (MVP.md 5.7)

| Kind | To | Subject | Primary button → URL |
|---|---|---|---|
| `invite` | Client | {Business} needs documents for {Title} | Add your documents → `/c/{token}` |
| `reminder` | Client | Reminder: {n} documents still needed for {Title} | Add your documents → `/c/{token}` |
| `change_request` | Client | {Business} asked you to update {n} document(s) | Update documents → `/c/{token}#item-{firstId}` |
| `receipt` | Client | You sent {n} documents to {Business} | none (summary only) |
| `digest` | Requester | {Client} added {x} of {y} documents | Open request → `/app/requests/{id}` |
| `submitted` | Requester | {Client} submitted documents for {Title} | Review now |
| `resubmitted` | Requester | {Client} updated {n} document(s) | Review now |
| `invite_bounced` | Requester | Your request to {email} bounced | Fix email → `/app/requests/{id}?fix=email` |
| `zip_ready` | Requester | Your ZIP for {Title} is ready | Download ZIP → `/app/requests/{id}?zip={exportId}` |
| `sign_in_link` | Requester | Your sign-in link for DocuLink | Sign in |
| `two_factor_changed` | Requester | Two-factor sign-in was turned off / A backup code was used / Several wrong codes were entered | Review security → `/app/settings/security` |

Every template:
- is single column and 600 px wide, with tokens as inline CSS: `paper` background, `surface` card, `action` button, `ink` text
- has one primary button, with the full link as text below it (MVP.md 5.7)
- includes a plain-text part
- uses the requester's business name on client emails, with "Sent via DocuLink" in the footer
- never contains file names (only the requester's item labels), and never attachments

Client emails also include the requester's personal message (`invite` only) and the due date when set. The final reminder says "This is the last reminder." Reminder footers add the link "Stop reminders for this request" (the same URL as the header).

The links in client emails contain the token. They're built by `requests` (`buildPortalUrl`) and passed in as props. `email` never decrypts tokens.

### Reminder opt-out (MVP.md 5.7, 9.3)

```ts
signUnsubscribeToken(requestId): string          // base64url(requestId) + "." + base64url(hmacSha256(EMAIL_LINK_SECRET, "unsub:" + requestId))
verifyUnsubscribeToken(t): { requestId } | null  // constant-time compare; no expiry, because the token can only stop reminders
```

`/api/email/unsubscribe?t=`:
- **GET** shows a small branded page: "Stop reminders for {title} from {business}?" with one **Stop reminders** button (a POST form). It changes nothing, because mail scanners open links.
- **POST** (the button, or a mail client's one-click with body `List-Unsubscribe=One-Click`) verifies `t` and calls `requests.stopRemindersByRecipient`. It works without cookies, always returns 200 for a valid `t` (idempotent, including revoked or completed requests), and shows "Reminders stopped. {business} can still email you about this request."
- An invalid `t` returns 400 with a plain page. Rate limited by IP (`portalPageLoad` policy).
- The token is never logged; Sentry scrubs the query string on this route.

### SES event handling (`handleSesEvent`)

| Event | Effect |
|---|---|
| `Delivery` | `email_messages.status = delivered` |
| `Bounce` with `bounceType = Transient` | Status unchanged (SES retries) |
| `Bounce` with `bounceType = Permanent` (including `OnAccountSuppressionList`) | Status `bounced`. If `kind` is `invite`, `reminder` or `change_request` and `request_id` is set: call `onRecipientBounced({ type: "bounce" })` |
| `Complaint` | Status `complained`. For the same kinds: call `onRecipientBounced({ type: "complaint" })` |
| `Reject` | Status `bounced`, Sentry warning |
| Unknown message ID | Ignored, logged at info (for example, mail sent before this table existed) |

How it's applied:
- Status updates are monotonic: `sent → delivered → bounced/complained`. A late `Delivery` after a `Bounce` doesn't downgrade the status.
- The handler is idempotent. Foundation's event-id dedupe applies, and the status updates are conditional.
- `onRecipientBounced` is implemented by `requests.markInviteBounced` and wired in `src/app/api/webhooks/aws/route.ts`.

## Data

Owns `email_messages` (MVP.md 7) and writes all of its rows. No schema changes.

## Code Style

```tsx
// src/server/email/templates/invite.tsx
export function InviteEmail({ businessName, recipientName, title, message, dueDate, portalUrl }: InviteProps) {
  return (
    <Layout preview={`${businessName} needs documents for ${title}`} brand={businessName}>
      <Text>Hi {recipientName},</Text>
      <Text>{businessName} uses DocuLink to collect documents securely. They need a few for <b>{title}</b>{dueDate ? ` by ${formatDate(dueDate)}` : ""}.</Text>
      {message ? <Quote>{message}</Quote> : null}
      <PrimaryButton href={portalUrl}>Add your documents</PrimaryButton>
      <LinkAsText href={portalUrl} />
    </Layout>
  );
}
```

## Testing Strategy

- **Unit:**
  - every template renders with its fixture props (snapshot of HTML and text)
  - subjects match the table exactly
  - the plain-text part contains the full link
  - no template prop type accepts `fileName`
  - display-name sanitizing
- **Integration** (OutboxTransport, Postgres, fake Redis):
  - `send` records `email_messages`
  - idempotency key: a second call is skipped
  - a transient failure retries and then throws, releasing the key
  - `handleSesEvent` applies the status table, including the monotonic rule, and calls `onRecipientBounced` only for the listed kinds
- **Unsubscribe:** sign and verify round-trip; tampered or truncated tokens fail; GET changes nothing; POST stops reminders once and is idempotent; the `reminder` template carries both headers and the footer link, and other kinds carry neither.
- **Contract:** recorded SES EventBridge payloads (Delivery, Bounce Permanent, Bounce Transient, Complaint, Reject) parse with the zod schema.
- **Manual (M5):** render every email in Gmail web and iOS, Outlook desktop and web, and Apple Mail, via staging sends. The checklist goes in the implementation doc.

MVP.md 14 acceptance criteria shared:

| Criterion | Test |
|---|---|
| An invite sent to the SES bounce simulator flags the request as "Invite bounced" within 5 minutes, pauses reminders and emails the requester | Staging E2E (`pnpm email:simulate -- bounce` plus polling), shared with `requests` |
| Rejecting 3 items sends exactly one email listing all 3 notes | Template test: the `change_request` fixture with 3 items lists 3 notes; the single-send part is in `review` |

## Edge Cases

- **The workspace has no reply-to email** → Reply-To falls back to the requester's sign-in email. It's never empty on client emails.
- **The business name contains an emoji, quotes or newlines** → sanitized in the display name. Rendered escaped in the body (React).
- **The recipient address is on the SES suppression list** → SES accepts the call and then emits a Permanent bounce (`OnAccountSuppressionList`), so the normal bounce path applies.
- **SES production access is still pending** (sandbox) → staging sends only to verified addresses. If production access is late, fall back to Postmark behind `EmailTransport` (MVP.md 15).
- **The SES event arrives before the `email_messages` row is inserted** (race in under a second) → the handler finds no row. Foundation releases the dedupe key on a "not found yet" error for events younger than 2 minutes, so EventBridge retries.
- **Very long item labels in `change_request`** → each label is wrapped. Notes are shown up to 500 characters, as entered.
- **A mail client posts the one-click unsubscribe months later** → still valid (no expiry). If the request is already finished, nothing changes and the response is still 200.

## Boundaries

- **Always:**
  - send through `send` (never call SES directly from other modules)
  - record every message
  - include a plain-text part
  - keep the subjects in MVP.md 5.7
- **Ask first:**
  - new email kinds
  - changing sender addresses or domains
  - adding open or click tracking (off; it would break the link-opened semantics in MVP.md 0 row 11)
- **Never:**
  - attach documents
  - include file names or file contents
  - log email bodies or tokens
  - send client emails without the requester's branding

## Success Criteria

- [ ] All 11 kinds render in the preview server and pass the snapshot tests.
- [ ] Staging sends through SES with the configuration set. `email_messages` rows match the SES message IDs.
- [ ] Bounce, complaint and delivery events update statuses and trigger `onRecipientBounced` for the right kinds, within 5 minutes on staging.
- [ ] Idempotency keys prevent duplicate sends on job retries.
- [ ] Inbox rendering checked in Gmail, Outlook and Apple Mail (M5).

## MVP.md changes (applied in v1.2, MVP.md 0 rows 38 and 43)

1. **MVP.md 5.7, 6.2:** the sender is `requests@mail.doculinkapp.com`.
2. **MVP.md 5.7, 7, 9.3:** reminder emails carry one-click unsubscribe, which stops reminders for that request only.

## Decisions (Oct 20, 2026)

1. **Sender address:** `requests@mail.doculinkapp.com`.
2. **List-Unsubscribe on reminders:** yes, one-click, scoped to the request.
