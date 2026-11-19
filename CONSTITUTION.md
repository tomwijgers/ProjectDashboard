# Project Constitution: project-dashboard

**Version:** 1.0  
**Last Updated:** 2026-11-19

This document defines the immutable architectural rules, technology stack, coding standards, and operational guardrails for `project-dashboard`. All AI agents and developers must strictly adhere to these rules when generating specifications, plans, tasks, and code.

# 1\. Project Overview & Domain Boundaries

- **Product Name:** project-dashboard  
- **Core Purpose:** A personal single-user executive web dashboard tracking all active software and business projects—aggregating monthly AI token usage, total and monthly active users, monthly and lifetime costs, expenses, profits, and shared fixed-cost service apportionments.  
- **Geographic / Domain Scope:** Single-owner personal administration tool (default currency AUD, stored as integer cents or fixed-precision decimal `NUMERIC(12, 2)`). All monthly rollups use calendar months (`YYYY-MM`).  
- **Primary Surfaces:**  
  1. Authenticated Next.js Web Dashboard (Desktop & Mobile responsive) hosted on Vercel.  
  2. Authenticated REST Ingestion API (`/api/ingest/tokens`) triggered by local Antigravity IDE lifecycle hooks.

# 2\. Technology Stack & Module Architecture

- **Language & Runtime:** TypeScript 5+ (strict mode enabled), Node.js 22 LTS on Vercel Serverless Functions, Python 3.11+ (standard library only) for local Antigravity hook scripts.  
- **Framework:** Next.js 15+ (App Router with asynchronous `params` and `searchParams` route contracts) with React 19 and Tailwind CSS v4 (CSS-first configuration with `@import "tailwindcss"` and semantic `@theme` mappings in `globals.css`).  
- **Database & ORM:** Neon Serverless Postgres (`@neondatabase/serverless`) with Drizzle ORM (`drizzle-orm`, `drizzle-kit`) for schema definition, batch transactions (`db.batch`), and SQL migrations.  
- **Authentication:** Single-User Session Cookie (`HttpOnly`, `Secure`, `SameSite=Lax`, signed via `jose` JWT/HMAC using `AUTH_SECRET`) + owner fallback password (`OWNER_PASSWORD_HASH`) and WebAuthn Passkey enrollment (`@simplewebauthn/server` / `@simplewebauthn/browser`), verified in Next.js `middleware.ts`. Ingestion endpoints authenticate via `Authorization: Bearer <INGEST_API_KEY>` using timing-safe comparison.  
- **Workspace / Module Structure:**  
  - `src/domain/` - Pure TypeScript domain models, apportionment calculators, rollup aggregators, and repository port interfaces. Zero framework, database, or HTTP imports allowed.  
  - `src/data/` - Drizzle schema (`schema.ts`), Neon database client, repository implementations, and SQL migrations (`drizzle/`).  
  - `src/app/` - Next.js App Router pages, Server Actions, Route Handlers (`/api/*`), and `middleware.ts`.  
  - `src/components/` - Reusable UI components strictly referencing semantic tokens defined in `DESIGN_SYSTEM.md`.  
  - `scripts/` - Local Antigravity hook scripts (`log_antigravity_tokens.py`) and local offline retry queue handlers.  
- **Concurrency & Time:** Store all timestamps in UTC (`timestamptz`). Group monthly ledger metrics by `billing_month` (`YYYY-MM` string) using `Australia/Melbourne` local calendar boundaries via `Intl.DateTimeFormat` at ingestion time.

# 3\. Data Sourcing & External Integrations

- **Antigravity Token Telemetry:** Local Stop lifecycle hook configured in `.agents/hooks.json` reads session transcript usage metadata (accounting for `input_tokens`, `output_tokens`, and `cache_read_tokens` using camelCase hook context), computes token deltas since the last cursor, and POSTs batches to `/api/ingest/tokens`. Unrecognized project slugs are preserved under unassigned records rather than dropped.  
- **Offline Hook Resilience:** If the network or Vercel deployment is unreachable, the hook script queues payloads to `~/.cache/antigravity-token-queue.jsonl` and flushes them on subsequent runs, always exiting with status `0` so the IDE is never blocked.  
- **Manual & Admin Entry:** Monthly user counts (total users, MAU), direct project expenses, gross revenue, and fixed-cost service subscriptions are entered and reconciled through authenticated dashboard Server Actions.

# 4\. Coding Standards & Hard Guardrails (Negative Constraints)

- **Strict Layer Separation:**  
  - **Do NOT** import `next/*`, `drizzle-orm`, `@neondatabase/serverless`, or browser APIs inside `src/domain/`.  
  - **Do NOT** execute raw SQL queries directly inside React Server Components or Client Components; route all reads/writes through `src/data/` repositories and Server Actions.  
- **Financial & Calculation Precision:**  
  - **Do NOT** use floating-point arithmetic (`Number` division without rounding) for final currency splits. Allocate remainder cents deterministically (largest-remainder method) so apportioned costs always sum exactly to the service's monthly cost.  
  - **Do NOT** compute historical AI token costs dynamically from mutable pricing tables during rollup queries; calculate and lock `cost_microcents` directly onto `ai_token_logs` rows at ingestion time to enforce `BR-012` snapshot immutability.  
  - **Do NOT** mutate historical closed `billing_month` snapshots when adding new projects or changing future service allocations. Execute recalculations atomically using `db.batch`.  
- **Error Handling & Security:**  
  - **Do NOT** leak stack traces, database connection strings, or raw SQL errors in API responses or UI views. Return typed `Result<T, DomainError>` objects from domain/data layers.  
  - **Do NOT** commit `.env`, `.env.local`, or API keys to Git.  
- **Testing Mandate (Test-First):**  
  - All domain logic in `src/domain/` (especially even/pro-rata cost apportionment, remainder cent distribution, and lifetime rollups) must have Vitest unit tests written before implementation code.

# 5\. Canonical Patterns (Few-Shot Anchors)

```ts
export type DomainError =
  | { code: 'UNAUTHORIZED' }
  | { code: 'NOT_FOUND'; entity: string; id: string }
  | { code: 'VALIDATION_ERROR'; message: string }
  | { code: 'PERSISTENCE_ERROR'; message: string };

export type Result<T, E = DomainError> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

# 6\. Commands & Verification Gates

- **Unit & Domain Tests:** `npm test` (Vitest)  
- **Type Check & Lint:** `npm run typecheck && npm run lint`  
- **Database Migrations:** `npx drizzle-kit generate && npx drizzle-kit migrate`  
- **Production Build Verification:** `npm run build`

# 7\. Security, Privacy & Agent Workflow Rules

- **Single-User Hardening:** Protect all dashboard routes (`/`, `/projects/*`, `/services/*`) via `middleware.ts`. Exempt only `/login`, `/api/health`, and `/api/ingest/*` (which enforces Bearer token auth).  
- **SDD Gate Rule:** Before writing application code, verify `CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `SPEC.md`, `PLAN.md`, and `TASKS.md` exist in `Gemini/SDD/project-dashboard` and align with zero terminology drift.  
- **One-Task Execution:** Execute one `[INCOMPLETE]` task per session, running the verification gate before marking `[COMPLETED]`.