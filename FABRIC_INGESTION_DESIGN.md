# Fabric Ingestion — Design Decisions & Landing Checklist

How the NCT feasibility workbooks feed Microsoft Fabric. Decisions reached
2026-06-16. Pairs with `DB_DESIGN.md` (target schema) and the revised ERD.

---

## 1. Decision record

| # | Decision | Why |
|---|---|---|
| D1 | Fabric reads **clean Excel Tables**, not the report-shaped sheets. No notebook parser. | Report shape (key-value, interleaved subtotals, merged cells) is brittle to parse and breaks when line items move. |
| D2 | Input region is **restructured into Excel Tables** (one row per coded line item, no interleaved subtotal rows). | Excel Tables (ListObjects) are read by name — position- and size-independent — so variable/growing line items are handled natively. |
| D3 | **The report's own detail rows ARE the Table** (option ①). Subtotals / PBT / IRR move out to `SUMIFS` or a display strip. Zero copy, zero drift. | Report and ingest source become the same object → no dual maintenance, no sync errors. User keeps editing the report as before. |
| D4 | **Type codes = shared master catalog** across all projects (cost item, assumption, sales/component type). **Instance codes (phase / node) = project-namespaced.** | Shared codes are the prerequisite for cross-project benchmarking and clean `DIM_*` business keys. Phases mean different things per project. |
| D5 | Code shape = **light-prefix mnemonic** (e.g. `C.PILE`). Prefix is decorative; authoritative classification lives in columns (`pillar`, `category`). Old `1.1 / 2.3` demoted to `report_no` / `display_order` (display only, never a join key). | Stable codes survive reorder / insert / reclassification; benchmarking joins always align; users still see familiar numbering. |
| D6 | Governance: **closed-set dropdown entry** from catalog; unknown code at ingest → **reject to quarantine + alert** (never auto-create). Format contract enforced. SCD Type 1 on name; soft-retire (`is_active=false`), never reuse a retired code. | Block errors at entry and at the gate — ten times cheaper than cleaning afterwards. Catalog integrity is the whole value. |
| D7 | Business version identity = **`version_code` in `tbl_META`** (option B). Fabric manages storage / immutability / dedup / partition-replace / retention. **One workbook = one (project × version × scenario) snapshot.** | Named cross-version comparison (V29 vs V30) needs a stable, named, queryable key in the data. Delta time-travel alone is retention-limited and unnamed. Storage mechanics still delegated to Fabric. |
| D8 | Pipeline = **one Data pipeline (trigger + orchestration) + one PySpark notebook (everything)**. Skip Dataflow Gen2. | Validation + idempotent replace are the notebook's home turf and can't be delegated; Dataflow Gen2 would add a tool without removing the notebook. |

---

## 2. Resulting architecture

```
WORKBOOK  (one per project × version × scenario)
  tbl_META    project_code | version_code | scenario_code | version_date | status | currency
  tbl_PHASE   code | nda_acre | plot_ratio | efficiency | construction_months | ...
  tbl_SALES   code | component_type | nfa | units | asp_psf | ...
  tbl_COST    code | pillar | category | report_no | budget | committed | actual | ...
  tbl_CURVE   code | curve_type | Y0 | Y1 | Y2 | ...
        │  (report still shows subtotals/PBT/IRR via SUMIFS — display only)
        ▼  land in OneLake / SharePoint folder
  DATA PIPELINE  — trigger on new file, orchestrate, loop files
        ▼
  PYSPARK NOTEBOOK
    1. read named tables (cached values; formula columns OK)
    2. unpivot  tbl_COST (budget/committed/actual) and tbl_CURVE (Y0..Yn) → long
    3. broadcast tbl_META (project/version/scenario) onto every row
    4. lookup shared-catalog dims by code → surrogate keys
    5. VALIDATE → quarantine on failure (see §4)
    6. idempotent load: replaceWhere (project, version, scenario) → Delta
        ▼
  LAKEHOUSE (Delta star schema — see DB_DESIGN.md / revised ERD)
        ▼
  POWER BI  (version-over-version, benchmarking)
```

---

## 3. Landing checklist

### A. Workbook restructure (per template)
- [ ] Add `tbl_META` (single row, the version label + scenario + status).
- [ ] Convert each input region to a real Excel Table: `tbl_PHASE`, `tbl_SALES`, `tbl_COST`, `tbl_CURVE`.
- [ ] Add `code` column to every table; add `pillar` / `category` / `report_no` columns to `tbl_COST`.
- [ ] Move pillar subtotals / GDC / PBT / IRR **out** of the data region → `SUMIFS` or a display strip.
- [ ] Wire `code` columns to **data-validation dropdowns** pointing at the shared catalog.
- [ ] Re-point `P1_CASHFLOW` curve rows to `tbl_CURVE` (keep the IRR/NPV engine intact).

### B. Catalog & governance (once)
- [ ] Nominate a **catalog owner**.
- [ ] Seed shared catalogs: `DIM_COST_ITEM`, `DIM_ASSUMPTION`, `DIM_COMPONENT` (start from existing `1.1..5.x` items → assign stable codes).
- [ ] Publish the **format contract**: `^[A-Z]\.[A-Z0-9_]{2,20}$`, uppercase, ASCII, prefix from a fixed domain list.
- [ ] Define new-code request flow (owner adds to catalog → re-run).

### C. Fabric pipeline (once)
- [ ] OneLake/SharePoint landing folder + file-arrival trigger.
- [ ] PySpark notebook: read tables → unpivot → broadcast meta → dim lookup → validate → `replaceWhere` load.
- [ ] Quarantine table + alert for failed loads.
- [ ] Delta retention policy so APPROVED versions persist (don't rely on VACUUM defaults).

### D. Validation rules (notebook, hard gate)
- [ ] Every `code` resolves in its catalog (else quarantine + list offenders).
- [ ] `tbl_META` complete; `version_code` present; `status` valid.
- [ ] Pillar `SUMIFS` totals == report totals (within rounding tolerance).
- [ ] Each curve in `tbl_CURVE` sums to 100% (collection/spend) or ≤100% (RoS).
- [ ] APPROVED version not silently overwritten unless intentional re-run.

---

## 4. Still open (next session)
1. **Target table list** — finalise the revised ERD incl. `FACT_PROJECT_CASHFLOW` (deletes: currency/source dims, bridge; adds: cashflow fact, tenure column). Then write the Lakehouse DDL.
2. **Rounding tolerance** for the report-vs-input reconciliation check.
3. **Quarantine workflow** — who reviews, how a fixed file is re-submitted.
4. **Initial catalog seed** — actual code list for the ~50 existing cost items.
