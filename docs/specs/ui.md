# UI design system: the shared look and parts of every screen

Status: Approved · Last updated: 2026-10-20
Full spec: [specs/SPEC-ui.md](../../specs/SPEC-ui.md) · Product spec: MVP.md 5.1, 5.2, 5.4, 5.8, 5.9 · Design source: `Designs/DocuLink Design System v2.html`

## In one paragraph
This module turns the DocuLink design system into code: the colors, fonts, buttons, badges, the "Received / Approved / Complete" seal and the page frames that every screen is built from. Building them once makes the product look the same everywhere: it reads like a document, with large serif titles, numbered items and hairline rules. It also means accessibility is handled in one place instead of on every screen. This module contains no actual screens. The dashboard, the client checklist and the review page are each built later by their own module, using these parts.

## Who it's for
- **Developers building screens.** They pick parts from this kit instead of styling things one by one.
- **Accountants (requesters)** get a consistent desktop app with a deep green header.
- **Their clients** get a phone-friendly page that leads with the accountant's own logo and name, with large text and touch targets, and statuses that never rely on color alone.

## How it works
1. **The design system file is the source.** Its colors, type sizes, spacing, corner radii and animation timings are copied into the code exactly once, under the same names. Nothing else in the code may use a raw color.
2. **Fonts.** Apple devices show Apple's own fonts (New York for titles, SF Pro for text, SF Mono for labels like "No. 0142"). Everyone else gets close free look-alikes that we host ourselves.
3. **Basic controls** (buttons, inputs, switches, menus, dialogs, a bottom sheet for phones) come from a well-tested open-source library and are restyled to match the design system.
   - Each view has one green button.
   - A disabled button always says why, next to it.
4. **Shared patterns** are built on top of those controls. Examples:
   - the status badge (always an icon plus words)
   - the round seal that lands with a small press when something is received or approved
   - the request label card
   - numbered checklist items that show "what" and "why"
   - file chips with upload progress
   - a sticky bottom bar
5. **Three page frames:**
   - the accountant's app, with the green header
   - the client's portal, with the firm's brand on top and a small "Secure request by DocuLink" footer
   - the sign-in page
6. **One place decides every status label**, such as "Needs review", "To review" or "Invite bounced", so labels can't drift between screens.
7. **A private gallery page** (`/dev/ui`, never live in production) mirrors the design system file using the real components. Automated tests check it for accessibility problems, keyboard use, zoom and phone widths.

## Key decisions
| Decision | Why |
|---|---|
| The design system file wins on visual details; MVP.md wins on behavior and copy | One clear tiebreaker; MVP.md 5.4 now copies the design system's values |
| Design system is its own module; screens live in feature modules | Screens change with their features; the kit stays stable and is tested once |
| shadcn/ui on Radix, restyled | Accessible keyboard and screen-reader behavior for free; we own the code |
| Apple fonts by name only, free look-alikes self-hosted | Apple's fonts can't be redistributed; self-hosting avoids a third-party request from the client portal |
| A gallery page instead of Storybook | One less tool to maintain for a solo build; the same page doubles as the test target |
| Contrast checked automatically from the color values | Accessibility can't silently regress when someone tweaks a color |
| Light mode only for the MVP | Saves time; the design system is light only |

## What changed when the design system arrived
- The colors moved to the design system's exact values and names (deep green "brand", fern "action", "seal", "mint" and so on). The old input-border problem is gone: the design system's control border passes accessibility rules.
- The fonts are now settled: New York, SF Pro and SF Mono, with Newsreader, Instrument Sans and JetBrains Mono as fallbacks.
- The seal animation is now 400 ms, with a drop-and-press motion.
- The request label card is now a shared part, because the design system defines it as one.

## Not included
- The actual screens (dashboard, builder, checklist, review, settings). They belong to their own modules.
- Email designs, which the email module owns.
- Dark mode and right-to-left languages.
- Drag-to-reorder, which the request builder owns.

## Decisions made on review
- **Dates** are written the American way ("Oct 17, 2026"), because customers are American for now. The design system's "17.10.2026" examples are not followed.
- **No "Clients" menu item** in the first version; it comes after launch.
- **Focus:** text fields get a dark border with a mint glow; buttons and links get a green ring.
- **Logo:** the cat now exists as a crisp vector file (`Designs/brand/doculink-mark.svg`).

## Open questions
- None.
