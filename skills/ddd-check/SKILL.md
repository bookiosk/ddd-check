---
name: ddd-check
description: DDD compliance check for Java projects with auto mode detection. Use when user mentions "DDD check", "DDD audit", "check my changes", "architecture review", "pre-commit check", "DDD compliance", "review this code", or "check entire project".
license: Apache-2.0
metadata: {"version": "2.0", "skill-author": "bookiosk"}
model: claude-sonnet-4-20250514
allowed-tools: Read, Grep, Glob, Bash
user-invocable: true
---

# DDD Compliance Check

Two-mode DDD compliance validation for Java projects. Auto-detects mode from user's trigger phrase.

## Prerequisites

This skill validates projects built on **ddd-framework** only.

| Item | Value |
|---|---|
| Framework | ddd-framework |
| Maven | `io.github.bookiosk:ddd-framework` |
| Package | `org.bookiosk.ddd` |
| Repository | https://github.com/bookiosk/bookiosk-ddd |

If the project does not use ddd-framework, the check results are meaningless — stop and ask the user before proceeding.

## Framework Detection (Run First)

Before either mode, verify the project actually uses ddd-framework — check both the declared dependency and the imported classes:

```bash
# 1) Dependency declared in build files
grep -rn "io\.github\.bookiosk" pom.xml build.gradle build.gradle.kts settings.gradle settings.gradle.kts 2>/dev/null

# 2) Framework classes actually imported in Java source (the real "uses the jar" signal)
grep -rln "import org\.bookiosk\.ddd\." --include="*.java" . 2>/dev/null | head -5
```

- **Source imports found** → the project genuinely uses the framework; continue with the detected mode.
- **Only build file declares it, no source imports** → the dependency is declared but unused; warn the user the check may be meaningless and ask before continuing.
- **Neither found** → stop and report: "This project does not use ddd-framework (`io.github.bookiosk:ddd-framework`). ddd-check validates ddd-framework conventions only — running it here would produce meaningless violations. Install the framework first, or skip this check."

## Mode Detection

| Trigger Phrases | Mode | Scope |
|---|---|---|
| "check my changes", "pre-commit", "review this code", "validate my code" | **Incremental** | `git diff` only |
| "full audit", "architecture review", "DDD compliance report", "check entire project", "pre-release" | **Full** | All Java files |

## Incremental Mode

### Workflow

0. **Verify framework dependency** — run the Framework Detection step above; stop if not found.
1. **Find changed files**: `git diff --cached --name-only --diff-filter=ACMR | grep '\.java$'` (also check unstaged and HEAD diff)
2. **Classify by layer**: Map package path to layer using layer rule files in `references/`.
3. **Load relevant rules**: Only load rule files matching changed layers. Always load `references/anti-patterns.md`.
4. **Check each file**: Validate structural rules (base class, package, naming), dependency rules (no forbidden imports), anti-patterns.
5. **Report**: Per-file violations with severity and fix. Skip "looks good" for clean files.

If no Java files changed: "No Java files changed — nothing to check."

## Full Mode

### Workflow

0. **Verify framework dependency** — run the Framework Detection step above; stop if not found.
1. **Discover**: `find . -name "*.java" -not -path "*/target/*" -not -path "*/test/*" | sort`
2. **Load all rules**: Read every layer rule file in `references/`. Load `references/base-classes-reference.md` and `references/exception-handling.md` as needed.
3. **Layer-by-layer audit**: Group by layer, check package, base class, naming, dependencies, structure, exception mode.
4. **Cross-layer**: Verify dependency direction (`grep -r "import org\.bookiosk\.ddd\."`), detect anemic models, check exception boundaries.
5. **Report**: Executive summary with compliance score, per-layer breakdown, anti-patterns section, dependency graph, ranked recommendations.

See `examples/violation-report.md` for sample output.

## Layer Checklists

See `references/shared-checks.md` for the complete severity table (CRITICAL/HIGH/MEDIUM/LOW) and per-layer checklists.

## Red Flags

These thoughts mean STOP — you're about to make a mistake:

| Thought | Reality |
|---|---|
| "This import looks wrong but the code compiles" | Compilation ≠ architecture compliance. Check the layer dependency rules. |
| "It's just one setter, what's the harm?" | Public setId on aggregates breaks identity protection. Every violation matters. |
| "I'll skip the cross-layer check, the file looks fine" | Domain→infrastructure imports are invisible without explicit grep. |
| "This exception pattern is different but it works" | Wrong exception mode breaks the entire error handling contract. |
| "The design pattern makes the code cleaner" | Design patterns in domain layer are FORBIDDEN — no exceptions. |
| "I'll report 'looks good' for files with no issues" | Only report problems. Silence means clean. |
| "A perfect score isn't possible, so I'll be lenient" | Architectural decay accelerates. Flag every real violation. |

## Common Mistakes

- **Flagging intentional violations as CRITICAL** — verify with author before escalating
- **Skipping cross-layer dependency check** — domain→infrastructure is the #1 architecture decay vector
- **Ignoring exception mode mismatch** — 阻断型 layer throwing to 分支型 caller breaks error handling contract
- **Incremental mode checking unchanged files** — only touch `git diff` output

## Quality Checklist

Before finalizing:
- [ ] Mode correctly detected from user's trigger phrase
- [ ] Only relevant rule files loaded (incremental) or all layers covered (full)
- [ ] Every violation references the specific rule file and section heading
- [ ] Each violation includes the exact fix, not just the problem
- [ ] CRITICAL issues listed first, prioritized by blast radius
- [ ] Anti-patterns cross-checked against `references/anti-patterns.md`
