---
name: ddd-check
description: DDD compliance checking skills for Java projects. Two skills: ddd-check-incremental (fast, git diff only) and ddd-check-full (complete architecture audit). Validates layer isolation, base classes, naming, dependency direction, exception handling, and anti-patterns.
user-invocable: false
disable-model-invocation: true
---

# DDD Check Skills

Two Claude Code skills for DDD compliance checking:

| Skill | Scope | Speed | Use Case |
|---|---|---|---|
| `ddd-check-incremental` | git diff Java files only | Fast | Pre-commit, PR review |
| `ddd-check-full` | All Java files | Thorough | Architecture review, pre-release |

## Install

```bash
# Clone and register both skills
git clone https://github.com/bookiosk/ddd-check.git

# Add to .claude/settings.json:
{
  "skills": {
    "ddd-check-incremental": {
      "path": "ddd-check/skills/ddd-check-incremental/SKILL.md"
    },
    "ddd-check-full": {
      "path": "ddd-check/skills/ddd-check-full/SKILL.md"
    }
  }
}
```

## Rules

Rule files in `rules/` cover all 6 DDD layers + anti-patterns. Reference docs in `references/` for education.
