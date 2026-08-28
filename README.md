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
