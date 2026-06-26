# Focus Areas

The focus areas document controls how we proceed through the codebase during a ralph refinement run.  It is intended to give us a structured approach to finding and fixing issues.

**Ordering.** Prioritize the list in this order: **newly-introduced test failures first** (regressions vs the recorded branch-point baseline), **then whole-pipeline wiring/provenance gaps** (below), **then gaps vs. the plans/designs**. Pre-existing baseline failures are NOT this work's job — do not let them displace the above.

There are 3 sections:

**Whole-Pipeline Wiring & Provenance**

The recurring failure mode is code that is present and green in isolation but **never wired-and-fed** in production. Seed concrete, named focus items here (do the greps and cite real findings — not generic reminders). For the feature under review, add focus items covering:

- For **every** new exported field / collaborator / config: `grep -rn <Symbol>` the whole repo — is it constructed/populated **outside `*_test.go`**? If it is only ever set in tests, that is a `[NOT-WIRED]` gap and is top priority.
- For **every** new component: is it assembled into the real composition root (server/DI wiring) so the live path reaches it? Name the wiring site, or flag it as `[NOT-WIRED]` if missing.
- Does any production call site pass `nil`/empty/hardcoded where a real producer should feed it (e.g. a scorer injected `nil`, an embedding set left empty, a bucket hardcoded)? List each.
- Do any tests pass only because a **fake ignores a load-bearing argument**, or because an "integration" test asserts an intermediate hop instead of the sink? Tag these `[GREEN-THEATER]`.

**Single-System Review**

When defining single systems to include on the list, you should review the commits and updates in the current branch (or defer to the user's input on scope).  Based on your review, you should include a seingle review system for each of the following:

- any time of domain concepts touched
- any system touched
- any app touched
- any API touched
- any contract touched

Each single review area will need 2 independent reviews approving completion before that single review focus area can be considered "done".

**Paired Reviews**

When defining paired reviews, pair each single review area with each other review area (except itself).  This will naturally create "reverse pairs".  During any given paired review, you should approach the review from the following mindset:

__I am reviewing the completeness, correctness, and spec alignment of {paired area 1} as it is impacted by updates in {paired area 2}.  Both should meet my stringent bar for "done" and work together correctly.__

Because of the existence of reverse pairs, each paired review onlyi needs 1 approval to be considered "done".

## Whole-Pipeline Wiring & Provenance (highest priority)

Each item is "done" only when the symbol/component is **wired-and-fed**: constructed at the real composition root AND fed real data from an existing producer (verified by grep, not by a green isolated test).

| # | Symbol / Component | Provenance check (grep outside `*_test.go` · composition-root site · call-site data source) | Status |
|---|---|---|---|


## Single-System Reviews

| # | Focus Area | Review 1 Status | Review 2 Status |
|---|---|---|---|


## Paired Integration Reviews

| # | Paired Review | Status |
|---|---|---|

