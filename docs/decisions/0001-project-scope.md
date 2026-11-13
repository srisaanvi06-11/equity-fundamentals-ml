# ADR 0001: Project scope

- Status: proposed
- Date: 2026-11-13

## Context

The final product needs one domain that supports a regression task, a
classification task, and an unsupervised or representation-learning task, using
lawful data that can be documented and partly fixtured for CI. The intended
users are equity analysts, shareholder-advisory teams, and allocators who need
to understand institutional holder behavior.

## Decision

Build a holder-behavior product on SEC Form 13F Data Sets:

- unsupervised fund and stock embeddings from the fund x stock weight matrix,
  with clustering of managers;
- position-exit classification for (fund, stock, quarter); and
- change-in-ownership-breadth regression for (stock, quarter).

Identity is by CUSIP and CIK. Features obey a 45-day filing-deadline
information boundary. Splits are by time.

## Alternatives considered

- **Stock-return prediction from fundamentals.** A common choice with a
  very low signal-to-noise ratio. Evaluation would mostly show that models
  barely beat a baseline, and price data raise licensing questions.
- **Data-driven comparable-company sets from XBRL fundamentals.** Viable and
  broadly useful, but its unsupervised task is less central to the product, and
  late-filing labels are rare.
- **Bank stress from FDIC call reports.** Viable, with a clear failure label,
  but less aligned with the intended users.

## Consequences

- Benefits: the unsupervised representation is the backbone of the
  supervised tasks rather than a separate exercise, and the data are public
  domain with a long history.
- Costs: significant data engineering (amendment deduplication, report-quarter
  re-keying, value-unit change, CUSIP lifecycle).
- Risks: matrix size (thousands of funds by thousands of CUSIPs per quarter)
  can exceed the memory and runtime of quadratic methods; mitigated by sparse
  storage and documented subsampling.
- Revisit if the first data pull shows the deduplication or unit issues cannot
  be resolved reliably within the data-boundary milestone.
