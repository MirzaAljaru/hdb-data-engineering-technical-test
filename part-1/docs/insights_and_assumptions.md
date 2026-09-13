# Data Quality Insights and Assumptions

This document records the processing decisions and observed results for Part 1.
Counts refer to the source snapshot used during development. A refreshed
snapshot or a changed exclusion rule requires updated results.

## Sources and Scope

The assignment points to data.gov.sg collection 189. Collection metadata supplies
the dataset IDs. All five complete CSVs are retained unchanged in `data/raw/`,
with metadata cached separately in `data/metadata/`.

Existing non-empty files are reused, assuming local files remain intact.
New downloads use temporary files, a size check where the server supplies
`Content-Length`, and an expected-header check. These are transfer checks;
they do not establish transaction-level completeness. Preparation is bounded
to ten polls. Opening HTTP responses uses a timeout and bounded retries;
streaming failures require rerunning extraction.

| Observed source coverage | Rows |
| --- | ---: |
| January 1990–December 1999 | 287,196 |
| January 2000–February 2012 | 369,651 |
| March 2012–December 2014 | 52,203 |
| January 2015–December 2016 | 37,153 |
| January 2017–September 2026 | 240,345 |
| **Combined sources** | **986,548** |

Three sources overlap January 2012–December 2016. Valid months outside this
period are scope exclusions. Missing or malformed months enter quarantine
because their scope cannot be determined reliably.

## Consolidation and Types

Concatenation retains the union of source columns: eleven attributes in total.
Absent source attributes become nulls. Categories, block identifiers, and
source remaining-lease values are loaded as text. Price, floor area, and
commencement year initially use numeric inference, followed by explicit checks.
Empty fields are treated as nulls; other text is not automatically interpreted
as missing. The original CSVs are unchanged.

The master contains 92,544 records and used approximately 42.2 MB in the
observed environment. All 60 required months have records, with monthly counts
between 886 and 2,360. The largest decrease is 45.74% in February 2013 and
the largest increase is 50.57% in March 2014. Volume changes are observations,
without an associated exclusion rule. No empty months were found; this does
not prove every transaction is present.

## Completeness

The 55,391 missing source `remaining_lease` values (59.85%) cover all records
from 2012–2014 and align with sources that omit the attribute. All 2015–2016
records supply it. These structural nulls are retained, with a calculated
balance added separately.

The other ten master columns contain no nulls or whitespace-only values.
The eight text fields checked after price routing have no outer whitespace.
Those findings establish completeness and formatting only for the properties
and populations examined.

## January 2012 Reference Rules

January 2012 provides exact allowed-value sets for town, flat type, flat model,
and storey range. The sets are captured before cleaning. No case conversion,
trimming, or reference expansion is applied.

For dates, January establishes `YYYY-MM` representation. The accepted period
remains January 2012–December 2016. Literal membership in January's date values
would contradict the assessment scope; this is the interpretation used here.

| Field | Master distinct values | January reference values | Failing records |
| --- | ---: | ---: | ---: |
| Town | 26 | 26 | 0 |
| Flat type | 7 | 7 | 0 |
| Flat model | 20 | 13 | 492 |
| Storey range | 25 | 12 | 6,975 |

The category rules fail for 7,411 distinct records. Fifty-six fail both model
and storey checks. Date routing finds no malformed or missing months in this
snapshot. Rechecking dates within master confirms the routing invariant.

One month's transactions cannot establish every legitimate category. These
exclusions follow the specified reference policy. Five-storey bands are left
unchanged because converting them to three-storey bands would require an
exact floor that the source does not supply.

## Numeric and Structural Checks

| Attribute | Minimum | Median | Maximum |
| --- | ---: | ---: | ---: |
| Resale price | $190,000 | $428,000 | $1,150,000 |
| Floor area | 31 m² | 95 m² | 280 m² |
| Lease commencement year | 1966 | 1988 | 2013 |

Master profiling finds complete numeric values in these three fields. It finds
no nonpositive prices or areas, nonpositive or fractional commencement years,
or commencement after the transaction year. Extreme prices and areas remain
subject to contextual review rather than fixed rejection thresholds.

Additional enforced rules require numeric, finite, positive prices and areas;
a nonblank street; a block containing an ASCII digit; and `NN TO NN` storey
ranges with positive, ordered bounds. No block suffix restriction is imposed
without an authoritative naming rule. All ten additional rules pass in the
retained population. No values are imputed or silently corrected.

Numeric enforcement currently follows price screening. Before processing
future invalid inputs, that enforcement should move ahead of arithmetic so
invalid metrics cannot influence thresholds. The current profile has no
such inputs. This remains a pipeline ordering limitation.

## Remaining Lease

The lease term is 99 years. Commencement is assumed to be 1 January of the
supplied year, with the balance measured at the start of the resale month.
The source has no commencement month, so the calculated month balance is
assumption-dependent rather than an exact contractual balance.

```text
elapsed_months = (resale_year - commencement_year) × 12 + (resale_month - 1)
remaining_months = 99 × 12 - elapsed_months
remaining_lease_years = remaining_months // 12
remaining_lease_months = remaining_months % 12
```

Whole-month arithmetic avoids rounding up. Missing, nonnumeric, fractional,
or nonpositive commencement years and balances outside the term are quarantine
failures. The source `remaining_lease` remains intact. All 85,133 reference-passing
records receive calculated balances, with no lease failures.

## Duplicate Selection

The key is fixed from every original master column except `resale_price`,
including source `remaining_lease`. Derived lease fields, screening statistics,
and source references do not extend it. Null key attributes participate in
comparisons and grouped analysis.

A stable descending price sort retains one maximum-priced record per key.
Equal maxima retain the first source record. All surplus records enter
quarantine, including exact duplicate copies.

Master profiling finds 3,154 records across 1,559 repeated keys, including
1,294 groups with conflicting prices. The 1,595 surplus records already
include 273 exact duplicate copies. On the reference-validated population,
duplicate selection excludes 1,383 and retains 83,750.

The source has no exact unit or transaction identifier. Matching these coarse
attributes may describe different real-world transactions. Selection follows
the assignment's explicit composite-key assumption.

## Price Screening

Price per square metre is compared within year, town, and flat type. Area
normalization and grouping provide a practical comparison, with yearly groups
supporting larger samples. The method does not fully account for within-town
location, model, floor level, lease balance, or intra-year market changes.

Eligible groups have at least 30 records and positive IQR. Fences are
`Q1 - 3 × IQR` and `Q3 + 3 × IQR`. These are screening parameters without
calibrated error probabilities. Small or zero-IQR groups remain unassessed.

The preliminary master screen flags 414 candidates. Refitting after validation
and deduplication gives:

| Final screening status | Records |
| --- | ---: |
| Within fences | 81,557 |
| Potential high price | 306 |
| Potential low price | 3 |
| Unassessed: insufficient group | 1,884 |
| **Screening input** | **83,750** |

The 309 candidates enter quarantine for review, with fences and high/low reasons.
Thresholds are applied once. Repeated exclusion and refitting would change the
policy. The 1,884 unassessed records remain accepted under the other rules,
with their screening status recorded in the audit output. Candidates are
statistically unusual, without establishing an incorrect transaction price.

## Record Routing and Reconciliation

Stages run in this order: date routing, reference validation, lease validation,
duplicate selection, price routing, and additional rules. Previously excluded
records do not proceed to later checks. Failure reasons cover all applicable
checks within the stage that excludes a record.

`source_record_id` is the combined source row index for this run. It depends
on source contents and ordering and is not persistent across changing snapshots.
Quarantine is rebuilt from stage outputs on reruns. Reference uniqueness and
separation from accepted records are checked.

| Final disposition | Records |
| --- | ---: |
| Cleaned | 83,441 |
| Quarantine: source dates | 0 |
| Quarantine: reference failures | 7,411 |
| Quarantine: lease failures | 0 |
| Quarantine: duplicate surplus | 1,383 |
| Quarantine: price candidates | 309 |
| Quarantine: additional failures | 0 |
| **Total quarantine** | **9,103** |
| Valid-date scope exclusions | 894,004 |

```text
83,441 cleaned + 9,103 quarantined = 92,544 in-scope records
92,544 in-scope + 894,004 scope exclusions = 986,548 combined source records
```

Per-rule profile counts may overlap. Final routing counts above are exclusive
and reconcile to the source population.

## Readable Identifier

The construction bullets printed under Requirement 10 are interpreted as
Requirement 9's readable identifier rules, followed by hashing. Transformation
confirms original-key uniqueness and uses the final cleaned population for
month–town–flat-type average prices. The in-memory `identifier_audit` records
the averages used; it is not currently exported.

| Component | Rule used |
| --- | --- |
| Prefix | Literal `S` |
| Block | First three ASCII digits after removing other characters; left-pad shorter values with zeros |
| Group price | First two digits of the integer part of the group average, without rounding up |
| Month | Two-digit transaction month |
| Town | First character of the source value |

Construction stops if block digits or two price digits are unavailable. Every
generated identifier matches `S[0-9]{7}[A-Z]`. Block 19, average $230,000,
January, and ANG MO KIO produce `S0192301A`.

| Identifier measure | Count |
| --- | ---: |
| Transformed records | 83,441 |
| Distinct readable identifiers | 71,591 |
| Identifiers shared by multiple records | 9,635 |
| Records sharing identifiers | 21,485 |

The format omits year and several key attributes. These repetitions are
construction collisions. Two distinct January 2012 ANG MO KIO block 314
records both receive `S3142501A`. The readable format is preserved as specified.

## Hash Input and Uniqueness

Hashing a repeating readable identifier alone cannot produce unique digests.
The implemented interpretation hashes that value together with the original
composite key. This extends the hash input to satisfy record uniqueness,
without changing the prescribed readable format.

SHA-256 operates on UTF-8 JSON containing a serialization version, the readable
identifier, and ordered column-name/value pairs for the original key. Column
names are sorted, nulls become JSON null, NumPy scalars become native values,
separators are fixed, and nonfinite serialization is rejected. Field boundaries
are explicit. Price, derived attributes, and source references are excluded
from the key.

The hashing cell checks for 83,441 unique, valid 64-character hexadecimal
digests and unchanged record count. The repeated block 314 example receives
different digests. SHA-256 collisions remain theoretically possible, so output
uniqueness is checked rather than inferred from algorithm choice.

Digests are deterministic for unchanged values and serialization. Updated
group averages or source representations may change them; they are not
persistent transaction identities across arbitrary revisions. Public or
guessable inputs also mean hashing is not a guarantee of anonymization.

## Output Contract

Paths below are relative to Part 1. Raw files remain unchanged. Accepted
groups have the same records, with attributes added during transformation.

| Group | Location | Contents |
| --- | --- | --- |
| Raw | `data/raw/` | Five source files, totaling 986,548 rows |
| Cleaned | `data/output/cleaned/cleaned.csv` | 83,441 accepted records with calculated lease fields |
| Transformed | `data/output/transformed/transformed.csv` | Accepted records with the readable identifier |
| Quarantined | `data/output/quarantined/quarantined.csv` | 9,103 excluded records, references, and reasons |
| Hashed | `data/output/hashed/hashed.csv` | Accepted records with readable identifiers and SHA-256 digests |

The price audit at `data/output/audit/price_screening.csv` holds the 83,750
records entering final screening, including thresholds and unassessed statuses.
It supports traceability alongside the five required groups. Quarantine unions
stage-specific diagnostics, with nulls where a diagnostic does not apply.

CSV exports include `source_record_id` explicitly, without adding it to the
composite key. Files use UTF-8 and omit the pandas index. CSV does not retain
pandas types, so downstream readers must apply the intended schema.

Exports replace destinations after each temporary write completes. This
protects individual files, without making the whole export a transaction.
Counts and accepted/quarantine separation are checked before writing.

## Remaining Submission Checks

Written-file verification and a fresh-kernel execution with the declared
dependencies remain pending. The notebook includes rationale comments and
saved development outputs; final outputs must be retained after verification.

The submission includes the notebook, input files, requirements, README, and
this document. The local environment is recreated from dependency declarations.
The assignment PDF remains outside the submitted repository.
