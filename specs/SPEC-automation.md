# Spec: Background jobs (digest, reminders, maintenance)

Module id: `automation`
Status: Approved
Implements: MVP.md 8 (conventions), 8.1, 8.2, 8.3, 9.3 (job routes), 14 (digest and reminder criteria)
Designs: none
Reading doc: [docs/specs/automation.md](../docs/specs/automation.md)
Last updated: 2026-10-20

## Objective

Do the chasing so the requester doesn't have to, without ever annoying anyone with duplicates:
- one digest email for a burst of uploads
- polite reminders to clients at 9 to 11 am in the requester's time zone, at most five
- nightly housekeeping: expire old requests, clean up abandoned uploads, delete files past retention, catch stuck scans and expire old exports

Success means:
- the MVP.md 14 digest and reminder criteria pass
- QStash retries never cause a duplicate email
- every job failure is visible in Sentry

## Tech Stack

Foundation's `createJobRoute`, `publishJob`, Redis, `withTx`, `logEvent`, `clock` and Sentry Cron monitors, plus:
- `requests`: rules, `nextLocalMorning`, `buildPortalUrl`, transitions, reminder functions, digest cursor
- `files`: deletion, `recheckScan`
- `email`: `send`
- `exports`: export expiry status

## Commands

```
Test:         pnpm vitest run src/server/automation
Integration:  pnpm vitest run --project integration src/server/automation
Run locally:  pnpm jobs:run -- reminders|maintenance|digest --rid <id>   # calls the handler directly with a fake signature bypass in dev only
Schedules:    pnpm jobs:schedule -- staging                               # creates/updates QStash schedules
```

## Project Structure

```
src/server/automation/index.ts
src/server/automation/digest.ts        → runDigest(rid)
src/server/automation/reminders.ts     → runReminders(), claimAndSend()
src/server/automation/maintenance.ts   → runMaintenance() with steps 1–6
src/app/api/jobs/digest/route.ts       → createJobRoute → runDigest
src/app/api/jobs/reminders/route.ts
src/app/api/jobs/maintenance/route.ts
scripts/jobs-schedule.ts               → QStash schedules: reminders "0 * * * *", maintenance "0 3 * * *" (UTC)
```

## Interfaces

All job routes:
- use `createJobRoute`: the QStash signature is verified, and the body is a zod object (`{ rid: uuid }` for digest, `{}` for the scheduled jobs)
- set `maxDuration = 300`
- are idempotent, return 200 when there's nothing to do, and throw to make QStash retry
- report to Sentry Cron monitors (`reminders-hourly`, `maintenance-daily`) with check-ins at start and finish

### Digest (`POST /api/jobs/digest`), MVP.md 8.1

The trigger, `requests.noteActivity`, is called by `portal` on confirm and remove. The job:

```
if request.status not in (in_progress, changes_requested): clear keys; return
quiet = now - dg:last ; age = now - dg:first
if quiet < 10 min and age < 30 min: publishJob(digest, { rid }, delay = 10 min - quiet); return
DEL dg:job, dg:first, dg:last
events = activity_events where request_id = rid and id > last_digest_event_id and type in (file_uploaded, file_removed)
if none: return
send email "digest" (idempotency "digest:{rid}:{max(events.id)}")
UPDATE document_requests SET last_digest_event_id = max(events.id) WHERE id = rid AND last_digest_event_id < max(events.id)
```

Digest content: "{Client} added {x} of {y} documents". It lists item labels that gained or lost files since the last digest, with counts, and never file names.

On submit or resubmit, `advanceDigestCursor` (requests) moves the cursor and clears the keys, so a pending digest finds nothing (MVP.md 8.1).

### Reminders (`POST /api/jobs/reminders`, hourly), MVP.md 8.2

```
loop until no rows or 250 s elapsed:
  due = reminder_schedules s join document_requests r join workspaces w
        where s.is_active and s.next_send_at <= now
          and r.status in (pending, in_progress, changes_requested)
          and (r.expires_at is null or r.expires_at > now)
          and r.invite_bounced_at is null
          and s.sent_count < s.max_count
        order by s.next_send_at limit 200
  for each (sequentially, 10 concurrent max):
    claimed = UPDATE reminder_schedules
              SET sent_count = sent_count + 1,
                  next_send_at = nextLocalMorning(now + interval_days days, w.time_zone),
                  is_active = (sent_count + 1 < max_count)
              WHERE request_id = $1 AND next_send_at = $old RETURNING *
    if claimed:
      send email "reminder" (link = buildPortalUrl(r); final = claimed.sent_count = max_count; idempotency "reminder:{rid}:{claimed.sent_count}")
      logEvent reminder_sent (system, meta: { n: claimed.sent_count, final })
```

Rules:
- The claim and the event go in one small transaction per row. The email goes after commit. If the email fails after 3 retries, the claim stands: one missed reminder is better than a duplicate. Sentry records it, and the next interval still fires.
- Reminder copy: "{n} documents still needed". The count is required items without active files, plus rejected items in changes_requested. The final reminder mentions the due date if set and says "This is the last reminder."
- When `is_active` becomes false, the request header shows "Reminders finished" (via `review`), or "{Client} turned off reminders" when `stopped_by_recipient_at` is set.
- Every reminder carries the one-click unsubscribe headers and footer link (`email`, MVP.md 5.7). A stopped schedule is `is_active = false`, so the query above never selects it.

### Maintenance (`POST /api/jobs/maintenance`, daily 03:00 UTC), MVP.md 8.3

Each step runs in its own try/catch, records counts, and is safe to rerun. A step failure doesn't stop later steps. The job throws at the end if any step failed, so Sentry and QStash flag it.

| # | Step | Rule | Effects |
|---|---|---|---|
| 1 | Expire | Status in (pending, in_progress, changes_requested) and `expires_at < now` | `applyTransition(EXPIRE)` per request (conditional); logs `request_expired` (system). Batches of 500 |
| 2 | Orphans | `uploaded_files` with status `pending` and `created_at < now - 2h` | `deleteObjectAllVersions(key)` (ignore missing), then delete the row. No event (never confirmed) |
| 3 | Removed files | Status `removed`, and the object still exists (`HeadObject`) or was removed less than 7 days ago | `deleteObjectAllVersions(key)` |
| 4 | Retention | Status `completed`, `completed_at + workspace.retention_days < now` (1 to 365 days, MVP.md 4.7), `files_deleted_at is null`, and no item with a drafted change note (a Reopen in progress) | `files.deleteRequestObjects(rid)`; files `removed`; `files_deleted_at = now`; logs `files_deleted` (system, meta `{ reason: "retention" }`). Must-have (MVP.md 1.1) |
| 5 | Stuck scans | `scan_status = pending` and `confirmed_at < now - 30 min` | `files.recheckScan(id)` applies the tag result through the same handler; still pending after 2 h → a Sentry alert per file (no names) |
| 6 | Exports | `zip_exports` status `ready` and `expires_at < now` | Status `expired` (the S3 lifecycle removes the objects); `queued` or `building` for more than 1 h → `failed` with "timed out" |

## Data

No owned tables. It writes:
- `reminder_schedules` (claims)
- `document_requests` (`last_digest_event_id`, expiry status, `files_deleted_at`) through `requests` and `files` functions
- `uploaded_files` (orphans, retention)
- `zip_exports` (expiry)
- `activity_events`

## Code Style

```ts
// src/server/automation/maintenance.ts
const STEPS = [expireRequests, cleanOrphans, purgeRemoved, applyRetention, recheckStuckScans, expireExports] as const;

export async function runMaintenance(now = clock.now()) {
  const results = [];
  for (const step of STEPS) {
    try { results.push({ step: step.name, ...(await step(now)) }); }
    catch (err) { captureError(err, { step: step.name }); results.push({ step: step.name, failed: true }); }
  }
  if (results.some((r) => r.failed)) throw new AppError("INTERNAL", "maintenance had failing steps");
  return results;
}
```

## Testing Strategy

- **Unit:** digest decision logic (quiet and age windows) with a frozen clock; reminder count copy; the final-reminder flag; step selection predicates.
- **Integration** (Postgres, fake Redis, fake QStash recording publishes, outbox, frozen clock):
  - **Digest:** 5 uploads within 10 minutes → exactly one digest email after the quiet window. Continuous uploads → a digest at 30 minutes. Submit during the window → no digest, and the submitted email is sent. A replayed job → no second email.
  - **Reminders:**
    - a schedule due → one email, and `next_send_at` lands at 09:00–10:59 in the workspace zone
    - two concurrent runs → one email (the claim)
    - a QStash retry after the email was sent → the idempotency key skips it
    - stops when submitted, resumes on changes_requested
    - stops after 5 (`is_active` false)
    - bounced → skipped
    - expired by time → skipped
    - the link in the email resolves to the request
  - **Maintenance:** each step on fixtures, rerun safety (a second run changes nothing), one failing step doesn't stop the others and the job throws.
- **E2E (staging, accelerated):** a test-only env var `REMINDER_DAY_SECONDS=60` makes "days" one minute on staging. Send a request with 2-day reminders → reminders arrive, then stop on submit.

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| Uploading 5 files within 10 minutes produces exactly one digest email; submitting produces one immediate email and no later digest | Integration above, plus a staging E2E with real QStash delays |
| Reminders arrive between 9 and 11 am in the requester's time zone, stop on submit, stop after 5, and are never duplicated when QStash retries | Integration above; `nextLocalMorning` unit tests (requests) |
| Reminder emails contain a working link | Integration: decrypt and resolve; E2E: click the link from the outbox |

## Edge Cases

- **The QStash schedule is missed** (outage) → the next hourly run picks up everything overdue (`next_send_at <= now`). Clients may get a reminder later in the day, never twice.
- **More than 200 due at once** → the loop processes batches until it reaches 250 seconds, and the next run continues.
- **The workspace changes its time zone** → the next claim computes the next send in the new zone.
- **DST "spring forward"** with 09:00–10:59 as the window → always exists (the gap is 02:00–03:00).
- **A request is revoked between selection and claim** → the claim still succeeds, but the email is skipped because sending re-checks the status inside the claim transaction (the claim query re-joins the status).
- **Retention deletes files while a background ZIP is building** → the ZIP job fails ("files were deleted"), and the export is marked `failed`.
- **The requester shortens retention** → the next run deletes every completed request now past the new date. The settings dialog warned with the count first (SPEC-settings).
- **A request was reopened and completed again** → `completed_at` is the new completion time, so the retention clock restarts.
- **S3 delete partially fails** → `files_deleted_at` is set only after `deleteRequestObjects` returns. Otherwise the request is retried the next night.

## Boundaries

- **Always:**
  - verify the QStash signature
  - claim before sending
  - use idempotency keys on every email
  - report check-ins to Sentry Cron
  - isolate steps
- **Ask first:**
  - changing schedules, windows or the reminder count
  - adding steps
  - the retention warning email (should-have, planned for week 1 after launch)
- **Never:**
  - send without a claim
  - delete objects for requests that are not completed or explicitly deleted
  - put file names in digest content or logs

## Success Criteria

- [ ] The three job routes are deployed, the QStash schedules are created by script, and the Sentry Cron monitors are green on staging for 3 days.
- [ ] The digest, reminder and maintenance integration tests cover every rule above.
- [ ] The MVP.md 14 digest and reminder criteria pass on staging.
- [ ] Retention deletion removes every version, and is visible in the activity log.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 40 and 45)

1. **MVP.md 1.1, 1.2 and 8.3 step 4:** automatic retention deletion is a must-have; only the 7-day warning email stays a should-have. The landing page, the client success screen and the completed view all promise automatic deletion, and it's one of the controls the security page maps to the FTC Safeguards Rule.
2. **MVP.md 8.3 step 6:** exports stuck in `queued` or `building` for over an hour are marked `failed`.

## Decisions (Oct 20, 2026)

1. **Retention warning email:** a should-have, built in week 1 after launch. Until then the completed view shows the deletion date.
