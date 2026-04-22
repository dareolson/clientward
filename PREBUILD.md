# Clientward — Pre-Build Plan

Everything that must exist before the first line of feature code is written. Linear order. Each phase must be complete before the next begins.

The discipline here is the point. A senior engineer does not open a code editor until the foundation is correct. Retrofitting any of this is either expensive, painful, or impossible.

---

## Phase 0 — Business and Legal Entity

Nothing else starts until the business exists as a legal entity. You cannot open a real Stripe account, sign customer contracts, or have clean IP ownership without this.

- [ ] Form LLC (registered in your state or Wyoming/Delaware for flexibility)
- [ ] Get EIN (free, IRS.gov, takes 10 minutes)
- [ ] Open business bank account (separate from personal — required for clean financials in due diligence)
- [ ] Purchase domain for Clientward
- [ ] Set up business email (Google Workspace or similar — not a personal Gmail)
- [ ] Draft Terms of Service (use a lawyer or a reputable template service like Bonterms or Clerky — do not wing this)
- [ ] Draft Privacy Policy (must be GDPR and CCPA compliant — covers EU and California users)
- [ ] Draft Data Processing Agreement (DPA) template — required for enterprise customers and GDPR
- [ ] Draft IP Assignment Agreement template — for any contractor who ever writes code

**Do not proceed to Phase 1 until these are done.** Legal hygiene after customers exist is a much harder problem.

---

## Phase 1 — Architecture Decisions (Write, Don't Build)

Before touching infrastructure, make and document all major decisions. A decision made in writing is a decision you can defend to an acquirer, a customer, or a new team member.

- [ ] Document the stack with rationale (why Next.js, why Supabase, why Vercel — not just what)
- [ ] Define environment strategy: local → staging → production. Three separate environments, no exceptions.
- [ ] Define branching strategy: `main` is always deployable, feature branches only, PRs required to merge, no direct pushes to main
- [ ] Define data classification tiers (Public / Internal / Confidential / Restricted) and assign expected classifications to anticipated data types
- [ ] Define subscription tiers on paper before Stripe: what each tier includes, pricing, feature gates, upgrade path
- [ ] Draw the data model on paper before writing any schema: teams, users, roles, permissions, and the core resource type(s) Clientward manages. Get this right before it's in a database.
- [ ] Document the RBAC permission matrix: which roles exist, what each can do, on which resource types
- [ ] Decision on audit log retention policy: how long are logs kept? (SOC 2 expects minimum 1 year)

---

## Phase 2 — Repository and CI/CD Foundation

- [ ] Create GitHub repo (private until launch)
- [ ] Add `.gitignore` — verify `.env*` is in it before the first commit. Non-negotiable.
- [ ] Add `README.md` — placeholder is fine, but the file must exist
- [ ] Commit `BUILDDOC.md` and `PREBUILD.md` to repo
- [ ] Set up GitHub branch protection on `main`:
  - Require pull request before merging
  - Require at least 1 approval
  - Require status checks to pass (lint, type check)
  - No force pushes
- [ ] Set up GitHub Actions workflow: runs on every PR
  - TypeScript type check (`tsc --noEmit`)
  - ESLint
  - (Later: tests)
- [ ] Configure Vercel project:
  - Connect to GitHub repo
  - Set up three environments: Production (main), Preview (PRs), Development
  - Configure custom domain
  - Enable Vercel Analytics
- [ ] Configure Sentry project — get DSN, do not write app code without error monitoring in place

---

## Phase 3 — Secrets and Environment Variables

- [ ] Decide on secrets management approach: Vercel env vars for now, Doppler when team grows
- [ ] Document every environment variable the app will need before writing code that uses them:
  - Supabase URL + anon key + service role key
  - Stripe publishable key + secret key + webhook secret
  - Sentry DSN
  - Any email/SMTP credentials
  - App URL per environment
- [ ] Create `.env.local.example` file in repo — lists every variable name with empty values and a comment explaining each. This is what a new developer copies to get started.
- [ ] Verify `.env.local` is gitignored before adding any real values

---

## Phase 4 — Supabase Foundation

Two separate Supabase projects: one for development, one for production. They must never share data.

- [ ] Create Supabase project: `clientward-dev`
- [ ] Create Supabase project: `clientward-prod`
- [ ] Configure auth settings on both:
  - Email confirmation enabled
  - Password minimum length (12+ characters)
  - MFA available (Supabase supports TOTP)
  - Allowed redirect URLs locked down to your domains
- [ ] Write base schema as SQL migration files committed to the repo (not just applied in the dashboard):
  - `teams` table
  - `roles` table (seed with system roles: owner, admin, member, viewer)
  - `team_members` table (user ↔ team ↔ role junction)
  - `permissions` table (role ↔ resource_type ↔ action)
- [ ] Enable RLS on every table immediately upon creation — never create a table without enabling it in the same migration
- [ ] Create `audit` schema (separate from `public`)
- [ ] Create `audit.logs` table with INSERT-only RLS policy (no UPDATE, no DELETE, ever)
- [ ] Write a local seed script: creates a dev team, dev users with different roles, sample data — so any developer can get a realistic local environment in one command
- [ ] Commit all migration files to repo under `/supabase/migrations/`

---

## Phase 5 — Auth and RBAC Implementation

This is the first actual code. It is not a feature — it is the scaffolding everything else runs on.

- [ ] Scaffold Next.js app (TypeScript, Tailwind, App Router)
- [ ] Install and configure Supabase client libraries
- [ ] Implement `src/proxy.ts` (auth middleware):
  - Refresh session on every request
  - Redirect unauthenticated users to `/login`
  - Protect all routes under `/dashboard/*` and `/app/*`
- [ ] Implement `/auth/callback/route.ts` — handles email confirmation redirect, exchanges code for session
- [ ] Implement `/login` page — email/password, signup, "check your email" state
- [ ] Implement RBAC utility function: `can(user, action, resourceType, teamId): boolean`
  - Queries `team_members` + `permissions` tables
  - Returns false by default (deny by default, never allow by default)
  - Must be callable from both server components and API routes
- [ ] Implement `usePermission()` client hook — wraps `can()` for component-level gating
- [ ] Implement team creation flow — first user who signs up creates a team and is assigned the owner role
- [ ] Implement team invite flow — owner/admin can invite by email, generates a pending invite record, sends email
- [ ] Write tests for `can()` — cover: owner can do everything, viewer cannot mutate, cross-team access returns false

---

## Phase 6 — Audit Logging

Must be in place before any feature that touches customer data.

- [ ] Implement `logAction(params)` server-side utility:
  - Inserts to `audit.logs`
  - Required params: `actorId`, `action`, `resourceType`
  - Optional params: `resourceId`, `teamId`, `beforeState`, `afterState`, `metadata`
  - Masks PII fields in `beforeState`/`afterState` before writing (define a `MASKED_FIELDS` constant: passwords, tokens, SSNs, payment data)
  - Never throws — log failures must not break the app, but must alert via Sentry
- [ ] Integrate `logAction()` into auth hooks (login, logout, password change)
- [ ] Document the contract: every server action and API route that mutates data must call `logAction()` — this is a code review requirement, not optional

---

## Phase 7 — Stripe and Billing Foundation

- [ ] Create Stripe account under the business entity (not personal)
- [ ] Create Products and Prices in Stripe matching the tier decisions from Phase 1
- [ ] Add `stripe_customer_id` and `subscription_status` and `plan_tier` columns to `teams` table
- [ ] Implement Stripe webhook handler (`/api/webhooks/stripe/route.ts`):
  - Verify webhook signature on every request
  - Handle: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
  - Update `teams` subscription status on every relevant event
- [ ] Implement checkout flow — team owner can subscribe, upgrades, downgrades
- [ ] Implement Stripe Customer Portal — self-serve billing management, cancel, update payment method
- [ ] Implement feature gating: `canAccess(team, feature)` utility that checks `plan_tier` — every premium feature calls this

---

## Phase 8 — Security Baseline

- [ ] Configure HTTP security headers in `next.config.ts`:
  - `Content-Security-Policy`
  - `X-Frame-Options: DENY`
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
  - `Permissions-Policy`
- [ ] Configure CORS — API routes only accept requests from your own domains
- [ ] Implement rate limiting on auth routes (`/api/auth/*`) — use Upstash Redis + `@upstash/ratelimit` or similar
- [ ] Install and configure input validation library (Zod) — every API route and server action validates input against a schema before touching the database
- [ ] Enable Dependabot on GitHub repo — automated dependency vulnerability alerts
- [ ] Verify service role key is never used in any client-side code — grep for it in CI as a check
- [ ] Configure Sentry: capture unhandled errors and unhandled promise rejections, set up alert rules for error spikes

---

## Phase 9 — Observability

- [ ] Sentry SDK integrated and tested — trigger a test error, confirm it appears in dashboard
- [ ] Set up uptime monitoring (BetterUptime, Checkly, or similar) — monitors `/api/health` endpoint
- [ ] Implement `/api/health` route — returns `{ status: 'ok', timestamp }`, checks Supabase connectivity
- [ ] Set up Vercel Analytics (already available, just needs to be enabled)
- [ ] Set up Stripe revenue tracking via Baremetrics or ChartMogul — connects to Stripe, tracks MRR, churn, LTV from day one
- [ ] Configure alerting: Sentry alerts on error rate spikes, uptime alerts on downtime, Stripe alerts on payment failures

---

## Phase 10 — Documentation Before Launch

- [ ] `README.md` complete: what Clientward is, local setup in under 30 minutes, environment variable list, how to run migrations, how to seed
- [ ] Architecture diagram committed to repo (`/docs/architecture.png` or similar) — one visual of Next.js → Supabase → Vercel, auth flow, data flow
- [ ] `BUILDDOC.md` up to date
- [ ] Runbook committed to repo (`/docs/runbook.md`): how to deploy, how to roll back, what to do when Supabase is down, how to handle a billing failure
- [ ] Supabase schema documented (`/supabase/README.md`): what each table is for, what RLS policies exist, how migrations work

---

## Phase 11 — Legal Goes Live

Before any user touches the product:

- [ ] Terms of Service live at `/terms`
- [ ] Privacy Policy live at `/privacy`
- [ ] Cookie consent banner if using any analytics that set cookies (Vercel Analytics is cookieless — no banner needed)
- [ ] User must check "I agree to Terms of Service" on signup — log the version they agreed to and the timestamp in the `users` table
- [ ] DPA template ready to send — enterprise customers will ask for it before signing

---

## Checkpoint: Ready to Write Feature Code

Before writing the first feature:

| Item | Status |
|---|---|
| LLC formed, EIN obtained | |
| Business bank account open | |
| ToS, Privacy Policy, DPA drafted | |
| GitHub repo with branch protection | |
| CI/CD pipeline running (lint, type check) | |
| Vercel: dev + staging + production environments | |
| Sentry integrated and tested | |
| Supabase: dev + prod projects, base schema, RLS on all tables | |
| audit.logs table, INSERT-only | |
| Auth flow: login, signup, email confirm, session middleware | |
| RBAC: `can()` utility + `usePermission()` hook, tested | |
| `logAction()` utility implemented and integrated into auth | |
| Stripe: products/prices created, webhook handler, feature gating | |
| Security headers, rate limiting, input validation (Zod) | |
| Uptime monitoring + health endpoint | |
| README: local setup in under 30 minutes | |
| ToS and Privacy Policy live | |

Every row checked = first feature can be built.

---

*Last updated: 2026-04-22*
