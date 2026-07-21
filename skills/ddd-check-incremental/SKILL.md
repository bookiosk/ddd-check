---
name: ddd-check-incremental
description: Incremental DDD compliance check scanning only changed Java files via git diff. Use when user mentions "check my changes", "DDD check", "review this code", "pre-commit check", or "validate my code". For full audits, see ddd-check-full.
license: Apache-2.0
metadata: {"version": "1.0", "skill-author": "bookiosk"}
model: claude-sonnet-4-20250514
allowed-tools: Read, Grep, Glob, Bash
user-invocable: true
---

# DDD Incremental Compliance Check

Check only **new or modified** Java files against DDD layer rules. Fast, pre-commit focused.

## Workflow

### Step 1: Find Changed Files

```bash
git diff --cached --name-only --diff-filter=ACMR | grep '\.java$'
git diff --name-only --diff-filter=ACMR | grep '\.java$'
git diff HEAD --name-only --diff-filter=ACMR | grep '\.java$'
```

If no Java files changed: "No Java files changed — nothing to check."

### Step 2: Classify by Layer

| Package Pattern | Layer | Rule File |
|---|---|---|
| `**/ddd/domain/**` | Domain | `rules/domain-layer.md` |
| `**/ddd/application/**` | Application | `rules/application-layer.md` |
| `**/ddd/adaptor/**` | Adaptor | `rules/adaptor-layer.md` |
| `**/ddd/infrastructure/**` | Infrastructure | `rules/infrastructure-layer.md` |
| `**/ddd/client/**` | Client | `rules/client-layer.md` |
| `**/ddd/model/**` | Model | `rules/model-layer.md` |
| `**/ddd/common/**` | Common | (structural only) |

### Step 3: Load Relevant Rules

Load only rule files matching changed layers. Always load `rules/anti-patterns.md`.

### Step 4: Check Each File

Validate each changed file against: structural rules (base class, package, naming), dependency rules (no forbidden imports), anti-patterns (`rules/anti-patterns.md`).

### Step 5: Report

```
## DDD Check: {branch} → {n} files changed

### {FilePath}.java — {Layer}
CRITICAL: {problem}. {fix}.
HIGH: {problem}. {fix}.

### Summary
- {n} files checked, {x} violations (C: {a}, H: {b}, M: {c}, L: {d})
```

See `examples/violation-report.md` for detailed sample output.

## Layer Checklists

See `references/shared-checks.md` for the complete severity table and per-layer checklists.

## Common Mistakes

- **Checking unchanged files** — incremental mode only touches `git diff` output
- **Reporting "looks good" for clean files** — only report problems found
- **Flagging intentional violations** — verify with author before marking CRITICAL

## Quality Checklist

Before reporting:
- [ ] Only changed files checked (verify via `git diff` output)
- [ ] Every violation references the specific rule section (e.g. "rules/domain-layer.md Section A.3")
- [ ] Each violation includes the exact fix, not just the problem
- [ ] No false positives from intentional design decisions
- [ ] Anti-patterns cross-checked against `rules/anti-patterns.md`
