# Public data feeds — catalog, Power BI URLs, and what the data revealed

Built 2026-08-28. Public feed repo commit `c1c21b3`, pipeline repo commit `4dd8c9c`.

---

## The six feeds

All URLs are anonymous, no gateway, no credentials. Prefix every path with:

`https://raw.githubusercontent.com/mikebarnas/public-data-feeds/main/`

| Feed | File | Rows | Size | Cadence |
| --- | --- | --- | --- | --- |
| Salmonella verification (existing) | `data/fsis-salmonella-verification/FACT_MONTHLY_SALMONELLA_CATEGORIES.csv` | 65,423 | 6.3 MiB | monthly |
| MPI establishments | `data/fsis-establishments/DIM_MPI_ESTABLISHMENTS.csv` | 8,171 | 2.9 MiB | weekly |
| Modified line speed | `data/fsis-modified-line-speed/DIM_FSIS_MODIFIED_LINE_SPEED.csv` | 44 | 4 KiB | on change |
| Raw poultry samples | `data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_SAMPLES_FY####.csv` ×13 | 318,175 | ~4 MiB each | quarterly |
| Raw poultry isolates | `data/fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_ISOLATES.csv` | 71,882 | 33.6 MiB | quarterly |
| CDC BEAM | `data/cdc-beam/FACT_CDC_BEAM.csv` | 255,779 | 30.5 MiB | monthly |
| CDC NORS | `data/cdc-nors/FACT_CDC_NORS.csv` | 66,713 | 10.0 MiB | annual, on change |

Plus two small lookups alongside the sampling data — `DIM_FSIS_SAMPLING_PROJECT.csv`
(34 project codes) and `DIM_FSIS_SAMPLE_SOURCE.csv` (11 sample sources) — and
`FACT_FSIS_RAW_POULTRY_SAMPLES_MANIFEST.csv`, which lists all 13 fiscal-year files and
their URLs so Power BI can combine them without hardcoding anything. The README in the
public repo has the M code.

---

## Six findings in the existing report

### 1. Campylobacter positivity is understated by roughly half

Not every sample is selected for Campylobacter analysis, and coverage collapsed from
about 95% of samples through FY2023 to roughly 52% from FY2024 onward. Unselected samples
return empty. The report divides both Salmonella and Campylobacter positives by the same
total sample count, so the Campylobacter rate is wrong. Correct rates, using only analyzed samples:

| Fiscal year | Correct | With the all-samples denominator |
| --- | --- | --- |
| FY2022 | 15.33% | 12.82% |
| FY2023 | 19.13% | 17.44% |
| FY2024 | 18.99% | 9.45% |
| FY2025 | 20.45% | 10.66% |
| FY2026 (part) | 18.34% | 9.86% |

The gap widens sharply from FY2024 because Campylobacter analysis coverage dropped —
only 14,398 of 28,941 samples in FY2024, versus 29,211 of 32,035 in FY2023.

Use `campylobacter_analyzed = 'Y'` as the denominator and `campylobacter_positive = '1'`
as the numerator. Same pattern for Salmonella.

### 2. The 0.00% Campylobacter rate for 2025 is a method column that went empty

FSIS moved from a 1 mL to a 30 mL Campylobacter method during FY2019.
`campylobacter_analysis_1ml` is empty on **every row from FY2021 onward**. Any measure
reading only the 1 mL column silently collapses to zero, which matches the 0.00% on the
Campylobacter Details page. The feed now carries `campylobacter_result` (coalesced across
both methods) and `campylobacter_method` so this cannot recur.

### 3. 6,054 pathogen isolates were being dropped

The upstream JSON is a one-to-many primary/secondary structure joined on `form_id`, and
FSIS documents it: *"a single secondary test can be performed multiple times... when
multiple Salmonella serotypes are identified from a single positive sample."*

Across FY2014–FY2026, **6,050 form_ids carry more than one isolate**, holding 12,104 isolate records between
them — so a one-row-per-sample flatten drops **6,054** of them. In FY2026 alone there are 246 such samples: 241 are a Salmonella isolate plus a
Campylobacter isolate from the same bird, and 5 are two different Salmonella serotypes
from one sample. One row per sample loses one of each pair. This directly affects the
serotype trend page, which is the point of that data.

Hence two tables. Top serotypes and species across the full history:

- Salmonella: Kentucky 9,948, Infantis 8,353, Enteritidis 5,972, Typhimurium 3,110, Schwarzengrund 2,461, Heidelberg 1,106
- Campylobacter: coli 17,719, jejuni 16,996, lari 83, not speciated 86

### 4. NORS figures in the report are exactly 7× the true values

The Poultry illness count of 80,073 is exactly 7.0× the true 11,439. Every other
category is also exactly 7.0× — Turtle 10,157 vs 1,451, Cattle 9,030 vs 1,290, total
118,573 vs 16,939 — and hospitalizations and deaths too. A uniform integer multiplier
across every category is a cardinality fan-out in the Power BI model, not a data problem.
The likely enabler is that NORS has no unique row identifier at all, so nothing anchors
the grain. The feed now supplies `RECORD_KEY`, a deterministic hash, which gives the
model something to key on.

Also: use `IFSAC Category` for commodity, not `Animal Type`. `Animal Type` is populated
on 587 of 66,713 rows and means the animal a person *touched*. The poultry number you
want is IFSAC Chicken 12,407 + Turkey 7,596 illnesses.

### 5. BEAM cannot attribute isolates to poultry

`source_type` and `source_site` were missing from the old extract entirely, which is why
the `Source` slicer was empty. They are now included and fully populated — but the values
are only `Human / Food / Animal / Environment` and `Stool / Blood / Urine / Other`. All
36,345 Food isolates are `Other`. CDC keeps commodity breakdowns in the interactive
dashboard only, so BEAM cannot answer "how much of this is chicken." Poultry attribution
has to come from the FSIS product class or NORS IFSAC Category. Worth saying plainly to
Phibro rather than implying BEAM supports it.

### 6. CDC mislabels a count as a percentage in BEAM

The BEAM column `% Isolates with clinically important antimicrobial resistance` is a
**count of resistant isolates, not a percentage**. Every one of the 2,604 populated values
is a whole number, every value is less than or equal to the `Number of sequenced isolates
analyzed by NARMS` on the same row, and two values exceed 100. This is what the
"Percent AMR Campy by State" visual was plotting when its axis read 9.8K / 8.7K / 8.1K.

The Power BI query renames it to `Resistant isolates` on import so it cannot be averaged
by accident, and the real rate is that column divided by the NARMS sequenced count.

The same denominator trap sits in the FSIS isolate table: a null resistance flag means the
isolate was never tested, which is not the same as "No". Only 27,855 of 71,882 isolates
have a tetracycline AST result at all, so dividing by total isolates understates
resistance by roughly 60%.

---

## Also worth knowing

**Establishment name standardization.** The 598-entry crosswalk is now applied in the
dimension as `standardized_name`, plus `company_name` from the DBA field for the roll-up.
It resolves the "Tyson Foods, Inc." / "Tyson Foods Inc" split — five spellings collapse to
one "Tyson Foods". Coverage is honest but incomplete: 774 of 8,171 establishment names
match the crosswalk and 507 resolve to a different spelling; the rest fall back to the raw
FSIS name. On the line speed list, 32 of 44 match. Unmatched names worth adding include
`Pilgrim's Pride Corp.`, `Case Farms`, and `Mar-Jac Poultry`.

**Leading zeros.** The previous establishment extract had lost leading zeros in `zip`,
`district`, and `fips_code` (`05` became `5`) — those columns matched the source on only
84–91% of rows. The feeds preserve them as text. If anything in the model joins on the
numeric form, it needs checking.

**FSIS restates history in bulk.** All twelve historical fiscal-year sampling files were
rewritten on 2026-04-03, the same pattern as the Salmonella file where eight posting
months were republished on 2026-06-11. The refresh watches `Last-Modified` on every
fiscal year, not just the current one, so a restatement is caught rather than missed.

**Upstream defects, re-verified every run.** 12 isolate records carry a blank
`pathogen_number`. 6 isolate records across 5 form_ids have no primary sample row in any
fiscal year file — an inner join drops them silently.

**Keys are not what they look like.** The sample key is `form_id` + `sample_number`, not
`form_id`: FSIS states a form can carry multiple sample numbers, and two form_ids in the
history do. The isolate table has no reliable natural key at all
(`secondary_sample_number` repeats on 14 rows and is blank on 12), so `isolate_key` is a
deterministic surrogate.

**Governance.** Publishing these raw feeds is fine — all inputs are public domain US
federal works, including the `dbas` field driving the company roll-up. But the report is
marked Confidential and the "Customer Trends" / "Customer Directory" framing is Phibro
work product. If a Phibro customer flag is ever added to the dimension, it must not go
into the public repo.

---

## How refresh works

One weekly task, Mondays at 9:15 AM Eastern, instead of five schedules on five guessed
cadences. It checks a cheap freshness signal per source — HTTP `Last-Modified` for the
FSIS files, Socrata `rowsUpdatedAt` for the CDC ones — compares against `state.json`, and
rebuilds and publishes only what actually moved. Unchanged sources cost one HEAD request.
Nothing moved means no commit and no notification.

Expected behaviour: the MPI directory changes most weekdays so it will usually rebuild;
BEAM lands monthly; raw poultry sampling is quarterly; the line speed page and NORS may
not move for a year. NORS has been stale since 2024-12-20 with a maximum year of 2023, so
the previous monthly reload was reprocessing identical data.

Two guards. A feed whose row count falls below 90% of the previous run is refused and not
published, because a sudden collapse is far more likely an upstream outage or a parser
regression than real change. And upstream state is only recorded for feeds that rebuilt
cleanly, so a failed run retries next week instead of marking itself done.

Manual commands:

```bash
cd /home/user/workspace/public_data_pipelines
python3 refresh.py --report-only          # what moved, change nothing
python3 refresh.py --publish              # rebuild what moved and push
python3 refresh.py --publish --force cdc_nors   # rebuild one regardless
python3 run_feed.py raw_poultry           # rebuild one feed, no publish
```

Sources: [FSIS MPI Directory](https://www.fsis.usda.gov/science-data/developer-resources/mpi-api),
[FSIS Raw Poultry Sampling](https://www.fsis.usda.gov/news-events/publications/raw-poultry-sampling),
[FSIS Modernization of Poultry Slaughter](https://www.fsis.usda.gov/inspection/inspection-programs/inspection-poultry-products/modernization-poultry-slaughter),
[CDC BEAM](https://data.cdc.gov/d/jbhn-e8xn),
[CDC NORS](https://data.cdc.gov/d/5xkq-dg7x),
[GitHub large file limits](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github).

### Incremental rebuild of the raw poultry feed

That feed is 13 fiscal-year zip files, and until now any rebuild re-downloaded and
re-derived all 13 even though FSIS normally only reposts the current year. Each year is now
cached in its derived form under `cache/derived/`, keyed on that file's upstream
`Last-Modified` header, so a rebuild only redoes the years that actually moved.

Measured on the full 318,175-sample series:

| Scenario | Time |
| --- | --- |
| Weekly check, nothing moved upstream | ~2 seconds |
| Rebuild, all years already cached | ~20 seconds |
| FSIS posts a new quarter (one year moved) | ~19 seconds |
| FSIS restates history (all 13 reposted) | ~1 minute 50 seconds |
| Cold start, no cached zips at all | ~20 minutes, nearly all of it downloading from FSIS |

The refactor was verified behaviour-preserving: a full rebuild from an empty cache produced
all 17 output files byte-for-byte identical to the published ones.

`cache/derived/` is about 205 MB and is not tracked in git. If it is ever cleared the next
rebuild simply takes the full path again — correctness never depends on it.

Two things guard the cache against serving stale data:

- Each entry is keyed on the upstream `Last-Modified`, so a repost invalidates that year.
- `DERIVE_VERSION` in `feeds/raw_poultry.py` is part of the cache filename. **Bump it whenever
  the derive logic changes**, or the cache will keep serving data built by the old code.

Files are also compared byte-for-byte before being written, so an unchanged file is not
rewritten and not archived. The run output marks each file `unchanged` or `WRITTEN`, which is
what lets the weekly notification name only the files that genuinely moved.

### Commands

```bash
# what the weekly scheduled task runs - checks freshness, rebuilds only what moved
python3 refresh.py --publish

# see what would happen, change nothing
python3 refresh.py --report-only

# USDA says historical data was restated: ignore every cache and re-derive everything
python3 refresh.py --publish --full --force raw_poultry

# one feed, no publish
python3 run_feed.py raw_poultry
```

Use `--full` after changing derive logic if you did not bump `DERIVE_VERSION`.
