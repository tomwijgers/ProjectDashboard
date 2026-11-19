# Feature Specification: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Status:** Approved v2.0  
**Governed By:** `CONSTITUTION.md`, `DESIGN_SYSTEM.md`

# 1\. Overview & Scope

- **Goal:** Provide a secure, single-user executive web dashboard to monitor Antigravity AI token consumption on a per-project basis (web, mobile, 3D asset, and creative workflows), manage monthly AI subscription costs (including credit top-ups), and apportion those costs across projects proportionally to their token usage using an extensible cost architecture.
- **In Scope (Phase 1 MVP):**  
  - Single-user authentication protecting all dashboard UI and administrative actions.  
  - Project registry: Create and edit projects (name, repo path, status). The project `slug` is the immutable unique identity used to correlate telemetry.  
  - Extensible cost management: Configurable monthly AI subscription cost (editable per month to incorporate credit top-ups), architected via an extensible cost ledger for seamless future additions of fixed and metered costs.  
  - Automated token-weighted cost apportionment: For each billing month, apportion the monthly AI budget across projects proportional to their token consumption using deterministic integer-cent allocation.  
  - Automated AI token ingestion: Local Antigravity Stop lifecycle hook transmitting input, output, and cached prompt tokens with local offline `.jsonl` buffering and idempotent deduplication.  
  - Executive dashboard views: Portfolio summary (total tokens, total AI spend, active projects), project token breakdown matrix (monthly tokens, token share %, monthly apportioned AI cost, lifetime tokens, lifetime apportioned cost), project editing modal, monthly AI price edit dialog, and unassigned telemetry banner.
- **Out of Scope (Deferred to Future Phases):**  
  - **Phase 2 (Business & Product Metrics):** Gross revenue, direct non-AI expenses, user growth metrics (Total Users, MAU), P&L net profit calculations, and non-AI recurring infrastructure services (Vercel, AWS, domains, SaaS).  
  - **Phase 3 (External API Metering):** Standalone script telemetry using direct API keys or Vertex AI endpoints, multi-currency FX.

# 2\. User Flows & Functional Requirements

## 2.1 Single-User Authentication & Access Control

- **FR-001 (Dashboard Authentication):** Unauthenticated requests to any dashboard route must redirect to `/login`. Only the configured single owner can authenticate (via owner fallback password or WebAuthn Passkey) to establish an encrypted session.  
- **FR-002 (Ingestion Endpoint Authorization):** External telemetry ingestion endpoints must reject requests lacking a valid Bearer credential without requiring an interactive browser session.  
- **FR-003 (Session Termination):** The authenticated owner must be able to sign out explicitly, immediately invalidating the active session cookie.

## 2.2 Project Portfolio & Cost Management

- **FR-004 (Project Registry & Metadata Editing):** The owner can create and update projects. The project `slug` is assigned upon creation and is strictly immutable. The owner can edit the project's display **Name**, **Repository Path**, and lifecycle **Status** (`active`, `paused`, `archived`) from the dashboard UI.  
- **FR-005 (Monthly AI Price Management):** The owner can view and edit the total AI cost for any selected calendar month (`YYYY-MM`) directly from the dashboard UI (e.g., updating the default \$33 base subscription to \$53 when a credit pack is purchased).  
- **FR-006 (Token-Weighted Cost Apportionment):** When viewing a billing month, the system calculates each project's total token consumption (`input + output + cache_read`), determines each project's share of total monthly tokens, and apportions the month's AI cost entry proportionally using the largest-remainder method.  
- **FR-007 (Portfolio & Project Financial Views):** The dashboard displays for each project and across the portfolio:  
  - **Selected Month:** Total Tokens Consumed (with breakdown of Input, Output, and Cache Read), Token Share Percentage, and Apportioned AI Cost.  
  - **Lifetime Cumulative:** Cumulative Historical Tokens Consumed and Cumulative Historical Apportioned AI Cost.

## 2.3 Antigravity AI Token Tracking

- **FR-008 (Automated Hook Telemetry):** When an Antigravity agent turn finishes in a workspace, a local Stop hook resolves the project slug (via `.dashboard-project.json` or fallback workspace folder name), extracts incremental input, output, and cached prompt (`cache_read_tokens`) tokens using camelCase parameters (`transcriptPath`), and transmits the usage record to the dashboard.  
- **FR-009 (Idempotent Ingestion & Unassigned Telemetry):** The ingestion endpoint deduplicates submissions using a unique message/event cursor key, resolves the target project by slug, attributes usage to the corresponding `Australia/Melbourne` calendar month, and preserves unrecognized slugs under `unassigned_project_slug` with `project_id = NULL` for retroactive assignment.  
- **FR-010 (Offline Queue & Non-Blocking Execution):** If the dashboard endpoint is unreachable or times out, the local hook persists unsent telemetry to `~/.cache/antigravity-tokens-queue.jsonl`, exits with status `0` without blocking the IDE, and flushes queued records on subsequent runs.

# 3\. Domain Rules & Business Logic

- **BR-001 (Extensible Cost Ledger):** All project costs must be modeled through an extensible cost ledger:  
  - `cost_items`: Cost definitions (for MVP: `'Antigravity AI Subscription'`, type `'fixed_subscription'`, apportionment `'token_weighted'`).  
  - `monthly_cost_entries`: Month-specific cost amounts (e.g., base subscription + credits purchased for month `M`).  
  - `project_cost_allocations`: Calculated project shares persisted per month.  
  This architecture guarantees that future fixed subscriptions (hosting, SaaS) and metered API costs integrate seamlessly without database redesign.  
- **BR-002 (Token-Weighted Apportionment Formula):**  
  When month `M` has total AI cost `C` (in integer cents) and `N` projects with token counts `t_1, t_2, ... t_N` where `T = sum(t_i) > 0`:  
  1. Calculate fractional share for each project: `share_i = t_i / T`.  
  2. Compute base integer cents: `base_cents_i = floor(C * share_i)`.  
  3. Compute fractional remainder: `rem_i = (C * share_i) - base_cents_i`.  
  4. Distribute any remaining cents (`C - sum(base_cents_i)`) one cent at a time to projects with the highest `rem_i` (breaking ties by project slug ascending) so `sum(allocated_cents) == C` exactly.  
- **BR-003 (Zero-Usage Month Handling):** If total token consumption across all projects in month `M` is `0` (`T = 0`), no project is allocated AI cost, and the total monthly cost entry remains in an unallocated pool.  
- **BR-004 (Monthly & Lifetime Rollup Equations):**  
  - `Monthly Project AI Cost = Apportioned Cents from project_cost_allocations for month M`  
  - `Lifetime Project Tokens = Sum(Tokens logged for project across all recorded months)`  
  - `Lifetime Project AI Cost = Sum(Monthly Project AI Cost across all recorded months)`  
- **BR-005 (Closed Month Snapshot Immutability):** Recalculating allocations or updating project metadata affects only the target month; historical closed months remain frozen.

# 4\. Edge Cases & Error States

| Scenario | Trigger Condition | Expected User-Facing Behavior |
| :---- | :---- | :---- |
| **Unknown Project Slug in Hook** | Antigravity hook sends a `projectSlug` not registered in the dashboard | Ingestion API saves record with `project_id = NULL` and `unassigned_project_slug = <slug>`, returning `200 OK` with `unassigned: 1`; dashboard displays an unassigned telemetry alert banner with an action to assign it to a project. |
| **Duplicate Hook Delivery** | Network retry resends an already-ingested cursor/event ID | Ingestion API returns `200 OK` (idempotent no-op) without double-counting tokens. |
| **Zero Tokens in Month** | Viewing a month with no recorded Antigravity turns | Displays `0` tokens cleanly; monthly AI cost shows as unallocated overhead with a prompt to run recalculation once activity occurs. |
| **Credit Top-Up for Month** | Owner buys additional AI credits during an intensive month | Owner clicks "Edit AI Cost" on the month selector and updates the amount (e.g. from \$33 to \$53); recalculation distributes the updated budget according to token usage. |
| **Missing `.dashboard-project.json`** | Workspace lacks project configuration file | Hook resolves slug from the workspace folder name or Git repository root. |

# 5\. Acceptance Criteria (Testable Definitions of Done)

1. **AC-1 (Authentication & Route Security):** Given an unauthenticated browser session, when navigating to `/`, the user is redirected to `/login`; after authenticating with owner credentials, the user accesses the dashboard and can sign out to revoke access.  
2. **AC-2 (Project Registry & Immutability):** Given an existing project, the owner can edit its display name, repo path, and status via the dashboard UI, while the project slug cannot be edited.  
3. **AC-3 (Monthly AI Cost Editing & Apportionment):** Given month `2026-11` with two active projects (`Project A` using 70,000 tokens, `Project B` using 30,000 tokens), when the monthly AI cost is set to `$50.00`, then running apportionment allocates `$35.00` to Project A and `$15.00` to Project B (summing to exactly `$50.00`).  
4. **AC-4 (Antigravity Hook Telemetry):** Given an active Antigravity workspace, when an agent turn completes, then `log_antigravity_tokens.py` posts the token delta (including `cache_read_tokens`) to `/api/ingest/tokens`, the dashboard increments that project's token count, and re-sending the same payload does not create duplicate entries.  
5. **AC-5 (Unassigned Telemetry Resolution):** Given telemetry received for an unregistered slug, the event is saved under `unassigned_project_slug`, an alert banner appears on the dashboard, and assigning that slug to a project attributes all unassigned records to that project.
