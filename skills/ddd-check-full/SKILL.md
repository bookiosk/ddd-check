---
name: ddd-check-full
description: Full-project DDD compliance audit scanning every Java file against all layer rules. Use when user mentions "full DDD audit", "architecture review", "DDD compliance report", "pre-release check", or "check entire project". For incremental checks, see ddd-check-incremental.
license: Apache-2.0
metadata: {"version": "1.0", "skill-author": "bookiosk"}
model: claude-sonnet-4-20250514
allowed-tools: Read, Grep, Glob, Bash
user-invocable: true
---

# DDD Full-Project Compliance Audit

Deep scan of **every** Java file against all DDD rules. For architecture review, refactoring assessment, or pre-release audit.

## Workflow

### Step 1: Discover All Java Files

```bash
find . -name "*.java" -not -path "*/target/*" -not -path "*/test/*" | sort
```

Report count: "Scanning {N} files across {M} layers..."

### Step 2: Load Rule Files

Read all files: `references/domain-layer.md`, `references/application-layer.md`, `references/adaptor-layer.md`, `references/infrastructure-layer.md`, `references/client-layer.md`, `references/model-layer.md`, `references/anti-patterns.md`. Load `references/base-classes-reference.md` and `references/exception-handling.md` as needed.

### Step 3: Layer-by-Layer Audit

Group files by layer. For each layer check: package correctness, base class usage, naming conventions, dependency direction, structural integrity, exception mode (阻断型 throws, 分支型 returns ResultDO).

### Step 4: Cross-Layer Checks

- **Dependency direction**: `grep -r "import org\.bookiosk\.ddd\."` per layer, verify no reverse deps
- **Anemic model**: Any domain class with only getters/setters and zero behavior?
- **Exception boundary**: APP catching Domain/Adaptor exceptions? Any layer throwing across boundaries?

### Step 5: Report

Format as comprehensive report. See `examples/violation-report.md` for sample output.

## Layer Checklists

See `references/shared-checks.md` for the complete severity table and per-layer checklists.

## Common Mistakes

- **Flagging intentional violations as CRITICAL** — verify with author before escalating
- **Skipping cross-layer dependency check** — domain→infrastructure is the #1 architecture decay vector
- **Ignoring exception mode mismatch** — 阻断型 layer throwing to 分支型 caller breaks error handling contract

## Quality Checklist

Before finalizing the audit report:
- [ ] All layers scanned (no skipped packages)
- [ ] Every violation references the specific rule section (e.g. "rules/domain-layer.md Section A.3")
- [ ] Each violation includes the exact fix, not just the problem
- [ ] CRITICAL issues listed first, prioritized by blast radius
- [ ] Anti-pattern section cross-referenced with `references/anti-patterns.md`
