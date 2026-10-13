# Gaps Identified

**Keep this file readable — every pass reads its open list.** Records are at most
ten lines with `file:symbol`. A closed item moves under `## Closed` with a one-line
note; its history does not stay in the open list. (One ledger grew to 1.4 MB and
18,000 lines because every pass appended a multi-page record, and no reviewer could
fit it in context.)

Format for an open item:

- [ ] **ISSUE TITLE** — what is wrong, where (`file:symbol`), and enough context for the next pass to act without rediscovering it.

Tags:
- `[NOT-WIRED]` — a production path that is present but unfed: constructed only in tests, or fed `nil`/empty/hardcoded. An inert path is an open gap, never "acceptable".
- `[GREEN-THEATER]` — a test that passes only because a fake ignores a load-bearing argument, or an "integration" test that asserts an intermediate hop instead of the sink.

## Open

- [ ] **Owner taken from the request body** — `api/handlers/saved_search.go:Create` reads `owner_id` from the JSON body (line 41), so any caller can store a search under another account. Read it from `auth.FromContext`. Area 1, Review 2.
- [ ] **[NOT-WIRED] Production mail sender is the no-op** — `config/defaults.yaml` sets `mail.sender: noop` and `cmd/worker/main.go` never overrides it, so the nightly job "sends" nothing in production. `grep -rn NewSMTPSender` finds only `mail/sender_test.go`. Area 2, Review 1.
- [ ] **[GREEN-THEATER] alert test asserts the matcher, not the email** — `jobs/alerts_test.go:TestRunNightly` checks the list returned by `match.Filter` (an intermediate hop); assert on the recorded message at `mail.Sender`. Area 2.

## Escalations

- Should an owner with 200 new matches get one digest or a capped list? The spec is silent; the code sends all 200. Needs a product decision.

## Closed

- **Duplicate saved searches on double submit** — unique index added in `migrations/0043`; `TestCreateSavedSearch_Idempotent`. (A1/R2, `3f2c1ab`)

## Won't Fix

- **Pre-existing:** `TestListingImport_Timezone` fails at the branch-point baseline; not this work.
