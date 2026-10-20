# Automation: digests, reminders and nightly clean-up

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-automation.md](../../specs/SPEC-automation.md) · Product spec: MVP.md 8.1–8.3, 9.3

## In one paragraph
Three background jobs do the chasing and tidying so the accountant doesn't have to:
- A **digest** bundles a burst of client uploads into one email.
- **Reminders** nudge clients politely in the morning, at most five times.
- A **nightly clean-up** expires old requests, removes abandoned uploads, deletes files whose retention period is over, re-checks any stuck malware scans and expires old ZIP downloads.

All three are built so that a retry never sends a second email.

## Who it's for
- **Accountants**, who get fewer, better emails and don't chase clients by hand.
- **Clients**, who get timely, polite reminders.

## How it works
1. **Digest.** After an upload, we wait until the client has been quiet for 10 minutes (or at most 30 minutes) and send one summary: "John Doe added 2 of 3 documents". If the client submits first, the digest is skipped and the accountant gets the "submitted" email instead.
2. **Reminders.** Every hour, we find reminders that are due and send each one between 9 and 11 am in the accountant's time zone.
   - They stop when the client submits, after 5 reminders, if the invite bounced, or if the request expired.
   - They restart if the accountant asks for changes.
3. **Nightly clean-up** (3 am UTC):
   1. Mark overdue requests as expired.
   2. Delete uploads that were never finished.
   3. Finish deleting removed files.
   4. Delete files from completed requests once the retention period ends.
   5. Re-check malware scans that seem stuck.
   6. Expire old ZIP links.

## Key decisions
| Decision | Why |
|---|---|
| Each reminder is "claimed" before sending | Two overlapping runs or a retry can never double-send |
| Mornings in the accountant's time zone | A 3 am reminder gets ignored |
| Automatic retention deletion at launch | The landing page and client screens promise "deleted after 90 days", so it has to be true |
| Each clean-up step runs on its own | One problem doesn't stop the rest; failures alert us |

## Not included
- The "files will be deleted in 7 days" warning email (planned soon after launch).
- Per-person notification settings (should-have).

## Decided
- The email that warns before files are deleted comes in the first week after launch. Until then, the completed request shows the deletion date.
