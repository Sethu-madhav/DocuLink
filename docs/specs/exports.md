# Exports: downloading everything as one ZIP

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-exports.md](../../specs/SPEC-exports.md) · Product spec: MVP.md 4.6, 5.5, 8.5

## In one paragraph
Once a client has submitted, the accountant can download every file as one ZIP, neatly named and in checklist order, ready to drop into their own filing system: `John_Doe - 2026 Tax Return/01_W-2_Form.pdf`. Smaller downloads (up to 200 MB) start straight away. Bigger ones are prepared in the background, and the accountant gets an email when they're ready.

## Who it's for
Accountants.

## How it works
1. **Download ZIP** appears from the moment the client submits. If some items aren't approved yet, a note says "Includes unreviewed files."
2. Only files that passed the malware scan are included. If any file is still being scanned, the button waits ("Finishing security scan...").
3. **Up to 200 MB:** the download starts immediately and is built on the fly.
4. **Over 200 MB:** the button reads **Prepare ZIP**. We build the ZIP in the background, email the accountant, and the download link works for 24 hours.
5. **Names:** the client name and request title for the folder; then the item number, item name, a page number when there are several files, and the real file type.

## Key decisions
| Decision | Why |
|---|---|
| Two paths, split at 200 MB | Fast for typical requests; reliable for big ones |
| No compression | PDFs and photos don't shrink; compressing only wastes time |
| File type taken from the real file, never the original name | Converted iPhone photos are real JPEGs that open on Windows |
| The same prepared ZIP is reused until the files change | No duplicate work if someone clicks twice |

## Not included
- A spreadsheet listing the files inside the ZIP (should-have).

## Open questions
- None.
