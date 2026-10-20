# Files: storing documents safely

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-files.md](../../specs/SPEC-files.md) · Product spec: MVP.md 4.2, 4.4, 4.7, 6.1, 8.4, 10

## In one paragraph
Client documents go straight from the client's phone into our private, encrypted storage in Amazon's US data centers. Each file is fingerprinted, so we know it arrived exactly as sent. Each one is scanned for malware before anyone can open it. Files can only be opened through links that expire in 10 minutes, and only by the accountant who asked for them, after two-factor sign-in. When files are deleted, every stored copy goes.

## Who it's for
- **Clients**, whose uploads must be safe and complete.
- **Accountants**, who must never be handed a virus or someone else's file.

## How it works
1. **Before upload.** The client's browser computes a fingerprint (SHA-256) of the file. The server checks the type and size (PDF or photo, up to 25 MB, 10 per item, 500 MB per request) and hands back a one-time upload link valid for 15 minutes.
2. **Upload.** The file goes directly to storage. If the bytes don't match the fingerprint, storage rejects them.
3. **Confirm.** The server double-checks the size, type and fingerprint before the file counts as uploaded.
4. **Scan.** Amazon's malware scanner checks every new file within seconds.
   - Clean files can be previewed.
   - Infected files are moved to a locked quarantine area and are never served. If the client hasn't submitted yet, they're asked to upload again.
5. **Opening a file.** The accountant gets a link valid for 10 minutes, and only for clean files.
6. **Deleting.** Deleting removes every stored version of the file, not just the latest.

## Key decisions
| Decision | Why |
|---|---|
| Direct upload to storage | Fast on mobile; the app server never handles the bytes |
| Fingerprint checked by storage itself | Corrupted or tampered uploads are impossible to accept |
| Nothing served until scanned | Accountants open files on office computers |
| No client names or file names in storage paths | A leaked path reveals nothing |
| A 50 GB per-workspace cap | The fair-use limit from the pricing page, actually enforced |

## Not included
- Checking a file's real type from its contents (should-have).
- Combining photos into one PDF (should-have).

## Decided
- Firm logos can be PNG, JPEG or WEBP. SVG isn't accepted, because it can carry code.
- While a client is uploading, they see the copy on their own device. On a later visit, a preview appears only once the file has passed the malware scan.
