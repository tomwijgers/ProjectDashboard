# Technical Design Plan: MVP Project Dashboard

**Project:** project-dashboard  
**Feature:** mvp-project-dashboard  
**Source Spec:** `SPEC.md` (v1.0)  
**Governed By:** `CONSTITUTION.md`, `DESIGN_SYSTEM.md`

---

# 1\. Architecture & Component Mapping

## Domain Layer (`src/domain/`)

- `models.ts`: Pure TypeScript interfaces for `Project`, `ServiceSubscription`, `MonthlyServiceAllocation`, `MonthlyProjectMetric`, `TokenUsageEvent`, `ModelPricingRate`, and `ProjectFinancialSummary`.  
- `apportionment.ts`: Pure deterministic apportionment engine (`calculateServiceApportionment`) implementing `BR-001` (annual-to-monthly normalization), `BR-002` (even split with deterministic slug-sorted remainder cents), `BR-003` (pro-rata largest-remainder cent distribution with automated metric binding support), and `BR-004` (zero-usage fallback to even split).  
- `rollups.ts`: Pure financial aggregator (`computeProjectFinancials`, `computePortfolioTotals`) implementing `BR-005` monthly and lifetime equations using locked token costs.  
- `tokenPricing.ts`: Pure cost calculator (`estimateTokenCostCents`) mapping model, `inputTokens`, `outputTokens`, and `cacheReadTokens` to micro-cent precision, plus deterministic `toMelbourneBillingMonth(date)` timezone helper using `Intl.DateTimeFormat`.  
- `ports.ts`: Repository interfaces (`ProjectRepository`, `ServiceRepository`, `LedgerRepository`, `TokenLogRepository`, `PasskeyCredentialRepository`).

## Data Layer (`src/data/`)

- `db.ts`: Neon HTTP serverless Drizzle client (`drizzle(neon(process.env.DATABASE_URL!))`), utilizing `db.batch([...])` for atomic multi-statement operations such as monthly apportionment recalculations.  
- `schema.ts`: Drizzle PostgreSQL schema defining `projects`, `services` (with `start_month`/`end_month`), `project_service_allocations`, `monthly_project_metrics`, `ai_token_logs` (with locked `cost_microcents`, `cache_read_tokens`, and `unassigned_project_slug`), `model_rates` (with `cache_input_microcents_per_1k`), and `owner_passkeys`.  
- `repositories/*.ts`: Concrete Drizzle implementations of `src/domain/ports.ts`.

## Application & API Layer (`src/app/`)

- `middleware.ts`: Verifies `pd_session` signed JWT cookie via `jose`; redirects unauthenticated browser requests to `/login`.  
- `app/login/page.tsx` & `app/api/auth/*`: Single-user login supporting owner password (`scrypt` hash check against `OWNER_PASSWORD_HASH`) as primary onboarding flow, with WebAuthn Passkey registration and authentication (`@simplewebauthn/server` / `@simplewebauthn/browser`) enrolled from authenticated settings.  
- `app/api/ingest/tokens/route.ts`: Bearer-authenticated (`INGEST_API_KEY`) idempotent ingestion endpoint (`FR-007`, `FR-008`). Locks calculated `cost_microcents` at ingestion time and safely buckets unrecognized slugs into `unassigned_project_slug`.  
- `app/api/health/route.ts`: Lightweight unauthenticated DB ping endpoint for deployment checks.  
- `app/(dashboard)/page.tsx`: Portfolio executive overview with month selector (`YYYY-MM`), unassigned telemetry alert banner, 4 KPI summary cards, and monthly vs. lifetime project comparison table. Built with Next.js 15 asynchronous `searchParams: Promise<{ month?: string }>`.  
- `app/(dashboard)/projects/page.tsx` & `projects/[slug]/page.tsx`: Project registry CRUD and monthly metrics entry form with Next.js 15 asynchronous `params: Promise<{ slug: string }>`.  
- `app/(dashboard)/services/page.tsx`: Shared services registry with date bounds and monthly apportionment matrix supporting **"Recalculate Splits"** Server Action with automated metric binding (`manual`, `mau`, `tokens`).

## Local Hook Integration (`scripts/`)

- `scripts/log_antigravity_tokens.py`: Python 3 standard-library script triggered by `.agents/hooks.json` (Stop event). Reads `cwd`, `workspacePaths`, and `transcriptPath` (camelCase) from stdin, resolves project slug via `.dashboard-project.json` (falling back to workspace directory name), extracts `input_tokens`, `output_tokens`, and `cache_read_tokens`, tracks processed message IDs in `~/.cache/antigravity-token-cursors.json`, queues failed requests to `~/.cache/antigravity-tokens-queue.jsonl`, and POSTs batches to `/api/ingest/tokens`.

---

# 2\. Domain Models & Data Contracts

```ts
export type ProjectStatus = 'active' | 'paused' | 'archived';
export type BillingCycle = 'monthly' | 'annual';
export type SplitMode = 'even' | 'pro_rata';
export type SplitMetricBasis = 'manual' | 'mau' | 'tokens';

export interface Project {
  id: string;
  slug: string;
  name: string;
  repoPath: string | null;
  status: ProjectStatus;
}

export interface ServiceSubscription {
  id: string;
  name: string;
  billingCycle: BillingCycle;
  rawCostCents: number;           // e.g., 2000 for $20.00/mo or 24000 for $240.00/yr
  normalizedMonthlyCents: number; // rawCostCents (monthly) or Math.round(rawCostCents / 12) (annual)
  startMonth: string | null;      // e.g., '2026-01' or null if open-ended
  endMonth: string | null;        // e.g., '2026-12' or null if currently active
  defaultSplitMode: SplitMode;
  defaultSplitMetric: SplitMetricBasis;
  isActive: boolean;
}

export interface AllocationInput {
  projectSlug: string;
  usageUnits: number; // >= 0 (derived from manual input, recorded MAU, or monthly token count)
}

export interface AllocationResult {
  projectSlug: string;
  effectiveSplitMode: SplitMode;
  fellBackToEven: boolean;
  usageUnits: number;
  shareRatio: number;
  apportionedCents: number;
}

export interface TokenUsageEvent {
  idempotencyKey: string;
  projectSlug: string;
  sessionId: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  cacheReadTokens: number;
  recordedAt: string; // ISO 8601 with offset
}
```

---

# 3\. Database Schema & External Contracts

## Database Schema Overview (`src/data/schema.ts`)

| Table Name | Primary Key | Key Columns & Indexes | Description |
| :---- | :---- | :---- | :---- |
| `projects` | `id` (uuid) | `slug` (text, unique, not null), `name` (text, not null), `status` (text, default `'active'`) | Core project registry |
| `services` | `id` (uuid) | `name` (text, not null), `billing_cycle` (text, not null), `raw_cost_cents` (integer, not null), `start_month` (text, nullable), `end_month` (text, nullable) | Shared subscriptions/services |
| `project_service_allocations` | `id` (uuid) | `(billing_month, service_id, project_id)` \[Unique Index\] | Apportioned service costs per project per month |
| `monthly_project_metrics` | `id` (uuid) | `(billing_month, project_id)` \[Unique Index\] | Manual metrics: users, MAU, direct cost, revenue |
| `ai_token_logs` | `id` (uuid) | `idempotency_key` (text, unique, not null), `project_id` (uuid, nullable), `unassigned_project_slug` (text, nullable), `cost_microcents` (int, not null), `cache_read_tokens` (int, default 0), `billing_month` (text, indexed) | Ingested AI token usage events with locked costs |
| `model_rates` | `model_pattern` (text) | `input_microcents_per_1k` (int), `output_microcents_per_1k` (int), `cache_input_microcents_per_1k` (int) | Pricing map for token cost calculations |
| `owner_passkeys` | `credential_id` (text) | `public_key` (text, not null), `counter` (bigint, not null) | WebAuthn credentials for owner auth |

> [!NOTE]
> All monthly apportionment recalculation writes must be executed atomically using `db.batch([...])` to prevent partial deletes or orphaned allocations during serverless execution.

## Ingestion REST Contract (`POST /api/ingest/tokens`)

- **Headers:**  
  - `Authorization: Bearer <INGEST_API_KEY>`  
  - `Content-Type: application/json`  
- **Request Body Payload:**

```json
{
  "events": [
    {
      "idempotencyKey": "session-abc:msg-004",
      "projectSlug": "sun-scanner",
      "sessionId": "session-abc",
      "model": "gemini-3-pro",
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

## Required Environment Variables

| Variable | Description |
| :---- | :---- |
| `DATABASE_URL` | Neon HTTP pooled connection string |
| `AUTH_SECRET` | 32+ byte random hex string for signing `pd_session` JWT cookies |
| `OWNER_PASSWORD_HASH` | Scrypt hash of initial owner login password |
| `WEBAUTHN_RP_ID` | Production domain (e.g., `project-dashboard.vercel.app`) |
| `WEBAUTHN_ORIGIN` | Full HTTPS origin URL |
| `INGEST_API_KEY` | Secret bearer token shared with `~/.config/project-dashboard/config.json` |

---

# 4\. Test Strategy & Fixtures

## Domain Unit Tests (`src/domain/__tests__/`)

- `apportionment.test.ts`:  
  - Tests `$10.00` split 3 ways (`334, 333, 333` cents using slug-sorted remainder logic).  
  - Pro-rata split with partial zero usage (`100, 300, 0`).  
  - Pro-rata all-zero fallback (`0, 0, 0` -> even split).  
  - Annual service normalization (`$100/yr` -> `833` cents/mo).  
  - Graceful zero handling for 0 linked projects.  
  - Automated metric binding calculation tests (`mau`, `tokens`).  
- `tokenPricing.test.ts`:  
  - Verifies micro-cent cost calculation with `inputTokens`, `outputTokens`, and discounted `cacheReadTokens`.  
  - Verifies `toMelbourneBillingMonth` across daylight saving time shifts (e.g., April and October boundary timestamps).  
- `rollups.test.ts`:  
  - Verifies monthly cost rollups and net loss handling using locked token costs.  
  - Verifies multi-month lifetime cumulative financial sums.

## API Integration Tests

- Validates `401 Unauthorized` response when supplying an invalid Bearer token.  
- Validates `200 OK` idempotent replay without duplicate inserts or double-counting when submitting an existing `idempotencyKey`.  
- Validates timezone-aware month boundary bucketing (`Australia/Melbourne`).  
- Validates unrecognized project slugs persist with `unassigned_project_slug` and return `unassigned: 1`.

## Hook Script Tests (`scripts/test_log_antigravity_tokens.py`)

- Tests parsing camelCase stdin parameters (`transcriptPath`, `workspacePaths`, `conversationId`).  
- Tests token extraction for `input_tokens`, `output_tokens`, and `cache_read_tokens`.  
- Tests fallback project slug resolution when `.dashboard-project.json` is missing.  
- Tests cursor state advancement in `~/.cache/antigravity-token-cursors.json`.  
- Tests offline `.jsonl` queue buffering during API network failures.  
- Tests automatic queue flushing upon successful execution using `unittest` and mock transcript JSON fixtures.