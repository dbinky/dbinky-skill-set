# PR Review Skill

A Claude Code plugin that orchestrates multi-persona PR reviews with language and framework-specific rules.

## Pipeline

```
Architect (R1) → 10x Engineer (R1) → Architect (R2) → 10x Engineer (R2) → Security → Manager → [You Choose Fixes] → Senior Engineer
```

Six specialized reviewers examine your PR sequentially, each building on previous feedback. The Architect and 10x Engineer debate twice before Security and Manager weigh in. You then choose which fix categories to apply, and the Senior Engineer implements them.

## Install

```bash
# Add the marketplace
/plugin marketplace add dbinky/dbinky-skill-set

# Install the plugin
/plugin install pr-review@dbinky-skill-set
```

## Usage

```
/pr-review              # Auto-detect PR from current branch
/pr-review 87           # Review specific PR number
```

## Reviewers

| Persona | Focus | Rounds |
|---------|-------|--------|
| **Ivory Tower Architect** | SOLID, DRY, naming, patterns, testability | 2 |
| **10x GSD Engineer** | Pragmatism, shipping, over-engineering pushback | 2 |
| **Security Expert** | OWASP, injection, auth, secrets, crypto | 1 |
| **Engineering Manager** | Final call — Must Fix / Should Fix / Consider / Defer | 1 |
| **Senior Engineer** | Implements the fixes you select | 1 |

## Comment Scaling

Comment limits scale with PR size so small PRs aren't over-reviewed and large PRs get thorough coverage. Limits apply to **new findings only** — responses between Architect and 10x during the debate are always unlimited.

| PR Size | Files | Architect | 10x | Security |
|---------|-------|-----------|-----|----------|
| Small   | 1–3   | 3–5       | 2–3 | Unlimited |
| Medium  | 4–10  | 5–10      | 3–6 | Unlimited |
| Large   | 11–25 | 8–16      | 5–10 | Unlimited |
| XL      | 26+   | 12–20     | 8–14 | Unlimited |

## Interactive Fix Selection

After the Manager categorizes all findings, you choose a tier:

1. **Must Fix only** — Critical issues only
2. **Must Fix + Should Fix** — Important improvements included
3. **All but Defer** — Everything except deferred items
4. **None** — Skip automated fixes

## Language & Framework Rules

Rules are auto-detected from the PR diff. Each language/framework has two rule files: general patterns and security-specific patterns.

**Languages:** Go, Python, TypeScript, JavaScript, C#, Rust, Java, Dart

**Frameworks:** Orleans, Spring Boot, React, Angular, Flutter

Multi-language PRs load all applicable rule sets.

## Requirements

- [GitHub CLI (`gh`)](https://cli.github.com/) — the plugin will check for it and guide installation if missing
- Claude Code with plugin support

## License

MIT
