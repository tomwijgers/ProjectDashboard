# Technical Design Plan: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Source Spec:** `SPEC.md` (v2.0)  
**Governed By:** `CONSTITUTION.md`, `DESIGN_SYSTEM.md`

---

# 1\. Architecture & Component Mapping

## Domain Layer (`src/domain/`)

- `models.ts`: Pure TypeScript interfaces for `Project`, `CostItem`, `MonthlyCostEntry`, `ProjectCostAllocation`, `TokenUsageEvent`, and `ProjectTokenSummary`.  
- `apportionment.ts`: Pure deterministic apportionment engine (`calculateTokenWeightedApportionment`) implementing `BR-002` (largest-remainder cent distribution proportional to token usage) and `BR-003` (zero-usage fallback).  
- `rollups.ts`: Pure aggregator (`computeProjectTokenSummary`, `computePortfolioSummary`) implementing `BR-004` monthly and lifetime metrics.  
- `dateUtils.ts`: Pure timezone helper `toMelbourneBillingMonth(date)` using `Intl.DateTimeFormat` with `timeZone: 'Australia/Melbourne'`.  
- `ports.ts`: Repository interfaces (`ProjectRepository`, `CostLedgerRepository`, `TokenLogRepository`, `PasskeyCredentialRepository`).

## Data Layer (`src/data/`)

- `db.ts`: Neon HTTP serverless Drizzle client (`drizzle(neon(process.env.DATABASE_URL!))`), utilizing `db.batch([...])` for atomic recalculation writes.  
- `schema.ts`: Drizzle PostgreSQL schema defining:
  - `projects` (immutable slug, display name, repo path, status).
  - `cost_items` (cost definition: AI subscription, fixed/metered flag, apportionment method).
  - `monthly_cost_entries` (month-specific cost amount, e.g. base subscription + credits).
  - `project_cost_allocations` (persisted monthly project cost shares with token basis units).
  - `ai_token_logs` (idempotency key, project ID, unassigned slug, model, input/output/cache tokens, billing month).
  - `owner_passkeys` (WebAuthn credentials).
- `repositories/*.ts`: Concrete Drizzle implementations of `src/domain/ports.ts`.

## Application & API Layer (`src/app/`)

- `middleware.ts`: Verifies `pd_session` signed JWT cookie via `jose`; redirects unauthenticated browser requests to `/login`.  
- `app/login/page.tsx` & `app/api/auth/*`: Owner password login (`scrypt` check against `OWNER_PASSWORD_HASH`) as primary onboarding flow, with WebAuthn Passkey registration/authentication.  
- `app/api/ingest/tokens/route.ts`: Bearer-authenticated (`INGEST_API_KEY`) idempotent ingestion endpoint (`FR-008`, `FR-009`).  
- `app/api/health/route.ts`: Lightweight unauthenticated DB ping endpoint for deployment health checks.  
- `app/(dashboard)/page.tsx`: Main dashboard overview with Next.js 15 asynchronous `searchParams: Promise<{ month?: string }>`:
  - Month selector (`YYYY-MM`) with **"Edit Monthly AI Budget"** modal trigger.
  - 3 Summary KPI cards: Total Tokens Consumed, Total Monthly AI Budget, Active Projects.
  - Project Token Matrix table: Project Name, Status, Monthly Tokens (Input, Output, Cache Read), Token Share %, Monthly Apportioned AI Cost, Lifetime Tokens, Lifetime Apportioned Cost.
  - Inline **"Edit Project Details"** modal (allows modifying name, repo path, status; slug is read-only).
  - Unassigned Telemetry Alert Banner for unknown slugs with one-click project assignment.
- `app/(dashboard)/actions.ts`: Authenticated Server Actions:
  - `updateProjectDetails(slug, data)`: Modifies name, repo path, or status (validating slug immutability).
  - `updateMonthlyAiCost(billingMonth, amountCents)`: Creates or updates the month's AI cost entry.
  - `recalculateMonthlySplits(billingMonth)`: Atomically computes and persists token-weighted allocations for that month via `db.batch`.
  - `assignUnassignedTelemetry(unassignedSlug, targetProjectId)`: Re-attributes unassigned logs.

## Local Hook Integration (`scripts/`)

- `scripts/log_antigravity_tokens.py`: Python 3 standard-library script triggered by `.agents/hooks.json` (Stop event). Reads `cwd`, `workspacePaths`, and `transcriptPath` (camelCase) from stdin, resolves project slug (via `.dashboard-project.json` or fallback workspace folder name), extracts `input_tokens`, `output_tokens`, and `cache_read_tokens`, tracks message IDs in `~/.cache/antigravity-token-cursors.json`, queues failed requests to `~/.cache/antigravity-tokens-queue.jsonl`, and POSTs batches to `/api/ingest/tokens`.

---

# 2\. Domain Models & Extensible Cost Contracts

```ts
export type ProjectStatus = 'active' | 'paused' | 'archived';
export type CostType = 'fixed_subscription' | 'metered';
export type ApportionmentMethod = 'token_weighted' | 'even' | 'manual' | 'direct';

export interface Project {
  id: string;
  slug: string;        // Immutable unique identifier
  name: string;        // Editable display name
  repoPath: string | null;
  status: ProjectStatus;
}

export interface CostItem {
  id: string;
  name: string;        // e.g., 'Antigravity AI Subscription'
  costType: CostType;  // 'fixed_subscription' (MVP) or 'metered' (future)
  apportionmentMethod: ApportionmentMethod; // 'token_weighted' (MVP)
  defaultAmountCents: number;              // e.g., 3300 for $33.00/mo
  isActive: boolean;
}

export interface MonthlyCostEntry {
  id: string;
  billingMonth: string; // 'YYYY-MM'
  costItemId: string;
  amountCents: number;  // Editable for that month (e.g. $33 base + $20 credits = 5300)
  notes: string | null;
}

export interface ProjectCostAllocation {
  id: string;
  billingMonth: string;
  costItemId: string;
  projectId: string;
  apportionedCents: number;
  basisMetric: string;   // 'tokens'
  basisUnits: number;    // Project's total tokens for that month
  basisRatio: number;    // (basisUnits / totalMonthUnits)
}

export interface ProjectTokenSummary {
  project: Project;
  monthlyInputTokens: number;
  monthlyOutputTokens: number;
  monthlyCacheReadTokens: number;
  monthlyTotalTokens: number;
  tokenShareRatio: number;
  apportionedCostCents: number;
  lifetimeTokens: number;
  lifetimeCostCents: number;
}
```

---

# 3\. Database Schema & External Contracts

## Database Schema Overview (`src/data/schema.ts`)

| Table Name | Primary Key | Key Columns & Indexes | Description |
| :---- | :---- | :---- | :---- |
| `projects` | `id` (uuid) | `slug` (text, unique, not null), `name` (text, not null), `repo_path` (text), `status` (text, default `'active'`) | Project portfolio registry (slug is immutable) |
| `cost_items` | `id` (uuid) | `name` (text, not null), `cost_type` (text, default `'fixed_subscription'`), `apportionment_method` (text, default `'token_weighted'`), `default_amount_cents` (int, not null), `is_active` (boolean, default true) | Extensible cost definitions |
| `monthly_cost_entries` | `id` (uuid) | `billing_month` (text, not null), `cost_item_id` (uuid, not null), `amount_cents` (int, not null), `notes` (text) — Unique on `(billing_month, cost_item_id)` | Month-specific cost overrides (sub + credits) |
| `project_cost_allocations` | `id` (uuid) | `billing_month` (text, not null), `cost_item_id` (uuid, not null), `project_id` (uuid, not null), `apportioned_cents` (int, not null), `basis_metric` (text), `basis_units` (bigint), `basis_ratio` (numeric) — Unique on `(billing_month, cost_item_id, project_id)` | Persisted monthly project cost splits |
| `ai_token_logs` | `id` (uuid) | `idempotency_key` (text, unique, not null), `project_id` (uuid, nullable), `unassigned_project_slug` (text, nullable), `model` (text, not null), `input_tokens` (int, not null), `output_tokens` (int, not null), `cache_read_tokens` (int, not null, default 0), `recorded_at` (timestamptz, not null), `billing_month` (text, not null, indexed) | Ingested Antigravity token usage records |
| `owner_passkeys` | `credential_id` (text) | `public_key` (text, not null), `counter` (bigint, not null) | WebAuthn credentials for owner auth |

> [!NOTE]
> All monthly apportionment recalculation writes must be executed atomically using `db.batch([...])` to prevent partial updates.

## Ingestion REST Contract (`POST /api/ingest/tokens`)

- **Headers:**  
  - `Authorization: Bearer <INGEST_API_KEY>`  
  - `Content-Type: application/json`  
- **Request Body Payload:**

```json
{
  "events": [
    {
      "idempotencyKey": "session-abc:step-004",
      "projectSlug": "sun-scanner",
      "sessionId": "session-abc",
      "model": "gemini-3.8-flash-high",
      "inputTokens": 4200,
      "outputTokens": 1150,
      "cacheReadTokens": 1024,
      "recordedAt": "2026-11-19T08:30:00+11:00"
    }
  ]
}
```

- **Response (`200 OK`):**

```json
{
  "inserted": 1,
  "duplicates": 0,
  "unassigned": 0
}
```

---

# 4\. Test Strategy & Fixtures

## Domain Unit Tests (`src/domain/__tests__/`)

- `apportionment.test.ts`:  
  - Proportional split: `$50.00` across 70,000 and 30,000 tokens yields `$35.00` and `$15.00`.  
  - Deterministic remainder cent distribution: `$10.00` across 3 projects with equal token counts yields `334, 333, 333` cents (slug ascending tie-breaker).  
  - Zero-usage fallback: If total month tokens = 0, allocations = 0 and cost remains unallocated.  
- `dateUtils.test.ts`:  
  - Verifies `toMelbourneBillingMonth` accurately groups timestamps across Daylight Saving Time shifts (AEST UTC+10 <-> AEDT UTC+11).  
- `rollups.test.ts`:  
  - Verifies monthly token totals and cumulative lifetime token and cost sums across multiple months.

## API Integration Tests

- Validates `401 Unauthorized` for missing/invalid Bearer token.  
- Validates `200 OK` idempotent deduplication for replayed `idempotencyKey`.  
- Validates timezone-aware month bucketing into `Australia/Melbourne`.  
- Validates unassigned slugs persist under `unassigned_project_slug` with `project_id = null`.

## Hook Script Tests (`scripts/test_log_antigravity_tokens.py`)

- Verifies parsing of camelCase stdin (`transcriptPath`, `workspacePaths`).  
- Verifies token extraction (`input_tokens`, `output_tokens`, `cache_read_tokens`).  
- Verifies fallback slug resolution from folder name when `.dashboard-project.json` is missing.  
- Verifies cursor advancement in `~/.cache/antigravity-token-cursors.json`.  
- Verifies offline `.jsonl` queue buffering during network failure and flushing on reconnect.
