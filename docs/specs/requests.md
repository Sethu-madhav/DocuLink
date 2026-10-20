# Requests: building, sending and tracking document requests

Status: Draft, revision 2 waiting for your review · Last updated: 2026-10-20
Full spec: [specs/SPEC-requests.md](../../specs/SPEC-requests.md) · Product spec: MVP.md 3, 4.1, 4.7, 4.8, 5.3, 9.1

## In one paragraph
This is the accountant's main workspace. They build a checklist of documents (from a ready-made template or from scratch), send it to a client as one secure link, and watch every request on a dashboard that answers "what needs me?". It also holds the rules for every request's life: what can happen when, what can be edited after sending, when reminders run, and when a link expires.

## Who it's for
Requesters: tax preparers, accountants, HR teams, lawyers and university offices. Clients only see the link and the emails it produces.

## How it works
1. **New request.**
   - Pick a template, such as "Individual tax return", or start blank. Templates are grouped as Tax files, Accounting, HR and Universities, each with its own icon, plus Custom for the ones you save yourself.
   - Fill in the client's name and email, the title, an optional due date and a message.
   - Each item has what's needed, instructions, and "why" it's needed. A live phone preview shows what the client will see.
2. **Send.** After a confirmation dialog, DocuLink creates a secret link, emails it from the firm's name and schedules reminders. Each request gets a number, like "No. 0142".
3. **Dashboard.**
   - "Two to review." at the top, with cards for requests that need review.
   - A table of requests waiting on clients, with progress and due dates; bounced invites appear first.
   - Counts for completed requests and drafts, and search.
4. **Managing a request:**
   - send a reminder now
   - fix a mistyped email and resend
   - get a fresh link (the old one stops working)
   - extend the deadline
   - revoke the request
   - delete all its files once it's finished
5. **Templates page.** Twelve ready-made templates in four groups, plus your own under Custom. Editing a ready-made one saves your own copy.
6. **Rules everyone uses.** A request moves through fixed steps: draft, sent, in progress, needs review, changes requested, completed, plus expired and revoked. Nothing can skip a step. A completed request can be reopened by asking for changes again.

## Key decisions
| Decision | Why |
|---|---|
| One tested set of status rules for the whole product | Three different status vocabularies would cause bugs, as MVP.md section 0 found |
| Request numbers per workspace | They're on every screen in the designs, and people quote them on calls |
| A "Why" line on every item | Clients are more willing to send sensitive documents when they know why |
| Links stored in scrambled form | Lets us resend links in reminders without storing them readable |
| Deleting a draft really deletes it | A draft never had a link or files; showing it as "revoked" would confuse |
| Templates grouped by who uses them | Five kinds of requesters; each should find their starting point in one look |
| A client can turn off reminders for one request | Better than being marked as spam; the accountant can see it happened |

## Not included
- The client's side (portal) and the review page (review): separate modules.
- Teams and shared templates (after launch).

## Decided
- The dashboard follows the design: a headline, cards for requests to review, and a table of requests waiting on clients.
- Item instructions can't be edited after sending.
- Templates are grouped as Tax files, Accounting, HR, Universities and Custom, each with its own icon.

## Open questions
- Are the twelve new starter templates right? They're new in this revision, and each group should be checked by someone who works in that field.
- Lawyers are in the audience but have no template group. Add a Legal group, or leave lawyers to build their own under Custom?
