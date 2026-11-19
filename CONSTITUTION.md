# Project Constitution: project-dashboard

**Version:** 1.0  
**Last Updated:** 2026-11-19

This document defines the immutable architectural rules, technology stack, coding standards, and operational guardrails for `project-dashboard`. All AI agents and developers must strictly adhere to these rules when generating specifications, plans, tasks, and code.

# 1\. Project Overview & Domain Boundaries

- **Product Name:** project-dashboard  
- **Core Purpose:** A personal single-user executive web dashboard tracking all active projects inside Antigravity (web apps, mobile apps, 3D assets, and creative workflows)—aggregating per-project token consumption, maintaining an extensible cost architecture for both fixed and metered expenses, and apportioning monthly AI subscription costs (including credit top-ups) proportionally based on token usage.  
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
  - Repository Root: Reserved for top-level documentation, specification suites (`CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `mvp-project-dashboard/`), hook integration scripts (`scripts/`), and future tools.  
  - `app/` - Dedicated application root containing the Next.js workspace:  
    - `app/src/domain/` - Pure TypeScript domain models, token-weighted apportionment calculator, extensible cost calculators, and repository port interfaces. Zero framework, database, or HTTP imports allowed.  
    - `app/src/data/` - Drizzle schema (`schema.ts`), Neon database client, repository implementations, and SQL migrations (`drizzle/`).  
    - `app/src/app/` - Next.js App Router pages, Server Actions, Route Handlers (`/api/*`), and `middleware.ts`.  
    - `app/src/components/` - Reusable UI components strictly referencing semantic tokens defined in `DESIGN_SYSTEM.md`.  
  - `scripts/` - Local Antigravity hook scripts (`log_antigravity_tokens.py`) and local offline retry queue handlers at repository root.  
- **Concurrency & Time:** Store all timestamps in UTC (`timestamptz`). Group monthly ledger metrics by `billing_month` (`YYYY-MM` string) using `Australia/Melbourne` local calendar boundaries via `Intl.DateTimeFormat` at ingestion time.

# 3\. Data Sourcing & External Integrations

- **Antigravity Token Telemetry:** Local Stop lifecycle hook configured in `.agents/hooks.json` reads session transcript usage metadata (accounting for `input_tokens`, `output_tokens`, and `cache_read_tokens` using camelCase hook context), computes token deltas since the last cursor, and POSTs batches to `/api/ingest/tokens`. Unrecognized project slugs are preserved under unassigned records rather than dropped.  
- **Offline Hook Resilience:** If the network or Vercel deployment is unreachable, the hook script queues payloads to `~/.cache/antigravity-token-queue.jsonl` and flushes them on subsequent runs, always exiting with status `0` so the IDE is never blocked.  
- **Admin Configuration:** The owner manages project metadata (editable name, repo path, status; slug is immutable) and configures monthly AI costs (base subscription + credit top-ups) directly through authenticated Server Actions.

# 4\. Coding Standards & Hard Guardrails (Negative Constraints)

- **Strict Layer Separation & Directory Guardrails:**  
  - **Do NOT** place application source files, package manifests, or build configs in the repository root; all Next.js application code resides strictly inside `app/`.  
  - **Do NOT** import `next/*`, `drizzle-orm`, `@neondatabase/serverless`, or browser APIs inside `app/src/domain/`.  
  - **Do NOT** execute raw SQL queries directly inside React Server Components or Client Components; route all reads/writes through `app/src/data/` repositories and Server Actions.  
- **Financial & Calculation Precision:**  
  - **Do NOT** allow modification of a project's `slug` after creation; the slug serves as the immutable foreign key for all incoming Antigravity telemetry.  
  - **Do NOT** use floating-point arithmetic for currency splits. Allocate remainder cents deterministically (largest-remainder method) so apportioned project costs always sum exactly to the monthly cost entry.  
  - **Do NOT** hardcode AI pricing logic in isolation; implement costs via the extensible cost ledger (`cost_items`, `monthly_cost_entries`, `project_cost_allocations`) to ensure future fixed subscriptions and metered API costs can be integrated without schema migrations.  
  - **Do NOT** mutate historical closed `billing_month` snapshots when adding new projects or updating future months. Execute recalculations atomically using `db.batch`.  
- **Error Handling & Security:**  
  - **Do NOT** leak stack traces, database connection strings, or raw SQL errors in API responses or UI views. Return typed `Result<T, DomainError>` objects from domain/data layers.  
  - **Do NOT** commit `.env`, `.env.local`, or API keys to Git.  
- **Testing Mandate (Test-First):**  
  - All domain logic in `app/src/domain/` (especially token-weighted cost apportionment and remainder cent distribution) must have Vitest unit tests written before implementation code.

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

Commands are executed from the `app/` directory (or with `--prefix app`):

- **Unit & Domain Tests:** `cd app && npm test` (Vitest)  
- **Type Check & Lint:** `cd app && npm run typecheck && npm run lint`  
- **Database Migrations:** `cd app && npx drizzle-kit generate && npx drizzle-kit migrate`  
- **Production Build Verification:** `cd app && npm run build`  
- **Hook Script Tests:** `python3 -m unittest scripts/test_log_antigravity_tokens.py` (run from repository root)

# 7\. Security, Privacy & Agent Workflow Rules

- **Single-User Hardening:** Protect all dashboard routes (`/`, `/projects/*`, `/services/*`) via `middleware.ts`. Exempt only `/login`, `/api/health`, and `/api/ingest/*` (which enforces Bearer token auth).  
- **SDD Gate Rule:** Before writing application code, verify `CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `SPEC.md`, `PLAN.md`, and `TASKS.md` exist in `Gemini/SDD/project-dashboard` and align with zero terminology drift.  
- **One-Task Execution:** Execute one `[INCOMPLETE]` task per session, running the verification gate before marking `[COMPLETED]`.