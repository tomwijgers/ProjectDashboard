# Feature Specification: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Status:** Draft v1.0  
**Governed By:** `CONSTITUTION.md`, `DESIGN_SYSTEM.md`

# 1\. Overview & Scope

- **Goal:** Provide a secure, single-user web dashboard to monitor all active projects in one place—tracking monthly AI token consumption, user growth (total and monthly active users), monthly and lifetime costs, expenses, and profits, alongside automated even or pro-rata apportionment of shared fixed-cost services.  
- **In Scope:**  
  - Single-user authentication protecting all dashboard UI and administrative actions.  
  - Project portfolio registry (create, edit, pause/archive projects with workspace identifiers).  
  - Automated AI token ingestion endpoint fed by Antigravity IDE Stop lifecycle hooks, with idempotent deduplication and local offline queuing.  
  - Monthly project metrics tracking (total users, monthly active users, direct project expenses, gross revenue) and cumulative lifetime rollups.  
  - Shared fixed-cost service registry (monthly and annualized subscriptions) with monthly **Even** and **Pro-Rata** cost apportionment across linked projects.  
  - Executive dashboard views: portfolio overview, per-project monthly vs. lifetime financial breakdown, and monthly reconciliation workflow.  
- **Out of Scope (Post-MVP):**  
  - Multi-tenant team accounts, role-based access control, or public registration.  
  - Automated billing API scrapers (e.g., pulling invoices automatically from Stripe, AWS, or Vercel APIs).  
  - Usage-tiered or variable utility service pricing models (MVP covers fixed-cost recurring subscriptions only).  
  - Multi-currency FX conversion (MVP operates in a single base currency).

# 2\. User Flows & Functional Requirements

## 2.1 Single-User Authentication & Access Control

- **FR-001 (Dashboard Authentication):** Unauthenticated requests to any dashboard route must redirect to a login screen. Only the configured single owner can authenticate (via WebAuthn Passkey or owner password) to establish an encrypted session.  
- **FR-002 (Ingestion Endpoint Authorization):** External telemetry ingestion endpoints must reject requests lacking a valid bearer credential without requiring an interactive browser session.  
- **FR-003 (Session Termination):** The authenticated owner must be able to sign out explicitly, immediately invalidating the active browser session.

## 2.2 Project Portfolio & Monthly Metrics Management

- **FR-004 (Project Registry):** The owner can create, update, and set lifecycle statuses (Active, Paused, Archived) for projects, assigning each a unique human-readable slug used to correlate incoming telemetry.  
- **FR-005 (Monthly Project Metrics Entry):** For any selected calendar month (`YYYY-MM`), the owner can record or update a project's **Total Users**, **Monthly Active Users (MAU)**, **Direct Expenses** (project-specific non-shared costs), and **Gross Revenue**.  
- **FR-006 (Portfolio & Project Financial Views):** The dashboard must display, for each project and across the combined portfolio:  
  - **Selected Month:** AI Token Count & Estimated AI Cost, Total Users, Monthly Active Users, Apportioned Service Costs, Direct Expenses, Total Monthly Cost, Gross Revenue, and Net Monthly Profit.  
  - **Lifetime Cumulative:** Cumulative Total Cost (all historical apportioned service costs \+ direct expenses \+ AI token costs), Cumulative Expenses, Cumulative Revenue, and Cumulative Net Profit.

## 2.3 Antigravity AI Token Tracking

- **FR-007 (Automated Hook Telemetry):** When an Antigravity agent turn completes inside a configured project workspace, a local hook must resolve the project slug (via `.dashboard-project.json` or fallback directory/git name), extract incremental input, output, and cached prompt (`cache_read_tokens`) token counts and model metadata using camelCase hook parameters (`transcriptPath`), and transmit the usage record to the dashboard.  
- **FR-008 (Idempotent Ingestion, Cost Locking & Unassigned Telemetry):** The ingestion endpoint must deduplicate repeated submissions using a unique event/message cursor key, resolve the target project by slug (or preserve the record with `unassigned_project_slug` if unmatched), compute the estimated token cost based on model rates (including discounted cache token rates), lock the calculated `cost_microcents` onto the record at ingestion time to guarantee immutability (`BR-012`), and attribute the usage to the corresponding `Australia/Melbourne` calendar month.  
- **FR-009 (Offline Queue & Non-Blocking Execution):** If the dashboard endpoint is unreachable or times out during an IDE session, the local hook must persist the unsent telemetry locally, exit cleanly without blocking or failing the IDE turn, and flush queued records on the next invocation.

## 2.4 Shared Services & Cost Apportionment

- **FR-010 (Fixed-Cost Service Registry):** The owner can create, edit, and deactivate recurring services (e.g., hosting, database, domain, AI subscriptions) with a **Fixed Monthly Cost** (or an annual cost normalized to a monthly equivalent), optional active date range (`start_month` to `end_month`), and a default apportionment mode (Even or Pro-Rata).  
- **FR-011 (Monthly Project-Service Linking & Recalculation):** For a selected billing month, the owner can link one or more projects to a service, choose Even or Pro-Rata mode for that month, select a split basis (`manual` usage units, recorded `mau`, or recorded monthly `tokens`), and trigger atomic apportionment recalculation.  
- **FR-012 (Immutable Historical Snapshots):** Recalculating service splits for a target month must persist the calculated apportioned amounts for that month only, leaving prior closed months unchanged even if projects are added or deactivated later.

# 3\. Domain Rules & Business Logic

- **BR-001 (Annual-to-Monthly Normalization):** Services billed annually must normalize to a monthly base cost of `annual_cost / 12` rounded to the nearest cent before apportionment.  
- **BR-002 (Even Apportionment Formula):** When a service has `N` linked projects in month `M` under Even mode (`N >= 1`), each project is allocated `service_monthly_cost / N`. Any remainder cents from integer division are distributed 1 cent at a time deterministically (ordered by project slug ascending) so the sum of project shares equals `service_monthly_cost` exactly.  
- **BR-003 (Pro-Rata Apportionment Formula):** When a service has `N` linked projects in month `M` under Pro-Rata mode, and project `i` has recorded usage `u_i >= 0` with total usage `U = sum(u_1 ... u_N) > 0`, project `i` is allocated `service_monthly_cost * (u_i / U)` using largest-remainder cent distribution so total shares equal `service_monthly_cost` exactly.  
- **BR-004 (Zero-Usage Pro-Rata Fallback):** If a service is set to Pro-Rata for month `M` but total usage across all linked projects is `0` (`U = 0`), the system must automatically fall back to Even apportionment (`BR-002`) for that month and flag the fallback in the UI.  
- **BR-005 (Monthly & Lifetime Financial Rollup Equations):**  
  - `Monthly Total Cost = Monthly Apportioned Service Costs + Monthly Direct Expenses + Monthly AI Token Cost`  
  - `Monthly Net Profit = Monthly Gross Revenue - Monthly Total Cost`  
  - `Lifetime Total Cost = Sum(Monthly Total Cost across all recorded months)`  
  - `Lifetime Expenses = Sum(Monthly Apportioned Service Costs + Monthly Direct Expenses across all recorded months)`  
  - `Lifetime Net Profit = Sum(Monthly Gross Revenue across all recorded months) - Lifetime Total Cost`

# 4\. Edge Cases & Error States

| Scenario | Trigger Condition | Expected User-Facing Behavior |
| :---- | :---- | :---- |
| **Unknown Project Slug in Hook** | Antigravity hook sends a `projectSlug` not registered in the dashboard | Ingestion API saves the payload with `project_id = NULL` and `unassigned_project_slug = <slug>`, returning `200 OK` with `unassigned: 1`; dashboard displays an unassigned telemetry alert banner with an action to assign it to an existing or new project. |
| **Prompt Cache Tokens** | Hook sends `cacheReadTokens` in addition to input/output tokens | Ingestion computes `cost_microcents` using discounted cache read rates and locks the computed cost onto the record. |
| **Duplicate Hook Delivery** | Network retry resends an already-ingested cursor/event ID | Ingestion API returns `200 OK` (idempotent no-op) without double-counting tokens or costs. |
| **Service Linked to Zero Projects** | Active fixed-cost service has 0 linked projects in month `M` | Service cost appears under an "Unallocated Overhead" summary line on the portfolio overview so total out-of-pocket spend is never hidden. |
| **Negative Profit / Net Loss** | Total costs exceed gross revenue for a project or month | Net profit renders with a minus sign (e.g., `-$35.00`) and loss status indicator; lifetime cumulative profit accurately deducts loss months. |
| **Empty Month / New Project** | Viewing a month with no recorded metrics or a newly created project | Displays `$0.00` / `0` values cleanly with a prompt to enter monthly metrics or run apportionment. |

# 5\. Clarifications & Resolved Ambiguities

| ID | Question / Ambiguity | Resolution / Default Assumed | Date |
| :---- | :---- | :---- | :---- |
| Q-1 | Should "Costs" vs. "Expenses" be tracked distinctly in monthly and lifetime metrics? | Yes: **Direct Expenses** represent project-specific non-shared outlays (e.g., ads, contractors, one-off fees), **Apportioned Service Costs** represent shared infrastructure subscriptions, **AI Token Cost** tracks LLM spend, and **Total Cost** is their combined sum subtracted from Gross Revenue to yield Net Profit. | 2026-11-19 |
| Q-2 | What happens if a project has 0 usage in a Pro-Rata service split while other projects have `> 0` usage? | That project receives `$0.00` (0% share), and 100% of the service cost is distributed proportionally among the projects with positive usage. If *all* linked projects have 0 usage, `BR-004` falls back to an Even split. | 2026-11-19 |
| Q-3 | Should Antigravity token costs be included inside Monthly/Lifetime Total Costs? | Yes (`BR-005`), calculated from ingested token counts multiplied by model rates (locked at ingestion time), while also broken out separately so AI burn is transparent. | 2026-11-19 |

# 6\. Acceptance Criteria (Testable Definitions of Done)

1. **AC-1 (Maps to FR-001, FR-003):** Given an unauthenticated browser session, when navigating to `/`, then the user is redirected to `/login`; after authenticating with valid owner credentials, the user accesses the portfolio dashboard and can sign out to revoke access.  
2. **AC-2 (Maps to FR-004, FR-005, FR-006, BR-005):** Given two projects with recorded monthly users, direct expenses, revenue, and token usage across two months, when viewing the dashboard for Month 2, then both Month 2 metrics and cumulative Month 1 \+ Month 2 lifetime costs, expenses, and net profits match `BR-005` to the exact cent.  
3. **AC-3 (Maps to FR-007, FR-008, FR-009):** Given an active Antigravity workspace containing `.dashboard-project.json`, when an agent turn finishes, then `log_antigravity_tokens.py` parses camelCase hook inputs and posts token deltas (including `cache_read_tokens`) to `/api/ingest/tokens`, the dashboard locks `cost_microcents` at ingestion time and increments that project's monthly token count and AI cost, and re-sending the exact same payload does not create duplicate rows.  
4. **AC-4 (Maps to FR-010, FR-011, FR-012, BR-002, BR-003, BR-004):** Given a `$10.00/mo` shared service linked to 3 projects under Even mode, when apportionment runs, then allocations equal `$3.34`, `$3.33`, and `$3.33` (summing to `$10.00`); when switched to Pro-Rata with usage `100, 300, 0`, allocations update to `$2.50`, `$7.50`, and `$0.00` without altering any prior closed month.  
5. **AC-5 (Maps to FR-008):** Given an ingestion payload referencing an unregistered `projectSlug`, when ingested, the record is stored with `unassigned_project_slug`, appears on the dashboard unassigned telemetry banner, and can be assigned to a project to update attribution without mutating raw event timestamps.