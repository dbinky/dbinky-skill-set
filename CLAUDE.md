# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PR Review Skill is a Claude Code plugin that orchestrates multi-persona PR reviews. It uses sequential AI agent personas (Architect, 10x Engineer, Security Expert, Engineering Manager, Senior Engineer) to debate and review pull requests, then lets users choose which fixes to implement.

**This is a pure Markdown project — no build, test, or lint commands.** All logic lives in Markdown instruction files that Claude interprets at runtime.

## Architecture

**Plugin entry**: `.claude-plugin/plugin.json` declares the plugin; `skills/pr-review/SKILL.md` is the user-facing skill invoked via `/pr-review`.

**Orchestrator** (`agents/review-orchestrator.md`): The core engine. Handles GitHub CLI interactions, auto-detects languages/frameworks from PR diffs, loads applicable rule files, and dispatches agents sequentially through 7 phases:

1. Pre-flight (gh CLI, auth, git remote)
2. PR resolution (explicit number or auto-detect from branch)
3. Language/framework detection → rule file selection
4. Pipeline execution: Architect R1 → 10x R1 → Architect R2 → 10x R2 → Security → Manager
5. Interactive fix selection (Must Fix / Should Fix / Consider / Defer tiers)
6. Senior Engineer implementation of selected fixes
7. Completion summary

**Sequential execution is critical** — each reviewer reads prior comments, enabling the architect-vs-pragmatist debate. This cannot be parallelized.

**Agent personas** (`agents/*.md`): Each file defines a reviewer's identity, comment format, tags, scope, and decision rules. Personas are instruction sets, not code.

**Rule files** (`rules/languages/*.md`, `rules/frameworks/*.md`): Technology-specific best practices split into general and security variants. Reviewers load only relevant rules based on detected tech. Languages: Go, Python, TypeScript, JavaScript, C#, Rust, Java, Dart. Frameworks: React, Angular, Flutter, Spring Boot, Orleans.

## Key Conventions

- Agent comment prefixes: `architect:`, `10x:`, `security:`, `manager:` (Senior Engineer writes code, not comments)
- Manager categorization determines implementation scope: Must Fix, Should Fix, Consider, Defer
- Security rules are separate files (`*-security.md`) loaded only by the Security Expert
- The orchestrator handles all `gh` CLI calls; personas focus on review logic

## Requirements

- GitHub CLI (`gh`) installed and authenticated
- Git repository with a configured remote
- Claude Code with plugin support
