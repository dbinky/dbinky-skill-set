# Gaps Identified

Issues found during review.

## Issue Pattern

When you add issues to this list, use the format:

- [ ] **ISSUE TITLE** - Issue Description that contains enough context for the reviewer to pick it up, get oriented in the code and then address the issue.

New issues are added to the `Open Issues` section. Resolved issues (where the [x] box is checked) are moved to the `Resolved Issues` section.

**Inert/no-op paths are OPEN gaps — never silently "acceptable".** Any production path that is present but unfed — fed `nil`/empty/hardcoded, or a degenerate case rationalized away (e.g. "empty set treated as maximally varied — acceptable") — MUST be logged here as an open `[NOT-WIRED]` issue. An unfed production path is a gap to fix or an explicit escalation; it is never a tolerated no-op and never a reason to call the work done. Likewise tag any test that only passes because a fake ignores a load-bearing argument or asserts an intermediate hop instead of the sink as `[GREEN-THEATER]`.

## Open Issues

_(none)_

## Resolved Issues

_(none yet)_

## Won't Fix (Beyond Current Scope)

_(none)_
