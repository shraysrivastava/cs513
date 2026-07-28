# CS513 Phase-II — submission contents (Team79, Chicago Food Inspections)

Sid Wanjara (swanj2) · Drew Patel (drewp4) · Shray Srivastava (ssriv5)

## What to put in the ZIP

| Rubric item | File(s) |
|---|---|
| **Workflow model** | `Workflow.yw` (YesWorkflow annotations for W1 + W2), `Workflow.gv` / `Workflow.pdf` (combined view), `Workflow_W1.gv` / `.pdf` / `.png`, `Workflow_W2.gv` / `.pdf` / `.png` |
| **Operation history** (we did **not** use OpenRefine) | `output/OtherToolHistory.json` — 16 logged operations, emitted by the notebook at run time; plus the executed notebook `CS513_Phase2_Cleaning.ipynb` itself |
| **Queries** | `queries.txt` — all 15 IC queries (before-on-`D` and after-on-`D'`, each annotated with the result it produced) plus the U1 and profiling queries |
| **Datasets** | *not in the ZIP* — see `DataLinks.txt` (Box links; Drew to fill in) |
| **Report** | `CS513_Phase2_Report_DRAFT.md` → export to `CS513_Phase2_Report.pdf` |
| Supporting tables behind the report | `output/ic_report.csv`, `output/change_summary.csv`, `output/derived_columns.csv`, `output/u1_result.csv` |

## How to reproduce

```bash
pip install pandas duckdb jupyter          # pandas>=2.0, duckdb>=1.0
# put Food-Inspections-20251023.csv in this directory, then:
jupyter nbconvert --to notebook --execute --inplace CS513_Phase2_Cleaning.ipynb
# or open it in Jupyter and Run All.  Runtime ~1 minute.
```

The notebook writes `output/clean_inspections.csv`, `output/clean_violations.csv`,
`output/ic_report.csv`, `output/change_summary.csv`, `output/derived_columns.csv`,
`output/u1_result.csv`, `output/OtherToolHistory.json` and `queries.txt`.

Re-render the workflow diagrams (optional, needs Graphviz):

```bash
dot -Tpdf Workflow_W1.gv -o Workflow_W1.pdf
dot -Tpdf Workflow_W2.gv -o Workflow_W2.pdf
```

## Headline numbers (all produced by the recorded notebook run)

| | |
|---|---|
| `D` | 298,869 rows × 17 cols, one relation |
| `D'` | 298,869 × 34 (inspections) + 974,597 × 8 (violations), two relations |
| Cells rewritten | 463,558 (9.12% of all cells in `D`); **no rows deleted, no values fabricated** |
| Violation facts recovered from free text | 974,597 across 110 categories, 100.00% parse rate |
| IC-violating rows (10 comparable constraints) | **220,254 → 0**; 4 further constraints only become checkable on `D'` |
| U1 | on `D`: 54,281 meaningless groups for 57,819 failures · on `D'`: a 2,296-cell ranked result |

## Open items for the team

1. **Drew** — paste the Box links into `DataLinks.txt`; export the report to PDF; build the ZIP.
2. **Sid** — the §2.1 and §2.2 tables in the report draft are copied verbatim from
   `output/change_summary.csv` and `output/ic_report.csv`; confirm the IC wording matches the
   column names in `clean_inspections.csv` / `clean_violations.csv`.
3. **All** — one decision is stated as an assumption, not a fact: `Risk = 'All'` (80 rows) is
   treated as `Unknown` and flagged with `risk_is_imputed`. Keep that caveat in the final report.
