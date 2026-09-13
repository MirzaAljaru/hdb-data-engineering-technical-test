# HDB Data Engineering Technical Test

Part 2 contains the [AWS ingestion and private Tableau architecture](part-2/README.md),
with an integrated PNG, a single-page editable draw.io file, and design assumptions.

The Part 1 pipeline is in `part-1/hdb_resale_pipeline.ipynb`. It processes
HDB resale transactions for January 2012 through December 2016. The workflow
covers extraction, profiling, validation, lease calculation, deduplication,
price screening, identifier construction, hashing, and CSV export.

Findings, assumptions, exclusion rules, and record counts are
documented in [insights_and_assumptions.md](part-1/docs/insights_and_assumptions.md).

## Requirement mapping

Notebook section numbers follow the assignment. Short descriptions below
show where each requirement is implemented or documented.

| Requirement | Implementation or evidence |
| --- | --- |
| 1. Combine datasets and retain all attributes | Notebook: Combine Sources and Retain All Attributes |
| 2. Profile the dataset | Notebook: Profile the Master Dataset |
| 3. Validate dates and reference categories | Notebook: Validate Dates and January 2012 Reference Categories |
| 4. Calculate remaining lease | Notebook: Calculate Remaining Lease; commencement assumption in the insights document |
| 5. Retain the highest price per composite key | Notebook: Retain the Highest Price per Composite Key |
| 6. Identify potentially anomalous prices | Notebook: Identify Potential Price Anomalies; thresholds and statuses in the audit output |
| 7. Add appropriate checks | Notebook: Apply Additional Validation Rules and structural checks |
| 8. Document insights and assumptions | `part-1/docs/insights_and_assumptions.md` |
| 9. Create the resale identifier | Notebook: Construct the Resale Identifier |
| 10. Hash and preserve uniqueness | Notebook: Hash the Identifier and Verify Uniqueness; hash-input interpretation in the insights document |
| 11. Optional intermediate tables or views | Stage DataFrames and displayed summaries support processing and review |
| 12. Produce five output groups | Notebook: Export the Five Required Output Groups; locations below |
| 13. Develop the pipeline in Python | Python code cells in `part-1/hdb_resale_pipeline.ipynb` |
| 14. Submit a notebook, inputs, comments, and outputs to Git | Notebook, raw inputs, rationale comments, and saved development outputs; final execution and GitHub upload pending |
| 15. Engineering practices and root README | Dependency declarations, bounded requests, staged routing, reconciliation, and this README; final verification pending |

## Setup and execution

Development started with Python 3.14. Final verification from a fresh kernel
is still pending. From the repository root, create an environment and install
the declared dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Open `part-1/hdb_resale_pipeline.ipynb` in VS Code with its Python and Jupyter
extensions. Select `.venv\Scripts\python.exe` as the notebook kernel and run
all cells in order. Run from the repository root or the `part-1` folder.
Run the cells in order and save their outputs. The "Run Source Extraction"
cell downloads missing source files. Later cells write processed outputs;
rerunning the export step replaces those files.

## Source files and reruns

- Original CSV files are stored in `part-1/data/raw/`, named by dataset ID.
- Collection metadata is cached in `part-1/data/metadata/collection_189.json`.
- Existing non-empty CSV files are skipped; empty or missing files are downloaded.
- A rerun with cached metadata and all CSV files present needs no network access.
- Set `REFRESH_COLLECTION = True` in the notebook to refresh collection metadata.
- To redownload a file, remove that specific CSV and rerun the download cell.
- Interrupted transfers never become final CSV files; only completed downloads
  with the expected CSV header are renamed from `.part` to `.csv`.

The initial run needs internet access. Download API calls are spaced 13 seconds
apart, with bounded retries for rate limits and temporary connection or server
failures. No API key is required for this assessment.

## Outputs

Paths below are relative to `part-1/`. Counts reflect the current source snapshot.

| Group | Location | Records |
| --- | --- | ---: |
| Raw | `data/raw/` | 986,548 across five files |
| Cleaned | `data/output/cleaned/cleaned.csv` | 83,441 |
| Transformed | `data/output/transformed/transformed.csv` | 83,441 |
| Quarantined | `data/output/quarantined/quarantined.csv` | 9,103 |
| Hashed | `data/output/hashed/hashed.csv` | 83,441 |

The price-screening audit is in `data/output/audit/price_screening.csv`.
It includes thresholds and assessment status for the 83,750 records entering
price screening. Exported file verification is still pending.

## Decisions affecting the results

- January 2012 defines allowed categorical values. Dates use `YYYY-MM` and
  the full assessment period.
- Lease commencement is assumed to be 1 January of the supplied year.
- Deduplication retains the highest price per original source key; equal
  maximum prices retain the first record in source order.
- Price screening uses three-IQR fences on price per square metre within
  year, town, and flat type. Small groups remain unassessed.
- The prescribed readable identifier repeats. SHA-256 includes that identifier
  and the original composite key to distinguish retained records. This
  interpretation of the uniqueness requirement is documented in the insights file.

The assignment PDF stays outside the submitted repository. The local `.venv`
is excluded from version control; reviewers recreate it from `requirements.txt`.
