# Capability Map: DocuLink MVP

Status: Approved (revision 2 approved 2026-10-20)
Source: MVP.md (version 1.2)
Last updated: 2026-10-20

Modules are vertical capabilities. Each feature module owns its own screens, actions, routes and tests. The `ui` module owns the design system those screens are built from, but not the screens themselves.

| Module id | Responsibility | MVP.md sections | Designs | Depends on |
|---|---|---|---|---|
| foundation | Next.js scaffold, CI, infrastructure-as-code (AWS accounts, S3, KMS, IAM/OIDC, SES identity, GuardDuty, EventBridge), Neon + Drizzle schema and migrations, env config, Sentry/PostHog wrappers, rate-limit helper, security headers, `/api/webhooks/aws` dispatcher | 6, 6.1, 6.2, 6.3, 7, 10, 13 M1 | — | — |
| ui | Design system: tokens, type roles, fonts, base and composite components (status badge, stamp mark, bottom sheet, sticky action bar, progress, toast, skeleton, empty state, confirm dialog), requester and portal app shells, loading/empty/error patterns, accessibility rules, component gallery | 5.1, 5.2, 5.4, 5.8, 5.9 | all (style reference) | foundation |
| auth | Better Auth (magic link, Google), two-factor (TOTP, backup codes, trusted devices), sessions, workspaces and owner membership, workspace and two-factor guards, onboarding step 1, forced 2FA setup, sign-in screens, Security settings panel | 5.3 (sign in, 2FA, onboarding 1, Security), 7, 10 | 4 Landing and sign-in - 02 | foundation, ui, email |
| email | SES client over OIDC, React Email layout and all templates, `email_messages` log, SES delivery/bounce/complaint handling exposed as events, reminder unsubscribe links | 5.7, 8.4 (SES), 7 | — | foundation |
| files | S3 key layout, presigned PUT with SHA-256 checksum, confirm (HEAD), GuardDuty scan results and `scan_status`, signed GET for preview and download, delete all versions, logo storage | 4.2 (storage steps), 6.1, 8.4 (GuardDuty), 9.1 file URL, 10 | — | foundation |
| requests | Request and item state machines (including Reopen), request number, builder with live preview, templates (page, grouped starter set, categories), drafts, send (token, invite), dashboard, duplicate, revoke, regenerate link, extend deadline, bounced invite and Fix email, reminder schedule state and client opt-out, digest trigger, activity log reads | 3, 4.1, 4.7, 4.8, 5.3 (dashboard, builder), 7, 9.1 | 2 Requester dashboard - 01, 02 | auth, ui, email, billing |
| portal | Client side: token + Turnstile session, checklist, desktop QR phone handoff, Add file sheet, camera capture, HEIC/resize, uploads with progress and retry, submit and resubmit, change-requested view, success and terminal states | 4.2, 4.3, 4.5, 5.6, 9.2 | 1 Client portal - 01, 02 | requests, files, ui |
| review | Request detail, preview panel with scan states, approve with stamp, draft and send change requests, Reopen, keyboard review, header actions | 4.4, 5.3 (detail and review), 9.1 | 3 Document review - 01, 02 | requests, files, email, ui |
| automation | Upload digest, hourly reminders, daily maintenance (expiry, orphan cleanup, retention), QStash job routes | 8.1, 8.2, 8.3, 9.3 | — | requests, files, email |
| exports | ZIP naming, direct streaming ZIP up to 200 MB, background ZIP via S3 multipart, ZIP-ready email | 4.6, 5.5, 8.5, 9.1 | — | requests, files, email |
| billing | Entitlements (trial, active, past due, lapsed), Stripe Checkout (monthly, annual), Stripe Tax, customer portal, webhooks, trial and lapsed banners, upgrade dialog, Billing settings panel | 11, 5.3 (billing states), 9.1, 9.3 | — | auth, ui |
| settings | Settings shell and tabs: Workspace (name, logo, reply-to, time zone, retention, default reminders), Account (sign-in methods, delete account); hosts the Security panel (auth) and Billing panel (billing); onboarding steps 2 and 3; account deletion | 5.3 (settings, onboarding 2–3), 10 (account deletion) | 5 Settings - 01 (02 is post-MVP, hidden) | auth, files, requests, billing, ui |
| site | Landing page, pricing, sample request, security page (FTC Safeguards Rule, IRS Pub 4557), "How is my data protected?", "Why did I get this?", terms, privacy, sub-processors | 1.1 (security page), 10, 13 M5 | 4 Landing and sign-in - 01 | ui, portal |

Build order: foundation → ui, email → auth → files → billing (entitlements only) → requests → portal → review → automation, exports → billing (Stripe) → settings → site

This follows the MVP.md 13 milestones: foundation, ui, email (sign-in link only) and auth in week 1; files, billing entitlements, requests and portal in weeks 2 to 3; review and automation in week 4; exports, billing Stripe and settings in week 5; site and hardening in weeks 6 to 7. Modules separated by a comma can be built in parallel.

## Boundary notes
- **Import rule.** Code in `src/server/<module>/` imports only the public `index.ts` of modules it depends on in this table. Routes and pages in `src/app/` may import any module and are where modules are wired together (for example, the AWS events route passes `requests.markInviteBounced` to `email`'s SES handler). A lint rule enforces this (SPEC-foundation.md).
- **ui vs feature modules.** `ui` provides tokens, components, shells and patterns with no data fetching and no business logic. Feature modules compose them into screens and own those screens' specs and tests. The contract (component APIs, token names) lives in SPEC-ui.md.
- **Workspace guard.** `auth` exposes the session, workspace and two-factor checks every requester action and file route uses. Contract in SPEC-auth.md.
- **AWS events.** `foundation` provides the route factory (secret check, idempotency, dispatch by source). `email` handles SES events; `files` handles GuardDuty results; their follow-up effects on requests are callbacks wired in the route. Contracts in SPEC-email.md and SPEC-files.md.
- **Request state.** `requests` owns the status and item-status transitions (MVP.md 3), the reminder schedule rows and the digest cursor. `portal`, `review`, `automation` and `exports` change that state only through its functions. Contract in SPEC-requests.md.
- **Entitlements.** `billing` exposes a pure `getEntitlements(workspace, now)`; `requests` calls it before creating or sending. The Stripe parts come later and don't change that contract.
- **Server conventions.** Validation, errors, Server Action and route wrappers, transactions, activity logging, rate limits, jobs and crypto helpers live in `foundation` and are described in SPEC-foundation.md. Every module uses them.
- **Bounced invite.** `email` detects the bounce and emits it; `requests` sets `invite_bounced_at`, pauses reminders and owns Fix email.
- **Emails.** `email` owns the layout and every template; the module that triggers an email calls `email`'s send function.
- **Analytics.** `foundation` provides the PostHog wrapper with masking; each module emits its own MVP.md 12 events.
- **SOC 2 readiness** (MVP.md 10) is not a module. `foundation` owns the engineering controls (branch protection, scanners, logs, staff access, runbooks); people controls are company processes.
- **Security hardening** (rate limits, CSP, scrubbing, device and accessibility passes in 13 M5) is not a module. Each module's spec carries its share; `foundation` provides the helpers.

## Not in any module (deliberately)
- Everything in MVP.md 1.2 (should-haves after launch) and 1.3 (later), including Teams and the `5 Settings - 02 Team` screen.

## Revision log
- **Revision 2 (2026-10-20, approved).** Written while drafting the module specs:
  - `auth` depends on `email` (sign-in link and security emails).
  - `requests` depends on `billing` (entitlements before create and send).
  - The Templates page moves from settings to `requests`; settings gains account deletion.
  - `site` gains the sample request, which reuses portal components, so `site` depends on `portal`.
  - The import rule and server conventions boundary notes are added.
  - The build order is adjusted to match.
  - After the Oct 20 decisions (MVP.md 1.2): the QR handoff goes to `portal`, "Why did I get this?" to `site`, Reopen to `review` and `requests`, and reminder unsubscribe to `email` and `requests`. These add no new dependencies.
