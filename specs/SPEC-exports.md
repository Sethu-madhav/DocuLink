# Spec: ZIP exports

Module id: `exports`
Status: Approved
Implements: MVP.md 4.6, 5.5 (naming), 7 (`zip_exports`), 8.5, 9.1 (ZIP routes), 12 (`zip_downloaded`)
Designs: `3 Document review - 02 Complete.png` ("Download all · ZIP"); states from MVP.md 5.3
Reading doc: [docs/specs/exports.md](../docs/specs/exports.md)
Last updated: 2026-10-20

## Objective

Give the requester every clean file of a request as one tidy ZIP, named so that it drops straight into their own filing system:
- requests up to 200 MB stream directly
- larger ones (up to the 500 MB cap) are built in the background and announced by email

Success means:
- the ZIP has the MVP.md 5.5 names, in checklist order
- infected and unscanned files are never included
- both paths stay under 100 MB of memory

## Tech Stack

Foundation (`defineRoute`, `createJobRoute`, `publishJob`, `s3()`), plus:
- `archiver` in store mode (zlib level 0)
- `@aws-sdk/lib-storage` `Upload` (multipart, `partSize` 8 MB, `queueSize` 2)
- `Readable.toWeb` for the direct stream
- `auth` (`requireTwoFactor`), `requests` (status, names), `files` (keys, scan rules), `email` (`zip_ready`)

## Commands

```
Test:         pnpm vitest run src/server/exports
Integration:  pnpm vitest run --project integration src/server/exports
Load:         pnpm test:load -- zip           # staging: 100 files / 500 MB background; 150 MB direct; reports peak RSS
```

## Project Structure

```
src/server/exports/index.ts
src/server/exports/naming.ts         → zipFolderName, zipEntryName, sanitizeSegment (pure)
src/server/exports/select.ts         → selectFiles(request): included files, totals, blockers
src/server/exports/stream.ts         → buildZip(files, sink) shared by both paths
src/server/exports/service.ts        → queueExport, runExport, getExport
src/app/api/requests/[id]/zip/route.ts           → GET direct stream (maxDuration 300)
src/app/api/requests/[id]/zip-exports/route.ts   → POST queue
src/app/api/zip-exports/[id]/route.ts            → GET status + URL
src/app/api/jobs/zip/route.ts                    → QStash job (maxDuration 300)
src/components/exports/ZipButton.tsx              → Download ZIP / Prepare ZIP / Preparing / Download (24 h)
```

## Interfaces

### Naming (pure, MVP.md 5.5)

```ts
zipFolderName(clientName, title)                 // "John_Doe - 2026 Tax Return"  (client name with spaces → underscores, " - ", title kept with spaces)
zipEntryName({ position, label, index?, mime })  // "02_Photo_ID_1.jpg"; index only when the item has > 1 included file (upload order)
sanitizeSegment(s, { spaces: "_" | " " })        // NFKD + strip diacritics → ASCII; strip \/:*?"<>| and control chars; collapse repeats; trim dots/spaces; max 80 chars; empty → "Untitled"
EXT_BY_MIME = { "application/pdf": "pdf", "image/jpeg": "jpg", "image/png": "png", "image/heic": "heic", "image/heif": "heif", "image/webp": "webp" }
```

- The extension always comes from the verified MIME type, never from the original name.
- Two items with the same label still get distinct names, because the position prefix differs.
- The entry path inside the ZIP is `{folder}/{entry}`. The archive's file name is `{folder}.zip`, sent with RFC 5987 `filename*`.

### File selection

`selectFiles(request)` returns `{ files, totalBytes, blockers }`:
- **Included:** files with status `active` and `scan_status = clean`, in item position order, then by `confirmed_at`.
- **Blockers:**
  - `scanning`: any active file still pending its scan; the button shows "Finishing security scan..."
  - `deleted`: `files_deleted_at` is set
- **Excluded silently:** infected and failed files. They're never included, and the review page already shows them as Blocked.
- **Availability:** `submitted_at` is not null and the status is not draft (MVP.md 4.6 "from submitted onwards").
- **Note:** "Includes unreviewed files" when any included item isn't approved.

### Direct stream: `GET /api/requests/{id}/zip`

| Step | Rule |
|---|---|
| Guard | `requireTwoFactor`; the request belongs to the workspace, else 404 |
| Select | Any blocker → 409 with its message. `totalBytes > 200 MB` → 413 "This ZIP is large. Prepare it instead." (the UI already shows Prepare ZIP) |
| Stream | `archiver("zip", { store: true })`. For each file, `GetObject` and append its body stream. Bodies are fetched sequentially (one open S3 stream at a time). Backpressure from the response, through `Readable.toWeb(archive)` |
| Headers | `Content-Type: application/zip`, `Content-Disposition: attachment; filename*=UTF-8''…`, `Cache-Control: no-store`. No `Content-Length` (streamed) |
| Log | `zip_downloaded` (requester, meta `{ files, bytes, mode: "direct" }`); PostHog `zip_downloaded` |

### Background export

**`POST /api/requests/{id}/zip-exports`:**
- Guard: `requireTwoFactor`.
- Select, then dedupe: if an export exists for this request with the same `fileSetHash` (sha256 of the sorted included file ids) and status `queued`, `building`, or `ready` and not expired, return it.
- Otherwise insert `zip_exports` (`queued`, `requested_by`) and `publishJob("/api/jobs/zip", { exportId }, { deduplicationId: exportId })`.
- Returns `{ exportId, status }`.

**`GET /api/zip-exports/{id}`:**
- Guard: `requireTwoFactor`, scoped through the request's workspace.
- Returns `{ status, fileCount, sizeBytes, error?, url? }`. When `ready`, `url` is a presigned GET for 10 minutes with `attachment` disposition, and the request logs `zip_downloaded` (mode `background`).

**Job `POST /api/jobs/zip`** (MVP.md 8.5):

```
claim: UPDATE zip_exports SET status = 'building' WHERE id = $1 AND status = 'queued' RETURNING *   // none → return (duplicate delivery)
files = selectFiles(request) (re-select at build time; if fileSetHash changed, continue with the new set)
upload = new Upload({ client: s3(), params: { Bucket, Key: exports/{requestId}/{exportId}.zip, ContentType: application/zip, SSE-KMS } })
buildZip(files, archive) piped into upload.Body; await upload.done()
on success: status ready, s3_key, size_bytes, file_count, ready_at, expires_at = now + 24 h; email zip_ready (idempotency "zip:{exportId}"); log zip_prepared
on failure: upload.abort(); status failed, error = safe message ("A file was missing from storage" / "Files were deleted" / "Timed out"); Sentry
```

**UI states (`ZipButton`):**

| Condition | Button |
|---|---|
| Total ≤ 200 MB | **Download ZIP** |
| Above 200 MB | **Prepare ZIP** → "Preparing your ZIP..." (polls every 5 s) → **Download ZIP** with "Link valid for 24 hours" |
| Failed | "Couldn't prepare the ZIP. {error}" plus per-item downloads (MVP.md 8.5) |
| Scanning | Disabled, with "Finishing security scan..." next to it (DS 07: say why) |

## Data

Owns `zip_exports`. Column (in MVP.md 7 since v1.2): `zip_exports.file_set_hash text not null`, used for dedupe and staleness.

## Code Style

```ts
// src/server/exports/naming.ts
export function zipEntryName({ position, label, index, mime }: EntryInput): string {
  const pos = String(position).padStart(2, "0");
  const base = sanitizeSegment(label, { spaces: "_" });
  const suffix = index === undefined ? "" : `_${index}`;
  return truncateTo80(`${pos}_${base}${suffix}`) + "." + EXT_BY_MIME[mime];
}
```

## Testing Strategy

- **Unit:**
  - naming: the MVP.md 5.5 example exactly; diacritics ("José Núñez" → "Jose_Nunez"); forbidden characters; an 80-character truncation that keeps the extension; an empty label; duplicate labels; a single file versus several in one item
  - selection: order, exclusions, blockers, "unreviewed" note
- **Integration** (Postgres plus an S3 mock serving generated streams):
  - direct route headers and entry order (unzip in the test and compare names)
  - 413 above 200 MB
  - 409 while scanning
  - 404 for another workspace
  - `TWO_FACTOR_REQUIRED`
  - export dedupe, a claim race (two deliveries → one build)
  - a failure path that aborts the multipart upload and sets `failed`
  - the `zip_ready` email sent once
- **Load (staging):** 100 files totaling 500 MB through the background path; a 150 MB direct download. Assert peak RSS under 100 MB (memory sampled in the function, logged to Sentry as a measurement) and completion within `maxDuration`.

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| A ZIP of 100 files totaling 500 MB is prepared in the background and downloads with correct names and order; a 150 MB ZIP streams directly; both jobs stay under 100 MB memory | Load test above, plus an integration check of names |
| EICAR file … excluded from the ZIP (shared with `files`) | Integration: an infected file is never selected; staging E2E with `files` |

## Edge Cases

- **A file is deleted from S3 but still `active` in the database** → the direct stream can't recover mid-stream. Before streaming, a HEAD on each file finds this and returns 409 with MVP.md 5.8 copy: "Couldn't generate the ZIP. The file '{name}' is missing from storage. Remove it from the request or ask the client to upload it again." The background job fails with the same message.
- **The client resubmits while a ready export exists** → the file set hash changes, the button shows **Prepare ZIP** again, and the old export expires naturally.
- **The connection drops during a direct download** → the archive is aborted, the S3 stream destroyed and nothing logged as downloaded. The requester retries.
- **Exactly 200 MB** → direct (≤ 200 MB).
- **ZIP64 needed?** No: under 4 GB and under 65,535 entries. Store mode at 500 MB is plain ZIP.

## Boundaries

- **Always:**
  - two-factor for every ZIP route
  - only clean, active files
  - store mode
  - streaming end to end (no buffering whole files)
  - names from the verified MIME type
- **Ask first:**
  - raising the 200 MB direct threshold
  - adding a CSV manifest (should-have)
  - moving the job to Lambda
- **Never:**
  - compress (it wastes CPU on PDFs and JPEGs)
  - write to `/tmp`
  - include infected or unscanned files
  - put the original file names inside the ZIP

## Success Criteria

- [ ] Direct and background paths work on staging, with the UI states above.
- [ ] The naming matches MVP.md 5.5 and passes the unit table.
- [ ] Load test results are recorded in the implementation doc: memory under 100 MB, within time limits.
- [ ] Exports expire after 24 hours, via the lifecycle rule and maintenance step 6.

## MVP.md changes (applied in v1.2, MVP.md 0 row 44)

1. **MVP.md 7:** `zip_exports.file_set_hash` (dedupe, and detecting stale exports after a resubmit or a Reopen).

## Open Questions

None.
