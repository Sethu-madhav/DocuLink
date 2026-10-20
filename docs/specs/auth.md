# Auth: signing in, two-factor and workspaces

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-auth.md](../../specs/SPEC-auth.md) · Product spec: MVP.md 5.3, 7, 9.1, 10

## In one paragraph
Accountants sign in with a link emailed to them, or with Google; there are no passwords. Their first sign-in creates their workspace and starts a 14-day trial. Before they can open any client file, they must set up a second lock: a six-digit code from an authenticator app. The FTC Safeguards Rule expects this of tax preparers, and firms will ask about it.

## Who it's for
Requesters (accountants). Clients never sign in; they use their link.

## How it works
1. **Sign in.** Enter your work email and get a link valid for 15 minutes, or click "Continue with Google".
2. **First sign-in.** We create your account and workspace and ask for your business name and time zone.
3. **Two-factor.** Google's own 2-Step Verification doesn't count; everyone sets up DocuLink's. Scan a QR code with an authenticator app and save 10 backup codes. You can skip this for now, but you won't be able to open client files until you do it.
4. **Later sign-ins.** Enter the code from your app, or tick "Trust this device for 30 days".
5. **Security settings.** See your devices and sessions, regenerate backup codes, and sign out everywhere. Turning two-factor off, or using a backup code, sends you a warning email.
6. **Behind the scenes.** Every page and action asks auth "who is this, and which workspace?". Anything belonging to another workspace simply looks like it doesn't exist.

## Key decisions
| Decision | Why |
|---|---|
| No passwords | Nothing to reuse or leak; the email and phone are the keys |
| Two-factor required for files, not for sending requests | Lets people try the product immediately, while protecting the sensitive part |
| Repeated wrong codes lock for 15 minutes and email you | Stops guessing and tells the owner |
| One workspace per person now, built for teams later | Teams are the first feature after launch, with no data migration needed |

## Not included
- Passkeys (planned soon after launch).
- Team members and roles (after launch).
- Self-service recovery when both the phone and the backup codes are lost: support handles it.

## Decided
- The emailed sign-in link opens a "Confirm sign-in" page with one button, so email security scanners can't use the link up.
- Google's own 2-Step Verification doesn't replace our two-factor.
