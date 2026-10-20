# Foundation: the base everything is built on

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-foundation.md](../../specs/SPEC-foundation.md) · Product spec: MVP.md 6, 7, 8, 10, 13 M1

## In one paragraph
Foundation is the plumbing: the app skeleton, the cloud setup, the database and a shared set of rules for how server code is written. It makes sure every feature, without extra effort, does these things:
- checks its input
- checks who is asking
- saves changes safely
- records what happened in the audit log
- limits abuse
- reports errors without leaking client data

Customers never see it, but every security answer we give depends on it.

## Who it's for
- **Developers.** Every other part of DocuLink is built on these conventions.
- **Customers and their compliance people,** indirectly. Encryption, access keys, audit logging and backups are set up here.

## How it works
1. **One app.** It's hosted on Vercel, with a staging copy and a production copy, kept apart down to separate AWS accounts and databases.
2. **The cloud setup is written as code** (Terraform): the private document storage, the encryption key, the email sender, the malware scanner and the event wiring. That makes it repeatable and reviewable. There are no stored AWS passwords; Vercel proves who it is to AWS for each request.
3. **The database** is created in one go from MVP.md section 7. The activity log is protected so it can only be added to, never edited.
4. **Shared server rules.** Every action follows the same steps:
   - check the input (for example, "is this a real email address?")
   - check who's asking and which workspace they belong to
   - apply rate limits
   - do the work in a single database transaction
   - record an audit event
   - only then send emails or schedule jobs
5. **Errors** come back to people as plain sentences that say what to do next. Technical details go to our error tracker with client information removed.

## Key decisions
| Decision | Why |
|---|---|
| Terraform for AWS | Most common, easy to audit, with security scanners that catch mistakes like public buckets |
| No AWS access keys anywhere | Nothing to leak or rotate; it's the first thing security questionnaires ask about |
| Emails and jobs only after a change is saved | A failed save never sends a misleading email |
| Status changes include "only if it's still in the state I expect" | Double clicks and two people acting at once can't corrupt a request |
| Strict import rules between modules | Keeps features independent, so one can change without breaking another |

## Not included
- Feature logic (requests, uploads, review): those are their own modules.
- The design system, which is the `ui` module.

## Decided
- The AWS setup is written in Terraform.
- The domain is doculinkapp.com, and email is sent from mail.doculinkapp.com.
- Developers use a sandbox in the staging AWS account rather than offline imitations of AWS.
