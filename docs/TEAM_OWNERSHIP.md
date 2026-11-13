# Team ownership and integration

## Participants

- @srisaanvi06-11 (solo)

## Shared core

As a solo product, the common end-to-end path (data boundary, experiments,
artifact, API, interface, deployment, and monitoring) is owned by
@srisaanvi06-11. Review of each pull request is self-review against the PR
checklist and CI until a reviewer arrangement is announced in class.

## Individual vertical slices

Solo products still add one substantial vertical slice beyond the shared core.

**Slice: manager fingerprints and holder-base analysis**

- Need: analysts and allocators want to see which managers actually behave
  alike, independent of self-described style.
- Owned interfaces: fund and stock embedding pipeline, clustering, the
  nearest-manager lookup, and the `/fund/{cik}` endpoint.
- Tests: reconstruction, stability, and deterministic-seed tests; differential
  comparison of student PCA/SVD against a trusted library.
- Related issues and pull requests: #7 (product and data contract).

## Integration and conflict history

To be recorded at the first consequential integration decision (expected: how
fund embeddings feed the exit classifier without leaking *t+1* information).

## Maintenance responsibility

@srisaanvi06-11 investigates defects in every component through final review.
