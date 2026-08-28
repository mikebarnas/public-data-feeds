# Power BI M queries for the public data feeds

Copy-ready. Create each as a blank query (Home → Transform data → New Source → Blank Query
→ Advanced Editor), paste, and name the query exactly as the heading says. Build them in
the order below — the first two are shared and everything else depends on them.

Every feed arrives as all-text on purpose, and each query then converts types
deliberately. That is what protects the leading zeros in `zip`, `district`, `fips_code`
and `sample_source_key` that the old extract was silently destroying.

---

## 1. `pFeedBase` — the one place the URL lives

```m
"https://raw.githubusercontent.com"
```

## 2. `fnFeedCsv` — shared loader

```m
let
    fnFeedCsv = (relativePath as text) as table =>
        let
            Raw = Web.Contents(
                      pFeedBase,
                      [ RelativePath = "mikebarnas/public-data-feeds/main/data/" & relativePath ]
                  ),
            Csv = Csv.Document(
                      Raw,
                      [ Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv ]
                  ),
            Promoted = Table.PromoteHeaders( Csv, [PromoteAllScalars = true] ),
            // CSV blanks arrive as empty strings; make them real nulls so
            // COUNT/DISTINCTCOUNT and relationships behave.
            Nulled = Table.ReplaceValue(
                         Promoted, "", null, Replacer.ReplaceValue,
                         Table.ColumnNames(Promoted)
                     )
        in
            Nulled
in
    fnFeedCsv
```

**Do not shortcut this by pasting a full URL into `Web.Contents`.** The base URL must be a
static literal and the changing part must go in `RelativePath`, otherwise the Power BI
Service refuses the credential and scheduled refresh fails with "the query references
other queries or steps, so it may not directly access a data source." Setting the base
once here is what makes all nine queries refresh unattended.

Credential, set once on first refresh: **Anonymous**, privacy level **Public**.

---

## 3. `DIM_MPI_ESTABLISHMENTS`

```m
let
    Source = fnFeedCsv("fsis-establishments/DIM_MPI_ESTABLISHMENTS.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"grant_date",           type date},
            {"load_date",            type date},
            {"LatestMPIActiveDate",  type date},
            {"category_start_date",  type date},
            {"category_end_date",    type date},
            {"grant_date_key",       Int64.Type},
            {"latitude",             type number},
            {"longitude",            type number}
        }
    ),
    // Everything else stays text on purpose. zip, district and fips_code carry
    // meaningful leading zeros ("07206", "05") that a numeric type destroys.
    Keys = Table.AddKey(Typed, {"establishment_number", "est_status"}, true)
in
    Keys
```

Grain is one row per establishment number per status, so the unique key is
`establishment_number` + `est_status` — 8,171 rows, 7,237 Active and 934 Inactive.

**If any existing relationship or merge in your model joins on `zip`, `district` or
`fips_code`, check it after this swap.** Those columns matched the source on only 84–91%
of rows in the old extract because leading zeros had been stripped. They are correct now,
which means a join built against the stripped version will stop matching.

## 4. `DIM_FSIS_MODIFIED_LINE_SPEED`

```m
let
    Source = fnFeedCsv("fsis-modified-line-speed/DIM_FSIS_MODIFIED_LINE_SPEED.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"date_granted",     type date},
            {"date_granted_key", Int64.Type}
        }
    ),
    Keys = Table.AddKey(Typed, {"establishment_number"}, true)
in
    Keys
```

44 rows. You do not need to relate this to anything — `modified_line_speed_flag` is
already resolved onto `DIM_MPI_ESTABLISHMENTS` for you, matched across all four components
of a compound grant so `P165S+V165S` is caught. Keep this table for the waiver date and
the `location` text.

## 5. `DIM_FSIS_SAMPLING_PROJECT`

```m
let
    Source = fnFeedCsv("fsis-raw-poultry-sampling/DIM_FSIS_SAMPLING_PROJECT.csv"),
    Keys   = Table.AddKey(Source, {"project_code"}, true)
in
    Keys
```

## 6. `DIM_FSIS_SAMPLE_SOURCE`

```m
let
    Source = fnFeedCsv("fsis-raw-poultry-sampling/DIM_FSIS_SAMPLE_SOURCE.csv"),
    Keys   = Table.AddKey(Source, {"sample_source_key"}, true)
in
    Keys
```

`sample_source_key` is text (`"0010"`). Leave it text on both sides of the relationship.

---

## 7. `FACT_FSIS_RAW_POULTRY_SAMPLES` — combines all 13 fiscal years

```m
let
    Manifest = fnFeedCsv("fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_SAMPLES_MANIFEST.csv"),

    // Pull each fiscal-year file named in the manifest. Using file_name (not the
    // manifest's url column) keeps the static-base rule above intact, so this
    // still refreshes in the Service.
    AddData = Table.AddColumn(
        Manifest, "Data",
        each fnFeedCsv("fsis-raw-poultry-sampling/" & [file_name]),
        type table
    ),
    Combined = Table.Combine(AddData[Data]),

    Typed = Table.TransformColumnTypes(
        Combined,
        {
            {"collection_date",                    type date},
            {"collection_date_key",                Int64.Type},
            {"fiscal_year",                        Int64.Type},
            {"salmonella_positive",                Int64.Type},
            {"campylobacter_positive",             Int64.Type},
            {"salmonellae_mpn_g",                  type number},
            {"salmonella_quantitative_cfu_g",      type number},
            {"salmonella_quantitative_cfu_ml",     type number},
            {"aerobic_plate_count_mpn_ml",         type number},
            {"aerobic_plate_count_mpn_g",          type number},
            {"aerobic_plate_count_mpn_100_sq_cm",  type number},
            {"enterobacteriaceae_mpn_ml",          type number},
            {"enterobacteriaceae_mpn_100_sq_cm",   type number}
        }
    ),
    Keys = Table.AddKey(Typed, {"sample_key"}, true)
in
    Keys
```

318,175 rows across FY2014–FY2026. When FSIS adds FY2027 the manifest gains a row and
this query picks it up with no edit.

The unique key is `sample_key`, **not `form_id`** — FSIS states a form can carry multiple
sample numbers and two form_ids in the history actually do. Use `sample_key` if you need a
one-row anchor.

## 8. `FACT_FSIS_RAW_POULTRY_ISOLATES`

```m
let
    Source = fnFeedCsv("fsis-raw-poultry-sampling/FACT_FSIS_RAW_POULTRY_ISOLATES.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"collection_date",     type date},
            {"collection_date_key", Int64.Type},
            {"fiscal_year",         Int64.Type}
        }
    ),
    // isolate_seq stays text ("01", "02"). The ~130 AST/WGS resistance columns
    // stay text as well: they are "Yes" / "No" / null, and null means the isolate
    // was never tested, which is not the same as "No".
    Keys = Table.AddKey(Typed, {"isolate_key"}, true)
in
    Keys
```

71,882 rows. This is a **second fact table at a finer grain**, not a replacement — 6,050
form_ids carry more than one isolate, and the old one-row-per-sample flatten was dropping
6,054 records, mostly a Salmonella and a Campylobacter isolate from the same bird.

Point every serotype and AMR visual at this table. Point sample counts and positivity
rates at `FACT_FSIS_RAW_POULTRY_SAMPLES`. Do not try to merge them back into one table.

## 9. `FACT_CDC_BEAM`

```m
let
    Source = fnFeedCsv("cdc-beam/FACT_CDC_BEAM.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"Year",                          Int64.Type},
            {"Month",                         Int64.Type},
            {"CALENDAR_KEY",                  Int64.Type},
            {"DATE",                          type date},
            {"date_key",                      Int64.Type},
            {"Load Date",                     type date},
            {"Number of isolates",            Int64.Type},
            {"Outbreak associated isolates",  Int64.Type},
            {"New multistate outbreaks",      Int64.Type},
            {"New multistate outbreaks - US", Int64.Type},
            {"Number of sequenced isolates analyzed by NARMS", Int64.Type},
            {"% Isolates with clinically important antimicrobial resistance", Int64.Type}
        }
    ),

    // CDC's column name is wrong: despite the "%" this is a COUNT of resistant
    // isolates, not a percentage. Every one of the 2,604 populated values is a
    // whole number, every value is <= the NARMS sequenced count on the same row,
    // and two exceed 100. Renaming it here is what stops it being averaged.
    Renamed = Table.RenameColumns(
        Typed,
        {{"% Isolates with clinically important antimicrobial resistance",
          "Resistant isolates"}}
    )
in
    Renamed
```

255,779 rows, 2018-01 through 2026-07. This is where your empty `Source` slicer gets
fixed: `Source Type` and `Source Site` were missing from the old extract entirely and are
now present and fully populated.

**But do not build a poultry visual on this table.** `Source Type` only takes
`Human / Food / Animal / Environment` and `Source Site` is the specimen site
(`Stool / Blood / Urine / Other`) — all 36,345 Food isolates are `Other`. BEAM cannot
attribute an isolate to chicken or turkey. That has to come from the FSIS product class or
the NORS IFSAC category.

## 10. `FACT_CDC_NORS`

```m
let
    Source = fnFeedCsv("cdc-nors/FACT_CDC_NORS.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"Year",             Int64.Type},
            {"Month",            Int64.Type},
            {"date_key",         Int64.Type},
            {"Load Date",        type date},
            {"Illnesses",        Int64.Type},
            {"Hospitalizations", Int64.Type},
            {"Deaths",           Int64.Type}
        }
    ),
    Keys = Table.AddKey(Typed, {"RECORD_KEY"}, true)
in
    Keys
```

**This is the query to load carefully.** Your current NORS figures are all exactly 7.0×
the truth — Poultry reads 80,073 against a real 11,439, and Turtle, Cattle, the total,
hospitalizations and deaths are every one of them 7.0× as well. A uniform integer
multiplier across every category is a fan-out in the model, and NORS ships with no
outbreak identifier at all, so nothing anchored the grain. `RECORD_KEY` gives you an
anchor at last. To be clear about what fixes what: the `Table.AddKey` line is only a hint
to the engine and does not enforce anything, so the actual fix is having `RECORD_KEY`
available as a unique row identifier and setting every relationship into this table to
single-direction many-to-one.

After loading, check the total: `SUM(Illnesses)` should be **16,939**, and IFSAC Chicken +
Turkey should be **12,407 + 7,596**. If you still see 118,573, a relationship is still
fanning out — look for a many-to-many or a bidirectional filter into this table.

Use `IFSAC Category` for commodity. `Animal Type` is populated on 587 of 66,713 rows and
means the animal a person touched.

## 11. `FACT_MONTHLY_SALMONELLA_CATEGORIES`

```m
let
    Source = fnFeedCsv("fsis-salmonella-verification/FACT_MONTHLY_SALMONELLA_CATEGORIES.csv"),
    Typed  = Table.TransformColumnTypes(
        Source,
        {
            {"Posted Date",      type date},
            {"Posted Date Key",  Int64.Type}
        }
    )
    // Establishment Number and District Number stay text - both keep leading zeros.
in
    Typed
```

Replacing your existing local-file query with this one is the change that ends the manual
monthly download. `Posted Date Key` keeps its historical unpadded-day form (`2024121`), so
existing measures against it still work.

---

## Model wiring

| From | To | Cardinality |
| --- | --- | --- |
| `FACT_FSIS_RAW_POULTRY_SAMPLES[establishment_number]` | `DIM_MPI_ESTABLISHMENTS[establishment_number]` | many-to-one |
| `FACT_FSIS_RAW_POULTRY_ISOLATES[establishment_number]` | `DIM_MPI_ESTABLISHMENTS[establishment_number]` | many-to-one |
| `FACT_FSIS_RAW_POULTRY_SAMPLES[project_code]` | `DIM_FSIS_SAMPLING_PROJECT[project_code]` | many-to-one |
| `FACT_FSIS_RAW_POULTRY_SAMPLES[sample_source_key]` | `DIM_FSIS_SAMPLE_SOURCE[sample_source_key]` | many-to-one |
| `FACT_FSIS_RAW_POULTRY_ISOLATES[sample_key]` | `FACT_FSIS_RAW_POULTRY_SAMPLES[sample_key]` | many-to-one |
| every `*_date_key` / `CALENDAR_KEY` | your date table's integer key | many-to-one |

Set all of these **single-direction**. Bidirectional filtering across two fact tables at
different grains is the most likely cause of the 7× fan-out described above.

One caveat on the establishment relationship: `DIM_MPI_ESTABLISHMENTS` is unique on
`establishment_number` + `est_status`, not on `establishment_number` alone, so a plain
single-column relationship will be rejected. Either filter the dimension to Active in a
separate query for that relationship, or relate on a composite you build. Filtering to
Active is usually what you want for current-state reporting:

```m
let
    Source = DIM_MPI_ESTABLISHMENTS,
    Active = Table.SelectRows(Source, each [est_status] = "Active"),
    Keys   = Table.AddKey(Active, {"establishment_number"}, true)
in
    Keys
```

All `*_date_key` columns are `yyyymmdd` integers, so one date table serves every fact.

---

## The measures that fix the wrong rates

The feeds now carry the correct denominators, but nothing changes until the measures use
them. These replace the ones producing 9.45% and 0.00%.

```dax
Samples = COUNTROWS ( FACT_FSIS_RAW_POULTRY_SAMPLES )

Salmonella Analyzed =
CALCULATE (
    COUNTROWS ( FACT_FSIS_RAW_POULTRY_SAMPLES ),
    FACT_FSIS_RAW_POULTRY_SAMPLES[salmonella_analyzed] = "Y"
)

Salmonella Positive = SUM ( FACT_FSIS_RAW_POULTRY_SAMPLES[salmonella_positive] )

Salmonella Percent Positive =
DIVIDE ( [Salmonella Positive], [Salmonella Analyzed] )

Campylobacter Analyzed =
CALCULATE (
    COUNTROWS ( FACT_FSIS_RAW_POULTRY_SAMPLES ),
    FACT_FSIS_RAW_POULTRY_SAMPLES[campylobacter_analyzed] = "Y"
)

Campylobacter Positive = SUM ( FACT_FSIS_RAW_POULTRY_SAMPLES[campylobacter_positive] )

-- The fix. Dividing by [Samples] instead of [Campylobacter Analyzed] is what
-- reported 9.45% for FY2024 when the real figure is 18.99%.
Campylobacter Percent Positive =
DIVIDE ( [Campylobacter Positive], [Campylobacter Analyzed] )

-- Worth putting on the page as a visible caveat. Coverage ran ~95% through
-- FY2023 and fell to ~52% from FY2024, which is why the gap widens.
Campylobacter Analysis Coverage =
DIVIDE ( [Campylobacter Analyzed], [Samples] )
```

Format the two percent measures as Percentage, 2 decimal places. After this, FY2024 reads
18.99% and FY2025 reads 20.45%.

For the AMR page, the same denominator trap applies — a null resistance flag means the
isolate was never tested, which is not "No":

```dax
Isolates = COUNTROWS ( FACT_FSIS_RAW_POULTRY_ISOLATES )

Tetracycline Tested =
CALCULATE (
    COUNTROWS ( FACT_FSIS_RAW_POULTRY_ISOLATES ),
    NOT ISBLANK ( FACT_FSIS_RAW_POULTRY_ISOLATES[salmonella_tetracycline_resistant_ast] )
)

Tetracycline Resistant =
CALCULATE (
    COUNTROWS ( FACT_FSIS_RAW_POULTRY_ISOLATES ),
    FACT_FSIS_RAW_POULTRY_ISOLATES[salmonella_tetracycline_resistant_ast] = "Yes"
)

Tetracycline Percent Resistant =
DIVIDE ( [Tetracycline Resistant], [Tetracycline Tested] )
```

Only 27,855 of 71,882 isolates have a tetracycline AST result at all, so dividing by
`[Isolates]` would understate resistance by roughly 60%. Repeat the pattern for whichever
of the other drugs the page shows.

And for BEAM, now that the mislabelled column is renamed:

```dax
BEAM Isolates = SUM ( FACT_CDC_BEAM[Number of isolates] )

BEAM Resistant Isolates = SUM ( FACT_CDC_BEAM[Resistant isolates] )

BEAM NARMS Sequenced = SUM ( FACT_CDC_BEAM[Number of sequenced isolates analyzed by NARMS] )

-- This is the actual percentage. The source column named "%" is a count, so the
-- old "Percent AMR Campy by State" visual was plotting 9.8K / 8.7K / 8.1K counts
-- on an axis labelled as a percent.
BEAM Percent Resistant =
DIVIDE ( [BEAM Resistant Isolates], [BEAM NARMS Sequenced] )
```

---

## Refresh settings

Set the dataset's scheduled refresh to **weekly, Monday afternoon Eastern** or later. The
feeds rebuild Monday at 9:15 AM Eastern, and `raw.githubusercontent.com` sends
`cache-control: max-age=300`, so anything from about 9:30 AM onward sees the new files.
Daily refresh is harmless if you prefer it — the files simply will not have changed.

No gateway, no credentials beyond Anonymous, nothing on your machine has to be running.

Sources: [Web.Contents and RelativePath for scheduled refresh](https://learn.microsoft.com/en-us/power-query/connectors/web/web),
[Power BI data refresh](https://learn.microsoft.com/en-us/power-bi/connect-data/refresh-data),
[CDC BEAM dataset](https://data.cdc.gov/d/jbhn-e8xn),
[CDC NORS dataset](https://data.cdc.gov/d/5xkq-dg7x),
[FSIS Raw Poultry Sampling](https://www.fsis.usda.gov/news-events/publications/raw-poultry-sampling).
