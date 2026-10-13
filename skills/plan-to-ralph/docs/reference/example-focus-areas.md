# Focus Areas — Single-System Reviews

Each area is a **behaviour**: one thing this work promises, traced from where it
starts (Entry) to where its effect lands (Sink), across every layer in between —
not a file, a class, a layer or a package.

Each area needs **two approvals**: Review 1 (the **wiring** lens — does the path
exist and is it fed?) and Review 2 (the **failure** lens — how does it fail?). Only
a fresh verifier subagent ticks a box; see `RALPH.md`. Rows and blocks are the approval
criteria — refiners never edit them; they add lines under `## Pending approvals`.

Rows are ordered **highest risk first**. Keep the table's five columns exactly. The
detail blocks use numbered lists, never checkbox syntax — `[ ]`/`[x]` outside the
table corrupts the approval count.

## Single-System Reviews

| # | Focus Area | Risk | Review 1 Status | Review 2 Status |
|---|---|---|---|---|
| 1 | a saved search is stored for its owner and refused for anyone else | high | [ ] | [ ] |
| 2 | the nightly job emails each owner the new listings matching their saved searches | high | [ ] | [ ] |
| 3 | the unsubscribe link turns alerts off for exactly one saved search | med | [ ] | [ ] |

## Area details

### Area 1 — a saved search is stored for its owner and refused for anyone else

- **Promise:** `POST /saved-searches` stores the caller's filters under the caller's account; a request naming another account is refused.
- **Entry:** `api/handlers/saved_search.go:Create`
- **Sink:** `store/saved_search_repo.go:Insert` (row in `saved_searches`)
- **Files:** `api/handlers/saved_search.go`, `domain/search/filters.go`, `store/saved_search_repo.go`, `migrations/0042_saved_searches.sql`, `cmd/server/main.go` (wiring), `api/routes.go`
- **Folded in:** `api/routes.go` — the new route is registered behind the auth middleware; `migrations/0042` — `owner_id` is `NOT NULL` with a foreign key.
- **Review 1 — wiring lens:**
  1. `NewSavedSearchRepo` is constructed in `cmd/server/main.go:buildServer` and injected into the handler — not only in tests.
  2. `Create` reads the owner from `auth.FromContext`, not from the request body.
  3. The route reaches `Insert` on the live router (`api/routes.go`), and `TestCreateSavedSearch_PersistsForOwner` reads the row back from the store.
- **Review 2 — failure lens:**
  1. Body naming a different `owner_id` → 403 and no row; `TestCreateSavedSearch_RejectsForeignOwner`.
  2. Empty filter set → 400, not an always-matching search; the test goes red if the validation is removed.
  3. Duplicate submit → one row (unique index on owner + filter hash); concurrent test.
- **Seed evidence:** `grep -rn NewSavedSearchRepo` → `cmd/server/main.go:88`, `store/saved_search_repo_test.go:14`; `grep -rn owner_id api/` → read from body at `saved_search.go:41` — logged as a gap.

### Area 2 — the nightly job emails each owner the new listings matching their saved searches

- **Promise:** once a night, every enabled saved search with new matching listings produces one email to its owner.
- **Entry:** `jobs/alerts.go:RunNightly` (scheduled in `cmd/worker/main.go`)
- **Sink:** `mail/sender.go:Send`
- **Files:** `jobs/alerts.go`, `domain/search/match.go`, `store/saved_search_repo.go:ListEnabled`, `store/listing_repo.go:SinceCursor`, `mail/sender.go`, `cmd/worker/main.go`
- **Folded in:** `config/defaults.yaml` — `alerts.cron` default; check it is read, not hardcoded.
- **Review 1 — wiring lens:**
  1. `RunNightly` is registered with the scheduler in `cmd/worker/main.go:registerJobs`.
  2. The real `mail.Sender` (not the no-op) is injected in production config.
  3. `TestRunNightly_EmailsOwner` asserts on the sender's recorded message, the sink.
- **Review 2 — failure lens:**
  1. The mail provider errors for one owner → the others still get their email and the cursor does not advance past the failed one.
  2. A crash mid-run → re-running does not double-send (cursor committed per owner).
  3. A saved search with zero matches sends nothing.
- **Seed evidence:** `grep -rn RunNightly` → `cmd/worker/main.go:52`; `mail.NewNoopSender` is the default in `config/defaults.yaml` — gap.

### Area 3 — the unsubscribe link turns alerts off for exactly one saved search

- **Promise:** the link in an alert email disables alerts for that saved search only.
- **Entry:** `api/handlers/unsubscribe.go:Handle`
- **Sink:** `store/saved_search_repo.go:SetEnabled`
- **Files:** `api/handlers/unsubscribe.go`, `domain/search/token.go`, `store/saved_search_repo.go`, `mail/templates/alert.html`
- **Folded in:** `mail/templates/alert.html` — the link carries the signed token, not the raw id.
- **Review 1 — wiring lens:**
  1. The handler is routed without auth (the link is opened from email) but verifies the signed token.
  2. `SetEnabled` is called with the id decoded from the token.
  3. `TestUnsubscribe_DisablesOneSearch` reads the flag back from the store.
- **Review 2 — failure lens:**
  1. A tampered or expired token → 400 and no change.
  2. Unsubscribing twice is idempotent.
  3. An owner's other saved searches stay enabled.
- **Seed evidence:** `grep -rn SetEnabled` → `unsubscribe.go:30`, repo tests only.

## Pending approvals

_(none)_
