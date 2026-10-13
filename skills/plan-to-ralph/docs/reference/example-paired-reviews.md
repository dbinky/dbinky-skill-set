# Paired Reviews — Integration Seams

A row exists only for a real **seam**: the two behaviours share persisted state, a
wire contract, a queue, a flag or an ordering constraint; they were built by
different tasks, phases or commits; and a break would be silent (each side green in
isolation).

Each seam is **one row**, named `A ↔ B`, and its single approval covers **both
directions**. There are no reverse rows. Paired reviews start only once every
single-system box in `focus-areas.md` is ticked (or its area escalated). Refiners never
edit the table or `## No seam`; demotions and promotions are proposed under
`## Pending approvals` and applied by the verifier.

Rows are ordered **highest risk first**. Keep the five columns exactly; no `|`
inside a cell.

## Paired Integration Reviews

| # | Paired Review | Seam | Risk | Status |
|---|---|---|---|---|
| 1 | saved search is stored ↔ nightly job emails matches | SHARED: the `saved_searches.filters` JSON shape, written by Area 1's handler and decoded by Area 2's matcher (built in plan tasks 1.2 and 2.3) · TRACE: create a search with a price range via the API, run `RunNightly` against one matching listing, observe the email · BREAK: the matcher silently ignores a filter key the handler writes (e.g. `priceMax` vs `max_price`), so alerts over-match | high | [ ] |
| 2 | unsubscribe turns alerts off ↔ nightly job emails matches | SHARED: the `enabled` flag, and the ordering that a disable before the run wins · TRACE: unsubscribe one of two searches, run `RunNightly`, observe exactly one email · BREAK: the job reads a cached search list loaded before the flag flipped and emails an unsubscribed owner | med | [ ] |

## No seam

### Inside one behaviour

- saved search is stored / the `0042` migration — the column constraints are check 3 of Area 1's failure lens.

### Genuinely unrelated

- saved search is stored / unsubscribe — share the table, but Area 1 never reads `enabled` and Area 3 never writes filters; no shared field or ordering.

### Deferred (budget)

_(none)_

## Pending approvals

_(none)_
