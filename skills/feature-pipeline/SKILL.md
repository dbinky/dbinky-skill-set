---
name: feature-pipeline
description: End-to-end feature automation — interactive brainstorm for spec and design, then automated alignment, planning, implementation, ralph refinement loop, and PR review
---

# Feature Pipeline

You are orchestrating the complete feature development pipeline. The user provides a feature description, you guide them through spec and design brainstorming (interactive Q&A), then run the entire remaining pipeline unattended — from alignment through ralph refinement to PR review.

**After the brainstorm sessions, the user walks away. Everything else is automated.**

## Arguments

Parse the following from the user's command:

- `DESCRIPTION` (required): Thorough description of the product feature
- `--slug SLUG` (optional): Override auto-derived slug (default: derived from description)
- `--max-iterations N` (optional): Override ralph iteration count (default: 200)
- `--priority LEVEL` (optional): Override ralph priority (default: high)
- `--spec-only` (optional): Stop after spec + design brainstorming (skip automated pipeline)

## Step 1: Derive Slug and Paths

Generate a URL-friendly slug from the feature description:
- "user authentication system" → `user-auth`
- "payment processing with Stripe" → `payment-stripe`
- Take the first 2-3 meaningful words, lowercase, hyphenated

Set all downstream paths:
- `SPEC_PATH` = `docs/specs/{SLUG}-spec.md`
- `DESIGN_GLOB` = `docs/superpowers/specs/{SLUG}-design-phase-*.md`
- `PLAN_GLOB` = `docs/superpowers/plans/{SLUG}-implementation-phase-*-task-*.md`

Create directories if they don't exist:
```bash
mkdir -p docs/specs docs/superpowers/specs docs/superpowers/plans
```

Announce to the user:
```
Feature pipeline initialized:
  Slug:    {SLUG}
  Spec:    {SPEC_PATH}
  Designs: {DESIGN_GLOB}
  Plans:   {PLAN_GLOB}
```

## Step 2: Brainstorm Feature Spec (INTERACTIVE)

Invoke the `superpowers:brainstorming` skill with this prompt:

> Use the superpowers:brainstorm skill to produce a product specification document at `{SPEC_PATH}`. We're going to concentrate on the product features and outcomes and will not bring any implementation details into the document. Here's what I'd like to discuss with you: {DESCRIPTION}

This enters interactive Q&A with the user. Wait for the brainstorming skill to complete — it will write the spec, get user approval, and commit it.

**After brainstorming completes,** verify the spec file exists:
```bash
test -f {SPEC_PATH} || echo "ERROR: Spec not written"
```

## Step 3: Brainstorm Implementation Design (INTERACTIVE)

Invoke the `superpowers:brainstorming` skill again with this prompt:

> Use the superpowers:brainstorm skill to produce an implementation design for the spec we just wrote at `{SPEC_PATH}`. The implementation design may be multiple phases of work and we want to create a separate design document for each logical phase. Those documents will be stored as `{DESIGN_GLOB}`. Read through the spec thoroughly for context and then let's figure out the high level implementation details for these design docs.

This enters interactive Q&A with the user. Wait for the brainstorming skill to complete.

**After brainstorming completes,** verify at least one design doc exists.

---

## ── USER INTERACTION ENDS HERE ──

**If `--spec-only` was passed:** Stop here. Output:

```
Spec and designs complete. Pipeline paused (--spec-only).

To resume the automated pipeline later:
  /spec-to-design --spec {SPEC_PATH}
```

**Otherwise:** Continue to the automated phases below.

---

## Step 4: Notify Teams — Automation Starting

```bash
ralph-o-matic notify --message "Feature pipeline running unattended for {SLUG} on branch $(git branch --show-current). Spec: {SPEC_PATH}"
```

## Step 5: Run spec-to-design (AUTOMATED)

Invoke the `spec-to-design` skill:

> Run /spec-to-design with SPEC_PATH={SPEC_PATH} and DESIGN_GLOB={DESIGN_GLOB} and PLAN_GLOB={PLAN_GLOB} and --slug {SLUG}. The user is unavailable for input.

This runs:
1. Design alignment
2. Implementation plan production
3. Plan alignment
4. Draft implementation via parallel subagents

**On failure:** The spec-to-design skill sends its own Teams notification with resume instructions. Stop the pipeline.

## Step 6: Run auto-ralph-prep (AUTOMATED)

Invoke the `auto-ralph-prep` skill:

> Run /auto-ralph-prep with SPEC_PATH={SPEC_PATH} and DESIGN_GLOB={DESIGN_GLOB} and PLAN_GLOB={PLAN_GLOB} and --slug {SLUG}. The user is unavailable for input.

This generates RALPH.md, focus-areas.md, and gaps-identified.md.

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed generating ralph loop files for {SLUG}. Error: {ERROR}. Resume: /auto-ralph-prep --spec {SPEC_PATH}"
```

## Step 7: Run auto-ralph-submit (AUTOMATED)

Invoke the `auto-ralph-submit` skill:

> Run /auto-ralph-submit with --priority {PRIORITY} --max-iterations {MAX_ITERATIONS} --slug {SLUG}. The user is unavailable for input.

This submits the job to ralph-o-matic.

**On failure:** Send Teams notification:
```bash
ralph-o-matic notify --message "Pipeline failed submitting to ralph for {SLUG}. Error: {ERROR}. Resume: /auto-ralph-submit"
```

## Step 8: Notify Teams — Handed Off to Ralph

```bash
ralph-o-matic notify --message "Pipeline handed off to ralph for {SLUG}. Job #{JOB_ID} with {MAX_ITERATIONS} max iterations. The post-completion hook will trigger PR review automatically."
```

## Step 9: Session Complete

Output:

```
Feature pipeline complete for {SLUG}:

  ✓ Spec:           {SPEC_PATH}
  ✓ Designs:        {N} phase docs
  ✓ Plans:          {M} task docs
  ✓ Implementation: committed and tested
  ✓ Ralph loop:     Job #{JOB_ID} submitted

What happens next (automated):
  1. Ralph runs the refinement loop ({MAX_ITERATIONS} max iterations)
  2. Post-completion hook triggers PR review automatically
  3. PR review applies all fixes except "Defer" ranked
  4. You'll get a Teams notification when the PR is ready

Nothing more to do — go to bed!
```

The Claude Code session can now end cleanly. The ralph server and post-completion hook handle everything from here.
