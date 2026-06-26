# Gaps Identified

Issues found during review iterations. Only the user can move items to the Won't Fix sections.

**Inert/no-op paths are OPEN gaps — never silently "acceptable".** A production path that is present but unfed (`nil`/empty/hardcoded, or a degenerate case rationalized away) is logged here as an open `[NOT-WIRED]` issue to fix or escalate — never recorded as tolerated. Tests that pass only because a fake ignores a load-bearing argument, or that assert an intermediate hop instead of the sink, are logged as `[GREEN-THEATER]`.

## Open Issues

- [ ] **[NOT-WIRED] EmbeddingSet never fed in production** — `RankRequest.EmbeddingSet` is only ever assigned in `rank_test.go`; the live `Handler.Rank` builds `RankRequest{}` and leaves it empty, so ranking always runs on an empty embedding set. `grep -rn EmbeddingSet` confirms no non-test producer. Wire the real embedder into the handler and feed it.
- [ ] **[NOT-WIRED] Empty embedding set treated as maximally varied** — when the embedding set is empty the scorer returns a max-diversity score and the path silently no-ops instead of erroring. This is an inert production path, NOT acceptable: either feed a real producer (see above) or escalate. Do not close as "acceptable".
- [ ] **[GREEN-THEATER] pipeline integration test asserts the wrong sink** — `pipeline_integration_test.go` asserts the value passed to `FakeStore.Save` (an intermediate hop) and `FakeStore.Save` ignores its `ctx` argument, so the test stays green even if the real store never persists. Make the test read back from the real store sink and stop ignoring load-bearing args.

## Fixed Previously

_(none yet — as issues are fixed during iterations, move them here with a summary of what was wrong and how it was fixed)_

<!--
Example of a fixed issue entry:

### COMP-A-001: Input validation missing for empty strings
**Area**: Component A (`path/to/component_a.py`)
**Fixed**: Iteration 3
**Detail**: The `process()` function accepted empty strings without validation, which caused a downstream `IndexError` in the parser. The code handled `None` but not `""`. Added an early return with a structured error response. Added 2 tests: empty string input returns error, whitespace-only input returns error.
-->

## Won't Fix (Beyond Current Scope)

_(items logged here are new functionality or enhancements that are outside the scope of the current review — only the user can move items to this section)_

<!--
Example of a won't-fix entry:

### COMP-C-W01: No retry logic for transient network failures
**Area**: Component C (`path/to/component_c.ts`)
**Detail**: The external API client has no retry/backoff for 5xx responses. Adding retry logic is a robustness improvement beyond the current feature scope. Low risk since failures are surfaced to the caller.
-->
