# public-data-feeds

Standardized fact tables built from **public government data sources**, published at
stable URLs so a BI tool can read them directly.

The point of this repo is the last mile. Power BI can refresh from an online source
without an on-premises data gateway, but it cannot reach every source directly — some
publishers sit behind bot protection that blocks Power Query, and some require a
transform step first. This repo is that step's output: a clean CSV at a fixed URL that
Power BI Service refreshes on a schedule with Anonymous credentials.

Every update is a commit, so each file also carries an immutable, timestamped record
of exactly what it contained on any given date.

---

## Feeds

### FSIS Salmonella Verification Testing — monthly establishment categories

| | |
| --- | --- |
| Path | `data/fsis-salmonella-verification/FACT_MONTHLY_SALMONELLA_CATEGORIES.csv` |
| Raw URL | `https://raw.githubusercontent.com/mikebarnas/public-data-feeds/main/data/fsis-salmonella-verification/FACT_MONTHLY_SALMONELLA_CATEGORIES.csv` |
| Grain | one row per establishment × product class × posting month |
| Coverage | 2019-01 → present, 90 posting months, ~65k rows |
| Update cadence | monthly, within days of the FSIS posting |
| Source | [USDA FSIS Salmonella Verification Testing Program Monthly Posting](https://www.fsis.usda.gov/science-data/data-sets-visualizations/microbiology/salmonella-verification-testing-program-monthly) |
| Licence | US federal government work, public domain |

Columns, in order:

| Column | Notes |
| --- | --- |
| Establishment Number | FSIS establishment number, e.g. `P1243` |
| Establishment Name | normalized via a curated crosswalk, so a plant keeps one label over time |
| City | |
| State | two-letter, uppercase |
| City State | `"City, ST"`, with a small override table for exceptions |
| District Number | zero-padded, e.g. `05` |
| Product Class | Young Chicken Carcasses, Young Turkey Carcasses, Chicken Parts, Comminuted Chicken, Comminuted Turkey |
| Salmonella Category | `1`, `2`, or `3`. Rows with a blank / `NA` category are excluded |
| Posted Date Key | year + zero-padded month + unpadded day, e.g. `2026081` |
| Posted Date | ISO, first of the posting month, e.g. `2026-08-01` |

Business key: `Establishment Number` + `Product Class` + `Posted Date Key`.

Built by [fsis-monthly-salmonella-verification](https://github.com/mikebarnas/fsis-monthly-salmonella-verification)
(private), which handles discovery, download, standardization, QA and stacking.

---

### FSIS MPI Establishment Directory — company roll-up dimension

| | |
| --- | --- |
| Path | `data/fsis-establishments/DIM_MPI_ESTABLISHMENTS.csv` |
| Grain | one row per establishment number per status (Active / Inactive) |
| Coverage | 8,171 rows — 7,237 active + 934 inactive — 54 columns |
| Update cadence | FSIS rewrites the directory most weekdays; **weekly** is plenty |
| Source | [MPI Directory by Establishment Number](https://www.fsis.usda.gov/sites/default/files/media_file/documents/MPI_Directory_by_Establishment_Number.csv), [Establishment Demographic Data](https://www.fsis.usda.gov/sites/default/files/media_file/documents/Dataset_Establishment_Demographic_Data.csv), [Inactive Directory](https://www.fsis.usda.gov/sites/default/files/media_file/documents/MPI_Inactive_Directory_by_Establishment_Number.csv) |

This is the conformed dimension every other feed joins to. Key columns:

| Column | Notes |
| --- | --- |
| `establishment_number` | normalized: unhyphenated, uppercase. Compound grants stay whole (`M1015+P1015+V1015`) |
| `EstNumber_0` … `EstNumber_3` | the compound grant split on `+`, so a join can match on any component |
| `standardized_name` | curated crosswalk applied; 507 rows resolve to a different spelling than FSIS ships |
| `company_name` | first DBA from the semicolon-delimited `dbas` field, falling back to `standardized_name` |
| `est_status` | `Active` or `Inactive`; `LatestMPIActiveDate` is populated on the 934 inactive rows |
| `modified_line_speed_flag` | `Y` on the 44 waiver establishments, matched on any grant component |

`Establishment Number` and `Modified Line Speed Flag` are retained as duplicates of
their snake_case equivalents purely so an existing Power BI model keeps working. New
work should use the snake_case columns.

### FSIS Modified Line Speed Waiver Participants

| | |
| --- | --- |
| Path | `data/fsis-modified-line-speed/DIM_FSIS_MODIFIED_LINE_SPEED.csv` |
| Grain | one row per participating establishment |
| Coverage | 44 establishments, all granted 2023-03-31 |
| Update cadence | **monthly**; FSIS edits the page only when a waiver changes |
| Source | [Modernization of Poultry Slaughter Inspection](https://www.fsis.usda.gov/inspection/inspection-programs/inspection-poultry-products/modernization-poultry-slaughter) |

Read from an HTML table on the page, not from a file. FSIS writes the numbers hyphenated
(`P-7927`); they are normalized here to match every other feed.

### FSIS Raw Poultry Product Sampling — samples and isolates

| | |
| --- | --- |
| Samples | `data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_SAMPLES_FY####.csv` (one per fiscal year) |
| Manifest | `data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_SAMPLES_MANIFEST.csv` |
| Isolates | `data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_ISOLATES.csv` |
| Lookups | `DIM_FSIS_SAMPLING_PROJECT.csv` (34 rows), `DIM_FSIS_SAMPLE_SOURCE.csv` (11 rows) |
| Coverage | FY2014–FY2026, 318,175 samples and 71,882 isolates, 2013-10-01 → 2026-03-31 |
| Update cadence | **quarterly**, and restated in bulk — see below |
| Source | [Raw Poultry Sampling](https://www.fsis.usda.gov/news-events/publications/raw-poultry-sampling) |

**Two tables, on purpose.** The upstream JSON holds a primary array (one record per
sample) and a secondary array (one record per pathogen isolate), joined on `form_id`,
and the relationship is one-to-many. FSIS says so in the file's own metadata: *"In rare
cases, a single secondary test can be performed multiple times. This occurs when
multiple Salmonella serotypes are identified from a single positive sample."*

It is not rare. Across FY2014–FY2026, **6,053 isolate records sit on a `form_id` that
carries more than one**, overwhelmingly a Salmonella isolate and a Campylobacter isolate
recovered from the same bird. Any flatten that produces one row per sample discards
them. That is the entire reason the isolate table is published separately.

**Keys.**

- Samples: `sample_key` = `form_id` + `sample_number`. **Not `form_id` alone** — the
  metadata states a form can carry multiple sample numbers, and two form_ids in the
  history really do.
- Isolates: `isolate_key`, a deterministic surrogate. The source has no reliable natural
  isolate key: `secondary_sample_number` repeats on 14 rows and is blank on 12, and
  `form_id` + `pathogen_number` repeats on 31 groups because two serotypes recovered
  from one sample share a `pathogen_number`.

**Derived columns that exist to prevent a specific wrong answer.** Only about 54% of
samples are selected for Campylobacter analysis; the rest come back empty. Dividing
Campylobacter positives by *all* samples understates the rate by roughly half. Use:

| Column | Use |
| --- | --- |
| `salmonella_analyzed` / `campylobacter_analyzed` | `Y`/`N` — **the correct denominator** |
| `salmonella_positive` / `campylobacter_positive` | `1`/`0` numerator |
| `salmonella_result` / `campylobacter_result` | the coalesced result |
| `campylobacter_method` | `1 mL` or `30 mL` |
| `pathogen` / `serotype_species` | on isolates: organism-agnostic, so one visual serves both |

`campylobacter_method` matters: FSIS switched from a 1 mL to a 30 mL method during
FY2019, and `campylobacter_analysis_1ml` is **empty on every row from FY2021 onward**.
A measure pointed at the 1 mL column alone silently collapses to zero.

**Known upstream defects**, re-verified on every run: 12 isolate records carry a blank
`pathogen_number`, and 6 isolate records across 5 form_ids have no primary sample row in
any fiscal year file — an inner join drops them silently.

**Why one file per fiscal year.** As a single table the samples fact is 91 MiB, past
GitHub's 50 MiB warning and near its 100 MiB block, and it grows about 4 MiB a year.
Split by fiscal year it is roughly 4 MiB per file, and a normal quarterly refresh
rewrites only the current year.

### CDC BEAM Dashboard

| | |
| --- | --- |
| Path | `data/cdc-beam/FACT_CDC_BEAM.csv` |
| Grain | year × month × state × source type × source site × pathogen × serotype/species |
| Coverage | 255,779 rows, 2018-01 → 2026-07 |
| Update cadence | **monthly** |
| Source | [CDC BEAM Dashboard, Socrata `jbhn-e8xn`](https://data.cdc.gov/d/jbhn-e8xn) |

`Source Type` and `Source Site` are new here and were missing entirely from the previous
extract, which is why a `Source` slicer built on it had nothing in it.

**Read this before using BEAM for poultry.** `Source Type` only takes the values
`Human`, `Food`, `Animal`, `Environment`, and `Source Site` is the specimen isolation
site (`Stool`, `Blood`, `Urine`, `Other`) — all 36,345 Food isolates are `Other`. **This
dataset cannot attribute isolates to chicken or turkey.** CDC keeps commodity breakdowns
in the interactive dashboard only. Poultry attribution has to come from the FSIS product
class or from the NORS IFSAC Category.

### CDC NORS — outbreak surveillance

| | |
| --- | --- |
| Path | `data/cdc-nors/FACT_CDC_NORS.csv` |
| Grain | one outbreak record (`RECORD_KEY`) |
| Coverage | 66,713 rows, through 2023 |
| Update cadence | **annual, on change** — see below |
| Source | [CDC NORS, Socrata `5xkq-dg7x`](https://data.cdc.gov/d/5xkq-dg7x) |

**This source is stale, and checking it monthly is wasted effort.** Socrata reports
`rowsUpdatedAt` of 2024-12-20 and the maximum `Year` is 2023. NORS is an annual release.
The scheduled check compares `rowsUpdatedAt` and only rebuilds when it moves.

`RECORD_KEY` is added here because the source has no outbreak identifier at all, and its
19-column composite collides on 1,428 rows. It is a deterministic hash of those 19
fields plus an occurrence ordinal.

**Use `IFSAC Category` for commodity, not `Animal Type`.** `Animal Type` is populated on
only 587 of 66,713 rows and means the animal a person *touched*, under
`Primary Mode = Animal contact`. The poultry figure you want is `IFSAC Category`:
Chicken 12,407 + Turkey 7,596 illnesses.

---

## Reading the fiscal-year sample files in Power BI

The manifest lists every sample file and its URL, so nothing needs hardcoding and new
fiscal years appear on their own:

```m
let
    Manifest = Csv.Document(Web.Contents(
        "https://raw.githubusercontent.com/mikebarnas/public-data-feeds/main/data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_SAMPLES_MANIFEST.csv"),
        [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    Header   = Table.PromoteHeaders(Manifest, [PromoteAllScalars=true]),
    Files    = Table.AddColumn(Header, "Data", each
                 Table.PromoteHeaders(
                   Csv.Document(Web.Contents([url]),
                     [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
                   [PromoteAllScalars=true])),
    Combined = Table.Combine(Files[Data])
in
    Combined
```

Set every column to Text on import, then convert deliberately. Leading zeros in `zip`,
`district` and `fips_code` are preserved in these feeds and Power BI will strip them if
it guesses the type.


## Reading a feed from Power BI

No gateway is needed. Microsoft's guidance is that semantic models on cloud sources
"don't require a data gateway if Power BI can establish a direct network connection to
the source," and the refresh support matrix covers all online sources shown in Power
Query. See [Data refresh in Power BI](https://learn.microsoft.com/en-us/power-bi/connect-data/refresh-data).

1. **Get data → Web**, and paste the raw URL exactly as listed above.
2. Authentication: **Anonymous**. Privacy level: **Public**.
3. Set the column types. `Posted Date` is ISO, so pick Date. Keep
   `Posted Date Key`, `District Number` and `Salmonella Category` as **Text** —
   `District Number` is zero-padded and will lose its leading zero as a number.
4. Publish, then **Semantic model → Settings → Scheduled refresh**.

### Two things that will bite you

- **Keep the URL static.** Do not build it in code from a date or parameter. Dynamic
  data sources generally cannot refresh in the Power BI Service, with narrow
  exceptions for `RelativePath` / `Query` on `Web.Contents`
  ([Microsoft Learn](https://learn.microsoft.com/en-us/power-bi/connect-data/refresh-data)).
  The paths here are deliberately fixed and do not carry a month in the filename.
- **`raw.githubusercontent.com` is cached.** A refresh within a few minutes of a push
  can serve the previous version. Refreshes scheduled hours after the update are
  unaffected.

### Reading a specific historical version

Substitute a commit SHA for `main` in the raw URL to pin the exact bytes as of that
commit. Useful for reproducing a number that was reported previously — see below for
why that matters.

---

## Provenance, and why the history is not immutable upstream

FSIS **rewrites already-published monthly files.** Checking the `Last-Modified` header
on the hosted `.xlsx` files on 2026-08-27 returned:

| Posting month | File last written upstream |
| --- | --- |
| 2026-08 | 2026-08-06 |
| 2026-07 | 2026-07-01 |
| 2025-11 | 2026-02-12 |
| 2025-10, 2025-12, 2026-01 … 2026-06 | **all 2026-06-11** |

Eight months of already-public data were bulk-republished on a single day. Values that
changed in that batch include an establishment's city and district number, several
establishment names, and at least two months gained a row.

The practical consequence: a figure pulled from the current upstream file will not
always match a figure reported from the same month last quarter. The commit history in
this repo is what makes the reported version recoverable.

---

## File size

Each feed is a plain CSV, kept well inside GitHub's limits — Git warns above 50 MiB and
GitHub blocks above 100 MiB
([GitHub Docs](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)).
The Salmonella feed is ~6 MB and grows ~70 KB per month.

## Adding another feed

One directory per source under `data/`, one CSV per grain, a fixed filename that never
encodes a date, and a row in the Feeds table above with the raw URL, grain, cadence and
upstream source link.
