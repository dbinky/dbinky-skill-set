---
name: feature-pipeline
description: End-to-end feature automation — interactive brainstorm for spec and design, then automated alignment, planning, implementation, ralph refinement loop, and PR review
---

# Feature Pipeline

Orchestrate the complete feature development pipeline from brainstorming through PR review. Steps 1-2 are interactive (user present for Q&A), steps 3-9 are fully automated (user walks away).

## Usage

```
/feature-pipeline {description of the product feature}
/feature-pipeline {description} --slug user-auth
/feature-pipeline {description} --spec-only
/feature-pipeline {description} --max-iterations 300 --priority normal
```

## Process

Read and follow the orchestrator instructions exactly:

**Orchestrator file:** `agents/feature-pipeline-orchestrator.md` (relative to this plugin's root)

The orchestrator will:
1. Parse arguments and initialize paths
2. Guide interactive spec brainstorming (user present)
3. Guide interactive design brainstorming (user present)
4. Dispatch automated steps as isolated subagents — alignment, planning, second alignment, implementation, ralph planning, ralph submission
5. Notify Teams at key milestones
6. Report final status

**IMPORTANT:** Read the orchestrator file using the path `${CLAUDE_PLUGIN_ROOT}/agents/feature-pipeline-orchestrator.md` and follow it exactly.
