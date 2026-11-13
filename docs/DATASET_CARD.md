# Dataset card

Status: draft for milestone 1 (issue #7). Fields marked **TO RECORD** are
filled in at the first real data pull.

## Identity and provenance

### Primary: SEC Form 13F Data Sets

- Dataset name: Form 13F Data Sets
- Source owner: U.S. Securities and Exchange Commission, Division of Economic
  and Risk Analysis (DERA)
- Location: <https://www.sec.gov/data-research/sec-markets-data/form-13f-data-sets>
- Documentation: SEC "Form 13F Data Sets" readme (PDF), which defines the
  SUBMISSION, COVERPAGE, and INFOTABLE tables
- License: public domain (U.S. government work)
- Coverage: XML-based 13F submissions from 2013 to the present
- Retrieval method and date: **TO RECORD** (download script and run date)
- Immutable version: **TO RECORD** (SHA-256 of each downloaded archive)

### Secondary: Official List of Section 13(f) Securities

- Source owner: SEC Division of Investment Management
- Location: <https://www.sec.gov/divisions/investment/13flists>
- Format: quarterly fixed-width text file (CUSIP, option indicator, issuer
  name, issuer description, added/deleted status)
- Retrieval method and date: **TO RECORD**

### Optional: OpenFIGI identifier mapping

- Source owner: Bloomberg OpenFIGI (public API, free; an API key raises rate
  limits)
- Use: map CUSIPs to tickers and security types for display only. Model
  identity uses CUSIP, so a missing mapping never changes a prediction.
- Stored output: a cached mapping table with its retrieval date.

## Observation and sampling

- **One raw row:** one holding line in one 13F filing (accession number plus
  info-table row key). A single position can span several lines (for example,
  split across sub-managers); lines are aggregated to fund x CUSIP x quarter.
- **Population:** institutional investment managers with at least $100 million
  in Section 13(f) securities, which are mostly US-listed equities, ETFs, ADRs,
  and listed options.
- **Repeated units:** the same fund appears every quarter; the same stock is
  held by many funds. Observations are not independent, so splits are by
  quarter and uncertainty uses quarter-block resampling.
- **Important boundaries:** report quarter (`PERIODOFREPORT`) versus filing
  date (`FILING_DATE`). Since March 2024 the SEC packages files by filing
  window, not by report quarter, so every row is re-keyed to its report
  quarter.

## Schema and measurement

Fields used (names from the SEC readme; types confirmed at first pull):

| Field | Table | Meaning | Use |
| --- | --- | --- | --- |
| ACCESSION_NUMBER | all | Filing identifier | Join key, provenance |
| CIK | SUBMISSION | Filer identifier | Fund identity |
| FILING_DATE | SUBMISSION | Date filed with the SEC | Information-boundary cutoff |
| PERIODOFREPORT | SUBMISSION | Quarter-end reported | Report quarter |
| SUBMISSIONTYPE | SUBMISSION | 13F-HR, 13F-HR/A, 13F-NT, 13F-NT/A | Keep holdings reports and amendments |
| CUSIP | INFOTABLE | Security identifier | Stock identity |
| VALUE | INFOTABLE | Reported market value | Portfolio weights. **Check units**: reported as thousands of dollars in older filings and, as far as we know, whole dollars after the 2023 format change. A unit test enforces the convention. |
| SSHPRNAMT, SSHPRNAMTTYPE | INFOTABLE | Share or principal amount and type | Keep shares (SH) only |
| PUTCALL | INFOTABLE | Option flag | Exclude option lines |

**Leakage risks**

- Amendments filed after the 45-day deadline: excluded from features.
- Forced exits (delisting, acquisition, CUSIP change): excluded from the exit
  label using the Official List deletions.
- Survivorship: funds that stop filing are removed from the panel for that
  transition, not labeled as exits.

## Data access and privacy

- Commit: download and validation code, schema documentation, and a small
  constructed fixture under `data/fixtures/` that mimics the SEC tables.
- Keep local: raw archives and the built database under `data/local/`
  (git-ignored).
- Credentials: none for SEC data. An optional OpenFIGI key goes in `.env`
  (git-ignored), never in source.
- CI: runs on the fixture only and never downloads from the SEC.
- SEC fair-access rules require a descriptive User-Agent header on automated
  requests; the download script sets one from configuration.

## Representativeness and limitations

- Long positions in 13(f) securities only. No shorts, bonds (except certain
  convertibles), non-US listings, or cash. A hedged fund can look long-only.
- Managers below $100 million do not file.
- Quarter-end snapshots only; trading within a quarter is invisible, and
  window dressing at quarter end is possible.
- Filed "as is": the SEC notes the data can contain redundancies and
  inconsistencies across amendments. A deduplication rule (latest
  pre-cutoff holdings report or amendment per fund per quarter) is required
  and tested.
- One legal filer can cover several advisers, and one adviser can file under
  several CIKs, so "fund" means filing entity, not investment team.
- Confidential-treatment requests let some managers delay disclosing
  positions, so some positions appear late or not at all.
- The data are not evidence about managers' intentions, returns, or full
  portfolios.
