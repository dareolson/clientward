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

## Summary: What Must Exist Before Writing Feature Code

| Requirement | Must Have Before... |
|---|---|
| RBAC schema + enforcement | Any protected route or user-facing data |
| Audit logging utility | Any mutation that touches customer data |
| RLS on all tenant tables | First table with customer data |
| UUID primary keys everywhere | First table, period |
| Separate audit schema | Before beta / any real customer data |

---

## Notes on Rate Guide as Funnel

Rate Guide will eventually serve as the top-of-funnel for Clientward. This means:
- Rate Guide users may eventually become Clientward users — plan for an import/upgrade path
- The `profiles` and `rate_history` tables on Rate Guide are not Clientward data — don't conflate them
- When Rate Guide starts collecting data that will migrate into Clientward, that's the moment to apply Clientward's data model standards retroactively

---

*Last updated: 2026-04-22*
