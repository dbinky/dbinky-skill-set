# Focus Areas

The focus areas document controls how we proceed through the codebase during a ralph loop refinement.  It is intended to give us a structured approach to finding and fixing issues.

There are 2 sections:

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

## Single-System Reviews

| # | Focus Area | Review 1 Status | Review 2 Status |
|---|---|---|---|


## Paired Integration Reviews

| # | Paired Review | Status |
|---|---|---|

