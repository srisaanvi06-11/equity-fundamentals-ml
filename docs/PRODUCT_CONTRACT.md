# Product contract

Status: draft for milestone 1 (issue #7). Items marked **OPEN** are decisions
to settle before the data-boundary milestone.

## Product in one sentence

A holder-behavior tool built from public SEC Form 13F filings. It learns a
behavioral "fingerprint" for every institutional manager, then uses those
fingerprints to estimate which holders of a stock are likely to exit and how
institutional ownership breadth is likely to change next quarter.

## Users and decisions

| User | Question the product supports |
| --- | --- |
| Equity analyst (buy side or sell side) | Who owns this stock, which behavioral groups dominate the holder base, and how crowded is it? |
| Shareholder-advisory, IR, or activism-defense team | Which current holders behave like sellers, and which managers look like the existing holders (targeting)? |
| Allocator, endowment, or OCIO analyst | Does a manager behave like its stated style, or like an index or a different peer group (style drift, closet indexing)? |

The product supports investigation. It must **not** automate:

- trading, rebalancing, or position sizing decisions;
- claims that a specific named manager *is* a closet indexer or has drifted
  from its mandate (the model gives a similarity measurement, not a finding);
- predictions of how a manager will vote in a proxy contest; or
- any claim about a manager's short, bond, derivative, or non-US exposure,
  which 13F filings do not report.

## Dataset suite

| Dataset | Role | Source and license | Observational unit |
| --- | --- | --- | --- |
| SEC Form 13F Data Sets (submission, cover page, info table) | Holdings, filers, filing dates | SEC DERA, public domain | One reported holding line in one 13F filing |
| SEC Official List of Section 13(f) Securities | Universe of reportable CUSIPs and quarterly additions/deletions | SEC Division of Investment Management, public | One CUSIP in one quarterly list |
| OpenFIGI mapping (optional) | CUSIP to ticker and security type for display | Bloomberg OpenFIGI public API | One CUSIP |

Full provenance, schema, and limits are in [`DATASET_CARD.md`](DATASET_CARD.md).

**Time boundary.** Proposed study window: report quarters 2015Q1 through the
latest quarter whose next-quarter filings are published. 13F XML data begin in
2013; the first two years are reserved as warm-up history for lagged features.

## Learning tasks

Notation: fund *f*, stock (CUSIP) *s*, report quarter *t*. *P(t)* is the panel
of funds that file a holdings report for both *t* and *t+1*.

### Task U: manager and stock representations (unsupervised)

- Input: the fund x stock matrix *W_t*, where *W_t[f, s]* is the market-value
  weight of *s* in fund *f*'s reported long equity portfolio (rows sum to 1).
- Method: low-rank factorization (PCA / truncated SVD) to produce *k*-dimensional
  fund and stock embeddings, then clustering of fund embeddings.
- Question: which behavioral groups of managers exist, and do they persist
  across quarters?
- No ground-truth labels. Evaluation uses reconstruction error, cluster
  stability across adjacent quarters and bootstrap resamples, and analyst
  interpretability. Any external labels (for example, index-fund names) are
  audit evidence only, never fitting targets.

### Task C: position exit (classification)

- Unit: one long equity position (*f*, *s*) held at *t* with *f* in *P(t)*.
- Target: `exit = 1` if *f* reports zero shares of *s* at *t+1*, else 0.
- Exclusions (to keep the label a decision, not an accident):
  - positions in CUSIPs deleted from the Official List between *t* and *t+1*
    (delisting, acquisition, CUSIP change), which force an exit;
  - put/call option lines; and
  - funds that stop filing (not in *P(t)*): their outcome is unobserved, not 0.

### Task R: change in ownership breadth (regression)

- Unit: one stock *s* at quarter *t*.
- Target: Δbreadth = (N_{t+1}(s) − N_t(s)) / |P(t)|, where N counts panel funds
  holding *s*. Breadth needs no shares-outstanding data, which 13F does not
  contain.
- **OPEN:** a secondary target, the log change in aggregate panel shares held.

### Information boundary (leakage rule)

13F reports are due 45 days after quarter end. Features for quarter *t* may use
only filings with `FILING_DATE` on or before the 45-day deadline for *t*. Late
filings and amendments filed after that cutoff are excluded from features even
though they exist in the data set. Labels come from *t+1* filings and are never
visible to features.

## Task-algorithm matrix

| Course family | Project task | Applicable? | Mathematical justification | Evidence |
| --- | --- | --- | --- | --- |
| Linear regression | R | Yes | Continuous target; interpretable baseline for feature effects | Planned |
| Logistic regression | C | Yes | Binary target; calibrated probabilities matter for the exit score | Planned |
| Nearest neighbors | C, R, U | Yes | Similar funds behave similarly; also powers "nearest managers" in the product | Planned |
| Support vector machines and kernels | C | Yes, with a subsample | Binary margin classifier. Kernel methods scale at least quadratically in rows, so fit on a documented stratified subsample of positions | Planned |
| Decision trees | C, R | Yes | Nonlinear thresholds (position age, size rank) and mixed feature scales | Planned |
| Ensembles | C, R | Yes | Variance reduction over trees; expected strongest tabular candidate | Planned |
| Neural networks | C, R | Yes | Nonlinear interactions between embeddings and position features | Planned |
| Clustering | U | Yes | Groups of managers in embedding space | Planned |
| Dimensionality reduction | U (and features for C, R) | Yes | Fund x stock matrix is sparse and very high-dimensional; low rank is the central modeling assumption | Planned |

No family is excluded at this stage. **OPEN:** confirm the full course algorithm
list against the course site and add rows for any family not shown here.

## Evaluation contract

Frozen before any final-test result is inspected.

- **Splits by time, never at random.** Proposed: development 2015Q1–2021Q4,
  validation 2022Q1–2023Q4, sealed final test 2024Q1 onward. Rows from the same
  quarter never appear in two roles.
- **Baselines.**
  - C: base-rate classifier, and the rule "exit if the position was cut by
    more than half last quarter."
  - R: zero change, and persistence (last quarter's Δbreadth).
- **Metrics.**
  - C: precision-recall AUC (exits are a minority class), log loss, Brier
    score, and a calibration curve.
  - R: MAE and RMSE against both baselines, plus Spearman rank correlation.
  - U: held-out reconstruction error by rank *k*, and adjusted Rand index of
    cluster assignments across adjacent quarters.
- **Uncertainty.** Quarter-block bootstrap intervals for every reported metric.
- **Slices.** Fund size bucket, portfolio concentration, stock size bucket, and
  market regime (for example, 2020Q1).
- **Selection rule.** Choose the best validation candidate on the primary metric
  (C: PR-AUC; R: MAE). Promote it only if it beats the strongest baseline
  outside the bootstrap interval. Open the final test once.

## System interfaces

| Interface | Plan |
| --- | --- |
| Data boundary | Download script into `data/local/`, then a validated local database keyed by accession number |
| Training command | `uv run dsm-project train` with a versioned TOML configuration |
| Experiment artifact | Versioned model bundle with configuration, data snapshot identity, and metrics |
| Prediction API | `GET /stock/{cusip}` holder base and exit scores; `GET /fund/{cik}` nearest managers and cluster |
| User interface | Small web page served with the API |
| Deployment | Container image built by CI |
| Logs, metrics, alerts | Structured request logs; data-freshness and score-drift metrics |
| Rollback | Previous model bundle retained and selectable by version |

Details move to [`OPERATIONS.md`](OPERATIONS.md) at the production milestone.

## Responsible-use boundary

- **Privacy.** Filers are institutions, and the data are public. The product
  stores no personal data. Manager names appear only as filed with the SEC.
- **Misuse.** Exit scores must not be presented as knowledge of a manager's
  intentions or used to front-run specific managers. The interface labels every
  score as a model estimate built from data up to 45 days stale.
- **Uncertainty.** Every score is shown with its calibration context and the
  as-of date of the underlying filings.
- **Human oversight.** Cluster labels such as "index-like" are written by a
  person after review, not generated automatically.
- **What a successful demonstration does not prove:** that the scores predict
  returns, that a manager's full portfolio looks like its 13F (shorts, bonds,
  and non-US holdings are invisible), or that past behavior patterns hold in a
  new market regime.
