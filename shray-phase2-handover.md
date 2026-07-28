# CS513 Phase-II Handover — Shray's Portion

**Team79 · Chicago Food Inspections · Owner of this doc: Shray**
Self-contained reference: what Phase-I delivered, what we decided this planning session, and the
concrete tasks that are mine for Phase-II.

---

## 1. Project context

- **Team79:** Sid Wanjara (swanj2), Drew Patel (drewp4), Shray Srivastava (ssriv5).
- **Dataset D:** Chicago Food Inspections — one flat CSV, ~298,869 rows × 17 cols, Jan 2010 – Oct 2025,
  one row per inspection (unique `Inspection ID`).
- **U1 (target use case — cleaning necessary & sufficient):** most common violation categories in
  **failed** inspections, broken down by **facility type** and **risk level**.
- **U0 (no cleaning needed):** count inspections by `Results`.
- **U2 (never enough):** whether specific establishments *caused* foodborne-illness outbreaks — the
  data has no illness/patient/lab records, so no cleaning can answer it.

---

## 2. What Phase-I delivered (recap)

Submitted report covered all four rubric sections:

- **Dataset description** — flat `FoodInspection` table modeled conceptually as Establishment /
  Inspection / Violation / Location entities; narrative on origin (Chicago Dept. of Public Health)
  and temporal/spatial extent.
- **Three use cases** — U1, U0, U2 as above.
- **Obvious DQ problems** documented with evidence:
  - Missing values by column (Violations 83,344; Facility Type 5,264; AKA 2,412; Lat/Long/Location
    1,022 each; City 162; Risk 87; State 58; Zip 42; License # 18; Address 3; Inspection Type 1).
  - Inconsistent City (`CHICAGO / Chicago / CCHICAGO / CHICAGOO / CHICAGOCHICAGO / ...`).
  - Unusual State values (blanks + IN/CA/WI/CO/NY).
  - Missing location (blank City/Zip with address present).
  - Ambiguous Facility Type (521 distinct values, 5,264 blanks).
  - Semi-structured Violations (many per cell, `|`-separated, `N. TITLE - Comments: text`).
  - Risk column has blanks and a mystery `All` category.
  - Repeated violation codes within a single inspection.
- **Phase-I plan S1–S5** with owners: S1 review (Drew/Shray), S2 profiling (Sid), S3 cleaning (Drew),
  S4 DQ checking (Sid/Shray), S5 documentation (all). *Timeline ran to ~8/1.*

> The Phase-II split below **reassigns** this: I now own all the cleaning (was Drew's S3), because I
> wanted it in Python. Note the change when we write the report.

---

## 3. Decisions made this planning session

- **Tooling:** cleaning done in **Python/pandas**; integrity-constraint checks in **DuckDB SQL run
  inside the same notebook**. **OpenRefine dropped.** Rationale: Python is my comfort zone and
  cleaning is the critical path; DuckDB still satisfies the "SQL or Datalog for `queries.txt`"
  requirement without adding a separate tool; the notebook doubles as operation-history provenance
  (replacing `OpenRefineHistory.json`).
- **3-person split** (see §5).
- **W2 inner workflow can't use OR2YW** (that's OpenRefine-only) — Drew builds it manually from my
  notebook's step structure.
- **Report will explicitly compare actual workflow vs the Phase-I plan** — turns the OpenRefine→Python
  switch into rubric credit instead of an unexplained deviation.
- **Gap flagged:** rubric 2a wants **cells-changed-per-column**, not distinct-value counts. I need to
  produce true per-column diff counts (rows where `*_clean != *_raw`), not just "200 city variants → 1".
- **Open decision to flag in report:** treating `Risk = All` as `Unknown` (cleanest for U1, but we
  weren't sure what `All` meant in Phase-I — call it out).
- **Re-derive Phase-I numbers from the real CSV** in the first profiling cell; don't trust the
  Phase-I estimates for the report tables (data may have drifted).

---

## 4. Artifacts already produced (this session)

- **`CS513_Phase2_Report_Template.docx`** — fill-in report template mapped to the Phase-II rubric,
  with owners, pre-built change/IC tables, and the plan-comparison section.
- **`CS513_Phase2_Cleaning_Plan.md`** — the detailed build brief I take into Claude Code for the
  notebook (schemas, per-step logic, violation-parsing regex, facility mapping, IC checks, guardrails).

---

## 5. Team split (for reference)

| Person | Owns |
|---|---|
| **Shray (me)** | All cleaning + notebook + Report §1 + provenance + cell-level change counts |
| **Sid** | Change-summary table (§2a) + IC-violation reports (§2b) + `queries.txt` |
| **Drew** | Workflow W1/W2 + Conclusions (§4) + report assembly + ZIP/Box upload |

---

## 6. MY Phase-II tasks (the actual to-do list)

**Primary — I own these:**

1. **Build the cleaning notebook** per `CS513_Phase2_Cleaning_Plan.md`:
   load & profile → standardize categoricals (City/State/Risk/Results/Inspection Type) → facility
   grouping → parse Violations into violation-level rows → missing-value flags → emit
   `clean_inspections.csv` + `clean_violations.csv`.
2. **Ship a rough v0 + lock the output schema early (~day 2–3)** so Sid and Drew aren't blocked
   waiting on final cleaning.
3. **Write Report Section 1** (40 pts): 1.1 overview + Phase-I-plan comparison, 1.2 high-level steps,
   1.3 rationale (necessary vs useful for U1, per step).
4. **Keep the notebook legible as provenance** — one markdown header per step + a one-line
   what/why. This IS our operation-history deliverable now.
5. **Produce cell-level change counts per column** (the gap fix) — feeds Sid's §2a change table.
6. **Hand my notebook step-list to Drew** so he can build W2.

**Shared / support:**

7. Sanity-check that the before/after numbers in the report match what the notebook actually
   produced (I was on S4 DQ-checking with Sid).
8. Write my row of the contributions table (§4.2) and review the merged report.

---

## 7. Handoff dependencies

**What others need FROM me:**
- v0 cleaned dataset + locked schema → Sid (to write after-queries) and Drew (to finalize workflow).
- Notebook step list → Drew (W2).
- Per-column cell-change counts → Sid (§2a table).

**What I need FROM others:**
- Sid: confirm his IC constraints reference my actual output column names.
- Drew: report assembly + final ZIP/Box; tell me the real Phase-II due date (check Key Dates on
  Coursera — not in the instructions PDF).

---

## 8. Watch-outs

- Don't over-clean fields U1 doesn't touch (dba_name, aka_name) — trim only.
- Don't delete rows to "fix" problems — use flags + filter in the analysis query, so counts stay
  auditable.
- Keep `clean_inspections` at exactly one row per `inspection_id`; fan-out lives only in
  `clean_violations`.
- Count each violation category **once per inspection** for U1 (handle in the query with `DISTINCT`,
  keep all parsed rows in the table).
- Only ONE submission per phase, manually graded — submit the FINAL version only.