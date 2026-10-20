# Spec: Authentication, two-factor and workspaces

Module id: `auth`
Status: Approved
Implements: MVP.md 1.1 (sign up, sign in, two-factor), 5.3 (sign in, two-factor setup, onboarding step 1, Security settings), 7 (workspaces, workspace_members, Better Auth tables), 9.1 (access rule), 10 (requester accounts, rate limits)
Designs: `4 Landing and sign-in - 02 Sign in.png`; design system DS 07 (fields, buttons)
Reading doc: [docs/specs/auth.md](../docs/specs/auth.md)
Last updated: 2026-10-20

## Objective

Let a requester (a tax preparer, accountant, HR team, lawyer or university office) sign up and sign in without a password, get a workspace automatically, and protect every client file behind a second factor, as the FTC Safeguards Rule requires (MVP.md 0 row 23). Every other module asks `auth` one question, "who is this and which workspace are they acting in?", and gets one answer.

User stories:
- **Sign in.** As a requester, I enter my work email and get a sign-in link, or use Google. Either way I land in my dashboard.
- **First sign-in.** As a new requester, my first sign-in creates my account and workspace and starts a 14-day trial. I'm asked for my business name and time zone, then to set up two-factor.
- **Two-factor.** I can create and send requests before setting up two-factor, but I can't preview, download or ZIP any file until I do.
- **Security settings.** I can see my two-factor status, regenerate backup codes, review trusted devices and active sessions, and sign out everywhere.

## Tech Stack

- **Better Auth** (1.x) with the Drizzle adapter and these plugins:
  - `magicLink`: 15-minute links, sent through `email`
  - `twoFactor`: TOTP with 6 digits and a 30-second step, 10 backup codes, trusted device for 30 days
  - social provider `google`
- **`qrcode`** renders the TOTP URI as an SVG on the server; the secret is never sent to a third-party QR service.
- It also relies on foundation (`defineAction`, `withTx`, `rateLimit`, `logEvent`, `track`), `ui` and `email`.

## Commands

The shared commands are in SPEC-foundation.md. Module-specific ones:

```
Auth schema:  pnpm dlx @better-auth/cli generate --output src/server/db/schema/auth.ts   # then pnpm db:generate
Test:         pnpm vitest run src/server/auth
Integration:  pnpm vitest run --project integration src/server/auth
E2E:          pnpm test:e2e e2e/auth
```

## Project Structure

```
src/server/auth/index.ts          → Public API: guards, getSession, auth instance export for the route
src/server/auth/config.ts         → betterAuth({...}) with plugins, hooks, session settings
src/server/auth/workspace.ts      → createWorkspaceForUser, getMembership
src/server/auth/guards.ts         → requireUser, requireWorkspace, requireTwoFactor
src/server/auth/rules.ts          → onboarding and two-factor state rules (pure)
src/server/auth/actions.ts        → completeOnboardingStep1, signOutEverywhere, revokeSession, revokeTrustedDevice
src/app/api/auth/[...all]/route.ts → Better Auth handler (toNextJsHandler)
src/app/(auth)/sign-in/page.tsx, check-email/page.tsx, confirm/page.tsx, two-factor/page.tsx
src/app/(app)/onboarding/page.tsx, setup-two-factor/page.tsx
src/components/auth/security-panel.tsx → Mounted by settings in the Security tab
```

## Interfaces

### Guards (used as `context` in `defineAction` and `defineRoute`)

```ts
type UserCtx = { user: { id: string; email: string; name: string | null }; session: { id: string; twoFactorVerified: boolean } };
type WorkspaceCtx = UserCtx & { workspace: Workspace; role: "owner" | "member" };

requireUser(req?): Promise<UserCtx>             // UNAUTHENTICATED if no valid session
requireWorkspace(req?): Promise<WorkspaceCtx>   // + membership; MVP: the user's only workspace
requireTwoFactor(req?): Promise<WorkspaceCtx>   // + user.twoFactorEnabled && session verified; else TWO_FACTOR_REQUIRED
requireOnboarded(ctx): void                     // redirects pages to /onboarding when workspace.onboardedAt is null
```

How the guards work:
- **Workspace selection.** In the MVP a user has exactly one membership. `requireWorkspace` loads it with a single join. When Teams ship, a `dl_ws` cookie selects among memberships without changing this interface.
- **Verified sessions.** With Better Auth's `twoFactor` plugin, a user who has two-factor enabled only gets a session after passing the TOTP or backup-code challenge, or from a trusted device. So `twoFactorVerified = user.twoFactorEnabled` for sessions created after enablement. A session created before the user enabled two-factor is treated as verified, because the user proved the code while enabling it.
- **Where `requireTwoFactor` is used:** the file URL route, the ZIP routes, the account deletion action, and the disable-two-factor action.

### Better Auth configuration (behavior)

**Sign-in methods:**
- **Magic link.** Single use, valid 15 minutes. The email (template `sign_in_link`) is sent through `email.send`. The emailed link is `${APP_URL}/sign-in/confirm#token=…` (built in the `sendMagicLink` callback from the token Better Auth passes). The confirm page shows "Sign in to DocuLink" and one **Sign in** button, which navigates to `/api/auth/magic-link/verify?token=…&callbackURL=/app`. The token sits in the URL fragment, so it isn't sent to our server or kept in logs until the button is pressed, and email scanners that open the link don't use it up (MVP.md 0 row 42). A missing or malformed fragment shows "This sign-in link isn't complete. Request a new one."
- **Google.** Scopes `openid email profile`. The account is linked automatically to an existing user only when Google reports the email as verified. Google's own 2-Step Verification doesn't count as our two-factor: Google users set up TOTP like everyone else (MVP.md 10).

**Sign-up:**
- The same flows create the user if the email is new. "Start free trial" on the sign-in page goes to the same form with copy changes.

**Sessions:**
- Expire after 7 days of inactivity (`expiresIn: 7d`, `updateAge: 1d`).
- Cookie: `__Secure-` prefixed, HttpOnly, SameSite=Lax.
- "Sign out everywhere" revokes all of the user's sessions, including the current one.

**Two-factor:**
- **Setup.** `twoFactor.enable` returns the TOTP URI and backup codes. Codes are stored hashed by the plugin and shown once, with Download (.txt) and Copy.
- **Trusted device.** Offered on the challenge as "Trust this device for 30 days" (off by default).
- **Backup codes.** "Regenerate backup codes" requires a current TOTP code.
- **Turning it off.** Disabling two-factor requires a current code and sends the `two_factor_changed` security email. The user then loses file access until they set it up again.
- **After a backup code is used:** a `two_factor_changed` email ("A backup code was used").

**Hooks:**
- `databaseHooks.user.create.after` → `createWorkspaceForUser(user)`, in the same transaction where Better Auth allows it, or idempotently otherwise. It creates:
  - a workspace: business name from the Google profile or the email domain, flagged not yet onboarded; time zone `America/New_York` until onboarding sets it; `subscription_status = trialing`; `trial_ends_at = now + 14 days`; `next_request_number = 1`
  - an `owner` membership
- `after sign-in` → PostHog `signed_up` on the first session only.

**Rate limits:** sign-in emails use `signInEmail` (5 per hour per address), applied before Better Auth sends anything; code attempts use `twoFactorAttempt` (below).

### Two-factor attempt limit

The `twoFactorAttempt` policy allows 5 failures per 15 minutes per user.
- At the 5th failure the account's two-factor challenge locks for 15 minutes, and a `two_factor_changed` email is sent ("Several wrong codes were entered").
- During the lock every attempt returns `RATE_LIMITED` with `retryAfterSeconds`.

### Actions

| Action | Input | Rules | Effects |
|---|---|---|---|
| `completeOnboardingStep1` | `businessName` (1–80, trimmed), `timeZone` (an IANA name in the supported list) | `requireWorkspace` | Updates the workspace and sets `onboarded_at`. Redirects to `/app/setup-two-factor` if two-factor is off, else `/app` |
| `signOutEverywhere` | — | `requireUser` | Revokes all sessions. Redirects to `/sign-in` |
| `revokeSession` | `sessionId` | Must belong to the user | Revokes it |
| `revokeTrustedDevice` | `deviceId` | Must belong to the user | Removes trust |

Two-factor enable, verify, disable and backup-code calls go through the Better Auth client SDK on the client; this module wraps them in the UI with the rate limit and emails above.

### Pages

**Sign in** (design `4 … 02`):
- Layout: a brand panel on the left ("Welcome *back.*" with the cat); on the right, the form ("Work email", **Email me a sign-in link**, "or", **Continue with Google**, "New to DocuLink? Start free trial").
- After submit: a "Check your inbox" state with the exact copy from MVP.md 5.3 and a "Use a different email" link.
- Errors: an invalid email gets a field error ("Enter an email address, like name@example.com"). Rate limited: "Too many sign-in links. Try again in N minutes."
- Expired or used link → `/sign-in?error=link_expired`, showing "That link has expired. We'll send you a new one." with the email prefilled.

**Two-factor challenge:**
- A 6-digit code field with `autocomplete="one-time-code"` and `inputmode="numeric"`.
- "Use a backup code" toggles to a backup-code field.
- "Trust this device for 30 days" checkbox, unchecked by default.

**Onboarding step 1:** business name and time zone (the US zones first, then all IANA zones), with the default guessed from the browser. Required and not skippable.

**Two-factor setup:**
- Copy: "Client documents need a second lock."
- A QR code (SVG), the manual key in mono groups of 4, a code field, then 10 backup codes with Download and Copy.
- Must confirm "I've saved my backup codes" to finish.
- Skippable ("Set up later"). Until it's done, the shell shows a banner: "Set up two-factor to view client files."

**Security panel** (mounted by settings): two-factor status, regenerate backup codes, turn off two-factor, trusted devices (browser, last used, revoke), active sessions (browser, IP city if available, last active, revoke), Sign out everywhere.

## Data

Owned tables:
- `workspaces`: identity columns, `onboarded_at` (proposed); billing columns are written by `billing`, settings columns by `settings`
- `workspace_members`
- the Better Auth tables `user`, `session`, `account`, `verification`, `twoFactor`, `trustedDevice` (generated by the CLI)

Column (in MVP.md 7 since v1.2): `workspaces.onboarded_at timestamptz null`. `business_name` is `not null`, so a workspace is created with a provisional name at sign-up. This column is how the app knows step 1 is done.

Better Auth tables are created in the first migration, from the CLI output, and kept in `src/server/db/schema/auth.ts`.

## Code Style

```ts
// src/server/auth/guards.ts
export async function requireTwoFactor(): Promise<WorkspaceCtx> {
  const ctx = await requireWorkspace();
  if (!ctx.user.twoFactorEnabled || !ctx.session.twoFactorVerified) {
    throw new AppError("TWO_FACTOR_REQUIRED", "Set up two-factor sign-in to view client files.");
  }
  return ctx;
}
```

## Testing Strategy

- **Unit** (`rules.test.ts`): onboarding redirects, two-factor gate decisions, time zone list validation, business-name normalization.
- **Integration** (Postgres plus Better Auth in memory over the real schema):
  - sign-up creates exactly one workspace and an owner membership, even when the hook runs twice
  - the magic link is single use and expires after 15 minutes (frozen clock)
  - fetching the emailed URL (as a scanner would) doesn't consume the token
  - the sign-in email rate limit
  - `requireWorkspace` returns NOT_FOUND-style failures for missing memberships
  - `requireTwoFactor` blocks a user without two-factor and allows a verified one
  - two-factor lock after 5 failures, plus the security email
- **E2E** (Playwright, staging with an outbox mail reader):
  - magic-link sign-up → onboarding → set up TOTP (codes generated in the test from the secret with `otpauth`) → dashboard
  - sign out, sign back in with TOTP
  - backup code sign-in sends a security email
  - Google sign-in on staging with a test Google account (manual in M5 if automation is blocked)

MVP.md 14 acceptance criteria owned or shared:

| Criterion | Test |
|---|---|
| A requester without two-factor set up, or with an unverified session, cannot preview, download or ZIP any file | Integration: guard tests; E2E: a user without 2FA gets `TWO_FACTOR_REQUIRED` from the file URL and ZIP routes (shared with `files` and `exports`) |
| A requester cannot read another workspace's request or files (ID tampering) | Integration: two workspaces; every guarded action and route returns NOT_FOUND across workspaces (a table-driven test that each module adds its routes to) |

## Edge Cases

- **The same email signs up with a magic link and later with Google** → the accounts are linked (Google-verified email). Still one user, one workspace.
- **A Google account with an unverified email** → not linked. "Sign in with your email link first, then connect Google in Settings."
- **The magic link is opened on another device** → it works (it's a bearer link). The session is created on that device.
- **The magic link is opened by an email security scanner first** → the scanner gets the confirm page, which does nothing without a click, and the fragment isn't sent to the server. The token survives for the user.
- **The user loses their authenticator and backup codes** → in the MVP, support-assisted reset: we verify by email and the Stripe billing email, then clear two-factor. This is documented in the internal runbook, with no self-service path.
- **The time zone isn't in the IANA list** → validation error. The browser's guess falls back to `America/New_York`.
- **Session expires mid-action** → the action returns `UNAUTHENTICATED`, and the client redirects to `/sign-in?next=<path>`.

## Boundaries

- **Always:**
  - every requester page and action goes through `requireWorkspace`
  - file access goes through `requireTwoFactor`
  - security events send the security email
- **Ask first:**
  - adding auth providers or passkeys (should-have)
  - changing session lifetime
  - self-service two-factor recovery
- **Never:**
  - store or log TOTP secrets or backup codes in plain text
  - put tokens in URLs other than the magic link
  - return 403 for another workspace's resources (always 404)

## Success Criteria

- [ ] New users get exactly one workspace and an owner membership, with a 14-day trial.
- [ ] Magic link and Google sign-in work on staging. Magic links go through the confirm page, expire after 15 minutes and are single use.
- [ ] Onboarding step 1 is enforced before the app is usable. Two-factor setup is offered right after it and can be deferred.
- [ ] File routes reject users without verified two-factor (tested in this module and in `files`/`exports`).
- [ ] The 2FA lock, the security emails and Sign out everywhere work.
- [ ] Cross-workspace access returns 404 for every guarded route (table-driven test).

## MVP.md changes (applied in v1.2, MVP.md 0 rows 42, 44 and 45)

1. **MVP.md 7:** `workspaces.onboarded_at timestamptz null`.
2. **MVP.md 10:** the account-recovery runbook for lost two-factor (support-assisted, MVP); the "Confirm sign-in" page; Google's 2-Step Verification doesn't count.

## Decisions (Oct 20, 2026)

1. **Magic link scanner protection:** yes. The link opens a "Confirm sign-in" page with one button.
2. **Google 2-Step Verification:** doesn't satisfy our two-factor requirement.
