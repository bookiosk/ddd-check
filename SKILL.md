---
name: ddd-check
description: DDD compliance check for Java projects — validates layer isolation, base class usage, naming conventions, dependency direction, identity protection, and exception handling against domain-driven design rules. Supports two modes: incremental (git diff, fast pre-commit) and full (all files, architecture review). Use when user mentions "DDD check", "DDD audit", "check my changes", "architecture review", "pre-commit check", "DDD compliance", "review this code", or "check entire project".
license: Apache-2.0
metadata: {"version": "2.0", "skill-author": "bookiosk"}
model: claude-sonnet-4-20250514
allowed-tools: Read, Grep, Glob, Bash
user-invocable: true
---

# DDD Compliance Check

Two-mode DDD compliance validation for Java projects. Auto-detects mode from user's trigger phrase.

## Mode Detection

| Trigger Phrases | Mode | Scope |
|---|---|---|
| "check my changes", "pre-commit", "review this code", "validate my code" | **Incremental** | `git diff` only |
| "full audit", "architecture review", "DDD compliance report", "check entire project", "pre-release" | **Full** | All Java files |

## Incremental Mode

### Workflow

1. **Find changed files**: `git diff --cached --name-only --diff-filter=ACMR | grep '\.java$'` (also check unstaged and HEAD diff)
2. **Classify by layer**: Map package path to layer using `references/domain-layer.md` etc.
3. **Load relevant rules**: Only load rule files matching changed layers. Always load `references/anti-patterns.md`.
4. **Check each file**: Validate structural rules (base class, package, naming), dependency rules (no forbidden imports), anti-patterns.
5. **Report**: Per-file violations with severity and fix. Skip "looks good" for clean files.

If no Java files changed: "No Java files changed — nothing to check."

## Full Mode

### Workflow

1. **Discover**: `find . -name "*.java" -not -path "*/target/*" -not -path "*/test/*" | sort`
2. **Load all rules**: Read every `references/` rule file (domain-layer, application-layer, adaptor-layer, infrastructure-layer, client-layer, model-layer, anti-patterns). Load base-classes-reference and exception-handling as needed.
3. **Layer-by-layer audit**: Group by layer, check package, base class, naming, dependencies, structure, exception mode.
4. **Cross-layer**: Verify dependency direction (`grep -r "import org\.bookiosk\.ddd\."`), detect anemic models, check exception boundaries.
5. **Report**: Executive summary with compliance score, per-layer breakdown, anti-patterns section, dependency graph, ranked recommendations.

See `examples/violation-report.md` for sample output.

## Layer Checklists

See `references/shared-checks.md` for the complete severity table (CRITICAL/HIGH/MEDIUM/LOW) and per-layer checklists.

## Common Mistakes

- **Flagging intentional violations as CRITICAL** — verify with author before escalating
- **Skipping cross-layer dependency check** — domain→infrastructure is the #1 architecture decay vector
- **Ignoring exception mode mismatch** — 阻断型 layer throwing to 分支型 caller breaks error handling contract
- **Incremental mode checking unchanged files** — only touch `git diff` output
- **Reporting "looks good" for clean files** — only report problems found

## Quality Checklist

Before finalizing:
- [ ] Mode correctly detected from user's trigger phrase
- [ ] Only relevant rule files loaded (incremental) or all layers covered (full)
- [ ] Every violation references the specific rule section (e.g. "references/domain-layer.md Section A.3")
- [ ] Each violation includes the exact fix, not just the problem
- [ ] CRITICAL issues listed first, prioritized by blast radius
- [ ] Anti-patterns cross-checked against `references/anti-patterns.md`
