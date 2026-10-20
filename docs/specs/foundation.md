# Foundation: the base everything is built on

Status: Approved (revised for SOC 2 readiness) · Last updated: 2026-10-20
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
6. **SOC 2 readiness, from day one.** We aren't getting audited yet, but we run the controls an auditor checks, so the evidence builds up from the start:
   - every code change is reviewed by a second person before it reaches the main branch
   - automatic scans for vulnerable libraries, risky code and leaked secrets, with fix deadlines
   - logs kept for a year, plus alerts and uptime checks
   - staff sign in to AWS and every vendor with their own account and a second factor; no shared logins
   - staff never open client documents; any look at production data is recorded, and production data never leaves production
   - written runbooks for incidents, key rotation and backup restores, and a quarterly review of who has access

## Key decisions
| Decision | Why |
|---|---|
| Terraform for AWS | Most common, easy to audit, with security scanners that catch mistakes like public buckets |
| No AWS access keys anywhere | Nothing to leak or rotate; it's the first thing security questionnaires ask about |
| Emails and jobs only after a change is saved | A failed save never sends a misleading email |
| Status changes include "only if it's still in the state I expect" | Double clicks and two people acting at once can't corrupt a request |
| Strict import rules between modules | Keeps features independent, so one can change without breaking another |
| SOC 2 controls now, audit later | A SOC 2 Type II report covers months of evidence; starting the controls late delays the first report by those months. The audit itself costs money only worth spending once firms ask |

## Not included
- Feature logic (requests, uploads, review): those are their own modules.
- The design system, which is the `ui` module.

## Decided
- The AWS setup is written in Terraform.
- The domain is doculinkapp.com, and email is sent from mail.doculinkapp.com.
- Developers use a sandbox in the staging AWS account rather than offline imitations of AWS.
- SOC 2 controls start on day one and work for a team; the audit comes when firms ask.

## Open questions
- Where do we keep a year of logs? Recommended: Axiom, connected to Vercel in about an hour. The alternative keeps them in our own AWS account but takes about 2 days.
