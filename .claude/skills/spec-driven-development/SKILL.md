---
name: spec-driven-development
description: Creates specs before coding, and a plain-language reading doc in docs/ for every spec and every implementation. Use when starting a new DocuLink module, feature or significant change and no spec exists yet; when drafting requirements with objectives and scope; when requirements are unclear, ambiguous or only a vague idea; or when one requirement spans several independently testable capabilities and needs a capability map of modules before specifying. Also use after implementing any spec'd task, to write its docs/implementation reading doc.
---

# Spec-Driven Development (DocuLink)

## Overview

Write a structured specification before writing any code. The spec is the shared source of truth between you and the human engineer. It defines what we're building, why, and how we'll know it's done. Code without a spec is guessing.

Every spec and every implementation also gets a **reading doc** in `docs/`: a short, plain-language explanation a person can read in five minutes without opening the code or the full spec. Specs are contracts for building; docs are for understanding.

## Project context

- **`MVP.md` is the product spec** (the PRD) for DocuLink. It holds product decisions, flows, UI, architecture, data model, jobs, security, billing, build plan and acceptance criteria. Module specs **reference MVP.md sections** (for example "MVP.md 4.2, 8.4") instead of copying them, and add only what's needed to build: interfaces, file layout, tests, edge cases.
- **Product decisions change in MVP.md first.** If a module spec needs to change a product decision (scope, flow, data model, pricing, security), update MVP.md, add a row to its section 0 change log, then update the module spec.
- **Stack** (MVP.md section 6): Next.js App Router, TypeScript, Tailwind, shadcn/ui on Radix, Drizzle on Neon Postgres, Better Auth, AWS S3/SES/GuardDuty/EventBridge (Vercel OIDC), Upstash Redis and QStash, Stripe, Sentry, PostHog. Don't introduce alternatives without asking.
- **Designs** live in `Designs/` as PNGs. Specs that touch UI name the design files they follow.

### Where files go

| Artifact | Path | Audience |
|---|---|---|
| Capability map | `specs/CAPABILITY-MAP.md` | Agent and human; the index of modules |
| Module spec | `specs/SPEC-<module-id>.md` | Agent and human; the build contract |
| Plan | `tasks/plan-<module-id>.md` | Agent and human |
| Task list | `tasks/todo-<module-id>.md` | Agent and human |
| Spec reading doc | `docs/specs/<module-id>.md` | Anyone; plain language |
| Implementation reading doc | `docs/implementation/YYYY-MM-DD-<module-id>-<slug>.md` | Anyone; plain language |
| Docs index | `docs/README.md` | Anyone |

Create folders as needed. Templates live next to this file in `templates/`.

## When to use

- Starting a new module or feature
- Requirements are ambiguous or incomplete
- The change touches multiple files or modules
- You're about to make an architectural decision
- The task would take more than 30 minutes to implement
- You just finished implementing a spec'd task (write its implementation reading doc)

**When not to use:** single-line fixes, typo corrections, or changes where requirements are unambiguous and self-contained. Those still get a commit message, not a spec.

## The gated workflow

Four phases, preceded by a scope check (Phase 0) that applies only when one request bundles several independently testable capabilities. Do not advance until the current phase is approved by the human.

```
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT
   │          │        │          │
   ▼          ▼        ▼          ▼
 Human      Human    Human      Human
 reviews    reviews  reviews    reviews
   │                              │
   ▼                              ▼
docs/specs/            docs/implementation/
```

### Phase 0: Scope check

Most requests describe one capability. If this one does, skip to Specify.

**Detection.** Decompose before specifying when a single requirement bundles several independently testable capabilities:

- It names distinct capabilities with their own consumers or data (for DocuLink: auth, requests, portal uploads, review, notifications, billing, exports)
- Acceptance criteria cluster into groups that could ship and be verified separately
- One capability could be cut or replaced without rewriting the others' requirements

Building the whole of MVP.md is the obvious case: map it into modules first.

**Propose a capability map before writing any spec.** A module table plus a build order, not a project plan. Use `templates/capability-map.md`.

```markdown
# Capability Map: DocuLink MVP

| Module id | Responsibility | MVP.md sections | Depends on |
|---|---|---|---|
| foundation | Repo, infra-as-code, schema, design tokens | 6, 7, 5.4 | — |
| auth | Sign-in, two-factor, workspaces | 5.3, 10 | foundation |
| requests | Builder, templates, send, dashboard | 4.1, 5.3 | auth |
| portal | Client checklist, uploads, submit | 4.2, 4.3, 5.6 | requests |
| ... | | | |

Build order: foundation → auth → requests → portal → ...
```

- **Stable module ids.** Kebab-case, chosen once, never renamed. Specs, plans, tasks and docs are selected by these ids.
- **Dependency direction, no cycles.** If two modules each need the other, they are one module.
- **Interfaces live at the boundary.** The map records that `portal` depends on `requests`; the contract between them belongs in the provider module's spec.

**The map is gated.** The human reviews module boundaries, dependency direction and build order before any module spec is written. Save it to `specs/CAPABILITY-MAP.md` and stop your turn.

**Then recurse per module** in dependency order: Specify → Plan → Tasks → Implement. The map, not filename guessing, is the index of what exists.

### Phase 1: Specify

Start from the vision (and the relevant MVP.md sections). Ask clarifying questions until requirements are concrete.

**Surface assumptions immediately**, before writing spec content:

```
ASSUMPTIONS I'M MAKING:
1. Portal uploads go direct to S3 with presigned PUTs (MVP.md 4.2)
2. Server Actions for requester mutations, Route Handlers for portal APIs
3. Tests use Vitest for units and Playwright for end-to-end
→ Correct me now or I'll proceed with these.
```

Don't silently fill in ambiguous requirements. Surfacing misunderstandings before code exists is the whole point.

**Write the spec to `specs/SPEC-<module-id>.md`** using `templates/spec.md`. It covers six core areas:

1. **Objective.** What we're building and why, who the user is, what success looks like. Cite MVP.md sections.
2. **Commands.** Full executable commands with flags (build, test, lint, dev, migrations). Until the app is scaffolded, write the commands the `foundation` module will create and mark them "planned".
3. **Project structure.** Where this module's source, tests and docs live.
4. **Code style.** One real snippet beats three paragraphs. Naming conventions, formatting, examples of good output.
5. **Testing strategy.** Framework, test locations, coverage expectations, which test level covers which concern. Map MVP.md section 14 acceptance criteria to tests.
6. **Boundaries.**
   - **Always:** run tests before commits, follow naming conventions, validate inputs, check workspace ownership on every requester action
   - **Ask first:** database schema changes, adding dependencies, changing CI or infrastructure, changing an MVP.md decision
   - **Never:** commit secrets or AWS access keys, log document contents or file names to Sentry/PostHog, weaken a security control from MVP.md section 10, remove failing tests without approval

**Reframe vague instructions as success criteria:**

```
REQUIREMENT: "Uploads should be reliable on mobile"

REFRAMED SUCCESS CRITERIA:
- Upload failure rate under 2% (MVP.md 12)
- A dropped connection retries 3 times with backoff, then shows Retry
- Reopening the link on another device shows every confirmed upload
→ Are these the right targets?
```

**External spec tools.** If the project later adopts OpenSpec or another spec system, keep that system's format and storage instead of creating duplicate `SPEC-*.md` files. This skill still owns clarification, content, approval gates and the reading docs.

**Write the spec reading doc.** After saving the spec, write `docs/specs/<module-id>.md` using `templates/reader-spec.md` and add or update its row in `docs/README.md` (status: Draft). Keep it short, plain-language and free of code: what this part of DocuLink does, who it's for, how it works step by step, the key decisions and why, what's out of scope and open questions. Link to the full spec.

**Stop after writing the spec (critical).** Once the spec and its reading doc are saved:

1. Summarize the spec and list any open questions.
2. Ask the human to approve it or request changes.
3. **Stop your turn immediately.** Don't start Phase 2 or write code in this turn. Planning starts only after the human approves the spec in a later turn.

On approval, change the spec and reading doc status to **Approved**, in both the files and `docs/README.md`.

### Phase 2: Plan

With the approved spec, write `tasks/plan-<module-id>.md`:

1. Identify the major components and their dependencies.
2. Choose the implementation order. Prefer vertical slices (a thin path through UI, API and data that can be tested end to end) over horizontal layers.
3. Note risks and mitigations.
4. Mark what can be built in parallel and what must be sequential.
5. Define verification checkpoints between phases.

The plan should be reviewable in a few minutes: the human can say "yes, that's the approach" or "no, change X". Stop for approval.

### Phase 3: Tasks

Break the plan into tasks in `tasks/todo-<module-id>.md`:

- Each task fits one focused session.
- Each has explicit acceptance criteria and a verification step (test, build, manual check).
- Tasks are ordered by dependency, not perceived importance.
- No task should change more than about 5 files.

```markdown
- [ ] Task: [Description]
  - Spec: [SPEC-<module-id>.md section, MVP.md section]
  - Acceptance: [What must be true when done]
  - Verify: [Test command, build, manual check]
  - Files: [Which files will be touched]
```

Stop for approval.

### Phase 4: Implement

Execute tasks one at a time:

1. Load only the spec sections, MVP.md sections and source files the task needs, not the whole spec.
2. Write or update the tests for the task's acceptance criteria first, then the code to make them pass.
3. Run the task's verification step and the module's test suite. Report failures honestly.
4. Tick the task in `tasks/todo-<module-id>.md`.
5. **Write the implementation reading doc** (below).
6. Commit only when the human asks. Reference the spec section and the implementation doc in the commit or PR description.

If implementation shows the spec is wrong, stop, update the spec (and MVP.md if it's a product decision), tell the human, then continue.

**Implementation reading doc.** For each implemented task, or each group of tasks finished together in one session, write `docs/implementation/YYYY-MM-DD-<module-id>-<slug>.md` using `templates/reader-implementation.md`, and add its row to `docs/README.md`. It explains in plain language:

- what was built and which spec sections it implements
- how it works, as a short walkthrough (a small Mermaid diagram if a flow has more than three steps)
- where the code lives (main files and folders, as links)
- how to run and verify it (exact commands, test names, manual steps)
- decisions made during implementation, and any spec or MVP.md updates they caused
- known limitations and follow-ups

Write for a new teammate or a non-engineer cofounder: short sentences, no unexplained jargon, no pasted code beyond a line or two when it truly helps. Use today's date in the filename.

## Docs index

`docs/README.md` is the reading index. Create it on first use from `templates/docs-readme.md`. Keep two tables up to date:

- **Specs:** module id, title, status (Draft, Approved, Implemented), reading doc link, spec link
- **Implementation notes:** date, module id, title, link

Newest implementation notes first. When every task in a module is done, mark its spec **Implemented**.

## Keeping the spec alive

- **Update when decisions change.** If the data model needs to change, update the spec first (and MVP.md for product decisions), then implement.
- **Update when scope changes.** Features added or cut appear in the spec and its reading doc.
- **Commit the spec and docs.** They belong in version control alongside the code.
- **Reference the spec in PRs.** Link the spec section and the implementation reading doc each PR covers.
- **Docs follow the spec.** When a spec changes after approval, update its reading doc in the same change.

## Common rationalizations

| Rationalization | Reality |
|---|---|
| "This is simple, I don't need a spec" | Simple tasks don't need long specs, but they need acceptance criteria. A two-line spec is fine. |
| "I'll write the spec after I code it" | That's documentation, not specification. The value is forcing clarity before code. |
| "The spec will slow us down" | A 15-minute spec prevents hours of rework. |
| "Requirements will change anyway" | That's why the spec is a living document. An outdated spec beats no spec. |
| "The user knows what they want" | Even clear requests carry implicit assumptions. The spec surfaces them. |
| "MVP.md already covers it" | MVP.md says what the product does. A module spec says how this module is built and tested. Reference MVP.md; don't skip the spec. |
| "It's one big feature; splitting it is overhead" | If acceptance criteria cluster into independently testable groups, a ten-line capability map is cheaper than one giant spec. |
| "I'll decompose during planning" | Module boundaries must be decided before the spec is written, not after. |
| "The code is the documentation" | The reading doc is for people who won't read the code. Five minutes to write, saves an hour of explaining. |

## Red flags

- Writing code without written requirements
- Asking "should I just start building?" before defining "done"
- Implementing features not in any spec or task list
- Architectural decisions made without being written down
- Writing the spec and starting the plan or code in the same turn
- One spec spanning several independently testable capabilities
- Module boundaries decided implicitly during implementation
- A module spec contradicting MVP.md without MVP.md being updated
- A spec or implementation with no reading doc in `docs/`, or `docs/README.md` out of date

## Verification

Before implementation starts:

- [ ] The spec covers all six core areas and cites the MVP.md sections it implements
- [ ] The human reviewed and approved the spec in a later turn than the one that wrote it
- [ ] Success criteria are specific and testable
- [ ] Boundaries (Always, Ask first, Never) are defined
- [ ] The spec is saved in `specs/`, and its reading doc in `docs/specs/` is listed in `docs/README.md`
- [ ] If the request bundled several capabilities, a capability map was approved before any module spec
- [ ] Every module spec traces to a module id in the approved map

After each implementation:

- [ ] Tests for the task's acceptance criteria pass, and the results were reported honestly
- [ ] The task is ticked in `tasks/todo-<module-id>.md`
- [ ] An implementation reading doc exists in `docs/implementation/` and is listed in `docs/README.md`
- [ ] Any spec or MVP.md changes made during implementation are reflected in the spec and its reading doc
