# Agent Guidelines & Repository Architecture (`AGENTS.md`)

This document defines the operational rules, architectural map, and Git workflow guardrails for all AI agents working inside this repository.

---

## 1. Branching & Git Workflow Rules (Strict Guardrails)

> [!CAUTION]
> **Zero Direct Commits or Pushes to `main`. No Automated Merging.**

1. **Always Work on a Feature Branch:**
   - Every modification (whether feature implementation, bug fix, or documentation update) **must** be performed on a dedicated branch branched off `main` (e.g., `feat/<name>`, `task/<number>-<description>`, or `fix/<name>`).
   - Never commit directly to `main`.
2. **Never Push Directly to `main`:**
   - The `main` branch is protected. Direct pushes (`git push origin main`) are strictly prohibited.
   - Pushes must target your working feature branch (e.g., `git push origin feat/...`).
3. **Always Create a Pull Request (PR):**
   - Once work and verification gates are complete, open a Pull Request targeting `main`.
4. **AI Agents Must Never Merge into `main`:**
   - The AI agent **must not** merge any branch or Pull Request into `main` (neither via `git merge`, CLI, nor GitHub API).
   - Merging into `main` is strictly reserved for the human repository owner after reviewing the PR.

---

## 2. Architecture & Governance File Map

Before generating code or executing tasks, review the authoritative governance documents:

| Document | Purpose & Authority | Key Rules to Obey |
| :--- | :--- | :--- |
| [CONSTITUTION.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/CONSTITUTION.md) | **Supreme Architectural Authority.** Immutable tech stack rules, domain boundaries, and coding standards. | • Zero framework/DB imports in domain layer.<br>• Integer cents for financial math.<br>• Slug is immutable project identity.<br>• Atomic `db.batch` recalculations. |
| [DESIGN_SYSTEM.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/DESIGN_SYSTEM.md) | Visual styling, design tokens, typography, and accessibility. | • Never hardcode hex colors; use CSS variables (`--surface-primary`, `--accent-primary`).<br>• Tabular numerals (`tabular-nums`) on all numeric readouts.<br>• WCAG AA+ minimum 44x44px touch targets. |
| [mvp-project-dashboard/SPEC.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/mvp-project-dashboard/SPEC.md) | Feature specification, functional requirements (`FR-xxx`), business rules (`BR-xxx`), edge cases, and acceptance criteria (`AC-xxx`). | • Token-weighted AI cost apportionment.<br>• Base subscription + credit pack adjustments.<br>• Unassigned telemetry preservation. |
| [mvp-project-dashboard/PLAN.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/mvp-project-dashboard/PLAN.md) | Technical implementation architecture, Drizzle schemas, domain model interfaces, API contracts, and test plans. | • Extensible cost ledger schema (`cost_items`, `monthly_cost_entries`, `project_cost_allocations`).<br>• Timezone bucketing in `Australia/Melbourne`. |
| [mvp-project-dashboard/TASKS.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/mvp-project-dashboard/TASKS.md) | Phased work breakdown structure with explicit verification gates. | • Execute **one task per session**.<br>• Run verification commands before marking complete. |
| [README.md](file:///Users/tom/code/splishsploshsplash/ProjectDashboard/README.md) | High-level repository overview, quickstart, and development instructions. | • Project setup and overview. |

---

## 3. Directory Layout Guardrails

The repository enforces strict physical directory separation:

```text
ProjectDashboard/
├── CONSTITUTION.md             # Supreme architecture & coding rules
├── DESIGN_SYSTEM.md            # Semantic design tokens & typography
├── AGENTS.md                   # Agent guidelines & Git workflow rules
├── README.md                   # Repo overview
├── mvp-project-dashboard/      # Specification suites (SPEC, PLAN, TASKS)
├── scripts/                    # Antigravity IDE hook scripts
└── app/                        # Dedicated Next.js web application root
    ├── package.json
    ├── tsconfig.json
    ├── drizzle.config.ts
    └── src/
        ├── domain/             # Pure TypeScript domain logic (NO framework imports)
        ├── data/               # Drizzle PostgreSQL schema & Neon client
        ├── app/                # Next.js App Router (pages, actions, APIs, middleware)
        └── components/         # Reusable UI components
```

- **Rule:** **Do NOT** create application source files or package configurations in the repository root. All application source code must be placed inside `app/`.
- **Rule:** **Do NOT** place business logic or domain math in UI components or route handlers; implement pure domain logic inside `app/src/domain/`.

---

## 4. Execution Workflow

When tasked with executing work:
1. **Branch Out:** `git checkout -b <branch-name> main`
2. **Review Docs:** Read relevant sections of `CONSTITUTION.md`, `DESIGN_SYSTEM.md`, `SPEC.md`, and `PLAN.md`.
3. **Tests First:** Write failing unit tests in `app/src/domain/__tests__/` before implementing code.
4. **Implement Code:** Write clean, typed, modular code adhering to all constraints.
5. **Run Verification Gates:**
   ```bash
   cd app && npm run typecheck && npm run lint && npm test && npm run build
   ```
6. **Commit & Push Branch:** Commit with clean semantic commit messages, then push your feature branch:
   ```bash
   git push origin <branch-name>
   ```
7. **Create Pull Request:** Create a PR to `main` and notify the owner. **Do not merge it.**
