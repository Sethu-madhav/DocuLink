# Spec: File storage, integrity and malware scanning

Module id: `files`
Status: Approved
Implements: MVP.md 4.2 (steps 4–8, storage side), 4.4 (scan states), 4.7 (delete files), 6.1, 8.3 (steps 2, 3, 5 helpers), 8.4 (GuardDuty), 9.1 (file URL route), 10 (storage, validation, malware), 11 (fair-use storage cap)
Designs: none of its own; states are shown by `portal` and `review` through `ui` patterns
Reading doc: [docs/specs/files.md](../docs/specs/files.md)
Last updated: 2026-10-20

## Objective

Store client documents so that:
- only the right person can reach them, through short-lived signed URLs only
- they arrive intact, with SHA-256 verified by S3
- nothing is served until GuardDuty says it's clean
- when deleted, they're actually gone, including all versions

`files` provides storage and file-row primitives. It does not decide when a client may upload or a requester may review: that's request state, owned by `requests` and used by `portal` and `review`.

## Tech Stack

- Foundation's `s3()` (OIDC), `@aws-sdk/s3-request-presigner`, `withTx`, `logEvent`, `defineRoute`.
- `auth` guards for the requester file URL route.

## Commands

```
Test:         pnpm vitest run src/server/files
Integration:  pnpm vitest run --project integration src/server/files
E2E:          pnpm test:e2e e2e/files          # staging: real S3, GuardDuty with the EICAR file
```

## Project Structure

```
src/server/files/index.ts         → Public API
src/server/files/rules.ts         → Allow-list, limits, key builders, filename sanitizing, disposition (pure)
src/server/files/uploads.ts       → createPendingFile, presignPut, confirmUpload, removeFile
src/server/files/urls.ts          → presignGet (inline/attachment)
src/server/files/scan.ts          → handleGuardDutyEvent, recheckScan
src/server/files/delete.ts        → deleteObjectAllVersions, deleteRequestObjects, deleteWorkspaceObjects
src/server/files/logos.ts         → presignLogoPut, confirmLogo, logoUrl
src/app/api/files/[fileId]/url/route.ts → Requester preview/download URL
```

## Interfaces

### Rules (pure)

```ts
ALLOWED_TYPES = ["application/pdf", "image/jpeg", "image/png", "image/heic", "image/heif", "image/webp"];
LIMITS = { fileBytes: 25 * MB, filesPerItem: 10, filesPerRequest: 100, requestBytes: 500 * MB, workspaceBytes: 50 * GB };
requestFileKey(requestId, itemId, fileId) → "requests/{requestId}/{itemId}/{fileId}"
quarantineKey(requestId, fileId)          → "quarantine/{requestId}/{fileId}"
logoKey(workspaceId, logoId)              → "logos/{workspaceId}/{logoId}"
sanitizeDisplayName(name) → string        // NFC, strip path and control chars, collapse spaces, max 200 chars, keep extension
checkUploadAllowed({ type, size, itemActiveOrPendingCount, requestCount, requestBytes, workspaceBytes }) → void | throws AppError
contentDisposition(mode, mime, displayName) → "inline" only for PDF/images in inline mode, otherwise attachment; RFC 5987 filename*
```

Limit errors, all `VALIDATION`, with MVP.md 5.6 copy:
- Too large: "This file is {x} MB. The limit is 25 MB."
- Wrong type: "This file type isn't accepted. Use PDF, JPG, PNG or HEIC."
- Too many on the item: "You can add up to 10 files here."
- Request or workspace full: "This request is full. Contact {business}."

### Uploads

```ts
createPendingFile(tx, { requestId, itemId, name, type, size, sha256 }): Promise<{ fileId, key }>
presignPut({ key, type, size, sha256 }): Promise<{ url: string; headers: Record<string, string>; expiresAt: Date }>
confirmUpload(tx, { requestId, fileId }): Promise<UploadedFile>
removeFile(tx, { requestId, fileId, actor }): Promise<UploadedFile>
```

**`createPendingFile`**:
- Re-checks the limits inside the transaction, counting `pending` and `active` files. It locks the item row (`SELECT … FOR UPDATE`), so parallel presigns can't exceed 10 per item.
- Inserts a `pending` row with `original_name = sanitizeDisplayName(name)` and `sha256` (base64 of the 32-byte digest, validated as exactly 44 characters).

**`presignPut`**:
- A `PutObjectCommand` with these signed: `ContentType`, `ContentLength`, `ChecksumAlgorithm: SHA256`, `ChecksumSHA256`, `ServerSideEncryption: aws:kms` and the key id.
- Expires in 15 minutes. Returns the headers the browser must send exactly as signed.

**`confirmUpload`**:
1. `HeadObject` with `ChecksumMode: ENABLED`.
2. Verifies that `ContentLength`, `ContentType` and `ChecksumSHA256` equal the row's values. On a mismatch: delete the object, mark the file `removed`, and return `CONFLICT` "Upload didn't finish. Try again."
3. On success: status `active`, `confirmed_at = now`, `scan_status = pending`.
4. Idempotent: already `active` → returns the row.
5. A missing object → `CONFLICT` (the client retries the upload).

**`removeFile`**: status `removed`, `removed_at`. Deletes the S3 object and all its versions after commit; failures are left to maintenance step 3. Logs `file_removed` (as the actor).

### Requester file URL route

`GET /api/files/{fileId}/url?mode=inline|download`:

| Step | Rule |
|---|---|
| Guard | `requireTwoFactor` |
| Lookup | The file joined to its request, filtered by `workspace_id` → otherwise 404 |
| State | The file is `active`, the request's `files_deleted_at` is null, `scan_status = clean`. Otherwise `CONFLICT`: "This file is still being scanned." / "This file was blocked by the security scan." |
| URL | `presignGet` for 10 minutes, with `ResponseContentDisposition` from `contentDisposition` and `ResponseContentType` = the stored MIME type |
| Log | `file_previewed` with `meta.mode`; PostHog none (`review_opened` is page level) |
| Response | `{ url, expiresAt }`, `Cache-Control: no-store` |

### Scan results

```ts
handleGuardDutyEvent(event, hooks: {
  onInfected: (args: { requestId: string; itemId: string; fileId: string }) => Promise<void>;
}): Promise<void>
recheckScan(fileId): Promise<"clean" | "infected" | "failed" | "pending">   // reads the GuardDutyMalwareScanStatus object tag
```

Mapping of the GuardDuty `scanResultDetails.scanResultStatus`:

| Result | Effect |
|---|---|
| `NO_THREATS_FOUND` | `scan_status = clean`, `scanned_at`, log `file_scan_clean` |
| `THREATS_FOUND` | `scan_status = infected`, `scanned_at`; copy the object to `quarantineKey`, delete all versions of the original; log `file_scan_infected` (meta: threat names, no content); Sentry warning; call `onInfected` |
| `UNSUPPORTED`, `ACCESS_DENIED`, `FAILED` | `scan_status = failed`, `scanned_at`; Sentry alert; treated as blocked (never served). Not quarantined: the object stays for a manual check and is deleted with the request |

How it's applied:
- Looked up by `s3_key = s3ObjectDetails.objectKey`. Events for keys outside `requests/` are ignored.
- The handler is idempotent: updates only apply `WHERE scan_status = 'pending'`.
- A file that is still `pending` (unconfirmed) when the scan arrives is still updated. Confirming later keeps the scan result: `confirmUpload` doesn't reset `scan_status` if it's already set.
- `onInfected` is implemented by `requests` (`handleInfectedFile`): if the request is not yet submitted, the file becomes `removed` and the item goes back to `pending` when it has no other active files. Otherwise nothing changes, and the requester sees "Blocked". It's wired in the AWS events route.

### Deletion

```ts
deleteObjectAllVersions(key): Promise<void>            // ListObjectVersions(prefix=key) → DeleteObjects (versions + delete markers)
deleteRequestObjects(requestId): Promise<{ deleted }>  // all versions under requests/{id}/ and exports/{id}/ ; quarantine left to lifecycle
deleteWorkspaceObjects(workspaceId): Promise<void>     // each request of the workspace, plus logos/{workspaceId}/
```

All three are batched 1,000 keys per call and resumable: calling them again after a partial failure finishes the job.

### Logos

- `presignLogoPut({ workspaceId, type, size })` accepts PNG, JPEG and WEBP up to 1 MB. SVG is not accepted (MVP.md 10); the settings field says "PNG, JPG or WEBP". It uses the `logos/` prefix, which isn't scanned.
- `confirmLogo` runs a HEAD check, then sets `workspaces.logo_key` and deletes the old logo.
- `logoUrl(workspace)` returns a presigned GET valid for 1 hour, used for the portal header and the builder preview. It's cached per request render.

## Data

- Owns `uploaded_files` and writes all of its columns.
- Reads `checklist_items` and `document_requests` for scoping and limits.
- Writes `workspaces.logo_key`.

No schema changes beyond foundation's proposals. Storage usage is computed with `SUM(size_bytes) WHERE status IN ('pending','active')` per request and per workspace, which is indexed enough at MVP scale. A cached counter is a later optimization.

## Code Style

```ts
// src/server/files/uploads.ts
export async function confirmUpload(tx: Tx, { requestId, fileId }: ConfirmInput) {
  const file = await getFileForUpdate(tx, requestId, fileId);
  if (file.status === "active") return file;
  if (file.status !== "pending") throw new AppError("CONFLICT", "This file was removed.");
  const head = await headWithChecksum(file.s3Key);
  if (!head) throw new AppError("CONFLICT", "Upload didn't finish. Try again.");
  if (head.size !== file.sizeBytes || head.type !== file.mimeType || head.sha256 !== file.sha256) {
    await markRemoved(tx, file.id, clock.now());
    tx.afterCommit(() => deleteObjectAllVersions(file.s3Key));
    throw new AppError("CONFLICT", "Upload didn't finish. Try again.");
  }
  return activate(tx, file.id, clock.now());
}
```

## Testing Strategy

- **Unit:**
  - limits (every boundary: 25 MB exactly, 10th and 11th file, 500 MB, 50 GB)
  - allow-list
  - key builders
  - `sanitizeDisplayName`: path traversal, control characters, RTL override characters, 300-character names, emoji
  - `contentDisposition` for every MIME type and mode
- **Integration** (Postgres plus `aws-sdk-client-mock`):
  - concurrent `createPendingFile` can't exceed 10 per item
  - confirm happy path, size, type and checksum mismatches, missing object, idempotent re-confirm
  - `removeFile` and the after-commit delete
  - GuardDuty handler for each result, idempotent replays, a scan before confirm, `onInfected` called only for `THREATS_FOUND`
  - the file URL route: 404 for another workspace, `TWO_FACTOR_REQUIRED`, `CONFLICT` for pending and infected, a 10-minute expiry
- **E2E (staging):** upload the EICAR test string as a `.pdf` through the portal, then:
  - it becomes `infected` and is quarantined
  - the URL route refuses it
  - it's excluded from the ZIP (with `exports`)
  - a clean PDF becomes previewable within 60 seconds
  - a PUT with a body that doesn't match the signed checksum gets a 400 from S3

MVP.md 14 acceptance criteria owned:

| Criterion | Test |
|---|---|
| EICAR file is marked infected, quarantined, never previewable or downloadable, and excluded from the ZIP; a clean file becomes previewable within a minute | Staging E2E above |
| An upload whose bytes don't match the signed checksum is rejected by S3 and shown as a failed upload | Staging E2E (S3 400) plus a portal UI test for the failed state |
| No S3 object is reachable without a signed URL; signed URLs expire as specified | Staging E2E: an unsigned GET returns 403; GET URLs expire at 10 minutes and PUT URLs at 15 (frozen clock in integration; a real expiry check on staging with a short override) |

## Edge Cases

- **The client uploads the same file twice** → two rows and two objects. Each is shown, and the client can remove one. No dedupe by hash in the MVP (a duplicate page is legitimate).
- **The browser sends different headers than signed** → S3 returns 403 `SignatureDoesNotMatch`. The portal shows the failed state and retries with a fresh presign.
- **A HEIC file that the browser failed to convert** → uploaded as `image/heic`, accepted. Review shows "Download to view" (no inline preview for HEIC).
- **GuardDuty scans an object deleted meanwhile** (the client removed it) → the result updates a `removed` row. Harmless; a clean result doesn't make it active.
- **Quarantine copy fails** → the original is still deleted (KMS-encrypted and never served). Sentry alert; `scan_status` is still `infected`.
- **The request's files were already deleted** (`files_deleted_at` set) → the URL route returns `CONFLICT` "Files for this request were deleted on {date}."
- **Versioned bucket delete** → plain `DeleteObject` only adds a delete marker. `deleteObjectAllVersions` removes every version and marker, which is what "actually gone" means in MVP.md 10.

## Boundaries

- **Always:**
  - every served URL is presigned and short-lived
  - check `scan_status = clean` before any requester GET
  - delete all versions
  - keep original names in the database only
- **Ask first:**
  - changing limits or the allow-list
  - scanning logos
  - a magic-byte check (should-have)
- **Never:**
  - public URLs
  - client names or original file names in S3 keys
  - serving `infected` or `failed` files
  - logging file names or contents to Sentry or PostHog
  - accepting SVG (or any scriptable format) for logos

## Success Criteria

- [ ] Presign, upload and confirm work end to end on staging, with the checksum enforced by S3.
- [ ] GuardDuty results flow through EventBridge to the handler. Infected files are quarantined and never served.
- [ ] The file URL route enforces workspace, two-factor and scan rules, with a 10-minute expiry.
- [ ] Deletion removes every version (verified with `ListObjectVersions` returning nothing).
- [ ] All limits, including the 50 GB workspace fair-use cap, are enforced at presign and covered by tests.

## MVP.md changes (applied in v1.2, MVP.md 0 row 45)

1. **MVP.md 10, Validation:** the 50 GB per-workspace cap (from MVP.md 11) is enforced at presign.
2. **MVP.md 8.4:** a scan result of `FAILED`, `UNSUPPORTED` or `ACCESS_DENIED` keeps the object (not quarantined) for a manual check. Only `THREATS_FOUND` is quarantined.
3. **MVP.md 6.1, 10:** the `quarantine/` key prefix; the IAM role's version, tagging and multipart permissions; logos are PNG, JPEG or WEBP only.

## Decisions (Oct 20, 2026)

1. **SVG logos:** not accepted. PNG, JPEG and WEBP only.
2. **Client thumbnails:** the browser's local preview during the session; on a return visit, server thumbnails only for clean files and a file icon for pending ones.
