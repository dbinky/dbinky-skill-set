---
name: spec-to-design
description: Automated pipeline from spec+designs through alignment, planning, plan alignment, dependency gate, test baseline, and draft implementation — the feature pipeline's unattended middle, with zero interaction
---

# Spec to Design

You are running the automated implementation pipeline. Given a feature spec and its
design docs, you will:

1. Align design docs to the spec
2. Produce detailed implementation plans
3. Align implementation plans to each other and the spec
4. Gate on unresolved dependencies
5. Capture the branch-point test baseline
6. Execute the implementation via parallel subagents

**This skill requires zero user interaction.** The user is unavailable. Do all work
autonomously, committing and pushing at each phase.

## One source of truth

These steps are the feature pipeline's Phases 5–8. Do not re-derive them: read
`${CLAUDE_PLUGIN_ROOT}/agents/feature-pipeline-orchestrator.md` and execute its
**Phase 5, Phase 6, Phase 7, Phase 7.2, Phase 7.5 and Phase 8** exactly as written there
— the same subagent dispatches, prompts, verifications, checkpoint pushes and failure
handling — with the arguments below substituted for the orchestrator's variables. Skip
everything else in that file (initialization, brainstorms, ralph prep and submit).

## Arguments

- `SPEC_PATH` (required): Path to the feature spec (e.g., `docs/specs/user-auth-spec.md`)
- `DESIGN_GLOB` (optional): Glob for design phase docs (default: derived from spec path — replace `-spec.md` with `-design-phase-*.md` in `docs/superpowers/specs/`)
- `PLAN_GLOB` (optional): Glob for plan docs (default: derived from spec path — replace `-spec.md` with `-implementation-phase-*-task-*.md` in `docs/superpowers/plans/`)
- `--slug SLUG` (optional): Feature slug for commit messages and file naming (default: derived from spec filename)

## Prerequisites

1. Verify `SPEC_PATH` exists. If not, notify Teams and stop.
2. Verify at least one file matches `DESIGN_GLOB`. If not, notify Teams and stop.
3. Read all input files to confirm they're valid markdown.

## The one difference: the dependency gate cannot ask

In the orchestrator, Phase 7.2.3 asks the human. Here nobody is present, so when the gate
counts `NEEDS > 0`:

```bash
git push --quiet -u origin HEAD
ralph-o-matic notify --message "spec-to-design for '{SLUG}' stopped at the dependency gate: {NEEDS} item(s) need a human decision before implementation. Items: {one-line list}. Resolve with: /feature-pipeline --resume --slug {SLUG}"
```

Report the BLOCKING and DECISION lines and stop. Do not implement against plans that
wait on a decision — `/feature-pipeline --resume` re-enters at plan alignment and runs the
gate interactively.

## Completion

```
spec-to-design complete for {SLUG}:
  Design alignment     ✓
  Plan production      ✓ ({N} plans across {M} phases)
  Plan alignment       ✓
  Dependency gate      ✓ ({K} non-blocking items noted)
  Test baseline        ✓ docs/test-baseline.json {(fallback: empty)}
  Implementation       STATUS: {IMPL_STATUS} — {NEW failures vs baseline, or "none"}

All work committed and pushed. Ready for /auto-ralph-prep.
```

If `IMPL_STATUS` is `DONE_WITH_CONCERNS` or `BLOCKED`, list the named blockers — and pass
the STATUS block to `/auto-ralph-prep` as `IMPL_STATUS` so they are seeded as gaps.
