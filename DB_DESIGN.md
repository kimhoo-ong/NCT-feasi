# NCT Feasibility — Database Design (ERD · Schema · Star Schema)

Backing store for the NCT property-development feasibility / costing system.
Mirrors the logic already in `NCT_Costing_Template.xlsx` and
`NCT_Master_Costing_MultiPhase.xlsx`, and extends it to the CEO-FZ depth seen in
the LIDO / FZ / Std-Feasi reference templates (cash flow, IRR, multi-component,
versioning).

---

## 0. Layering

```
  INPUT (app / Excel import)          ANALYTICS (dashboards, BI)
  ──────────────────────────          ───────────────────────────
  OLTP  (3NF, normalised)   ──ETL──►  STAR SCHEMA (dimensional marts)
  write-optimised, 1 row/fact         read-optimised, conformed dims
```

- **OLTP / 3NF** — what the data-entry app writes to. One source of truth, no
  duplication, referential integrity. Section 2.
- **Star schema** — what dashboards query. De-normalised dimensions + numeric
  fact tables. Section 3. *This is the part you asked to prioritise.*
- Keep both. Don't run dashboards on the OLTP tables; don't enter data into the
  star schema.

Core design principle: **a feasibility study is a versioned document.**
Every fact carries a `version_key` (FZ V29 vs V30, budget snapshot vs actuals
snapshot, scenario A vs B). This is the single most important decision — it lets
you compare versions and freeze board-approved numbers.

---

## 1. Grain decisions (decide these first)

| Fact | Grain (1 row = …) | Additive? |
|---|---|---|
| `fact_cost_line` | one cost line item, per version × phase × component | budget/committed/actual: **yes**; psf: no |
| `fact_cashflow` | one period (year), per version × phase | outflow/inflow/net: **yes**; cumulative: semi |
| `fact_feasi_summary` | one phase snapshot, per version | totals: yes across phases; **IRR/NPV/margin: NO** |
| `fact_revenue` *(opt.)* | one revenue component, per version × phase | gdv/units: yes |

> ⚠️ `irr`, `npv`, `profit_margin`, `efficiency_pct`, any `*_psf` and `*_pct`
> are **non-additive ratios**. Store them at summary grain only and never `SUM()`
> them — re-derive from the additive base measures when aggregating.

---

## 2. Operational model (3NF) — ERD

Entities and their key columns. `PK` primary key, `FK→` foreign key.

```
developer(developer_id PK, name, type)
   └─< project

project(project_id PK, developer_id FK→developer, name, location,
        gross_land_acre, country, region, created_at)
   ├─< jv_partner
   ├─< fz_version
   └─< phase

jv_partner(partner_id PK, project_id FK→project, name,
           equity_pct, land_contribution, profit_share_pct)

fz_version(version_id PK, project_id FK→project, version_label,        -- 'FZ V30'
           status_flag, wacc, is_current, snapshot_type,               -- status: 精算/概念/TBC
           snapshot_date, created_by, notes)                           -- snapshot: budget|committed|actual

phase(phase_id PK, project_id FK→project, name, maturity_flag,         -- 精算✓/概念◑/TBC○
      target_launch, target_vp, construction_months,
      nda_acre, plot_ratio, efficiency_pct)
   ├─< plot
   └─< component

plot(plot_id PK, phase_id FK→phase, name, area_acre, plot_ratio,       -- optional (LIDO-style)
     land_psf, land_value)

component(component_id PK, phase_id FK→phase, classification_code,      -- IP1.. / HR1.. / IND.. / R..
          classification_group, product_type, tenure,                  -- tenure: sell|lease|hold
          gfa_sqft, efficiency_pct, nfa_sqft, units, asp_psf, ratio_pct)

cost_pillar(pillar_id PK, code, name, target_pct_low, target_pct_high) -- P1..P5
   └─< cost_item

cost_item(item_id PK, pillar_id FK→cost_pillar, parent_item_id FK→cost_item,
          code, name, level, sort_order)                               -- self-ref hierarchy

cost_entry(entry_id PK, version_id FK→fz_version, phase_id FK→phase,
           item_id FK→cost_item, component_id FK→component NULL,
           budget, committed, actual, note)                            -- the cost OLTP grain

revenue_assumption(rev_id PK, version_id FK→fz_version, phase_id FK→phase,
           component_id FK→component, gross_gdv, bumi_discount,
           affordable_subsidy, promo_packages, carpark_revenue, net_gdv)

curve(curve_id PK, version_id FK→fz_version, phase_id FK→phase,
      curve_type, period_index, pct)                                   -- spend|collection|billing|ros

cashflow_entry(cf_id PK, version_id FK→fz_version, phase_id FK→phase,
      period_index, outflow, inflow)                                   -- derivable from curve+totals

assumption(assumption_id PK, version_id FK→fz_version, key, value, unit)-- gearing, rates, fees

benchmark(benchmark_id PK, category, item, low, mid, high, basis,      -- the BENCHMARK sheet
          source, as_of)
```

Relationship summary: `developer 1─< project 1─< phase 1─< component`;
`project 1─< fz_version`; every transactional table hangs off
(`version`, `phase`[, `component`]).

---

## 3. Star schema (dimensional marts)

Three fact tables sharing conformed dimensions = a **fact constellation**
(galaxy schema). Dimensions are reused across all facts.

### 3.1 Dimensions

```sql
dim_project   (project_key PK, project_id, project_name, developer_name,
               location, country, region, gross_land_acre)
dim_phase     (phase_key PK, phase_id, project_key FK, phase_name,
               maturity_flag, product_mix, construction_months,
               nda_acre, plot_ratio, efficiency_pct)
dim_version   (version_key PK, version_id, version_label, status_flag,
               snapshot_type, wacc, is_current, snapshot_date)         -- SCD-2 friendly
dim_component (component_key PK, classification_code, classification_group,
               product_type, tenure)                                   -- sell|lease|hold
dim_pillar    (pillar_key PK, pillar_code, pillar_name,
               target_pct_low, target_pct_high)
dim_cost_item (cost_item_key PK, item_code, item_name, pillar_code,
               parent_item_name, level)
dim_date      (date_key PK, full_date, year, quarter, month, month_name)
dim_period    (period_key PK, period_index, period_label, is_relative) -- 'Y0','Y1'… relative timeline
```

Conformed (shared by every fact): `dim_project`, `dim_phase`, `dim_version`.

### 3.2 Fact tables

```sql
fact_cost_line                         -- grain: version × phase × component × cost_item
  (cost_line_key PK,
   version_key FK, project_key FK, phase_key FK, component_key FK,
   pillar_key FK, cost_item_key FK, date_key FK,
   budget_amt, committed_amt, actual_amt, variance_amt,    -- additive
   budget_psf_nfa, actual_psf_nfa, pct_of_gdv)             -- non-additive

fact_cashflow                          -- grain: version × phase × period
  (cashflow_key PK,
   version_key FK, project_key FK, phase_key FK, period_key FK, date_key FK,
   outflow, inflow, net_cf,                                -- additive
   cumulative_cf, discount_factor, discounted_net_cf)      -- semi/non-additive

fact_feasi_summary                     -- grain: version × phase (one snapshot row)
  (feasi_key PK,
   version_key FK, project_key FK, phase_key FK, date_key FK,
   gross_gdv, net_gdv, gdc, tcc, pbt, total_units, nfa_sqft,  -- additive across phases
   profit_margin, irr, npv, peak_funding, payback_years)      -- NON-additive

fact_revenue                           -- (optional) grain: version × phase × component
  (revenue_key PK,
   version_key FK, project_key FK, phase_key FK, component_key FK,
   gross_gdv, total_discounts, net_gdv, units, nfa_sqft, asp_psf)
```

---

## 4. DDL (PostgreSQL)

```sql
-- ============ DIMENSIONS ============
CREATE TABLE dim_version (
  version_key    SERIAL PRIMARY KEY,
  version_id     INT,
  version_label  TEXT,
  status_flag    TEXT,          -- 精算 / 概念 / TBC
  snapshot_type  TEXT,          -- budget | committed | actual
  wacc           NUMERIC(6,4),
  is_current     BOOLEAN DEFAULT FALSE,
  snapshot_date  DATE
);
CREATE TABLE dim_project (
  project_key      SERIAL PRIMARY KEY,
  project_id       INT,
  project_name     TEXT,
  developer_name   TEXT,
  location         TEXT,
  country          TEXT,
  region           TEXT,
  gross_land_acre  NUMERIC(12,3)
);
CREATE TABLE dim_phase (
  phase_key           SERIAL PRIMARY KEY,
  phase_id            INT,
  project_key         INT REFERENCES dim_project,
  phase_name          TEXT,
  maturity_flag       TEXT,
  product_mix         TEXT,
  construction_months INT,
  nda_acre            NUMERIC(12,3),
  plot_ratio          NUMERIC(6,2),
  efficiency_pct      NUMERIC(5,4)
);
CREATE TABLE dim_component (
  component_key        SERIAL PRIMARY KEY,
  classification_code  TEXT,    -- IP1.. HR1.. IND.. R..
  classification_group TEXT,    -- Investment | HighRise | Industrial | Landed
  product_type         TEXT,
  tenure               TEXT     -- sell | lease | hold
);
CREATE TABLE dim_pillar (
  pillar_key      SERIAL PRIMARY KEY,
  pillar_code     TEXT,         -- P1..P5
  pillar_name     TEXT,
  target_pct_low  NUMERIC(5,4),
  target_pct_high NUMERIC(5,4)
);
CREATE TABLE dim_cost_item (
  cost_item_key    SERIAL PRIMARY KEY,
  item_code        TEXT,
  item_name        TEXT,
  pillar_code      TEXT,
  parent_item_name TEXT,
  level            INT
);
CREATE TABLE dim_date (
  date_key   INT PRIMARY KEY,  -- yyyymmdd
  full_date  DATE,
  year       INT, quarter INT, month INT, month_name TEXT
);
CREATE TABLE dim_period (
  period_key   SERIAL PRIMARY KEY,
  period_index INT,            -- 0,1,2…
  period_label TEXT,           -- 'Y0','Y1'…
  is_relative  BOOLEAN DEFAULT TRUE
);

-- ============ FACTS ============
CREATE TABLE fact_cost_line (
  cost_line_key  BIGSERIAL PRIMARY KEY,
  version_key    INT REFERENCES dim_version,
  project_key    INT REFERENCES dim_project,
  phase_key      INT REFERENCES dim_phase,
  component_key  INT REFERENCES dim_component,
  pillar_key     INT REFERENCES dim_pillar,
  cost_item_key  INT REFERENCES dim_cost_item,
  date_key       INT REFERENCES dim_date,
  budget_amt     NUMERIC(18,2),
  committed_amt  NUMERIC(18,2),
  actual_amt     NUMERIC(18,2),
  variance_amt   NUMERIC(18,2),   -- budget - actual
  budget_psf_nfa NUMERIC(12,4),
  actual_psf_nfa NUMERIC(12,4),
  pct_of_gdv     NUMERIC(8,6)
);
CREATE TABLE fact_cashflow (
  cashflow_key     BIGSERIAL PRIMARY KEY,
  version_key      INT REFERENCES dim_version,
  project_key      INT REFERENCES dim_project,
  phase_key        INT REFERENCES dim_phase,
  period_key       INT REFERENCES dim_period,
  date_key         INT REFERENCES dim_date,
  outflow          NUMERIC(18,2),
  inflow           NUMERIC(18,2),
  net_cf           NUMERIC(18,2),
  cumulative_cf    NUMERIC(18,2),
  discount_factor  NUMERIC(10,8),
  discounted_net_cf NUMERIC(18,2)
);
CREATE TABLE fact_feasi_summary (
  feasi_key      BIGSERIAL PRIMARY KEY,
  version_key    INT REFERENCES dim_version,
  project_key    INT REFERENCES dim_project,
  phase_key      INT REFERENCES dim_phase,
  date_key       INT REFERENCES dim_date,
  gross_gdv      NUMERIC(18,2),
  net_gdv        NUMERIC(18,2),
  gdc            NUMERIC(18,2),
  tcc            NUMERIC(18,2),
  pbt            NUMERIC(18,2),
  total_units    INT,
  nfa_sqft       NUMERIC(16,2),
  profit_margin  NUMERIC(8,6),   -- non-additive
  irr            NUMERIC(8,6),   -- non-additive
  npv            NUMERIC(18,2),  -- non-additive across phases
  peak_funding   NUMERIC(18,2),
  payback_years  NUMERIC(5,2)
);
```

---

## 5. Sample analytical queries

```sql
-- Budget vs actual variance by pillar, current version
SELECT pl.pillar_code, SUM(f.budget_amt) budget, SUM(f.actual_amt) actual,
       SUM(f.variance_amt) variance
FROM   fact_cost_line f
JOIN   dim_pillar pl  USING (pillar_key)
JOIN   dim_version v  USING (version_key)
WHERE  v.is_current
GROUP  BY pl.pillar_code ORDER BY pl.pillar_code;

-- Phase KPI scoreboard (ratios read straight from summary — never summed)
SELECT ph.phase_name, ph.maturity_flag,
       s.net_gdv, s.gdc, s.pbt, s.profit_margin, s.irr, s.npv, s.peak_funding
FROM   fact_feasi_summary s
JOIN   dim_phase ph   USING (phase_key)
JOIN   dim_version v  USING (version_key)
WHERE  v.is_current;

-- Cumulative cash-flow curve for one phase
SELECT p.period_label, f.outflow, f.inflow, f.net_cf, f.cumulative_cf
FROM   fact_cashflow f
JOIN   dim_period p   USING (period_key)
JOIN   dim_phase ph   USING (phase_key)
WHERE  ph.phase_name = 'Phase 1' ORDER BY p.period_index;

-- Version comparison: GDC drift V29 → V30
SELECT v.version_label, SUM(f.budget_amt) gdc_budget
FROM   fact_cost_line f JOIN dim_version v USING (version_key)
GROUP  BY v.version_label ORDER BY v.snapshot_date;
```

---

## 6. ETL mapping (Excel → schema)

| Excel source | Target |
|---|---|
| `Pn_INPUT` rows (project/area/GDV) | `dim_phase`, `dim_component`, `revenue_assumption` → `fact_revenue` |
| `Pn_COSTING` line items | `cost_entry` → `fact_cost_line` (one row per item × snapshot column) |
| `Pn_COSTING` Budget/Committed/Actual columns | measures `budget_amt`/`committed_amt`/`actual_amt` (NOT a dimension) |
| `Pn_CASHFLOW` curves + CF rows | `curve`, `cashflow_entry` → `fact_cashflow` |
| `MASTER` roll-up | derived = `SUM(fact_*)` grouped by project (don't import; compute) |
| `BENCHMARK` sheet | `benchmark` (reference table, not a fact) |
| Workbook file itself = one FZ version | one `fz_version` / `dim_version` row |

Rule of thumb: each saved workbook (or "FZ Vxx") becomes one `dim_version` row;
re-importing a newer version inserts new facts, never overwrites old ones.
