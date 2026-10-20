# Spec: UI design system

Module id: `ui`
Status: Approved
Implements: MVP.md 5.1, 5.2, 5.4, 5.8, 5.9 (and the visual rules every screen in 5.3 and 5.6 depends on)
Designs:
- **Source of truth:** `Designs/DocuLink Design System v2.html`. Open it in a browser; it needs JavaScript. Its sections 01 to 09 are referenced below as "DS 01" to "DS 09".
- **Style reference:** the ten screen PNGs in `Designs/`. Screen layouts belong to the feature modules (see Capability Map).
Reading doc: [docs/specs/ui.md](../docs/specs/ui.md)
Last updated: 2026-10-20

## Objective

Turn the DocuLink design system into code once, so that every screen is assembled from parts that already exist, look like the design system, and meet WCAG 2.2 AA.

Users:
- **Feature-module developers** (requests, portal, review, auth, settings, billing, site). They import tokens, components, shells and formatters, and never restyle them per screen.
- **Requesters and clients**, who see the result. The design system's four principles (DS 01) apply:
  - read like a document
  - one green action per view
  - show the ask (what is needed and why)
  - keep only what's needed

What `ui` delivers:
1. **Tokens** from MVP.md 5.4 / DS 03, 05 and 08: colors, type scales, spacing, radius, elevation and motion, as Tailwind theme values. No raw hex anywhere else in the codebase.
2. **Type** (DS 04): system font stacks (New York, SF Pro, SF Mono) with self-hosted fallbacks (Newsreader, Instrument Sans, JetBrains Mono). Display weight 500; tabular figures for counts, dates and sizes.
3. **Brand** (DS 02): the cat mark and wordmark component, with clear space and a 24 px minimum.
4. **Primitives** (shadcn/ui on Radix, restyled; DS 07):
   - button, input, textarea, select, switch, checkbox, segmented control
   - dialog, bottom sheet, dropdown menu, popover, tooltip, tabs
   - toast, skeleton, progress, avatar, kbd
5. **Patterns** from DS 07 that two or more modules use:
   - status badge
   - seal
   - request label
   - folio number
   - request item frame
   - progress count
   - data table
   - display heading
   - meta label
   - file chip
   - drop zone (visual only)
   - sticky action bar
   - empty state
   - confirm dialog
   - unsaved-changes bar
   - numbered side nav
   - banner
6. **Shells** (DS 07, "App header and portal footer"):
   - requester shell: `brand` header 68 px tall, active nav underlined in `accent-on-brand`
   - portal shell: the firm's header on `surface`, with `brand` used only in the footer
   - auth shell
7. **Formatters** for dates, times, relative times, file sizes and request numbers ("No. 0142").
8. **Status presentation maps** for `request_status`, item status, `scan_status` and `export_status`. This is the only place a status gets its label, icon and tone (MVP.md 3.1, 3.2, 5.9).
9. **A component gallery** at `/dev/ui`, which mirrors DS sections 02 to 08 using the real components. It returns 404 in production. It is also the target for accessibility and visual tests.

Out of scope for `ui`:
- Screens, data fetching and Server Actions: feature modules own these.
- Email templates: the `email` module owns them, reusing the token values but not the React components.
- The builder's phone preview frame: `requests` owns it.
- The document viewer toolbar: `review` owns it.

## Tech Stack

Relies on `foundation` for Next.js (App Router), TypeScript (strict) and Tailwind CSS v4. This module adds:

| Package | Use |
|---|---|
| shadcn/ui (CLI, components copied into the repo) | Primitives source |
| `@radix-ui/*` (pulled in by shadcn) | Accessible behavior |
| `class-variance-authority`, `clsx`, `tailwind-merge` | Variants and `cn()` |
| `lucide-react` | Icons: 24 grid, 2 px stroke, round caps (DS 06) |
| `sonner` | Toasts (shadcn default) |
| `next/font/google` | Self-hosted fallback fonts: Newsreader (500, 500 italic), Instrument Sans (400, 500, 600), JetBrains Mono (400), Latin subset |
| Dev: `vitest`, `@testing-library/react`, `@testing-library/user-event`, `vitest-axe` | Component tests |
| Dev: `@playwright/test`, `@axe-core/playwright` | Gallery accessibility, keyboard and screenshot tests |

Other choices:
- **Bottom sheet:** built on Radix Dialog (MVP.md 6), not a drawer library.
- **Seal animation:** CSS keyframes, no animation library.
- **Drag-to-reorder:** not part of `ui`; `requests` owns it.
- **Apple fonts:** referenced by name in the stacks only. Never embed or redistribute them (DS 04).

## Commands

All of these are planned. The `foundation` module creates the scripts, and `ui` adds the gallery and contrast tests to them.

```
Dev:       pnpm dev                                # gallery at http://localhost:3000/dev/ui
Build:     pnpm build
Test:      pnpm test                               # vitest run (unit + component + token contrast)
Test one:  pnpm vitest run src/components/ui/button.test.tsx
E2E:       pnpm test:e2e --project=chromium --grep @ui
Lint:      pnpm lint && pnpm typecheck             # eslint, tsc --noEmit
Add prim.: pnpm dlx shadcn@latest add <component>  # then restyle to tokens; never leave defaults
```

## Project Structure

```
src/app/globals.css                 → Tailwind import, @theme tokens, base layer (body, focus, reduced motion, tabular figures)
src/app/(dev)/dev/ui/page.tsx       → Component gallery, sections mirror DS 02–08; notFound() in production
src/lib/fonts.ts                    → next/font fallbacks exposed as CSS variables used inside the system stacks
src/lib/cn.ts                       → cn() helper
src/lib/format.ts                   → Date, time, relative time, size, request-number formatters
src/components/brand/               → Mark and Wordmark
src/components/ui/                  → Restyled shadcn primitives (one file each)
src/components/patterns/            → Shared patterns (status-badge, seal, request-label, ...)
src/components/shells/              → requester-shell, portal-shell, auth-shell
src/components/status/              → Status presentation maps (request, item, scan, export)
public/brand/mark.svg               → Cat mark, copied from Designs/brand/doculink-mark.svg
src/**/*.test.tsx                   → Colocated component tests (Vitest)
e2e/ui/                             → Playwright specs for the gallery, tagged @ui
```

## Interfaces

Feature modules import only from these paths. Anything not listed here is internal and may change.

### Tokens (Tailwind theme)

**Colors.** These names come from MVP.md 5.4:
- Brand and action: `brand`, `on-brand`, `accent-on-brand`, `action`, `action-pressed`, `on-action`, `done`, `highlight`
- Surfaces and lines: `paper`, `surface`, `mist`, `line`, `line-strong`, `border`
- Text: `ink`, `ink-2`, `ink-3`
- Status: `attention`, `attention-tint`, `danger`, `danger-tint`

They're used as `bg-brand`, `text-ink-2`, `border-border` and so on.

**Fonts:**

| Utility | Stack |
|---|---|
| `font-display` | `ui-serif, "New York", var(--font-newsreader), Georgia, serif` |
| `font-sans` | `-apple-system, BlinkMacSystemFont, "SF Pro Text", var(--font-instrument-sans), system-ui, sans-serif` |
| `font-mono` | `ui-monospace, "SF Mono", SFMono-Regular, Menlo, var(--font-jetbrains-mono), monospace` |

**Type scale** (size / line height / tracking, from DS 04):

| Group | Utility | Value |
|---|---|---|
| Display (`font-display`, weight 500) | `text-display-1` | 160 / 0.84 / −5% |
| | `text-display-2` | 128 / 0.86 / −4.5% |
| | `text-display-3` | 96 / 0.88 / −4% |
| | `text-headline-1` | 64 / 0.92 / −3.5% |
| | `text-headline-2` | 48 / 1.0 / −3% |
| | `text-headline-3` | 32 / 1.1 / −2% |
| | `text-quote` | 24 / 1.4 / −1%, italic |
| Interface (`font-sans`) | `text-title` | 28/34, 600 |
| | `text-title-2` | 22/28, 600 |
| | `text-lead` | 19/28 |
| | `text-headline` | 17/22, 600 |
| | `text-body` | 17/24 |
| | `text-subhead` | 15/20 |
| | `text-badge` | 13/18, 600 |
| | `text-footnote` | 13/18 |
| Metadata | `text-mono` | 12/16 |

Display sizes step down below 768 px (display-3 becomes headline-1, and so on), so that nothing overflows at 375 px.

**Other tokens:**
- **Spacing:** Tailwind's 4 pt scale. The design system steps are 4, 8, 12, 16, 24, 32, 48, 64 and 96.
- **Radius:** `rounded-label` (4), `rounded-input` (10), `rounded-button` (12), `rounded-card` (18), `rounded-full` for badges.
- **Elevation:**
  - `shadow-card`: `0 1px 3px rgb(20 40 28 / .2)`
  - `shadow-float`: `0 2px 6px rgb(20 40 28 / .08), 0 40px 70px -36px rgb(20 50 35 / .55)`
  - Flat surfaces get a hairline instead (`ring-1 ring-line`).
- **Motion:**
  - Durations: `duration-fast` (120 ms), `duration-base` (200 ms), `duration-slow` (400 ms).
  - Easing: `ease-standard` `cubic-bezier(.2,.7,.2,1)`, `ease-stamp` `cubic-bezier(.2,1.4,.4,1)`.
  - Every transition is wrapped in `motion-safe:`.
- **Layout:** `container-app` gives 80 px margins at 1440 and up, and a 12-column grid with 32 px gutters. Review's 260 / fluid / 380 split is a `review` module concern.

### Brand (`@/components/brand`)

```ts
<Mark size={40} />                              // cat head from public/brand/mark.svg; min 24 px; never recolored, outlined, rotated or stretched
<Wordmark tone="on-paper" | "on-brand" />       // mark + "DocuLink" in display Medium 24; clear space = half the mark's height
```

### Primitives (`@/components/ui/*`)

`Button` (DS 07, "Buttons"):

| Prop | Value | Look |
|---|---|---|
| `variant` | `primary` | `action` fill, `on-action` text, `action-pressed` on hover. One per view; the gallery and code review enforce this. |
| | `secondary` | 1 px `border` on `surface` |
| | `ink` | `ink` fill, for file actions such as choose and download |
| | `text` | `action-pressed` text, for low-weight actions and links |
| | `on-brand` | `on-brand` fill, `brand` text, for the header's primary action |
| | `danger` | |
| `size` | `s` | 36 px, inside rows, requester only |
| | `m` | 44 px, the default |
| | `l` | 52 px, page actions |

Other `Button` props:
- `loading` and `asChild`.
- `kbd`: an optional shortcut chip, for example "A".
- `disabledReason`: required whenever `disabled` is true. It renders next to the button (DS 07: "say why next to it"), and the disabled button gets the `mist` fill.

Fields (`Input`, `Textarea`, `Select`), from DS 07, "Fields":
- 1 px `border` on `surface`, radius 10.
- On focus: an `ink` border with a 3 px `highlight` halo.
- On error: a `danger` border plus an error message under the field (`role="alert"`, linked by `aria-describedby`). The message says what to enter, for example "Enter an email address, like name@example.com".
- Props: `label`, `hint`, `error`.

Other primitives follow the shadcn APIs, restyled to the tokens:
- `Switch`, `Checkbox`
- `SegmentedControl`: Radix ToggleGroup, single selection, `mist` track
- `Dialog`, `DropdownMenu`, `Popover`, `Tooltip`, `Tabs`
- `Skeleton`, `Progress`, `Avatar` (initials on `highlight`), `Kbd`

`Sheet` is the bottom sheet:
```ts
<Sheet open onOpenChange title="Photo ID" description="Front and back of your driver's license">
  …
</Sheet>
// Bottom-anchored under 640 px, centered dialog at 640 px and up; respects the safe-area inset; focus trapped; Escape and swipe-down close.
```

`toast(message, { tone?: "default" | "danger", action?: { label, onClick } })`: re-exported from sonner and styled. Copy rule (MVP.md 5.8, DS 09): a toast uses the button's verb.

**Focus:** fields use the ink border and halo described above. Every other interactive element gets a 2 px `action` ring with a 2 px offset via `focus-visible` (MVP.md 5.9).

### Patterns (`@/components/patterns/*`)

```ts
type Tone = "needed" | "neutral" | "review" | "expired" | "approved" | "added";
// needed: ink outline on surface · neutral: ink on mist · review: attention on attention-tint
// expired: danger on danger-tint · approved: done on highlight · added: on-action on done

<StatusBadge tone="review" icon={Eye} label="To review" />          // icon and label are required props
<Seal label="Received" | "Approved" | "Complete" sub="Oct 17 · SL" size="sm" | "lg" animate?: boolean />
    // round double ring in `done`; animate = drop from 1.8× at −16°, press to 0.93×, settle 1×, 400 ms ease-stamp, once
<RequestLabel number="0142" firm="Acme Accounting" title="2026 Tax Return"
              preparedFor="John Doe" due={date} sealSlot?={<Seal …/>} />   // the request's identity card (DS 07)
<FolioNumber n={2} required={true} active?: boolean />  // "02", display 48; `action` when required, `ink-3` when optional
<RequestItemFrame n={2} required title="W-2 form" ask="All pages, from every employer."
                  why="it shows your wages and the tax already withheld." open?: boolean
                  trailing={<StatusBadge …/>}>{children}</RequestItemFrame>
    // hairline above each item; `ink` rule above the open item; renders "Why:" in semibold
<ProgressCount done={1} total={2} label="required items added" />   // display count with tabular figures + bar (`action` fill on `line`)
<DataTable columns rows />                                          // mono 12 headers over an `ink` rule, `line` row dividers
<DisplayHeading text="Two to" italic="review." as="h1" size="display-3" />  // at most one italic word; `action` on paper, `accent-on-brand` on brand
<MetaLabel>No. 0142</MetaLabel>                                     // mono 12, ink-2, tabular
<FileChip name="w2_2025.pdf" size={188416} state="uploading" | "done" | "failed" | "scanning" | "blocked"
          progress?={0.42} thumbnailUrl? onRemove? onRetry? onCancel? />
<DropZone title="Drop your W-2 here" hint="PDF, JPG, PNG or HEIC, up to 25 MB" onFiles={(files) => …}
          accept="application/pdf,image/*" multiple disabled? />    // dashed `line-strong` border; drag/drop + file input; no upload logic
<StickyActionBar primary={<Button …/>} reason="1 required item left: W-2 form" />  // safe-area aware
<EmptyState title="Send your first request" body="Pick a template or start from scratch." actions={…} />
<ConfirmDialog title body confirmLabel tone?="danger" requireTypedText?="DELETE" onConfirm />
<UnsavedChangesBar count={2} onDiscard onSave saving? />
<NumberedNav items={[{ n: 1, label: "Workspace", href, hidden? }]} /> // settings side nav, folio numerals
<Banner tone="review" | "expired" | "neutral" title body? actions? />
```

### Status presentation (`@/components/status`)

```ts
requestStatusBadge(status: RequestStatus, opts?: { inviteBounced?: boolean }): BadgeSpec
itemStatusBadge(status: ItemStatus, view: "client" | "requester", opts?: { changeDrafted?: boolean }): BadgeSpec
scanStatusBadge(status: ScanStatus): BadgeSpec | null
exportStatusBadge(status: ExportStatus): BadgeSpec
type BadgeSpec = { label: string; icon: LucideIcon; tone: Tone };
```

Request status labels match the MVP.md 3.1 dashboard labels exactly:

| Status | Label | Tone |
|---|---|---|
| `draft` | Draft | needed |
| `pending` | Sent | needed |
| `in_progress` | In progress | neutral |
| `submitted` | Needs review | review |
| `changes_requested` | Changes requested | review |
| `completed` | Completed | approved |
| `expired` | Expired | expired |
| `cancelled` | Revoked | neutral |
| any status, with `inviteBounced` | Invite bounced | review |

Item status:

| Status | Client sees | Requester sees |
|---|---|---|
| `pending` | Required / Optional (tone `needed`) | Not provided |
| `uploaded` | Added (`added`) | To review (`review`) |
| `approved` | Approved (`approved`) | Approved (`approved`) |
| `rejected` | Needs a fix (`review`) | Change requested (`review`); "Change drafted" when not yet sent |

Scan status:
- `pending` → "Scanning..." (`neutral`)
- `clean` → no badge
- `infected` and `failed` → "Blocked" (`expired`)

The enum types are imported from `foundation`'s Drizzle schema.

### Shells (`@/components/shells/*`)

```ts
<RequesterShell nav={[{ label: "Requests", href: "/requests" }, …]} workspaceName="Acme Accounting"
                primaryAction?={<Button variant="on-brand">New request</Button>} user={{ initials: "SL" }}
                breadcrumb?={[…]} meta?="No. 0142 · 1 of 3 approved" banner?={<Banner …/>}>{children}</RequesterShell>
<PortalShell requester={{ name, logoUrl?, replyTo }} requestNumber="0142"
             footerLinks={[{ label: "How is my data protected?", href }]}>{children}</PortalShell>
<AuthShell>{children}</AuthShell>
```

How the shells behave:
- **RequesterShell:**
  - `brand` header, 68 px tall
  - nav text in `on-brand`; active item underlined in `accent-on-brand`
  - nav turns into a menu sheet below 768 px
  - the MVP nav is Requests, Templates, Settings; the designs' "Clients" item is post-MVP (MVP.md 1.3)
  - renders the trial or lapsed banner slot under the header (`billing` supplies the content)
- **PortalShell:**
  - the firm's logo and name lead on a `surface` header, with "Questions?" and the reply-to link
  - DocuLink signs only the `brand` footer ("Secure request by DocuLink" plus links; MVP.md 5.1.3, DS 02)
  - sets `lang="en"` and the 17 px base

### Formatters (`@/lib/format`)

```ts
formatDate(d, tz)            // "Oct 17, 2026"         sentences, headings, due dates
formatDateShort(d, tz)       // "Oct 17"
formatDateNumeric(d, tz)     // "10/17/2026"           mono metadata only
formatDateTime(d, tz)        // "Oct 16, 2026, 4:12 pm"
formatRelative(d, now)       // "12m ago", "2h ago", "Tomorrow", "Overdue"
formatBytes(n)               // "184 KB", "1.8 MB"
formatRequestNumber(n)       // "No. 0142"
```

All formatters are US English (`en-US`) and use the requester's time zone. Their output is rendered with tabular figures. Day-first dates ("17.10.2026") are never used, even though the design system's examples show them.

## Data

None. `ui` owns no tables and makes no migrations. It reads only the enum types from `foundation`'s schema.

## Code Style

```tsx
// src/components/patterns/status-badge.tsx
import type { LucideIcon } from "lucide-react";
import { cva } from "class-variance-authority";
import { cn } from "@/lib/cn";

const badge = cva(
  "inline-flex items-center gap-1 rounded-full px-3 py-1 text-badge",
  {
    variants: {
      tone: {
        needed: "bg-surface text-ink ring-1 ring-ink",
        neutral: "bg-mist text-ink",
        review: "bg-attention-tint text-attention",
        expired: "bg-danger-tint text-danger",
        approved: "bg-highlight text-done",
        added: "bg-done text-on-action",
      },
    },
  },
);

export type Tone = NonNullable<Parameters<typeof badge>[0]>["tone"];

export function StatusBadge({ tone, icon: Icon, label, className }: {
  tone: Tone; icon: LucideIcon; label: string; className?: string;
}) {
  return (
    <span className={cn(badge({ tone }), className)}>
      <Icon aria-hidden className="size-4" strokeWidth={2} />
      {label}
    </span>
  );
}
```

Rules:
- **Naming:** files are kebab-case (`status-badge.tsx`). Components are PascalCase named exports with no default exports. Props types are inline, or `<Name>Props` when exported.
- **Tokens only:** no hex, `rgb()` or arbitrary Tailwind colors (`bg-[#...]`) outside `globals.css`. A lint rule enforces this.
- **Presentational:** patterns take no fetchers or Server Actions and read no context except toast.
- **Text:**
  - sentence case everywhere, no all caps
  - display type never lighter than 500
  - mono only for metadata, never for sentences
  - at least 13 px sans and 12 px mono
- **Interactive elements:**
  - a visible focus style, as described in Primitives
  - at least 44 × 44 px in portal sizes
  - an accessible name
- **Icons:** `aria-hidden` next to a text label. An icon-only button needs an `aria-label`.
- **Motion:** `motion-safe:` only. With reduced motion, everything appears in its end state, including the seal. No loops.
- **Copy inside components** follows DS 09 and MVP.md 5.8. Plain, specific, calm: name the thing and say what happens next ("1 required item left: W-2 form", not "You're almost there!"). No emoji.

## Testing Strategy

| Level | Tool | Covers | Location |
|---|---|---|---|
| Unit | Vitest | Formatters; status maps (every enum value has a label, icon and tone, and labels match MVP.md 3.1); `cn` | `src/lib/*.test.ts`, `src/components/status/*.test.ts` |
| Token contrast | Vitest | Reads the `@theme` values and checks every MVP.md 5.9 pair: text ≥ 4.5:1; `border` on surface and paper ≥ 3:1; the `action` focus ring on paper and surface ≥ 3:1 | `src/app/tokens.test.ts` |
| Component | Vitest + Testing Library + vitest-axe | Each primitive and pattern renders, has no axe violations, and works by keyboard. Specific checks: Sheet traps focus and closes on Escape; SegmentedControl uses arrow keys; a disabled Button without `disabledReason` throws in development; field errors use `role="alert"`; FileChip failed state is announced; progress uses `aria-live="polite"` | colocated `*.test.tsx` |
| Gallery E2E | Playwright + axe | `/dev/ui` has zero axe violations at 375 px and 1280 px; works keyboard-only (Tab through everything, focus always visible); reduced motion leaves the seal static; 200% zoom has no horizontal scroll at 375 px; screenshots of every gallery section for side-by-side review against the design system file | `e2e/ui/*.spec.ts` (`@ui`) |
| Browser matrix | Playwright projects | Chromium, WebKit (Safari: checks the New York and SF stacks), Firefox (checks the fallback fonts) | same |

Coverage: 90% of lines in `src/lib/format.ts` and `src/components/status/`. Every pattern needs at least one render test and one axe test.

MVP.md 14 acceptance criteria this module owns or enables:

| Criterion | Test |
|---|---|
| "Every status and badge is understandable without color." | Status map unit test (every value has an icon and label); a gallery E2E check with a grayscale filter, asserting each badge has visible text |
| MVP.md 5.9 contrast pairs (supports the accessibility pass in 13 M5) | Token contrast test |
| Keyboard-only run of both apps (13 M5) | Gallery keyboard E2E covers the parts; feature modules cover their screens |
| VoiceOver and TalkBack on the portal (13 M5) | Manual: gallery checklist in the implementation reading doc |

## Edge Cases

- **Long names** (client, business, item label of 80 characters) → truncate with an ellipsis in table cells and chips, wrap in headings; the full text goes in `title` and is available to screen readers.
- **Long titles in display type** ("Year-end bookkeeping and payroll reconciliation") → `DisplayHeading` steps down one size above 24 characters and never overflows at 375 px.
- **Fonts on different systems:**
  - Safari on Apple devices renders New York, SF Pro and SF Mono.
  - Chrome on a Mac renders SF Pro but falls back to Newsreader for display.
  - Windows and Android get all three fallbacks.
  - The fallbacks load with `display: swap` and metric-adjusted fallbacks, so that layout doesn't jump.
- **No logo** → the portal header and avatar fall back to initials on `highlight`. A broken logo URL falls back the same way.
- **Request number above 9999** → "No. 10342" (no fixed width).
- **Reduced motion** → the seal, skeleton shimmer and toast slides all appear in their end state.
- **200% zoom or a 320 px wide screen** → no horizontal scroll; the sticky action bar wraps its reason above the button.
- **iOS safe area and on-screen keyboard** → the sticky bar and sheet use `env(safe-area-inset-bottom)`, and the sheet stays visible when the keyboard opens.
- **Dark mode** → not supported in the MVP. The pages declare `color-scheme: light` so that form controls don't flip.
- **RTL** → out of scope (US launch); spacing uses logical properties where it's free.
- **More than one primary button on a view** → allowed only in the gallery; a dev-only console warning fires in app routes when two `primary` buttons mount in the same shell.

## Boundaries

- **Always:**
  - follow the design system file and MVP.md 5.1 to 5.9
  - use tokens only
  - keep components presentational
  - run `pnpm test` and the `@ui` E2E before commits
  - add every new component and state to the gallery
  - keep status labels in the status maps only
- **Ask first:**
  - adding or changing a token, font or motion value (this is an MVP.md 5.4 change)
  - adding a dependency beyond the table above
  - adopting Storybook or a visual-regression service
  - changing a primitive's public props once a feature module uses them
  - supporting dark mode
- **Never:**
  - hardcode colors or font sizes in feature modules
  - convey status by color alone
  - ship an interactive element without visible focus
  - embed or redistribute Apple's font files
  - load fonts from a third-party CDN at runtime
  - recolor or distort the cat mark
  - put analytics or Sentry calls inside `ui` components
  - expose `/dev/ui` in production

## Success Criteria

- [ ] Every MVP.md 5.4 token exists as a Tailwind theme value with the exact hex. A lint rule fails on raw hex outside `globals.css`.
- [ ] The token contrast test passes for every MVP.md 5.9 pair.
- [ ] Every primitive and pattern listed in Interfaces exists, renders in `/dev/ui` in every listed state, and has a component test with no axe violations.
- [ ] Status maps cover every value of `request_status`, item status, `scan_status` and `export_status`, each with a label, an icon and a tone. Request labels match MVP.md 3.1.
- [ ] The `/dev/ui` Playwright run has zero axe violations at 375 px and 1280 px, works keyboard-only, and has no horizontal scroll at 200% zoom.
- [ ] The seal runs the DS 08 animation in 400 ms, once, and renders statically under `prefers-reduced-motion`.
- [ ] The human signs off on the gallery screenshots side by side with the design system file, and on the three shells against their PNGs:
  - requester shell: `2 Requester dashboard - 01`
  - portal shell: `1 Client portal - 01`
  - auth shell: `4 Landing and sign-in - 02`
- [ ] `/dev/ui` returns 404 in a production build.
- [ ] Fallback fonts are self-hosted via `next/font`, there are no requests to fonts.googleapis.com at runtime, and CLS on the gallery is under 0.05.

## Decisions (resolved open questions, 2026-10-20)

1. **Dates use US formats.** Customers are American for now: "Oct 17, 2026" in text, "10/17/2026" in mono metadata. This deliberately differs from the design system's day-first examples (MVP.md 5.4).
2. **No "Clients" nav item in the MVP.** The nav is Requests, Templates, Settings. The Clients screen is post-MVP (MVP.md 1.3).
3. **Focus.** Fields use the design system's ink border with a mint halo. Every other control uses a 2 px `action` ring with a 2 px offset (MVP.md 5.9).
4. **Cat mark.** A vector redraw of the design system's PNG is at `Designs/brand/doculink-mark.svg`. `ui` copies it to `public/brand/mark.svg`.

## Open Questions

None for this module.

Differences between the designs and MVP.md that belonged to other modules, all decided on Oct 20, 2026 (MVP.md 0 rows 37 to 45):
- **portal:** "Download receipt" and "Add something else" are not built (MVP.md 5.6 wins). The QR phone handoff and the "Why did I get this?" link are must-haves.
- **requests:** the dashboard follows the design (hero count, review cards, count list as tabs). Templates are grouped by category, each with a Lucide icon (SPEC-requests).
- **settings:** "Delete files after" and "Default reminders" stay on the Workspace tab, as designed. Retention is 30 days, 90 days or a custom 1 to 365 days, which neither design shows; the Custom option adds a number field.
