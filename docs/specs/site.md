# Site: landing page and trust pages

Status: Draft, revision 2 waiting for your review · Last updated: 2026-10-20
Full spec: [specs/SPEC-site.md](../../specs/SPEC-site.md) · Product spec: MVP.md 1.1, 10, 11

## In one paragraph
The public website. It sells DocuLink to tax preparers, accountants, HR teams, law firms and universities: "Ask once. Get every *document.*" It also explains, honestly and in plain language, how client documents are protected. One version is for firms, mapped to the FTC Safeguards Rule and IRS Publication 4557 so tax and accounting firms can cite us in their own security program; other audiences get the same plain descriptions, with no claims about their own rules. Another is for clients wondering whether a link is safe. It includes a sample request anyone can try without signing up.

## Who it's for
- **Tax preparers, accountants, HR teams, law firms and universities deciding whether to try DocuLink,** and their compliance people.
- **Clients** who clicked "How is my data protected?" or "Why did I get this?" in the portal.

## How it works
1. **Landing:** headline, how it works (Ask, Gather, Review), a "Who it's for" section with the five audiences, what the client sees, the retention promise, and **Start free trial**.
2. **Sample request:** a fully clickable demo of the client's view. Nothing leaves the browser.
3. **Pricing:** Solo, monthly or annual; Team "coming in 2027"; an FAQ.
4. **Security:** each protection in plain words, what it covers in the FTC and IRS rules, and a clear statement that we're not SOC 2 or HIPAA certified.
5. **Data care and "Why did I get this?":** short pages for clients, written so anyone can follow them.
6. **Legal:** terms, privacy, data processing addendum, sub-processor list.

## Key decisions
| Decision | Why |
|---|---|
| Never claim certifications we don't have | Honesty is the product; an automated check blocks phrases like "bank-level" |
| Static pages, no third-party scripts | Fast, private, nothing to break |
| The sample request runs entirely in the browser | People can try it with zero risk and no sign-up |
| No cookie banner | Analytics on these pages run without cookies |

## Not included
- A blog, testimonials, or contact and waitlist forms (later).

## Decided
- The landing page speaks to tax preparers, accountants, HR, law firms and universities.
- Contact is by email only: hello@doculinkapp.com.

## Open questions
- Who writes the terms, privacy policy and data processing addendum, and by when? Decided later, but they must be ready before the mid-December beta.
- Please review the new audience wording and the "Who it's for" section.
