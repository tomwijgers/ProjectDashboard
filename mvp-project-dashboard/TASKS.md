# Execution Tasks: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Source Docs:** `CONSTITUTION.md` (v2.0), `DESIGN_SYSTEM.md` (v2.0), `SPEC.md` (v2.0), `PLAN.md` (v2.0)  
**Workflow Rule:** Execute **one task per session**. Read `CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `SPEC.md`, and `PLAN.md` -> write failing unit/integration tests matching acceptance criteria -> implement code -> run verification gate -> mark `[COMPLETED]` -> commit.

---

# Phase 1: Project Scaffolding & Core Domain Engine

## Task 1.1: Initialize Next.js 15 + Drizzle ORM + Tailwind v4 Project Structure `[INCOMPLETE]`

- **Traces To:** `CONSTITUTION.md` §1–§2, `DESIGN_SYSTEM.md` §1–§3  
- **Goal:** Scaffold the Next.js 15 App Router workspace in the `app/` directory with TypeScript strict mode, extensible cost ledger Drizzle schema (`app/src/data/schema.ts`), and Tailwind v4 CSS-first tokens (`app/src/app/globals.css`).  
- **Deliverables:**  
  - `app/package.json`, `app/tsconfig.json` (strict), `app/drizzle.config.ts`, and `app/src/app/globals.css` with semantic tokens from `DESIGN_SYSTEM.md` §1.  
  - Drizzle PostgreSQL schema in `app/src/data/schema.ts` defining:
    - `projects` (slug immutable, name, repo_path, status).
    - `cost_items` (name, cost_type, apportionment_method, default_amount_cents, is_active).
    - `monthly_cost_entries` (billing_month, cost_item_id, amount_cents, notes).
    - `project_cost_allocations` (billing_month, cost_item_id, project_id, apportioned_cents, basis_metric, basis_units, basis_ratio).
    - `ai_token_logs` (idempotency_key, project_id, unassigned_project_slug, model, input_tokens, output_tokens, cache_read_tokens, recorded_at, billing_month).
    - `owner_passkeys` (credential_id, public_key, counter).
- **Verification Gate:** `cd app && npm run typecheck && npm run lint && npx drizzle-kit generate` pass with zero errors.

## Task 1.2: Implement Pure Domain Models, Token-Weighted Apportionment & Rollups `[INCOMPLETE]`

- **Traces To:** `FR-006`, `FR-007`, `BR-001`–`BR-004`, `AC-3`  
- **Goal:** Implement pure TypeScript domain logic in `app/src/domain/` with zero framework or database imports.  
- **Deliverables:**  
  - `app/src/domain/models.ts`, `app/src/domain/ports.ts`, `app/src/domain/apportionment.ts`, `app/src/domain/rollups.ts`, and `app/src/domain/dateUtils.ts` (with `toMelbourneBillingMonth` via `Intl.DateTimeFormat`).  
  - Token-weighted apportionment algorithm (`BR-002`) using largest-remainder integer cent distribution, zero-usage fallback (`BR-003`), and monthly/lifetime token and cost aggregators (`BR-004`).  
- **Tests First (`app/src/domain/__tests__/`):**  
  - `apportionment.test.ts` covering proportional token split (`$50.00` split 70k/30k tokens -> `$35.00` / `$15.00`), remainder cent distribution (slug ascending tie-breaking), and zero-token month handling.  
  - `dateUtils.test.ts` verifying `toMelbourneBillingMonth` across daylight saving time shifts (AEST <-> AEDT).  
  - `rollups.test.ts` verifying monthly project token summaries and multi-month lifetime cumulative totals.  
- **Verification Gate:** `cd app && npm test` passes with 100% pass rate.

---

# Phase 2: Authentication & Ingestion API

## Task 2.1: Implement Single-User Authentication & Route Protection `[INCOMPLETE]`

- **Traces To:** `FR-001`, `FR-003`, `CONSTITUTION.md` §7, `AC-1`  
- **Goal:** Protect all dashboard routes via `app/src/middleware.ts` using signed `pd_session` JWT cookies (`jose`), supporting owner password login (`scrypt` against `OWNER_PASSWORD_HASH`) and WebAuthn Passkey registration/login.  
- **Deliverables:**  
  - `app/src/middleware.ts`, `app/src/app/login/page.tsx`, and `app/src/app/api/auth/*` handlers (login, passkey registration, passkey authentication, logout).  
  - `app/src/app/api/health/route.ts` unauthenticated health endpoint.  
- **Verification Gate:** `cd app && npm run typecheck && npm test` pass; unauthenticated requests to `/` redirect to `/login`.

## Task 2.2: Implement Idempotent Token Ingestion API & Antigravity Hook Script `[INCOMPLETE]`

- **Traces To:** `FR-002`, `FR-008`, `FR-009`, `FR-010`, `AC-4`, `AC-5`  
- **Goal:** Build `POST /api/ingest/tokens` with Bearer token auth, idempotency deduplication, unassigned slug retention, `Australia/Melbourne` month bucketing, and local offline-resilient Python hook script.  
- **Deliverables:**  
  - `app/src/app/api/ingest/tokens/route.ts` and `app/src/data/repositories/` implementations.  
  - `scripts/log_antigravity_tokens.py` parsing camelCase hook stdin (`transcriptPath`, `workspacePaths`), extracting `input_tokens`, `output_tokens`, and `cache_read_tokens`, resolving `.dashboard-project.json` or fallback workspace name, tracking cursors (`~/.cache/antigravity-token-cursors.json`), and buffering offline requests (`~/.cache/antigravity-tokens-queue.jsonl`).  
- **Tests First:**  
  - API integration tests for `401 Unauthorized`, `200 OK` idempotent replay, timezone month boundary bucketing, and `unassigned_project_slug` persistence.  
  - `scripts/test_log_antigravity_tokens.py` verifying cursor advancement, fallback slug resolution, and offline retry queue flushing.  
- **Verification Gate:** `cd app && npm test && python3 -m unittest scripts/test_log_antigravity_tokens.py` pass.

---

# Phase 3: Executive Dashboard UI & Cost Management Workflows

## Task 3.1: Build Executive Dashboard UI & Project Details Editing `[INCOMPLETE]`

- **Traces To:** `FR-004`, `FR-007`, `DESIGN_SYSTEM.md` §1–§5, `AC-2`, `AC-5`  
- **Goal:** Build the main executive dashboard (`app/src/app/(dashboard)/page.tsx`), Project Token Matrix table, and project editing modal with immutable slug enforcement adhering to Next.js 15 async route props and WCAG AA+ compliance.  
- **Deliverables:**  
  - Month selector (`YYYY-MM`), 3 portfolio KPI cards (Total Tokens, Monthly AI Cost, Active Projects), Project Token Matrix table (Monthly Tokens, Share %, Apportioned Cost, Lifetime Tokens, Lifetime Cost).  
  - Project Details editing modal allowing modification of display name, repo path, and status, while displaying the project slug as read-only.  
  - Unassigned telemetry resolution banner with action to assign unknown slug to an existing or new project.  
- **Verification Gate:** `cd app && npm run typecheck && npm run lint && npm run build` pass.

## Task 3.2: Build Monthly AI Price Management & Apportionment Recalculation `[INCOMPLETE]`

- **Traces To:** `FR-005`, `FR-006`, `BR-001`, `BR-002`, `BR-005`, `AC-3`  
- **Goal:** Build the modal to view/edit the month's AI subscription price (base fee + credit top-ups) and the **"Recalculate Splits"** Server Action executing atomic token-weighted apportionment via `db.batch`.  
- **Deliverables:**  
  - "Edit Monthly AI Budget" dialog to create or update `monthly_cost_entries` for the selected month.  
  - **"Recalculate Splits"** Server Action that aggregates monthly project tokens, computes proportional integer cents with largest-remainder distribution, and persists `project_cost_allocations` in an atomic transaction.  
- **Verification Gate:** `cd app && npm test && npm run build` pass with zero errors.
