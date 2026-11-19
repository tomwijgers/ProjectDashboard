# Project Dashboard

A personal single-user executive web dashboard designed to track **Google Antigravity** AI token consumption on a per-project basis (web applications, mobile apps, 3D asset generation, and creative media workflows), manage monthly AI subscription costs (including credit top-ups), and apportion those costs across active projects proportionally based on token usage.

Built on an **extensible cost ledger architecture**, the dashboard is designed to seamlessly accommodate future fixed subscriptions (hosting, databases, SaaS tools) and metered API token costs without database redesigns.

---

## Architecture & Directory Layout

To maintain clear separation between architectural governance, specification suites, background hooks, and implementation code, all application source code resides in the dedicated `app/` subfolder:

```text
ProjectDashboard/
├── CONSTITUTION.md             # Immutable architectural guardrails & tech stack standards
├── DESIGN_SYSTEM.md            # Semantic design tokens, typography, and WCAG AA+ guidelines
├── README.md                   # Repository overview & development guide
│
├── mvp-project-dashboard/      # Specification-Driven Development (SDD) feature suite
│   ├── SPEC.md                 # Functional requirements, edge cases & acceptance criteria
│   ├── PLAN.md                 # Architecture, domain models, Drizzle schema & API contracts
│   └── TASKS.md                # Phase-by-phase task checklists & verification gates
│
├── scripts/                    # Antigravity IDE lifecycle integration scripts
│   ├── log_antigravity_tokens.py       # Stop hook: reads transcripts, buffers offline, sends deltas
│   └── test_log_antigravity_tokens.py  # Unit test suite for hook & offline retry queue
│
└── app/                        # Next.js 15 Web Application Root
    ├── package.json            # Node.js dependencies & scripts
    ├── tsconfig.json           # Strict TypeScript configuration
    ├── drizzle.config.ts       # Drizzle ORM database migration configuration
    └── src/
        ├── domain/             # Pure TypeScript models, apportionment engine & rollups (zero-dependency)
        ├── data/               # Drizzle PostgreSQL schema, Neon serverless client & repositories
        ├── app/                # Next.js App Router (pages, Server Actions, Route Handlers, middleware)
        └── components/         # Reusable UI components strictly styled with semantic tokens
```

---

## Core Features (Phase 1 MVP)

- **Antigravity Token Intelligence:**
  - Automated tracking of `input_tokens`, `output_tokens`, and prompt `cache_read_tokens` per project.
  - Standard-library Python Stop hook running in `.agents/hooks.json` with local cursor management (`~/.cache/antigravity-token-cursors.json`).
  - Resilient offline queue buffering (`~/.cache/antigravity-tokens-queue.jsonl`) that never blocks or fails IDE turns during network outages.
- **Project Portfolio Management:**
  - Project registry with editable display names, repository paths, and lifecycle statuses (`active`, `paused`, `archived`).
  - **Immutable Slug Identity:** The project slug cannot be modified after creation, serving as the permanent foreign key for telemetry events.
- **Extensible Cost Management:**
  - Tracks monthly AI subscription spend (defaults to base subscription, e.g. \$33 AUD).
  - Inline monthly budget editor in the dashboard UI to adjust for AI credit pack purchases (e.g. updating a month to \$53 AUD).
- **Deterministic Token-Weighted Apportionment:**
  - Proportional cost distribution based on each project's share of total tokens in that billing month.
  - Integer-cent calculation using the **Largest Remainder Method** to ensure apportioned amounts sum exactly to the monthly cost entry without floating-point drift.
- **Single-User Perimeter:**
  - Strict route protection via Next.js `middleware.ts` using signed HTTP-only JWT cookies (`jose`).
  - Owner password login (`scrypt`) with WebAuthn Passkey registration/authentication from authenticated settings.
  - Timing-safe Bearer token authorization (`INGEST_API_KEY`) for external telemetry ingestion.

---

## Technology Stack

- **Framework:** Next.js 15+ (App Router with asynchronous route contracts)
- **UI & Styling:** React 19, Tailwind CSS v4 (CSS-first `@theme` tokens)
- **Language:** TypeScript 5+ (strict mode enabled)
- **Database & ORM:** Neon Serverless Postgres (`@neondatabase/serverless`) with Drizzle ORM
- **Testing:** Vitest (domain & integration tests) and Python `unittest` (hook script tests)
- **Timezone Boundary:** `Australia/Melbourne` local calendar month bucketing via `Intl.DateTimeFormat`

---

## Development & Verification Workflow

All application commands are executed from the `app/` directory:

```bash
# Navigate to application workspace
cd app

# Install dependencies
npm install

# Run type checking & linting
npm run typecheck
npm run lint

# Run domain & integration tests
npm test

# Generate database migrations
npx drizzle-kit generate

# Run Next.js production build verification
npm run build
```

Hook script unit tests are run from the repository root:

```bash
python3 -m unittest scripts/test_log_antigravity_tokens.py
```

---

## Specification-Driven Development (SDD) Rules

1. **One Task Per Session:** Follow the phased checklist in [mvp-project-dashboard/TASKS.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/mvp-project-dashboard/TASKS.md).
2. **Test-First Implementation:** Write failing unit tests in `app/src/domain/__tests__/` before implementing domain code.
3. **Verification Gates:** Run type check, linting, tests, and builds before marking any task `[COMPLETED]`.
4. **Separation of Concerns:** Keep `app/src/domain/` completely free of framework, database, or HTTP imports.
