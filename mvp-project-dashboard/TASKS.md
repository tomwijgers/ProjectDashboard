# Execution Tasks: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Source Docs:** `CONSTITUTION.md` (v1.0), `DESIGN_SYSTEM.md` (v1.0), `SPEC.md` (v1.0), `PLAN.md` (v1.0)  
**Workflow Rule:** Execute **one task per session**. Read `CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `SPEC.md`, and `PLAN.md` \-\> write failing unit/integration tests matching acceptance criteria \-\> implement code \-\> run verification gate \-\> mark `[COMPLETED]` \-\> commit.

---

# Phase 1: Project Scaffolding & Core Domain Engine

## Task 1.1: Initialize Next.js 15 + Drizzle ORM + Tailwind v4 Project Structure `[INCOMPLETE]`

- **Traces To:** `CONSTITUTION.md` §1–§2, `DESIGN_SYSTEM.md` §1–§3  
- **Goal:** Scaffold the Next.js 15 App Router workspace with TypeScript strict mode, Drizzle ORM schema (`src/data/schema.ts`), and CSS-first Tailwind v4 custom properties (`globals.css` with `@import "tailwindcss"` and `@theme`).  
- **Deliverables:**  
  - `package.json`, `tsconfig.json` (strict), `drizzle.config.ts`, and `src/app/globals.css` with all light/dark semantic tokens from `DESIGN_SYSTEM.md` §1.  
  - Drizzle PostgreSQL schema in `src/data/schema.ts` defining:
    - `projects` (slug, name, status, repo_path).
    - `services` (name, billing_cycle, raw_cost_cents, start_month, end_month, default_split_mode, default_split_metric).
    - `project_service_allocations` (billing_month, service_id, project_id, apportioned_cents, fell_back_to_even).
    - `monthly_project_metrics` (billing_month, project_id, total_users, mau, direct_expenses_cents, gross_revenue_cents).
    - `ai_token_logs` (idempotency_key, project_id, unassigned_project_slug, model, input_tokens, output_tokens, cache_read_tokens, cost_microcents, recorded_at, billing_month).
    - `model_rates` (model_pattern, input_microcents_per_1k, output_microcents_per_1k, cache_input_microcents_per_1k).
    - `owner_passkeys` (credential_id, public_key, counter).
- **Verification Gate:** `npm run typecheck && npm run lint && npx drizzle-kit generate` pass with zero errors.

## Task 1.2: Implement Pure Domain Models, Apportionment Engine & Rollups `[INCOMPLETE]`

- **Traces To:** `FR-006`, `FR-010`, `FR-011`, `FR-012`, `BR-001`–`BR-005`, `AC-2`, `AC-4`  
- **Goal:** Implement pure TypeScript domain logic in `src/domain/` with zero framework or database imports.  
- **Deliverables:**  
  - `src/domain/models.ts`, `src/domain/ports.ts`, `src/domain/apportionment.ts`, `src/domain/rollups.ts`, and `src/domain/tokenPricing.ts` (with `toMelbourneBillingMonth` via `Intl.DateTimeFormat`).  
  - Deterministic annual-to-monthly normalization (`BR-001`), slug-sorted even remainder distribution (`BR-002`), largest-remainder pro-rata apportionment with automated metric binding support (`BR-003`), zero-usage even fallback (`BR-004`), and monthly/lifetime financial rollups using locked token costs (`BR-005`).  
- **Tests First (`src/domain/__tests__/`):**  
  - `apportionment.test.ts` covering `$10.00` 3-way split (`334, 333, 333`), partial zero usage (`100, 300, 0`), all-zero fallback, annual normalization (`$100/yr` -> `833` cents), metric binding (`mau`, `tokens`), and 0 linked projects.  
  - `tokenPricing.test.ts` verifying input/output and discounted `cache_read_tokens` microcent calculations, plus `toMelbourneBillingMonth` across daylight saving shifts.  
  - `rollups.test.ts` covering monthly net loss and multi-month cumulative lifetime totals.  
- **Verification Gate:** `npm test` passes with 100% pass rate.

---

# Phase 2: Authentication & Ingestion API

## Task 2.1: Implement Single-User Authentication & Route Protection `[INCOMPLETE]`

- **Traces To:** `FR-001`, `FR-003`, `CONSTITUTION.md` §7, `AC-1`  
- **Goal:** Protect all dashboard routes via `middleware.ts` using signed `pd_session` JWT cookies (`jose`), supporting initial owner password login (`scrypt` against `OWNER_PASSWORD_HASH`) and WebAuthn Passkey registration/login.  
- **Deliverables:**  
  - `src/middleware.ts`, `src/app/login/page.tsx`, and `src/app/api/auth/*` handlers (login, passkey registration, passkey authentication, logout).  
  - `src/app/api/health/route.ts` unauthenticated health endpoint.  
- **Verification Gate:** `npm run typecheck && npm test` pass; unauthenticated requests to `/` redirect to `/login`.

## Task 2.2: Implement Idempotent Token Ingestion API & Antigravity Hook Script `[INCOMPLETE]`

- **Traces To:** `FR-002`, `FR-007`, `FR-008`, `FR-009`, `AC-3`, `AC-5`  
- **Goal:** Build `POST /api/ingest/tokens` with Bearer token auth, idempotency deduplication, locked `cost_microcents`, unassigned slug retention, `Australia/Melbourne` month bucketing, and local offline-resilient Python hook script.  
- **Deliverables:**  
  - `src/app/api/ingest/tokens/route.ts` and `src/data/repositories/` implementations.  
  - `scripts/log_antigravity_tokens.py` parsing camelCase hook stdin (`transcriptPath`, `workspacePaths`), extracting `input_tokens`, `output_tokens`, and `cache_read_tokens`, resolving `.dashboard-project.json` or fallback workspace name, tracking cursors (`~/.cache/antigravity-token-cursors.json`), and buffering offline requests (`~/.cache/antigravity-tokens-queue.jsonl`).  
- **Tests First:**  
  - API integration tests for `401 Unauthorized`, `200 OK` idempotent replay, timezone month boundary bucketing, locked token cost calculation, and `unassigned_project_slug` persistence.  
  - `scripts/test_log_antigravity_tokens.py` verifying cursor advancement, fallback slug resolution, and offline retry queue flushing.  
- **Verification Gate:** `npm test && python3 -m unittest scripts/test_log_antigravity_tokens.py` pass.

---

# Phase 3: Executive Dashboard UI & Service Apportionment Workflows

## Task 3.1: Build Portfolio Executive Overview & Project Metrics Management `[INCOMPLETE]`

- **Traces To:** `FR-004`, `FR-005`, `FR-006`, `DESIGN_SYSTEM.md` §1–§5, `AC-2`, `AC-5`  
- **Goal:** Build the main executive dashboard (`/`), project registry CRUD (`/projects`), unassigned telemetry alert banner, and monthly metrics entry (`/projects/[slug]`) adhering to Next.js 15 async route props and WCAG AA+ compliance.  
- **Deliverables:**  
  - Month selector (`YYYY-MM`), 4 portfolio KPI cards, unassigned telemetry resolution widget, Selected Month vs. Lifetime Cumulative comparison table, and monthly metrics Server Actions.  
- **Verification Gate:** `npm run typecheck && npm run lint && npm run build` pass.

## Task 3.2: Build Shared Services Registry & Monthly Apportionment Matrix `[INCOMPLETE]`

- **Traces To:** `FR-010`, `FR-011`, `FR-012`, `BR-001`–`BR-004`, `DESIGN_SYSTEM.md` §1–§5, `AC-4`  
- **Goal:** Build `/services` allowing recurring service management (with `start_month`/`end_month`), per-month project linking, Even/Pro-Rata mode selection, metric binding (`manual`, `mau`, `tokens`), and atomic monthly snapshot recalculation using `db.batch`.  
- **Deliverables:**  
  - Service subscription CRUD, monthly apportionment matrix UI with fallback badges, and **"Recalculate Splits"** Server Action executing batch atomic writes.  
- **Verification Gate:** `npm test && npm run build` pass with zero errors.