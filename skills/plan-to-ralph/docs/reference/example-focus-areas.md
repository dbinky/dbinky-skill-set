# Focus Areas Review Tracking

This document tracks the systematic review of [YOUR FEATURE/SYSTEM] for correctness, security, and robustness.

Each **single focus area** requires **2 full review passes** before being considered done.
Each **paired focus area** requires **1 full review pass** before being considered done.

A review pass is only checked when the checklist produces a passing result for that area.

**Progression order:** Address newly-introduced test failures first (regressions vs the recorded branch-point baseline), then the whole-pipeline wiring/provenance gaps (Table 0), then gaps vs. the plans/designs. Within the review tables: complete all Pass 1 single area reviews first, then paired area reviews (which catch integration issues while the code is fresh), then Pass 2 single area reviews.

---

## Table 0: Whole-Pipeline Wiring & Provenance (highest priority)

Each item is done only when the symbol/component is **wired-and-fed**: constructed at the real composition root AND fed real data from an existing producer — verified by `grep`, not by a green isolated test.

| # | Symbol / Component | Provenance check | Status |
|---|---|---|---|
| W1 | `EmbeddingSet` field on `RankRequest` (`path/to/rank/request.go`) | `grep -rn EmbeddingSet` — only assigned in `rank_test.go`; the real `Handler.Rank` constructs `RankRequest{}` with the field left empty. `[NOT-WIRED]` — wire the live embedder into the handler. | [ ] |
| W2 | `Scorer` collaborator on `Pipeline` (`path/to/pipeline/pipeline.go`) | `grep -rn NewScorer` — constructed in tests; `cmd/server/main.go` composition root passes `nil` for the scorer arg, so the prod path scores nothing. `[NOT-WIRED]` — construct and inject the real scorer at the composition root. | [ ] |
| W3 | `cache.Bucket` config (`path/to/config/cache.go`) | `grep -rn Bucket` — value is hardcoded to `"default"` at the call site instead of read from loaded config; the env var loads but is never fed to the client. List as a call-site provenance gap. | [ ] |
| W4 | `FakeStore` in `pipeline_integration_test.go` | The "integration" test asserts the value handed to `FakeStore.Save` (an intermediate hop) instead of reading it back from the real store sink; also `FakeStore.Save` ignores its `ctx` arg. `[GREEN-THEATER]` — assert the sink and stop ignoring load-bearing args. | [ ] |

---

## Table 1: Single Area Reviews (2 passes each)

| # | Focus Area | Pass 1 | Pass 2 | Status |
|---|---|---|---|---|
| 1 | Component A (`path/to/component_a.py` — initialization, validation, core logic) | [ ] | [ ] | |
| 2 | Component B (`path/to/component_b.go` — request handling, error paths, response formatting) | [ ] | [ ] | |
| 3 | Component C (`path/to/component_c.ts` — external API client, retry logic, response parsing) | [ ] | [ ] | |
| 4 | Data Layer (`path/to/db/` — schema, queries, migrations, connection handling) | [ ] | [ ] | |
| 5 | Configuration (`path/to/config/` — env var loading, defaults, validation) | [ ] | [ ] | |
| 6 | Test Coverage (`path/to/tests/` — test isolation, mocking strategy, missing edge cases) | [ ] | [ ] | |

---

## Table 2: Paired Area Reviews (1 pass each)

| # | Pair | Pass | Status |
|---|---|---|---|
| 7 | Component A + Component B (data flow across boundary, error propagation, contract alignment) | [ ] | |
| 8 | Component B + Component C (request/response contract match, header naming, error mapping) | [ ] | |
| 9 | Component A + Data Layer (query correctness, transaction boundaries, null handling) | [ ] | |
| 10 | Configuration + All Components (env vars actually used, defaults match behavior, missing config) | [ ] | |
| 11 | Test Coverage + Components (tests cover seams between pairs, not just unit-level happy paths) | [ ] | |

---

## Review Guidance

### Single Area Reviews — What To Check

**Pass 1 (Correctness & Coverage):**
- All public functions have test coverage (happy path, failure, error, edge cases)
- The code does what the design doc says it should
- Error handling is thorough — no silent failures
- Module boundaries are clean (no leaking types across boundaries)

**Pass 2 (Robustness & Extensibility):**
- Code aligns to the spirit of the design, not just surface conformance
- The component is ready to be extended by future work
- Test assertions test real risks, not just happy-path ceremony
- Edge cases are handled (empty inputs, concurrent access, boundary values)

### Paired Area Reviews — What To Check

Paired reviews verify that two components integrate correctly:
- Data flows cleanly across the boundary
- Error propagation works end-to-end
- No assumptions in one component that the other doesn't satisfy
- The integration tests cover the seams between the pair
