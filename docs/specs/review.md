# Review: checking each document against the ask

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-review.md](../../specs/SPEC-review.md) · Product spec: MVP.md 4.4, 4.5, 5.3, 9.1

## In one paragraph
When a client submits, the accountant opens one page with three parts: the list of items on the left, the document in the middle, and what they asked for on the right. For each item they either approve it (an "Approved" seal lands on it) or write a short note asking for a fix. All fixes go to the client in one email. When everything is approved, the request is complete and the files download as one ZIP.

## Who it's for
Accountants. It's the page they use most.

## How it works
1. **Open the request.** Two-factor sign-in is required to see files.
2. **Look at a file.**
   - PDFs show page by page, with zoom and rotate. Photos show directly.
   - Files still being scanned say "Scanning...". Files that failed the scan say "Blocked" and can't be opened.
3. **Approve** with the button or the A key. The seal animates once.
4. **Request a change** with the R key, or by typing a note. Quick picks such as "Blurry or unreadable" or "Pages missing" help. The change is saved as a draft first.
5. **Send change requests.** One click sends a single email to the client listing every fix, and reminders restart.
6. **Complete.** When every provided item is approved, the request completes automatically: "Complete." with a seal, Download all as a ZIP, and the date the files will be deleted.
7. **Keyboard.** J and K move between items, A approves, R requests a change, ? shows all shortcuts.

## Key decisions
| Decision | Why |
|---|---|
| Fixes are drafted, then sent together | Rejecting three items sends one email, not three |
| Can't approve while a file is scanning or blocked | Never approve something nobody could open safely |
| PDFs drawn page by page with scripts turned off | Enables spread view and zoom from the design without running anything inside the PDF |
| Approvals can be undone until the request completes | Mistakes are easy to fix before it's final |

## Not included
- Approving before the client submits (revisit after beta).
- "Approve all" in one click.
- The ZIP itself, which is the exports module.

## Decided
- "Reopen" on a completed request lets the accountant write notes on the items that need another look and send them as change requests. The client gets one email and fixes only those items.
- Activity history sits in a collapsible "History" section under the item list.
