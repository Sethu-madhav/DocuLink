# Spec: Foundation and server conventions

Module id: `foundation`
Status: Approved (revision 2: SOC 2 readiness controls, 2026-10-20)
Implements: MVP.md 6, 6.1, 6.2, 6.3, 7, 8 (job conventions), 10, 13 M1
Designs: none
Reading doc: [docs/specs/foundation.md](../docs/specs/foundation.md)
Last updated: 2026-10-20

## Objective

Give every other module a working, secure base, so that module specs only have to describe their own business logic. Foundation covers four things:

1. **Repository and tooling.** A Next.js app with TypeScript, linting, tests, CI and deploys to Vercel staging and production.
2. **Infrastructure as code.** Everything in AWS (S3, KMS, IAM with Vercel OIDC, SES, GuardDuty, EventBridge, CloudTrail), in separate staging and production accounts. Also the provisioning checklist for Neon, Upstash, Stripe, Sentry, PostHog and Turnstile.
3. **Database.** The full MVP.md 7 schema as the first Drizzle migration, including the append-only trigger, plus the database clients.
4. **Server conventions.** One way to:
   - validate input
   - check access
   - run business logic in transactions
   - log audit events
   - rate limit
   - publish and receive jobs
   - talk to AWS
   - encrypt tokens
   - report errors and analytics

   All other module specs refer to these conventions instead of redefining them.

Success means a new module can add a Server Action or route in about ten lines that is validated, authorized, rate limited, transactional, audited and observable by default.

## Tech Stack

| Area | Choice | Notes |
|---|---|---|
| Runtime | Node.js 24 LTS, pnpm 10 | `engines` pinned; `.nvmrc` |
| Framework | Next.js 16 (App Router), React 19, TypeScript strict | Node.js runtime for all routes. The request interceptor is `proxy.ts` (Next 16's replacement for `middleware.ts`), used for security headers and the auth redirect only. Confirm the exact file convention at scaffold time. |
| Styling | Tailwind CSS v4 | Tokens owned by `ui` |
| Validation | `zod` v4 | Every external input: env, Server Actions, route bodies and params, webhooks, job payloads, JSON columns |
| Database | `drizzle-orm`, `drizzle-kit`, `@neondatabase/serverless` | `neon-http` driver for single-statement reads; `neon-serverless` `Pool` with `attachDatabasePool` (`@vercel/functions`) for transactions |
| Cache, limits, jobs | `@upstash/redis`, `@upstash/ratelimit`, `@upstash/qstash` | |
| AWS | `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, `@aws-sdk/lib-storage`, `@aws-sdk/client-sesv2`, `@vercel/functions/oidc` | No access keys anywhere |
| Time zones | `date-fns`, `@date-fns/tz` | Local-morning scheduling, due-date expiry |
| Observability | `@sentry/nextjs`, `posthog-js`, `posthog-node` | Scrubbing rules below |
| Infrastructure as code | Terraform 1.9+ with the AWS provider, state in S3 with locking, one root per environment | Chosen over CDK (decided Oct 20, 2026) |
| Tests | Vitest (unit, integration, component projects), Playwright, `aws-sdk-client-mock`, Postgres 17 in Docker for integration | |
| Lint | ESLint (flat config) with `eslint-plugin-boundaries` for module imports, Prettier | |
| CI | GitHub Actions | |

## Commands

```
Install:    pnpm install
Dev:        pnpm dev                                   # next dev on :3000, uses .env.local
DB up:      pnpm db:up                                 # docker compose up -d postgres (local Postgres 17)
Generate:   pnpm db:generate                           # drizzle-kit generate (SQL migration from schema)
Migrate:    pnpm db:migrate                            # drizzle-kit migrate against DATABASE_URL
Studio:     pnpm db:studio
Seed:       pnpm db:seed                               # local only: one workspace, sample requests
Test:       pnpm test                                  # vitest run --project unit --project component
Integr.:    pnpm test:int                              # vitest run --project integration (needs pnpm db:up)
E2E:        pnpm test:e2e                              # playwright test (against pnpm dev or PLAYWRIGHT_BASE_URL)
Lint:       pnpm lint && pnpm typecheck                # eslint . ; tsc --noEmit
Build:      pnpm build
Infra plan: pnpm infra:plan -- staging                 # terraform -chdir=infra/envs/staging plan
Infra app.: pnpm infra:apply -- staging                # never run against production from a laptop
Jobs:       pnpm jobs:schedule -- staging              # create or update QStash schedules (idempotent)
```

## Project Structure

```
proxy.ts                          → Security headers, auth redirect (cookie presence only)
src/app/                          → Pages and route handlers. The only layer that wires modules together.
src/app/api/health/route.ts       → Liveness: DB ping, Redis ping, version
src/app/api/webhooks/aws/route.ts → Composes createAwsEventsRoute with email and files handlers
src/server/env.ts                 → Zod-validated env, server only
src/server/db/                    → schema/*.ts (all MVP.md 7 tables), client.ts (db, withTx), enums.ts
src/server/core/                  → action.ts, route.ts, errors.ts, result.ts, clock.ts, ids.ts
src/server/audit/                 → logEvent, event type union
src/server/limits/                → rateLimit, policies
src/server/jobs/                  → publishJob, verifyQStash, createJobRoute
src/server/aws/                   → credentials.ts (OIDC), s3.ts, ses.ts, events.ts (createAwsEventsRoute)
src/server/crypto/                → randomToken, sha256, encryptToken/decryptToken (versioned AES-256-GCM), hmac
src/server/observability/         → sentry scrubbing, track() server analytics
src/server/<module>/              → One folder per feature module; public API in index.ts
src/lib/                          → Isomorphic helpers (no secrets)
drizzle/                          → Generated SQL migrations (committed)
infra/modules/, infra/envs/{staging,production}/ → Terraform
e2e/                              → Playwright specs
tests/                            → Test helpers: db reset, factories, fakes (redis, qstash, clock)
.github/workflows/                → ci.yml, migrate.yml, codeql.yml
.github/CODEOWNERS, .github/dependabot.yml
docs/compliance/                  → control list, access reviews, onboarding and offboarding checklists
docs/ops/                         → incident response, secret rotation, restore drills, access log rules
```

Module layout (every feature module follows this):

```
src/server/<module>/
  index.ts        → the public API; the only file other modules may import
  schemas.ts      → zod input schemas (shared with forms on the client)
  rules.ts        → pure business rules (no IO): state transitions, limits, calculations
  service.ts      → use cases: load, check rules, write in withTx, log events, side effects after commit
  queries.ts      → read models for pages
  actions.ts      → "use server" Server Actions built with defineAction (thin)
  *.test.ts       → unit tests for rules; *.int.test.ts for service against Postgres
```

## Interfaces

### Environment (`@/server/env`)
`env` is parsed once at boot with zod from MVP.md 6.2. It also includes:
- `NODE_ENV`, `VERCEL_ENV`
- `APP_URL` (validated as https outside development): `https://doculinkapp.com` in production, `https://staging.doculinkapp.com` on staging
- `EMAIL_LINK_SECRET` (32 bytes, base64): signs reminder unsubscribe links (SPEC-email)
- `EMAIL_TRANSPORT` (`ses` | `outbox`; `outbox` stores sent mail in memory or the database in development and tests)
- `DEV_UI_ENABLED`

If a variable is missing, the build fails with the variable's name. Secrets never get a `NEXT_PUBLIC_` prefix, and `src/server/**` files import `server-only`.

### Errors and results (`@/server/core`)

```ts
type ErrorCode =
  | "VALIDATION"        // 400, with fieldErrors
  | "UNAUTHENTICATED"   // 401, redirect to /sign-in
  | "TWO_FACTOR_REQUIRED" // 403, UI routes to setup or challenge
  | "NOT_FOUND"         // 404, also used for other workspaces' resources (never 403)
  | "CONFLICT"          // 409, invalid state transition or stale write
  | "PAYMENT_REQUIRED"  // 402, opens the upgrade dialog
  | "PAYLOAD_TOO_LARGE" // 413
  | "RATE_LIMITED"      // 429, with retryAfterSeconds
  | "UPSTREAM"          // 502, AWS/Stripe/SES failure after retries
  | "INTERNAL";         // 500

class AppError extends Error { code: ErrorCode; userMessage: string; fieldErrors?: Record<string, string[]>; retryAfterSeconds?: number }
type Result<T> = { ok: true; data: T } | { ok: false; error: { code: ErrorCode; message: string; fieldErrors?: Record<string, string[]>; retryAfterSeconds?: number } };
```

Rules:
- **`userMessage` follows MVP.md 5.8:** say what happened and what to do. It never contains internal detail. `INTERNAL` always shows "Something went wrong. Try again, and contact us if it keeps happening."
- **Business rules throw `AppError`.** Wrappers convert it to a `Result` for actions, or to JSON with a status code for routes.
- **Unknown errors** are reported to Sentry and returned as `INTERNAL`.

### Server Actions (`defineAction`)

```ts
export const approveItem = defineAction({
  input: z.object({ itemId: z.uuid() }),
  context: requireWorkspace,               // from auth; resolves { user, workspace, session } or throws
  rateLimit: (ctx) => ({ policy: "requesterWrite", key: ctx.user.id }), // optional
  handler: async ({ input, ctx }) => review.approveItem(ctx, input.itemId),
});
// returns Promise<Result<Awaited<ReturnType<handler>>>>
```

`defineAction` always does the following:
1. Parses the input (zod). On failure it returns `VALIDATION` with field errors.
2. Resolves the context. Foundation ships `publicContext`; auth supplies `requireUser`, `requireWorkspace` and `requireTwoFactor`.
3. Applies the rate limit.
4. Runs the handler inside a Sentry span.
5. Maps errors to a `Result`.
6. Never throws to the client.

Mutating actions call `revalidatePath` or `revalidateTag` inside the handler, not in the wrapper.

### Route handlers (`defineRoute`)

```ts
export const POST = defineRoute({
  params: z.object({ id: z.uuid() }), body: schema, query: schema,
  context: requirePortalSession,           // or requireWorkspace, verifyQStash, ...
  rateLimit, maxBodyBytes: 16_384,
  handler: async ({ params, body, ctx, req }) => Response | json,
});
```

Same pipeline as `defineAction`. Errors come back as `{ error: { code, message, fieldErrors? } }` with the mapped status code. Every route exports `runtime = "nodejs"`. Routes that need it set `maxDuration`.

### Database (`@/server/db`)

```ts
db            // drizzle(neon-http) — single statements and reads
withTx(fn)    // Pool-backed db.transaction(fn); retries once on serialization failure; fn gets tx
schema.*      // all tables and enums from MVP.md 7
```

Rules:
- **Multi-statement writes** always go through `withTx`.
- **Side effects run after commit.** Emails, QStash publishes, S3 deletes and PostHog calls are collected with `tx.afterCommit(cb)`, a small helper that runs them in order after the transaction commits and skips them on rollback.
- **Workspace scoping.** Every query on customer data filters by `workspace_id`, directly or through `document_requests`. Repository helpers take `workspaceId` as their first argument. There are no unscoped "by id" lookups outside system jobs.
- **Optimistic state checks.** State-changing updates include the expected status in the `WHERE` clause, for example `UPDATE ... WHERE id = $1 AND status = 'submitted' RETURNING *`. Zero rows means `CONFLICT`. This makes double clicks and races safe.
- **IDs and timestamps.** IDs are `uuid` with the `gen_random_uuid()` default. Timestamps are `timestamptz` and stored in UTC.

### Audit log (`@/server/audit`)

```ts
logEvent(tx, { requestId, actor: "requester" | "recipient" | "system", type: EventType, itemId?, fileId?, meta? })
type EventType = /* the 25 types in MVP.md 7 */;
```

- **`meta` is typed per event type.** It never contains file contents, and never file names beyond the sanitized `original_name` already stored.
- **Append-only.** No update or delete code path exists. A trigger blocks `UPDATE` and `DELETE` unless the transaction has set `SET LOCAL app.allow_event_delete = 'on'`. Only account deletion and `requests.deleteDraft` set it (MVP.md 7).

### Rate limits (`@/server/limits`)

```ts
rateLimit(policy: PolicyName, key: string): Promise<void>   // throws RATE_LIMITED with retryAfterSeconds
```

Policies (from MVP.md 10, sliding window):

| Policy | Limit | Key |
|---|---|---|
| `portalPageLoad` | 60 per 10 min | IP |
| `portalPresign` | 60 per 10 min | request id |
| `invalidToken` | 20 per 10 min, then a 1-hour block | IP |
| `signInEmail` | 5 per hour | email address |
| `sendRequest` | 100 per day | user |
| `twoFactorAttempt` | 5 per 15 min, then a 15-minute lock | user |
| `reminderNow` | 1 per 12 hours | request |
| `requesterWrite` | 300 per 10 min (abuse guard) | user |
| `portalWrite` | 120 per 10 min | request |

Client IP comes from `x-forwarded-for` as set by Vercel; the first entry only.

### Jobs (`@/server/jobs`)

```ts
publishJob(path: JobPath, body: unknown, opts?: { delaySeconds?: number; deduplicationId?: string })
createJobRoute({ body: schema, handler })   // verifies the QStash signature (current and next keys) on the raw body
```

How job routes behave:
- They are idempotent.
- They return 200 for "nothing to do".
- They throw for transient failures, so QStash retries 3 times with backoff.
- A Sentry alert fires when the final attempt fails.
- Locally, `EMAIL_TRANSPORT=outbox` together with a dev QStash (`npx @upstash/qstash-cli dev`) runs jobs end to end.

### AWS (`@/server/aws`)

```ts
s3(): S3Client      // credentials: awsCredentialsProvider({ roleArn: env.AWS_ROLE_ARN }) on Vercel; dev uses a staging read-write sandbox role via `aws sso` profile
ses(): SESv2Client
createAwsEventsRoute(handlers: { "aws.ses"?: Handler; "aws.guardduty"?: Handler })
```

`createAwsEventsRoute` does the following, in order:
1. Compares the `x-doculink-events-secret` header to `AWS_EVENTS_SECRET` in constant time.
2. Parses the EventBridge envelope (zod).
3. Deduplicates on the event `id` with Redis `SET NX EX 604800`.
4. Dispatches to the handler for the event's `source`.
5. Answers 200 for unknown sources, which are logged.

The dedupe key is deleted again if the handler throws, so that EventBridge's retry is processed.

### Crypto (`@/server/crypto`)

```ts
randomToken(bytes = 32): string                 // base64url
sha256Hex(input: string): string
encryptToken(token): string                     // "v1:" + base64(iv(12) | tag(16) | ciphertext), key = TOKEN_ENCRYPTION_KEY
decryptToken(ciphertext): string                // dispatches on version prefix; unknown version throws
hmacSign(payload, secret) / hmacVerify(...)     // SHA-256, constant-time compare; used by the portal cookie
```

### Observability

- **Sentry** runs on the server, the client and in jobs. A `beforeSend` scrubs:
  - request bodies, query strings and cookies on `/c/*` and `/api/portal/*`
  - every `Authorization`, `Cookie` and `x-*-secret` header
  - every field named `token`, `email`, `name`, `original_name` or `note`

  Releases are tagged with the commit SHA. Sentry Cron monitors cover the hourly and daily jobs.
- **`track(event, props)`** is server-side PostHog with a typed event map (MVP.md 12). Its props are allow-listed per event, so file names, emails and notes are compile-time errors. `distinct_id` is the user id for requesters and the request id for recipients.
- **Client PostHog:**
  - autocapture of inputs off, `mask_all_text` for the portal
  - replay masks all inputs, with a `ph-no-capture` class on preview components
  - `persistence: "memory"` on the portal (no cookies for clients)

### Security headers (`proxy.ts`)

- **On every response:** HSTS (2 years, preload), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` (camera=self, nothing else), `X-Frame-Options: DENY`.
- **CSP with a per-request nonce:**
  - `default-src 'self'`
  - `script-src 'self' 'nonce-…' https://challenges.cloudflare.com` and the PostHog host
  - `connect-src 'self' https://{bucket}.s3.us-east-1.amazonaws.com` and the PostHog and Sentry hosts
  - `img-src 'self' data: blob:` and the S3 origin
  - `frame-src https://challenges.cloudflare.com`
  - `frame-ancestors 'none'`
- **Portal routes (`/c/*`)** add `Referrer-Policy: no-referrer`, `Cache-Control: no-store` and `X-Robots-Tag: noindex, nofollow` (MVP.md 4.2, 10).
- **Auth redirect.** Requester routes (`/app/*`) with no session cookie redirect to `/sign-in`. The real session check happens on the server in `requireUser`.

## Infrastructure (Terraform, per environment)

| Resource | Settings |
|---|---|
| S3 bucket `doculink-{env}-files` | Block Public Access (all four), BucketOwnerEnforced, versioning on, SSE-KMS with the app CMK and Bucket Keys, policy denying `aws:SecureTransport = false` and any principal other than the app role and GuardDuty's role. CORS: `PUT, GET, HEAD` from `APP_URL` only, `ExposeHeaders: ETag`. Lifecycle: non-current versions expire after 7 days; `exports/` after 1 day; `quarantine/` after 30 days; incomplete multipart uploads abort after 1 day |
| KMS CMK `alias/doculink-{env}-files` | Rotation on; key policy grants use to the app role, GuardDuty's role and the account admin role |
| IAM role `doculink-{env}-app` | OIDC trust: `oidc.vercel.com/{team}`, `aud = https://vercel.com/{team}`, `sub = owner:{team}:project:doculink:environment:{production\|preview}`. Permissions: `s3:PutObject, GetObject, GetObjectVersion, DeleteObject, DeleteObjectVersion, CopyObject (via Put/Get), GetObjectTagging, ListBucket, ListBucketVersions, AbortMultipartUpload` on the bucket; `kms:Encrypt, Decrypt, GenerateDataKey` on the CMK; `ses:SendEmail` on the identity and configuration set |
| IAM role `doculink-{env}-guardduty` | For the Malware Protection plan: tag and read objects under `requests/`, use the CMK, publish to EventBridge |
| GuardDuty Malware Protection for S3 | Plan on the bucket, prefix `requests/`, tagging enabled |
| SES | Domain identity `mail.doculinkapp.com` (staging: `mail.staging.doculinkapp.com`) with Easy DKIM, custom MAIL FROM, DMARC record `p=quarantine` (start at `p=none` for 2 weeks); configuration set `doculink-{env}` with an event destination to EventBridge for `DELIVERY, BOUNCE, COMPLAINT, REJECT`; account-level suppression for bounces and complaints |
| EventBridge | Rules: `source = aws.guardduty` with detail-type "GuardDuty Malware Protection Object Scan Result" and bucket = ours; `source = aws.ses` from our configuration set. Target: an API destination `POST {APP_URL}/api/webhooks/aws` with a connection using an API-key header `x-doculink-events-secret`; retry 24 h; DLQ (SQS) with a CloudWatch alarm |
| CloudTrail | Org or account trail with S3 data events for the bucket, logs to a separate locked bucket |
| Budgets and alarms | Monthly budget alert; alarms on DLQ depth and the SES bounce rate (over 4%) |

Domains (MVP.md 0 row 38):

| Name | Use |
|---|---|
| `doculinkapp.com`, `www` redirects to it | Production app and site (Vercel) |
| `staging.doculinkapp.com` | Staging (Vercel), `noindex` |
| `mail.doculinkapp.com` | Production SES identity and custom MAIL FROM; sender `requests@mail.doculinkapp.com` |
| `mail.staging.doculinkapp.com` | Staging SES identity |

Mailboxes `hello@`, `security@` and `abuse@doculinkapp.com` are needed before launch (SPEC-site). They're ordinary inboxes at the email host, not SES.

Production and staging live in separate AWS accounts. Vercel production deploys use the production role; preview and staging deploys use the staging role. The DNS records for SES are output by Terraform and applied at the DNS host.

Provisioning checklist (not Terraform, recorded in `infra/README.md`):
- **Neon:** a project in `aws-us-east-1` with point-in-time restore. Branch `main` is production, branch `staging` is staging, and preview deploys get a branch each via the Vercel integration.
- **Upstash:** one Redis and one QStash per environment.
- **Stripe:** a live account and a test-mode account, with Tax on.
- **Others:** a Sentry project, a PostHog project, a Turnstile site.
- **Vercel Pro project:** region `iad1`, Fluid compute on, OIDC on (team issuer).

## SOC 2 readiness controls (MVP.md 10)

These run from the first commit, so their evidence exists whenever an audit starts. They work for a solo developer today and switch to team rules with one setting when the second engineer joins. Where evidence is a document, it lives in `docs/compliance/` (the control list and where each control's evidence is) and `docs/ops/` (runbooks).

| Control | How | Evidence |
|---|---|---|
| Change management | GitHub organization. `main` is protected: pull request required; required checks `ci`, `codeql`, `gitleaks`, `terraform`; no force pushes or deletion; rules apply to admins. Production deploys only from the Release workflow. **While there's one engineer**, GitHub can't require an approval (authors can't approve their own PRs), so each PR uses the template checklist (what changed, how it was tested, security impact) and the owner signs off the week's merged PRs in `docs/compliance/change-reviews/{yyyy-ww}.md`. **When a second engineer joins**, turn on "1 approval from a code owner who isn't the author" and "dismiss stale approvals"; the weekly sign-off then stops | PR history; weekly sign-offs; branch protection settings export (quarterly) |
| Emergency changes | An admin may bypass the required checks only for a production incident. The PR is labeled `emergency` and is reviewed within 2 business days (in the weekly sign-off while solo) | PRs with the label, each with a later review |
| Vulnerability management | Dependabot security updates on, weekly grouped version updates; `pnpm audit --audit-level high` in `ci`; CodeQL (JavaScript/TypeScript) on every PR and weekly; `gitleaks`; `checkov` and `tflint` on `infra/`. Fix critical findings within 7 days and high within 30 | Dependabot and CodeQL alert history, with close dates |
| Application logs | Axiom through its Vercel integration. Free Personal plan (30-day retention) during the build; Axiom Cloud with 12-month retention from the closed beta in mid-December, when real client data arrives. Logs are JSON with a request id and follow the same scrubbing rules as Sentry: no document contents, file names, tokens or client emails | Log store retention setting; a sample query |
| Infrastructure logs | CloudTrail (management and S3 data events) in a separate locked bucket, kept 1 year, with Object Lock | Terraform; bucket lifecycle |
| Monitoring and alerting | Sentry issue alerts and Cron monitors (existing); Sentry Uptime monitors on `/api/health`, `/sign-in` and an invalid portal link, every minute; AWS alarms (existing). Alerts go to the on-call channel | Alert rules; incident log |
| Staff access to AWS | IAM Identity Center with MFA required. Permission sets: `ReadOnly` (both accounts), `Developer` (staging only), `ProductionAdmin` (production, 1-hour sessions). No IAM users; `pnpm infra:check-users` fails if any exist | Identity Center assignments; CloudTrail sign-ins |
| Staff access to vendors | GitHub, Vercel, Neon, Upstash, Stripe, Sentry, PostHog, Cloudflare and the email host: named company accounts, MFA enforced at the organization level, SSO where the plan allows, roles by need. No shared logins | Quarterly access review |
| Production data | No admin panel or staff path to customer files in the app. Production database reads use a Neon read-only role, only for an incident or a customer's support request, each recorded as a GitHub issue labeled `prod-access` (who, when, why, what was read). Production data is never copied to staging, previews or laptops; those use seeds and factories | `prod-access` issues; Neon audit of role use |
| Secrets | Runtime secrets only in Vercel environment variables, separate per environment. Staff credentials in the company password manager. `docs/ops/secret-rotation.md` covers every secret, including the `TOKEN_ENCRYPTION_KEY` re-encryption | Rotation records |
| Incident response | `docs/ops/incident-response.md`: severity levels, roles, the first-hour checklist, customer notification without undue delay (so customers can meet their own deadlines, such as the FTC Safeguards Rule's 30 days for 500 or more consumers), and a post-incident review | Incident log; reviews |
| Backup and recovery | Neon point-in-time restore; S3 versioning. A restore drill on staging before launch and every 6 months | Drill records in `docs/ops/` |
| Access reviews | Quarterly: list every person's access in each system above, remove what isn't needed, sign off | `docs/compliance/access-reviews/{yyyy-qn}.md` |

People controls (background checks, agreements, training, managed laptops, offboarding within one business day) are company processes in MVP.md 10, not code. This module only provides the checklists in `docs/compliance/`.

## Data

Foundation owns the schema files and migrations for every MVP.md 7 table and enum. Feature modules own the behavior on their tables (listed in their specs) and request schema changes through their specs.

The first migration (`0000_init.sql`) contains:
- all tables, enums and indexes in MVP.md 7, plus the Better Auth tables generated by `@better-auth/cli generate`
- the append-only trigger on `activity_events`, with the bypass setting described below
- the column additions proposed in the module specs, once approved: `workspaces.onboarded_at`, `next_request_number`, `past_due_since` and `deleting_at`; `document_requests.number`; `checklist_items.reason`; `zip_exports.file_set_hash`

Migration rules:
- Migrations are generated with `drizzle-kit generate` and committed. Never use `push` against a shared database.
- **Staging:** the `migrate.yml` GitHub Action runs `pnpm db:migrate` against the staging branch on merge to `main`, before the staging deploy is promoted.
- **Production:** migrations run on a manual "Release" workflow with approval.
- Migrations must be backward compatible with the previous deploy (expand, then contract).

## Code Style

```ts
// src/server/review/service.ts — the shape every use case follows
export async function sendChangeRequests(ctx: WorkspaceCtx, requestId: string) {
  return withTx(async (tx) => {
    const request = await getRequestForUpdate(tx, ctx.workspace.id, requestId);   // scoped; NOT_FOUND otherwise
    const drafted = await listDraftedChanges(tx, request.id);
    if (drafted.length === 0) throw new AppError("CONFLICT", "There are no change requests to send.");
    const next = requestRules.transition(request.status, "SEND_CHANGES");         // pure; throws CONFLICT
    const sentAt = clock.now();
    await markItemsRejected(tx, drafted, sentAt);
    await updateRequestStatus(tx, request.id, { from: request.status, to: next }); // WHERE status = from
    await requests.resumeReminders(tx, request.id, sentAt);
    await logEvent(tx, { requestId: request.id, actor: "requester", type: "change_requests_sent", meta: { itemIds: drafted.map((d) => d.id) } });
    tx.afterCommit(() => email.send({ kind: "change_request", requestId: request.id, idempotencyKey: `cr:${request.id}:${sentAt.toISOString()}` }));
    tx.afterCommit(() => track("change_requests_sent", { distinctId: ctx.user.id, count: drafted.length }));
    return { sent: drafted.length };
  });
}
```

Conventions:
- **Naming.** Files are kebab-case. Functions are verbs in camelCase. Zod schemas are `<thing>Schema` with types inferred via `z.infer`. Database columns are snake_case and TypeScript fields camelCase (Drizzle `casing: "snake_case"`).
- **Time.** No `Date.now()` or `new Date()` in business code; use `clock.now()`, so tests can freeze time.
- **No `any`.** `unknown` plus zod at every boundary.
- **Pure rules.** `rules.ts` functions are pure and exhaustively unit tested. Services stay thin over them.
- **Comments** explain why, not what.

## Testing Strategy

| Level | Tool | What | Location |
|---|---|---|---|
| Unit | Vitest (`unit` project, node) | Pure rules, crypto round-trips, the error mapping in `defineAction` and `defineRoute`, the env schema, CSP builder, rate-limit policy table | `src/**/*.test.ts` |
| Integration | Vitest (`integration` project) with Postgres 17 in Docker | Migrations apply cleanly from empty; the append-only trigger blocks UPDATE/DELETE and allows the bypass; `withTx` rollback skips `afterCommit`; services against a real database with fake Redis, QStash and clock; AWS calls via `aws-sdk-client-mock` | `src/**/*.int.test.ts`, helpers in `tests/` |
| Contract | Vitest | `createAwsEventsRoute` with recorded EventBridge payloads (GuardDuty and SES samples): secret check, dedupe, dispatch | `src/server/aws/*.int.test.ts` |
| E2E | Playwright against staging | Health check, security headers present, portal headers, CSP has no violations on key pages | `e2e/foundation/*.spec.ts` |
| Infra | `terraform validate`, `tflint`, `checkov` in CI | Public access blocked, TLS-only policy, encryption, no wildcard IAM | `infra/` |

Factories in `tests/factories.ts` build workspaces, users, requests, items and files. The integration database is truncated between test files, not between tests, and each test file runs in its own schema for parallelism.

CI (`ci.yml`) runs on every PR: install → lint → typecheck → unit → integration (Postgres service container) → build → Playwright smoke against the Vercel preview. The PR can't merge unless all of these are green.

MVP.md 14 acceptance criteria owned here:

| Criterion | Test |
|---|---|
| No AWS access keys exist in Vercel or the repository; the app works with OIDC credentials only | CI secret scan (`gitleaks`); a script lists Vercel env var names and fails on `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY`; staging E2E upload succeeds |
| No S3 object is reachable without a signed URL | Staging E2E: unsigned GET of a known key returns 403; HTTP (non-TLS) request denied; `checkov` policy checks |
| Sentry receives errors from both apps; PostHog replays show no document content | Staging: a test-only route throws in each app and Sentry receives it; manual replay review in M5 (checklist in the implementation doc) |

## Edge Cases

- **OIDC token unavailable** (local dev, misconfigured project) → the AWS client throws `UPSTREAM` at first use, with a clear message naming the missing piece. It never falls back to keys.
- **Neon cold start or a connection error** → `withTx` retries once on a connection error before the first statement. It never retries after a write.
- **Serialization failure (40001) or deadlock (40P01)** → retried once with jitter. After that, `CONFLICT` with "Someone else changed this. Reload and try again."
- **QStash delivers twice** → handlers are idempotent through the conditional updates and dedupe keys described in each module.
- **EventBridge retries a webhook after a partial failure** → the dedupe key is released on failure, and the handler is idempotent anyway.
- **Clock skew between Vercel and Neon** → business time comes from `clock.now()` in the app. Database defaults (`now()`) are used only for `created_at`.
- **Large JSON bodies on portal routes** → `maxBodyBytes` defaults to 16 KB and returns 413 above that.
- **Requests from preview deployments** → previews use the staging AWS role, staging Stripe and staging Neon branches. They can never reach production data.

## Boundaries

- **Always:**
  - validate every external input with zod
  - scope every query by workspace
  - use `withTx` for multi-statement writes and `afterCommit` for side effects
  - log audit events for every MVP.md 7 event type
  - run lint, typecheck and tests before commits
- **Ask first:**
  - schema changes after the first migration
  - adding dependencies
  - changing CI, Terraform, IAM permissions or the CSP
  - changing a rate-limit policy
  - adding a new external service
  - changing branch protection or required checks
- **Never:**
  - AWS access keys in any environment
  - `drizzle-kit push` against staging or production
  - logging document contents, file names, tokens or client emails to Sentry, PostHog or the console
  - disabling the append-only trigger outside account deletion
  - wildcard IAM actions or resources
  - running `infra:apply` against production from a laptop
  - merging to `main` without passing checks (or, once there are two engineers, without review), except a labeled emergency change
  - copying production data out of production, or reading it without a `prod-access` record
  - shared logins or IAM users

## Success Criteria

- [ ] `pnpm dev`, `pnpm test`, `pnpm test:int`, `pnpm lint`, `pnpm typecheck` and `pnpm build` all pass on a fresh clone with only `.env.example` filled in for local services.
- [ ] Staging is deployed from `main`, and production from the Release workflow. Both use OIDC with no AWS keys present.
- [ ] Terraform applies cleanly to both accounts. `checkov` reports no high findings.
- [ ] The first migration creates every MVP.md 7 table. The append-only trigger is proven by an integration test.
- [ ] `defineAction`, `defineRoute`, `withTx`/`afterCommit`, `logEvent`, `rateLimit`, `publishJob`/`createJobRoute`, `createAwsEventsRoute` and the crypto helpers exist with unit or integration tests, and an example use in the health route or tests.
- [ ] The lint boundary rule fails when a module imports another module's non-index file or a module it doesn't depend on.
- [ ] Security headers and the CSP are present on staging (E2E), with no CSP violations reported on sign-in and portal pages.
- [ ] Sentry receives a test error from the server and the browser with scrubbed fields. PostHog receives a typed test event.
- [ ] SES production access has been requested (day 1), and SPF, DKIM and DMARC records verify.
- [ ] Branch protection, CODEOWNERS, Dependabot and CodeQL are on, and a test PR with a failing check can't merge.
- [ ] Production logs arrive in Axiom (12-month retention from the beta). Sentry Uptime monitors are green.
- [ ] Staff reach AWS only through IAM Identity Center with MFA, and `infra:check-users` passes in both accounts.
- [ ] `docs/compliance/` and `docs/ops/` hold the control list, the incident response runbook and the access review template.

## MVP.md changes (applied in v1.2, MVP.md 0 rows 38 and 45)

1. **MVP.md 10, AWS access.** The app role also needs `s3:GetObjectVersion`, `DeleteObjectVersion`, `ListBucketVersions`, `GetObjectTagging` and `AbortMultipartUpload`. Without them, "Delete files" and retention can't remove all versions, the stuck-scan check can't read the GuardDuty tag, and failed background ZIPs can't be cleaned up.
2. **MVP.md 6.1.** The `quarantine/{requestId}/{fileId}` key prefix (30-day lifecycle, not scanned) used in 8.4.
3. **MVP.md 7, append-only trigger.** Deletes are allowed only when the transaction sets `app.allow_event_delete` (account deletion and Delete draft). A Postgres trigger also fires on cascaded deletes, so without a bypass both would fail.
4. **MVP.md 6.** Next.js 16 renames middleware to `proxy.ts`, and it runs on Node.js.
5. **MVP.md 6, 6.2, 13.** Terraform for the AWS stack; the domain `doculinkapp.com`; `EMAIL_LINK_SECRET`.
6. **MVP.md 10, 13, 14 (row 46).** SOC 2 readiness controls from day one.

## Decisions (Oct 20, 2026)

1. **Infrastructure as code:** Terraform.
2. **Domain:** `doculinkapp.com`, with email from `mail.doculinkapp.com`.
3. **Local AWS:** developers use a sandbox role in the staging account (AWS SSO) for S3 and SES. No LocalStack.
4. **SOC 2:** build the controls from day one; the audit comes later (MVP.md 0 row 46). While solo, a weekly sign-off stands in for a second reviewer.
5. **Log store:** Axiom. Free during the build; Axiom Cloud ($25 a month platform fee plus usage, per its pricing page) with 12-month retention from the beta. It has a native Vercel integration and its own SOC 2 Type II report. Chosen over our own AWS setup because it's about an hour of work instead of about 2 days, and the 7-week plan has no slack.

## Open Questions

None.
