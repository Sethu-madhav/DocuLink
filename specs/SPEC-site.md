# Spec: Marketing site and trust pages

Module id: `site`
Status: Approved
Implements: MVP.md 1.1 (public security page), 10 (honest positioning, compliance support), 11 (pricing), 13 M5 (privacy page, terms, privacy policy), 15 Q6–Q7 (client-facing pages)
Designs: `4 Landing and sign-in - 01 Landing.png`; design system DS 01–09
Reading doc: [docs/specs/site.md](../docs/specs/site.md)
Last updated: 2026-10-20

## Objective

Convince a requester (a US tax preparer, accountant, HR team, law firm or university office; MVP.md 0 row 37) to start a free trial, and reassure the people they ask that a DocuLink link is safe. These are static, fast pages that tell the truth about security (MVP.md 10: no "bank-level", no certification claims). They map our controls to the FTC Safeguards Rule and IRS Publication 4557, so tax and accounting firms can cite us in their own written security program. Other audiences get the same plain descriptions of the controls, with no claims about rules specific to them.

Pages:

| Path | Page | Audience |
|---|---|---|
| `/` | Landing (design `4 … 01`) | Requesters |
| `/pricing` | Solo plan, Team "coming soon", FAQ | Requesters |
| `/sample` | "See a sample request": a read-only demo portal | Requesters |
| `/security` | "How we protect client documents", with the control map | Requesters, their IT and compliance people |
| `/data-care` | "How is my data protected?", plain language | Clients (linked from the portal footer) |
| `/why` | "Why did I get this?" (launch scope, MVP.md 15 Q7) | Clients (linked from the portal footer) |
| `/legal/terms`, `/legal/privacy`, `/legal/dpa`, `/legal/subprocessors` | Legal | Everyone |

Success means:
- Lighthouse 95+ on Performance, Accessibility, Best Practices and SEO for every page
- the landing page converts visitors to trial starts (tracked; no MVP target)
- every security claim is true and maps to a control in another spec

## Tech Stack

- Next.js static rendering (`export const dynamic = "force-static"`), content in TSX with `ui` components.
- No new dependencies. Legal pages are TSX with prose components.
- PostHog page views on marketing pages only, with cookieless persistence.
- `next/og` for Open Graph images.

## Commands

```
Test:         pnpm vitest run src/components/site
E2E:          pnpm test:e2e e2e/site
Lighthouse:   pnpm lhci autorun          # @lhci/cli in CI against the preview URL
```

## Project Structure

```
src/app/(site)/layout.tsx         → Marketing header (brand bar: How it works, For clients, Data care, Sign in, Start free trial) and footer
src/app/(site)/page.tsx           → Landing
src/app/(site)/pricing/page.tsx
src/app/(site)/sample/page.tsx    → Demo portal using portal components with fixture data and no API calls
src/app/(site)/security/page.tsx
src/app/(site)/data-care/page.tsx
src/app/(site)/why/page.tsx
src/app/(site)/legal/*/page.tsx
src/app/sitemap.ts, robots.ts     → robots disallows /c/, /app/, /api/
src/components/site/              → Hero, HowItWorks, ClientSection, RetentionSection, ControlMap, Faq
```

## Interfaces

### Landing (design `4 … 01`)

1. **Header:** brand bar with the wordmark and the nav. **Start free trial** goes to `/sign-in?trial=1`.
2. **Hero:**
   - mono kicker "Document requests for tax preparers, accountants, HR, law firms and universities"
   - "Ask once. Get every *document.*"
   - lead copy from the design
   - **Start free trial** (primary) and **See a sample request** (secondary)
   - "14 days free. No card needed."
   - the request label card with the Complete seal
3. **How it works:** 01 Ask, 02 Gather, 03 Review, with the design copy.
3a. **Who it's for** (new section, after How it works, in the design's type and spacing): five short rows, each an icon, an audience and two example requests:
   - `Receipt` Tax preparers: individual returns, self-employed clients
   - `Calculator` Accountants: new client onboarding, monthly bookkeeping
   - `Users` HR teams: new hire paperwork, contractor onboarding
   - `Scale` Law firms: new client intake, estate planning
   - `GraduationCap` Universities: admissions documents, financial aid verification
4. **For your clients:** a phone mockup and "No account. No password. *One link.*"
5. **Keep only what's needed:** a, b, c points. Retention "Deleted 90 days after completion by default, or when you choose" (automation step 4 is a must-have).
6. **Footer band:** big wordmark, **Start free trial**, and links to Privacy, Terms, Data care, Contact (`mailto:hello@doculinkapp.com`).

The lead copy speaks to all five audiences; the examples on the page start with tax because launch falls in tax season (MVP.md header).

### Sample request (`/sample`)

- A fixed demo: "Acme Accounting · 2026 Tax Return".
- It renders the real portal components in a `demo` mode: uploads stay in the browser (object URLs) and never call the API; Submit shows the success screen.
- A banner reads "This is a sample. Nothing you add leaves your browser."
- `noindex`.

### Security page (`/security`)

Plain-language sections. Each control maps to the FTC Safeguards Rule (16 CFR 314.4) element and the IRS Pub 4557 topic it supports, and links to nothing internal:

| Control | What we do | Source spec |
|---|---|---|
| Multi-factor authentication | Required before any file can be viewed (TOTP + backup codes) | auth |
| Encryption in transit | TLS everywhere, HSTS | foundation |
| Encryption at rest | AWS KMS customer-managed key for documents; database encrypted at rest | foundation, files |
| Access control | Private storage, links expire in 10 minutes, workspace isolation | files |
| Malware scanning | Every upload scanned before anyone can open it | files |
| Integrity | SHA-256 checksum verified on every upload | files |
| Audit log | Every view, upload, approval and deletion recorded | foundation |
| Retention and deletion | Automatic deletion after your retention period; deleting removes every copy | automation, settings |
| Vendor management | SOC 2 Type II reports on file for every sub-processor | site (sub-processors page) |
| Incident response | Contact `security@doculinkapp.com`; we notify affected customers without undue delay | runbook |

Required statements:
- "DocuLink is not SOC 2 or HIPAA certified."
- "This page describes our controls so you can include DocuLink in your written information security program."
- No "bank-level" or "military-grade" wording, enforced by a content test.
- "The FTC Safeguards Rule and IRS Publication 4557 apply to tax and accounting firms. If you're an HR team, law firm or university, use this page to check our controls against your own requirements." DocuLink makes no FERPA, HIPAA or bar association claims.

### Client pages

- **`/data-care`:** who can see your files (only the firm), where they're stored (US, encrypted), how long they're kept, how to ask the firm to delete them, and that we never sell or share data. Written at an 8th-grade reading level.
- **`/why`:** "{Business} asked you for documents using DocuLink. If you don't know them, don't upload anything and contact them directly." Plus how to report phishing (`abuse@doculinkapp.com`). It's generic, with no request data.

### Pricing (`/pricing`)

- **Solo:** $29 a month or $290 a year (toggle). One user, unlimited requests, 500 MB per request, 50 GB storage. "Plus sales tax where applicable."
- **Team:** "Coming in 2027", with a "Tell me when" mailto (no form in the MVP).
- **FAQ:** trial, cancellation, what clients need, security link.

## Data

None.

## Code Style

Pages are server components with no client JavaScript, except the pricing toggle and the sample portal. Copy lives inline in the TSX, following DS 09 voice rules (sentence case, specific, no exclamation marks or emoji).

## Testing Strategy

- **Component:** sections render; the pricing toggle; the sample portal never calls `fetch` (spy).
- **Content tests:**
  - forbidden phrases ("bank-level", "military-grade", "SOC 2 certified", "HIPAA compliant", "FERPA compliant", "SOC 2 compliant", "SOC 2 ready") are absent from the rendered HTML of every page
  - the security page includes the "not certified" statement
- **E2E:** every page returns 200 with axe checks clean; robots and sitemap; portal footer links resolve; the sample flow (add a file, submit, success) works offline (network blocked except static assets).
- **Lighthouse CI** budgets: 95+ in all four categories, LCP under 2.0 s on mobile simulation.

## Edge Cases

- **Visitors from the EU or UK** → no cookie banner is needed for cookieless PostHog. The privacy policy covers it. No geo-blocking.
- **A logged-in requester visits `/`** → the header shows "Open app" instead of "Sign in".
- **`/why` is linked from a phishing email that imitates us** → the page says how to verify (contact the firm directly) and doesn't confirm any request exists.

## Boundaries

- **Always:**
  - statements match what the specs actually implement
  - content tests for forbidden claims
  - static rendering
  - accessibility checks
- **Ask first:**
  - adding forms (waitlist, contact)
  - third-party scripts (chat, ads, pixels)
  - testimonials (none until beta firms agree)
- **Never:**
  - compliance or certification claims we don't hold
  - tracking pixels on client-facing pages
  - showing real client data in the sample

## Success Criteria

- [ ] Every page in the table exists, matches the design and DS, and passes axe checks and Lighthouse budgets.
- [ ] The security page maps every control to its implementation and to the FTC/IRS topic, with the honest-positioning statements.
- [ ] The sample request works with no network calls.
- [ ] Legal pages exist with reviewed text (a lawyer review is outside engineering scope; placeholder text is never shipped to production).

## MVP.md changes (applied in v1.2, MVP.md 0 rows 37 and 38)

1. **MVP.md 1.1:** the `/sample` demo portal (it's on the landing design) and `/why` (MVP.md 15 Q7) are launch scope.
2. **MVP.md header, 2, 10:** the audience is tax preparers, accountants, HR teams, lawyers and universities, with the FTC and IRS mapping aimed at tax and accounting firms.

## Decisions (Oct 20, 2026)

1. **Audience copy:** for tax preparers, accountants, HR, law firms and universities.
2. **Contact:** email only, `hello@doculinkapp.com`. No form.
3. **Legal text:** decided later.

## Open Questions

1. **Legal text owner and date.** Who drafts the terms, privacy policy and DPA, and by when? It doesn't block building the pages, but placeholder text never ships, so it blocks the mid-December beta.
