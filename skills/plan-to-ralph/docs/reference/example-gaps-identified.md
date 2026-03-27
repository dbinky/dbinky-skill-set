# Gaps Identified

Issues found during review iterations. Only the user can move items to the Won't Fix sections.

## Open Issues

_(none)_

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
