# Clientward — Build Documentation

This document tracks foundational architectural decisions that must be in place **from day one**. These are not features — they are structural requirements. Retrofitting them later is extremely costly and, in some cases (audit logs, data isolation), effectively impossible without a full data migration.

Lesson learned from Rate Guide: do not build first and think about this stuff later.

---

## 1. Role-Based Access Control (RBAC)

### What it is
Every action in the app — reading a record, writing to a field, deleting an entry, accessing a report — must be gated by a permission check. Permissions are granted through roles. Roles are assigned to users within a team context.

### Why team + data type + action scoping matters
A flat "admin / member / viewer" model breaks quickly at enterprise scale. Users may be an admin of one team and a viewer on another. They may be allowed to read invoices but not contracts. The permission model must express:

- **Who** — the user, through their role
- **In what context** — the team they belong to
- **On what kind of data** — the resource type (e.g., `invoice`, `project`, `client`)
- **Doing what** — the action (`read`, `create`, `update`, `delete`, or custom like `export`, `approve`)

### Data model

```sql
-- Core tables
teams (id, name, slug, created_at)
roles (id, name, description, is_system)  -- e.g., "owner", "admin", "member", "viewer"
team_members (id, team_id, user_id, role_id, created_at)

-- Granular permissions (optional: used when roles alone aren't enough)
permissions (id, role_id, resource_type, action, created_at)
-- resource_type: 'invoice' | 'project' | 'client' | '*'
-- action: 'read' | 'create' | 'update' | 'delete' | 'export' | '*'
```

### Enforcement layers
1. **Row Level Security (RLS) in Postgres/Supabase** — database enforces tenant and team scoping at the query level, not just app logic
2. **Middleware check** — before any server action or API route resolves, verify the user has the required permission for the resource type and action
3. **UI layer** — hide/disable buttons and routes the user can't access (UX, not security — never rely on this alone)

### Key rules
- Users have roles *within a team*, not globally
- A user can belong to multiple teams with different roles on each
- Permission checks must be composable: `can(user, 'create', 'invoice', teamId)`
- Build a `usePermission()` hook on the frontend early so every component can gate itself without custom logic

---

## 2. Audit Logs

### What it is
Every meaningful action in the system — login, data access, record creation, update, deletion, permission change, export — must be logged with full context: who did it, from where, to what, and when. These logs are immutable and queryable.

### Why this is non-negotiable
- **SOC 2 Type II requirement** — auditors will ask to see these
- **Customer trust** — enterprise buyers will ask "can I see who accessed my data and when?" before signing
- **Incident response** — when something goes wrong, these logs are how you find out what happened
- **Legal/compliance** — in some industries (healthcare, finance), these are legally required

### What to log

Every audit entry should capture:

| Field | Description |
|---|---|
| `id` | UUID |
| `timestamp` | UTC timestamp, immutable |
| `actor_id` | User who performed the action |
| `actor_ip` | IP address |
| `actor_user_agent` | Browser/client string |
| `team_id` | Team context of the action |
| `action` | What was done: `login`, `create`, `update`, `delete`, `export`, `view`, `permission_change` |
| `resource_type` | What kind of thing was acted on: `invoice`, `user`, `project`, etc. |
| `resource_id` | The specific record ID |
| `before_state` | JSON snapshot of the record before (for updates/deletes) |
| `after_state` | JSON snapshot of the record after (for creates/updates) |
| `metadata` | Freeform JSON for extra context |

### Immutability requirements
- **No UPDATE or DELETE on audit_logs, ever.** Enforce this at the database level with RLS policies that allow `INSERT` only.
- Store in a **separate schema or separate database** from app data — this makes it easier to prove immutability to auditors and prevents accidental access via app queries
- Consider append-only log storage services for long-term archival (AWS CloudTrail, Datadog Logs, Papertrail) in addition to database logs

### Data model

```sql
-- In a separate schema: audit.*
CREATE TABLE audit.logs (
  id           UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  timestamp    TIMESTAMPTZ NOT NULL DEFAULT now(),
  actor_id     UUID REFERENCES auth.users(id),
  actor_ip     INET,
  actor_agent  TEXT,
  team_id      UUID REFERENCES public.teams(id),
  action       TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id  UUID,
  before_state JSONB,
  after_state  JSONB,
  metadata     JSONB
);

-- Deny all mutations
CREATE POLICY "no_update" ON audit.logs FOR UPDATE USING (false);
CREATE POLICY "no_delete" ON audit.logs FOR DELETE USING (false);

-- Useful indexes
CREATE INDEX ON audit.logs (actor_id, timestamp DESC);
CREATE INDEX ON audit.logs (team_id, timestamp DESC);
CREATE INDEX ON audit.logs (resource_type, resource_id);
```

### Implementation pattern
Write a single `logAction()` server-side utility that every mutation in the app calls. Never log from the client side. Log from:
- Server actions
- API routes
- Background jobs
- Auth hooks (Supabase auth hooks for login/logout)

---

## 3. Data Isolation (Multi-Tenancy)

### What it is
When multiple customers (teams/orgs) share the same application, their data must be structurally isolated from each other. One tenant must never be able to access, see, or influence another tenant's data — not through bugs, not through API manipulation, not by guessing IDs.

### Why architecture-level matters
App-level filtering (adding `WHERE team_id = $1` to queries) is not enough on its own. A single missed filter, a new developer, a rushed feature — and tenant data leaks. Architecture-level isolation means the database itself enforces the boundaries.

### Three models — tradeoffs

| Model | Isolation | Cost | Complexity | Best For |
|---|---|---|---|---|
| Separate database per tenant | Strongest | High | High | Regulated industries, very high-value contracts |
| Separate schema per tenant | Strong | Medium | Medium | Mid-market SaaS, SOC 2 targets |
| Shared schema + RLS | Good | Low | Low-medium | Startup/growth stage, Supabase-native |

### Recommended approach for Clientward: Shared Schema + Postgres RLS

Given the Rate Guide stack (Supabase/Next.js), the path of least resistance that still meets enterprise requirements is **row-level security enforced at the database level**.

Every table that holds tenant data gets:
1. A `team_id` column (foreign key to `teams`)
2. An RLS policy that restricts reads and writes to the current user's team

```sql
-- On every tenant-scoped table:
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY "team_isolation" ON invoices
  USING (team_id = (
    SELECT team_id FROM team_members
    WHERE user_id = auth.uid()
    LIMIT 1
  ));
```

With Supabase this runs at the Postgres level — the app cannot return data from another team even if the query omits the filter.

### Non-negotiables
- **Every new table** that holds customer data must have `team_id` and RLS enabled from day one — do not create a table without it
- **Service role key** (which bypasses RLS) must never be used in client-facing code — only in trusted server-side contexts
- **No sequential integer IDs** on tenant-visible records — use UUIDs to prevent enumeration attacks
- **Cross-tenant admin views** (e.g., internal dashboards) must be explicitly scoped and logged as audit events

### If Clientward grows to need stronger isolation
If a customer is large enough / regulated enough to require schema-per-tenant:
- Supabase supports multiple schemas — all app tables move to `tenant_{slug}.*`
- Each schema gets its own RLS + access controls
- The migration from shared-schema to schema-per-tenant is painful but doable if `team_id` is consistently applied from the start

This is why applying `team_id` correctly from day one matters even if we start shared-schema — it's the migration path.

---

## 4. Security Foundations (SOC 2 — Architectural)

These three items belong in the plan from day one because they affect how you write code and design the schema. Getting them wrong is either a migration or a security incident.

### Encryption

**At rest:** Supabase encrypts all data at rest by default at the infrastructure level. That covers you for storage — but you're still responsible for not undermining it. Never store sensitive fields (SSNs, payment details, health data) in plaintext columns. If Clientward ever touches regulated data, field-level encryption on specific columns may be required on top of disk-level encryption.

**In transit:** HTTPS everywhere, enforced at the platform level (Vercel + Supabase handle this). Never allow HTTP fallback. Never pass sensitive data as URL query parameters — they end up in logs.

**In logs:** Never log PII in plaintext. This means audit log `before_state` / `after_state` fields must mask or exclude sensitive fields (passwords, tokens, payment info, SSNs). Build the masking into `logAction()` from the start.

### Secrets Management

API keys, service credentials, database URLs, and tokens must never appear in source code or git history. One committed secret invalidates your SOC 2 audit.

Rules:
- All secrets live in environment variables (Vercel env vars for production, `.env.local` locally)
- `.env*` files are in `.gitignore` — verify this before the first commit
- Use a secrets manager (Doppler, Infisical, or AWS Secrets Manager) as you scale past solo development
- Rotate any key that is ever accidentally committed — do not just delete the commit, treat the key as compromised
- Separate keys per environment: dev, staging, production never share credentials

### Data Classification

Before writing schema, decide which fields are sensitive. This drives encryption decisions, log masking rules, export controls, and eventually your data processing agreement with customers.

A simple starting framework:

| Class | Examples | Treatment |
|---|---|---|
| Public | Team name, plan tier, created_at | No restrictions |
| Internal | Usage metrics, feature flags | Internal access only |
| Confidential | User emails, names, billing address | Masked in logs, access-logged |
| Restricted | Payment data, SSNs, health records | Field-level encryption, strict audit, may require HIPAA/PCI |

For most early Clientward data (team info, project data, rates, contacts), Confidential is the right classification. Build your logging and export features with that assumption.

---

## 5. SOC 2 — Operational Requirements (Add As You Go)

These are real SOC 2 requirements but they are process and policy based, not architectural. They do not affect how you write code from day one. Add them when you are actively pursuing certification or when an enterprise customer asks for them.

**Access controls:**
- MFA enforced for all internal admin access (Supabase dashboard, Vercel, GitHub, DNS)
- Access reviews — quarterly review of who has access to what, revoke anything stale
- Principle of least privilege — engineers get the minimum access needed, no shared root credentials

**Policies (written documents):**
- Security policy — how you handle access, incidents, and vulnerabilities
- Incident response plan — what you do when something goes wrong, who gets notified, how you communicate to customers
- Change management policy — how code gets reviewed and deployed (PR review, no direct pushes to main)
- Business continuity / disaster recovery — what happens if Supabase or Vercel goes down

**Vendor management:**
- Verify that your critical vendors have their own SOC 2: Supabase (yes), Vercel (yes), Resend (check), any others
- Keep a vendor list — auditors will ask for it

**Vulnerability management:**
- Dependency scanning (GitHub Dependabot covers this at no cost)
- Penetration testing before launch to any regulated industry customer
- Process for patching critical CVEs within a defined window (e.g., 72 hours for critical)

**Monitoring and alerting:**
- Error monitoring (Sentry or similar) so you know when something breaks before customers do
- Uptime monitoring with alerting
- Log retention policy — how long do you keep audit logs? (SOC 2 typically expects 1 year minimum)

**When to actually pursue certification:**
SOC 2 Type I (point-in-time snapshot) takes ~3 months and costs $10–30k with an auditor. Type II (6-month observation period) is what enterprise buyers actually want. Don't pursue it until you have a prospect who is requiring it — use that deal as the forcing function and cost-justify it against the contract value.

---

## Summary: What Must Exist Before Writing Feature Code

| Requirement | Must Have Before... |
|---|---|
| RBAC schema + enforcement | Any protected route or user-facing data |
| Audit logging utility | Any mutation that touches customer data |
| RLS on all tenant tables | First table with customer data |
| UUID primary keys everywhere | First table, period |
| Separate audit schema | Before beta / any real customer data |
| Secrets in env vars, .env gitignored | First commit |
| Data classification decision | Schema design |
| No PII in logs (masking in logAction) | Audit logging implementation |

---

## Notes on Rate Guide as Funnel

Rate Guide will eventually serve as the top-of-funnel for Clientward. This means:
- Rate Guide users may eventually become Clientward users — plan for an import/upgrade path
- The `profiles` and `rate_history` tables on Rate Guide are not Clientward data — don't conflate them
- When Rate Guide starts collecting data that will migrate into Clientward, that's the moment to apply Clientward's data model standards retroactively

---

*Last updated: 2026-04-22 — added SOC 2 architectural foundations (encryption, secrets, data classification) and operational requirements*
