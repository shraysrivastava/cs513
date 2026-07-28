# CS513 Phase-II — Data Cleaning Notebook Build Brief

**Team79 · Chicago Food Inspections · Owner: Shray**
Use this as the spec for building the cleaning notebook in Claude Code.

---

## 0. Goal & context

Main use case **U1**: *identify the most common violation categories associated with failed
inspections, broken down by facility type and risk level.* Everything in this notebook exists to
make U1 answerable. If a step doesn't serve U1 (or a sanity check for it), keep it minimal.

Raw dataset `D`: one flat CSV, ~298,869 rows × 17 columns, one row per inspection (unique
`Inspection ID`). Coverage Jan 2010 – Oct 2025, primarily Chicago, IL.

**Deliverables from this notebook:**
- `clean_inspections.csv` — one row per inspection, standardized fields + `facility_group`.
- `clean_violations.csv` — one row per parsed violation per inspection, linked by `inspection_id`.
- The notebook itself is the **provenance / operation-history** artifact (we're not using
  OpenRefine), so structure it cleanly — see §7.
- IC-check SQL that also gets pasted into `queries.txt`.

---

## 1. Environment

- Python + **pandas** for all cleaning.
- **DuckDB** (`import duckdb`) for the integrity-constraint checks, run directly over the
  dataframes inside the notebook (`duckdb.sql("... df ...")`). No separate SQL install — this keeps
  the whole thing in one Python environment while still satisfying the "SQL or Datalog" requirement
  for `queries.txt`.
- No OpenRefine. No OR2YW.

---

## 2. Target output schemas

**clean_inspections.csv** (one row per inspection)

| column | source | notes |
|---|---|---|
| `inspection_id` | Inspection ID | primary key, must stay unique |
| `dba_name` | DBA Name | trim/upper-normalize whitespace only |
| `aka_name` | AKA Name | keep; blanks allowed |
| `license_num` | License # | keep as-is; flag blanks |
| `facility_type_raw` | Facility Type | keep original for traceability |
| `facility_group` | derived | broad category — see §5 |
| `risk_clean` | Risk | one of Risk 1 / Risk 2 / Risk 3 / Unknown |
| `address`, `city_clean`, `state_clean`, `zip` | address fields | canonicalized — see §4 |
| `inspection_date` | Inspection Date | parse to ISO date |
| `inspection_type` | Inspection Type | standardized casing |
| `result` | Results | leave categories intact (already clean — U0) |
| `latitude`, `longitude` | coords | keep; blanks allowed, exclude from maps |
| `has_violation_text` | derived | boolean; False when Violations blank |
| `fail_missing_violations` | derived | boolean flag: result=Fail AND no violation text |

**clean_violations.csv** (one row per violation per inspection)

| column | notes |
|---|---|
| `inspection_id` | FK to clean_inspections |
| `violation_number` | int, 1–44 or 70 |
| `violation_title` | parsed title text |
| `violation_comment` | parsed inspector comment (may be empty) |

---

## 3. Notebook structure (build in this order)

Each numbered step = one markdown header + its code cell(s). See §7 for why the headers matter.

1. Load & profile raw D
2. Standardize categorical fields (City, State, Risk, Results, Inspection Type)
3. Normalize Facility Type → `facility_group`
4. Parse Violations → violation-level table
5. Handle missing values + derive flags
6. Emit `clean_inspections.csv` and `clean_violations.csv`
7. IC checks (DuckDB) — before vs after
8. Change summary (per-column diff counts for the report table)

---

## 4. Step 2 — Standardize categoricals

**City.** The core mess. Approach: uppercase + strip, then map known Chicago variants to `CHICAGO`.
Observed variants to fold in (not exhaustive — build from `value_counts()` first):
`CHICAGO, Chicago, chicago, CHicago, CCHICAGO, CHICAGOCHICAGO, CHICAGOO, CHICAGO., CHCICAGO, CHCHICAGO`.
Strategy: explicit map for the known typos + a rule that any value whose cleaned form starts with
`CHICAG` and is within an edit-distance threshold collapses to `CHICAGO`. Leave genuinely different
cities (e.g. real suburbs) alone. Keep a `city_clean` column; do **not** overwrite the raw silently
— we need before/after counts.

**State.** Most are `IL`. Uppercase/strip. Blanks + non-IL (`IN, CA, WI, CO, NY`) → keep but flag;
for U1 we can filter to `IL` in the analysis query rather than deleting rows here.

**Risk.** Map to exactly {`Risk 1 (High)`, `Risk 2 (Medium)`, `Risk 3 (Low)`, `Unknown`}.
Blank → `Unknown`. `All` → `Unknown` (call this out in the report; we're treating it as
non-informative). Normalize casing/spacing variants.

**Results.** Already clean (this is our U0 case). Just trim/standardize casing; do not re-bucket.

**Inspection Type.** Standardize casing/whitespace; no heavy re-mapping needed for U1.

---

## 5. Step 3 — Facility Type → facility_group

521 distinct raw values → a small set of broad groups. Keep `facility_type_raw`, add
`facility_group`. Start from an explicit mapping dict for the common ones, then bucket the long tail
with keyword rules; anything unmatched → `Other`; blank → `Unknown`.

Starter mapping (extend after inspecting `value_counts()`):
```
Restaurant                          -> Restaurant
Grocery Store                       -> Grocery
School                              -> School
Children's Services Facility        -> Childcare
Daycare Above and Under 2 Years     -> Daycare
Daycare (2 - 6 Years)               -> Daycare
Bakery                              -> Bakery
Catering                            -> Catering
Long Term Care                      -> Long Term Care
Liquor                              -> Liquor
Mobile Food Preparer                -> Mobile Food
Mobile Food Dispenser               -> Mobile Food
```
Keyword fallback examples: contains "DAYCARE"/"DAY CARE" → Daycare; "MOBILE" → Mobile Food;
"SCHOOL" → School; "GROCERY" → Grocery. Document the final group list for the report.

---

## 6. Step 4 — Parse Violations (the important one)

Format: multiple violations in one cell, separated by `|`. Each chunk usually looks like:
```
<NUMBER>. <TITLE> - Comments: <COMMENT TEXT>
```

Parsing logic:
1. If Violations is blank → contributes **zero** rows to `clean_violations`; set
   `has_violation_text=False` on the inspection.
2. Split the cell on `|`.
3. For each chunk, strip and match:
   `^\s*(\d+)\.\s*(.*?)\s*(?:-\s*Comments:\s*(.*))?$`
   - group 1 → `violation_number` (int)
   - group 2 → `violation_title`
   - group 3 → `violation_comment` (may be empty when a chunk has no "- Comments:" segment)
4. Valid numbers are 1–44 and 70 — flag anything outside that range for review rather than dropping
   silently.
5. Emit one row per parsed chunk into `clean_violations`.

**Repeated-violation decision:** some inspections repeat the same `violation_number` with different
comments. Keep all parsed rows in `clean_violations` (full provenance), but for U1 counting **count
each violation category once per inspection** — do this with `DISTINCT (inspection_id,
violation_number)` in the analysis query, not by deleting rows here.

---

## 7. Step 5 — Missing values & flags

Distinguish acceptable blanks from problematic ones:
- Blank Violations on a **Pass** → acceptable, no flag.
- Blank Violations on a **Fail** → set `fail_missing_violations=True` (these distort U1 if counted
  as "no violations"; report their count, exclude from violation-frequency analysis).
- Blank Facility Type → `facility_group='Unknown'`.
- Blank coordinates → keep the row, just exclude from any map.
- Blank Zip / City → keep + flag; only infer if reliable (don't fabricate).

---

## 8. Provenance requirements (don't skip)

Because we're not using OpenRefine, **the notebook is our operation history**. Make it legible:
- One markdown header per high-level step, matching §3 numbering.
- Under each header, a one-line statement of *what* the step does and *why it's needed for U1*
  (this text feeds Report section 1.3 almost verbatim).
- Print a small before/after count after each transform (e.g. distinct city values before → after)
  so the change-summary table in Report section 2.1 is a copy job, not a re-derivation.
- Keep raw columns alongside cleaned ones (`*_raw`) so every change is auditable.

---

## 9. Step 7 — IC checks via DuckDB (feeds queries.txt)

Write each as a query that returns the count (or the offending rows) of constraint violations. Run
each one **on the raw data and on the cleaned data** and record both numbers. Paste the final SQL
into `queries.txt`.

Constraints to implement:
1. `inspection_id` is unique (count of duplicate ids).
2. Every `Fail` inspection has ≥ 1 parsed violation (count of fails with none).
3. `city_clean = 'CHICAGO'` for all rows where `state_clean = 'IL'` (count of exceptions — expect
   this to drop sharply after cleaning).
4. `risk_clean` ∈ {Risk 1, Risk 2, Risk 3, Unknown} — no `All`/blank (count of violations).
5. Referential integrity: every `clean_violations.inspection_id` exists in `clean_inspections`
   (inclusion dependency; count of orphans → should be 0).
6. Every non-blank `facility_type_raw` maps to a defined `facility_group` (count of `Other`/unmapped).

Also run the **U1 analysis query** before vs after to prove the point:
```sql
SELECT facility_group, risk_clean, violation_number, violation_title, COUNT(*) AS n
FROM clean_inspections i
JOIN clean_violations v ON i.inspection_id = v.inspection_id
WHERE result = 'Fail'
GROUP BY 1,2,3,4
ORDER BY n DESC;
```
(Before cleaning this is essentially impossible because violations are trapped in free text — say so.)

---

## 10. Scope guardrails

- Don't over-clean fields U1 doesn't touch (dba_name spelling, aka_name, etc.). Trim only.
- Don't delete rows to "fix" problems — prefer flags + filtering in the analysis query, so counts
  stay auditable and reversible.
- Don't fabricate missing zips/coords.
- Keep `clean_inspections` at exactly one row per `inspection_id`; the fan-out lives only in
  `clean_violations`.