# NCT Feasibility Input Tool — Spec

## What it is

Single-file browser tool (`NCT_Feasi_Input.html`) for entering property development feasibility numbers at sub-phase and component granularity. Saves versioned snapshots to Supabase for downstream analytics in Fabric / Power BI.

No backend, no framework, no build step. Open the file directly in a browser or via `npx serve`.

---

## How to use

1. Open `NCT_Feasi_Input.html` (or `index.html` which redirects to it).
2. Fill **Project Details** (L1–L3 header: company, plot code, location, project code/name).
3. Select **Stage**: BD → R1 → R2 → R3 (sequential revisions).
4. Add **Phases** (L3) via the `+` tab button.
5. Each phase has one or more **Sub-phases** (L4). Add via `+ Add Sub-phase`.
6. Enter **Common Cost** (RM total) above the matrix — auto-allocated to components by NFA ratio.
7. Add **Components** (L5) via `+ Add`. For each component column, fill:
   - Type, Units, NFA/Unit, ASP (RM psf)
   - Bumi Discount %, Cross-subsidy (RM), Carpark Revenue (RM)
8. Expand pillars (P1–P5) to enter cost line items per component.
9. Click **Save** to write a new snapshot to Supabase.
10. Click **Load** to browse and restore any saved snapshot.
11. **Report** exports an Excel summary; **Pipeline** exports flat tables ready for Fabric import.

---

## Data model (4-table Supabase schema)

Join key between tables: `(snapshot_id, l3_phase, l4_subphase, l5_component)`

### `snapshots`
Lightweight version index. One row per save.

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | auto |
| project_name | TEXT | |
| project_code | TEXT | |
| stage | TEXT | BD / R1 / R2 / R3 |
| scenario | TEXT | Base / Upside / Downside |
| status | TEXT | Draft / Final / Approved |
| version_date | DATE | |
| wacc | NUMERIC(6,4) | decimal e.g. 0.08 |
| created_at | TIMESTAMPTZ | auto |

### `hierarchy`
One row per L4 subphase. Carries the L1–L4 path.

| Column | Type | Notes |
|---|---|---|
| snapshot_id | UUID FK | → snapshots |
| l1_company | TEXT | |
| l2_plot_code | TEXT | |
| l2_plot_name | TEXT | |
| l3_phase | TEXT | e.g. "Phase 1" |
| l4_subphase | TEXT | e.g. "Sub-phase A" |
| l3_sort_order | INT | for ordering phases |
| l4_sort_order | INT | for ordering subphases |

### `sales`
One row per L5 component. Revenue inputs only — no computed columns stored.

| Column | Type | Notes |
|---|---|---|
| snapshot_id | UUID FK | |
| l3_phase / l4_subphase / l5_component | TEXT | join key |
| product_type | TEXT | Residential / Commercial / etc. |
| units | INT | |
| nfa_per_unit | NUMERIC(12,2) | sqft |
| asp_psf | NUMERIC(12,2) | RM per sqft |
| bumi_pct | NUMERIC(5,2) | percentage 0–100 |
| cross_subsidy_amt | NUMERIC(18,2) | RM |
| carpark_revenue | NUMERIC(18,2) | RM |
| sort_order | INT | column order |

Derived values (Gross GDV, Bumi Amt, Net GDV) are computed in the app and in Fabric — not stored.

### `cost`
One row per cost line item. Two special cases:

| Sentinel | Meaning |
|---|---|
| `l5_component = ''`, `pillar_code = 'COMMON'` | Common cost total for that subphase |
| `pillar_code = 'P1'…'P5'` | Standard pillar line item per component |

| Column | Type | Notes |
|---|---|---|
| snapshot_id | UUID FK | |
| l3_phase / l4_subphase / l5_component | TEXT | join key |
| pillar_code | TEXT | P1–P5 or COMMON |
| item_code | TEXT | e.g. L.PURCHASE, C.PILE |
| description | TEXT | |
| budget_amt | NUMERIC(18,2) | RM |
| sort_order | INT | |

Common cost allocation per component = `budget_amt × (comp_nfa / total_nfa)` — computed at read time, not stored.

---

## Supabase connection

**URL:** `https://vtqjcfiobexntgjyiteu.supabase.co`  
**Frontend key (publishable):** in `NCT_Feasi_Input.html` as `SB_KEY` — safe to commit.  
**Secret key:** server-side only, never in frontend HTML.

Auth: disabled (pilot phase). No RLS policies required for internal pilot.

---

## Save / Load flow

**Save:**
1. POST to `/rest/v1/snapshots` with `Prefer: return=representation` → get `id`
2. Parallel POST to `/rest/v1/hierarchy`, `/rest/v1/sales`, `/rest/v1/cost` with `snapshot_id` injected

**Load:**
1. GET `/rest/v1/snapshots?select=id,project_name,stage,version_date&order=created_at.desc`
2. User picks a row
3. Parallel GET of all 4 tables filtered by `id=eq.<snapshot_id>`
4. UI reconstructed from flat data

**No update / delete in the app.** Each Save creates a new snapshot. Comparison between stages is done in Fabric, not here.

---

## Computation rules (app-side)

| Value | Formula |
|---|---|
| Total NFA | `units × nfa_per_unit` per component |
| Gross GDV | `total_nfa × asp_psf` |
| Bumi Amt | `gross_gdv × bumi_pct / 100` |
| Net GDV | `gross_gdv − bumi_amt − cross_subsidy + carpark_revenue` |
| CC Alloc | `cc_total × (comp_nfa / subphase_total_nfa)` |
| Total GDC | `cc_alloc + sum(all pillar item budgets)` |
| PBT | `net_gdv − total_gdc` |
| Margin | `pbt / net_gdv` |

---

## Fabric migration notes

Schema uses only standard PostgreSQL — no Supabase-specific extensions beyond `gen_random_uuid()` (available as `uuid_generate_v4()` in most Postgres installs). When migrating to Fabric:
- Replace `gen_random_uuid()` with `uuid_generate_v4()` or application-generated UUIDs
- Drop `ON DELETE CASCADE` if Fabric doesn't support it; handle orphan cleanup in ETL
- All `NUMERIC` types and `TIMESTAMPTZ` are standard and port directly

---

## File inventory

| File | Purpose |
|---|---|
| `NCT_Feasi_Input.html` | Main tool (the only app file) |
| `index.html` | Redirect to input tool |
| `NCT_Feasi_Demo.html.bak` | Archived demo view (not used) |
| `SPEC.md` | This document |
| `DB_DESIGN.md` | Full ERD + star schema for future Fabric analytical layer |
