# Capability Map: DocuLink MVP

Status: Approved
Source: MVP.md (version 1.1)
Last updated: 2026-10-20

Modules are vertical capabilities. Each feature module owns its own screens, actions, routes and tests. The `ui` module owns the design system those screens are built from, but not the screens themselves.

| Module id | Responsibility | MVP.md sections | Designs | Depends on |
|---|---|---|---|---|
| foundation | Next.js scaffold, CI, infrastructure-as-code (AWS accounts, S3, KMS, IAM/OIDC, SES identity, GuardDuty, EventBridge), Neon + Drizzle schema and migrations, env config, Sentry/PostHog wrappers, rate-limit helper, security headers, `/api/webhooks/aws` dispatcher | 6, 6.1, 6.2, 6.3, 7, 10, 13 M1 | — | — |
| ui | Design system: tokens, type roles, fonts, base and composite components (status badge, stamp mark, bottom sheet, sticky action bar, progress, toast, skeleton, empty state, confirm dialog), requester and portal app shells, loading/empty/error patterns, accessibility rules, component gallery | 5.1, 5.2, 5.4, 5.8, 5.9 | all (style reference) | foundation |
| auth | Better Auth (magic link, Google), two-factor (TOTP, backup codes, trusted devices), sessions, workspaces and owner membership, workspace guard, onboarding step 1, forced 2FA setup, sign-in screens | 5.3 (sign in, 2FA, onboarding 1), 7, 10 | 4 Landing and sign-in - 02 | foundation, ui |
| email | SES client over OIDC, React Email layout and all templates, `email_messages` log, SES delivery/bounce/complaint handling exposed as events | 5.7, 8.4 (SES), 7 | — | foundation |
| files | S3 key layout, presigned PUT with SHA-256 checksum, confirm (HEAD), GuardDuty scan results and `scan_status`, signed GET for preview and download, delete all versions, logo storage | 4.2 (storage steps), 6.1, 8.4 (GuardDuty), 9.1 file URL, 10 | — | foundation |
| requests | Request and item state machines, builder with live preview, templates, drafts, send (token, invite), dashboard, duplicate, revoke, regenerate link, extend deadline, bounced invite and Fix email, activity log | 3, 4.1, 4.7, 4.8, 5.3 (dashboard, builder), 9.1 | 2 Requester dashboard - 01, 02 | auth, ui, email |
| portal | Client side: token + Turnstile session, checklist, Add file sheet, camera capture, HEIC/resize, uploads with progress and retry, submit and resubmit, change-requested view, success and terminal states | 4.2, 4.3, 4.5, 5.6, 9.2 | 1 Client portal - 01, 02 | requests, files, ui |
| review | Request detail, preview panel with scan states, approve with stamp, draft and send change requests, keyboard review, header actions | 4.4, 5.3 (detail and review), 9.1 | 3 Document review - 01, 02 | requests, files, email, ui |
| automation | Upload digest, hourly reminders, daily maintenance (expiry, orphan cleanup, retention), QStash job routes | 8.1, 8.2, 8.3, 9.3 | — | requests, files, email |
| exports | ZIP naming, direct streaming ZIP up to 200 MB, background ZIP via S3 multipart, ZIP-ready email | 4.6, 5.5, 8.5, 9.1 | — | requests, files, email |
| billing | Stripe Checkout (monthly, annual), Stripe Tax, customer portal, webhooks, trial banner, lapsed gating | 11, 5.3 (billing states), 9.1, 9.3 | — | auth, ui |
| settings | Settings shell and tabs: Workspace (logo, reply-to, time zone), Defaults, Templates, Account; hosts the Security tab (from auth) and Billing tab (from billing); onboarding steps 2 and 3 | 5.3 (settings, onboarding 2–3) | 5 Settings - 01 (02 is post-MVP, hidden) | auth, files, requests, billing, ui |
| site | Landing page, pricing, security page (FTC Safeguards Rule, IRS Pub 4557), "How is my data protected?", terms, privacy | 1.1 (security page), 10, 13 M5 | 4 Landing and sign-in - 01 | ui |

Build order: foundation → ui → auth → email, files → requests → portal → review → automation, exports → billing → settings → site

This follows the MVP.md 13 milestones: foundation, ui and auth in week 1; email, files, requests and portal in weeks 2 to 3; review and automation in week 4; exports, billing and settings in week 5; site and hardening in weeks 6 to 7. Modules separated by a comma can be built in parallel.

## Boundary notes
- **ui vs feature modules.** `ui` provides tokens, components, shells and patterns with no data fetching and no business logic. Feature modules compose them into screens and own those screens' specs and tests. The contract (component APIs, token names) lives in SPEC-ui.md.
- **Workspace guard.** `auth` exposes the session, workspace and two-factor checks every requester action and file route uses. Contract in SPEC-auth.md.
- **AWS events.** `foundation` owns `POST /api/webhooks/aws` (secret check, dispatch by source). `email` handles SES events; `files` handles GuardDuty results. Contracts in SPEC-email.md and SPEC-files.md.
- **Request state.** `requests` owns the status and item-status transitions (MVP.md 3). `portal`, `review`, `automation` and `exports` change state only through its functions. Contract in SPEC-requests.md.
- **Bounced invite.** `email` detects the bounce and emits it; `requests` sets `invite_bounced_at`, pauses reminders and owns Fix email.
- **Emails.** `email` owns the layout and every template; the module that triggers an email calls `email`'s send function.
- **Analytics.** `foundation` provides the PostHog wrapper with masking; each module emits its own MVP.md 12 events.
- **Security hardening** (rate limits, CSP, scrubbing, device and accessibility passes in 13 M5) is not a module. Each module's spec carries its share; `foundation` provides the helpers.

## Not in any module (deliberately)
- Everything in MVP.md 1.2 (should-haves after launch) and 1.3 (later), including Teams and the `5 Settings - 02 Team` screen.
- Open questions in MVP.md 15 (QR phone handoff, "Why did I get this?" page) until decided.
