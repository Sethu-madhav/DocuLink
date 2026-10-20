# Portal: the client's side

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-portal.md](../../specs/SPEC-portal.md) · Product spec: MVP.md 4.2, 4.3, 4.5, 5.6, 9.2

## In one paragraph
The client taps a link in an email and sees their accountant's name, logo and a short checklist. They photograph documents or pick files, watch them upload, and tap Submit. No account, no app, no password. It works on a phone with a weak signal and picks up where they left off on any device. If the accountant needs something fixed, the client sees exactly which item and why, and changes only that.

## Who it's for
Clients (recipients): people with no reason to trust a new website, often on a phone, handing over identity and financial documents.

## How it works
1. **Open the link.** The page shows the firm's branding, the request, the due date and the accountant's message. A quiet bot check runs in the background, with no puzzles.
2. **Add files.** Tap "Add file" on an item, then "Take photo" or "Choose file".
   - iPhone photos are converted to standard JPEGs and resized.
   - Each page photo gets a "Use photo / Retake" check, with a tip about sharp text and visible corners.
3. **Upload.** Each file shows a progress bar.
   - A dropped connection retries by itself three times, then shows Retry.
   - Up to 3 files upload at once.
4. **Submit.** The button stays disabled, and says why ("1 required item left: W-2 form"), until the required items are in. A confirmation lists what will be sent.
5. **Done.** A "Received" seal, a thank-you, and a receipt email.
   - On a computer, a "Paper copy? Use your phone" box shows a QR code. Scanning it opens the same link on the phone, so paper documents can be photographed there.
6. **Fixes.** If changes are requested, the items to fix come first, with the accountant's note. Everything else is locked. Resubmit when done.
7. **Other states.** Locked after submitting, completed, expired, revoked and invalid links each have one clear screen that names the accountant and how to contact them.

## Key decisions
| Decision | Why |
|---|---|
| The firm's brand first, DocuLink only in the footer | Clients trust their accountant, not us |
| One bot check per visit, not per upload | Fast on mobile data |
| The session is tied to the current link | Getting a fresh link instantly ends old sessions |
| Uploads go straight to storage with a fingerprint check | Fast, and nothing corrupted gets through |
| No marketing, sign-up prompts or chat widgets | Calm, and nothing that looks like a trick |

## Not included
- Client accounts or a list of all their requests (later).
- Combining photos into one PDF (should-have).

## Decided
- No extra buttons on the success screen; the receipt arrives by email.
- The QR code for continuing on a phone is part of launch.
- The "Why did I get this?" page is written for launch.
- Clients see their own on-device preview until the malware scan finishes.
