# Design System & UI Guidelines: project-dashboard

**Version:** 1.0  
**Governed By:** `CONSTITUTION.md`

This document defines the visual tokens, spacing scale, typography, accessibility rules, and data presentation standards for `project-dashboard`. All UI components must strictly reference these CSS custom properties and Tailwind semantic tokens.

# 1\. Semantic Color Tokens (Light & Dark Mode)

- **Rule:** Never hardcode raw hex color literals inside feature pages or components. Always reference semantic CSS variables (`--surface-primary`, `--text-primary`, etc.) mapped in `globals.css`.

| Semantic Token | Purpose | Light Mode Hex | Dark Mode Hex | Min Contrast Ratio |
| :---- | :---- | :---- | :---- | :---- |
| `--surface-primary` | Main viewport background | `#F8FAFC` | `#090D16` | `>= 4.5:1` |
| `--surface-card` | Elevated KPI cards & data tables | `#FFFFFF` | `#111827` | `>= 4.5:1` |
| `--surface-subtle` | Table row hover & muted panels | `#F1F5F9` | `#1E293B` | `>= 4.5:1` |
| `--border-subtle` | Card outlines & table dividers | `#E2E8F0` | `#1F2937` | `>= 3.0:1` |
| `--text-primary` | Primary headings & metric values | `#0F172A` | `#F8FAFC` | `>= 7.0:1` |
| `--text-secondary` | Labels, timestamps & subtext | `#475569` | `#94A3B8` | `>= 4.5:1` |
| `--accent-primary` | Primary CTAs, active tabs & links | `#0284C7` | `#38BDF8` | `>= 4.5:1` |

## Domain Financial & Status Palette

| Semantic Token | Domain State | Light Badge Fill | Light Foreground | Dark Badge Fill | Dark Foreground |
| :---- | :---- | :---- | :---- | :---- | :---- |
| `--status-profit` | Net Profit (`> $0`) / Active | `#DCFCE7` | `#166534` | `#052E16` | `#4ADE80` |
| `--status-loss` | Net Loss (`< $0`) / Ingestion Error | `#FEE2E2` | `#991B1B` | `#450A0A` | `#F87171` |
| `--status-neutral` | Break-even (`$0`) / Paused | `#F1F5F9` | `#334155` | `#1E293B` | `#CBD5E1` |
| `--status-prorata` | Pro-Rata Apportionment Mode | `#E0F2FE` | `#075985` | `#082F49` | `#38BDF8` |
| `--status-even` | Even Split Apportionment Mode | `#FEF3C7` | `#92400E` | `#451A03` | `#FBBF24` |

# 2\. Spacing & Layout Scale

- **Grid Unit:** `4px` (`0.25rem`) base unit using standard Tailwind spacing steps:  
  - `space-xs`: `4px` (`gap-1`) — inline icon-to-label gaps  
  - `space-sm`: `8px` (`gap-2`) — compact badge & form label spacing  
  - `space-md`: `16px` (`p-4` / `gap-4`) — internal card padding & form grids  
  - `space-lg`: `24px` (`p-6` / `gap-6`) — major dashboard grid gutters  
  - `space-xl`: `32px` (`mb-8`) — page header to content separation  
- **Shapes & Elevation:**  
  - KPI & Project Cards: `12px` (`rounded-xl`) with `1px solid var(--border-subtle)`.  
  - Pills, Mode Badges & Status Chips: `6px` (`rounded-md`) or full pill (`rounded-full`).  
- **Responsive Breakpoints:**  
  - Mobile (`< 768px`): Single-column stacked KPI cards; horizontally scrollable financial tables with sticky project name column.  
  - Desktop (`>= 1024px`): 4-column executive summary strip above a 2- or 3-column project portfolio grid / dense comparison matrix.

# 3\. Typography & Numeric Formatting Scale

- **Font Stack:** `Inter` (UI sans-serif) and `JetBrains Mono` (monospace for slugs, token counts, and API keys).  
- **Tabular Numerals (`font-variant-numeric: tabular-nums`):** Mandatory on all currency readouts, token counts, user counts, and percentage split columns so numbers align vertically across rows and tables.  
- **Type Scale:**  
  - `display-kpi`: `1.875rem` (`30px`), `font-bold`, `tabular-nums` — Top-level portfolio KPIs.  
  - `title-section`: `1.25rem` (`20px`), `font-semibold` — Page and section headers.  
  - `title-card`: `1rem` (`16px`), `font-semibold` — Project card titles and modal headers.  
  - `body-primary`: `0.875rem` (`14px`), `font-normal` — Table cells, forms, and descriptions.  
  - `label-caption`: `0.75rem` (`12px`), `font-medium` — Column headers, timestamps, and mode badges.

# 4\. Accessibility Guardrails (WCAG AA+)

1. **Color Independence:** Never convey net profit/loss or split mode solely by color. Always prefix net loss values with `-` (e.g., `-$42.50`) and pair status colors with explicit text badges (`Pro-Rata`, `Even Split`, `Active`, `Paused`).  
2. **Interactive Targets:** All buttons, month selectors, table action triggers, and form inputs must have a minimum hit area of `44px x 44px` on touch viewports and clear `:focus-visible` outlines (`2px solid var(--accent-primary)`).  
3. **Keyboard & Screen Reader Semantics:** Use semantic `<table>`, `<thead>`, `<th scope="col">`, and `<caption>` elements for financial ledgers; provide `aria-label` attributes on icon-only buttons and month navigation arrows.  
4. **Reduced Motion:** Respect `prefers-reduced-motion: reduce` by disabling non-essential card transitions.

# 5\. Data Presentation & Formatting Standards

- **Currency Formatting:** Display standard monetary totals to 2 decimal places (`$124.50`); display micro AI token costs under `$1.00` to 4 decimal places (`$0.0412`) so small session increments remain visible.  
- **Token Formatting:** Format large token totals cleanly (`1.42M` for `>= 1,000,000`; `48.2K` for `>= 1,000`) with exact integer counts available in hover tooltips or detail views.  
- **Monthly vs. Lifetime Pairing:** Always visually distinguish **Selected Month** metrics from **Lifetime Cumulative** metrics using clear column groupings or segmented card sections so monthly burn is never confused with lifetime totals.