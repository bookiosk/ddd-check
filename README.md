# ddd-check

DDD compliance check skills for Claude Code. Two modes — fast incremental and thorough full audit.

[![agentskills.io](https://img.shields.io/badge/agentskills.io-compliant-6e3bf0)](https://agentskills.io)

## Skills

### `ddd-check-incremental`

Scans **only changed/new Java files** (via `git diff`) against DDD rules. Fast, pre-commit focused.

Trigger: "check my changes", "DDD check", "review this code", "pre-commit check"

### `ddd-check-full`

Audits **every Java file** in the project against all DDD rules. For architecture reviews and pre-release checks.

Trigger: "full DDD audit", "architecture review", "DDD compliance report", "check entire project"

## Install

```bash
git clone https://github.com/bookiosk/ddd-check.git
```

Add to `.claude/settings.json`:

```json
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

## Usage

```
/ddd-check-incremental     # Check changed files
/ddd-check-full            # Full project audit
```

## What It Checks

- Layer isolation (no wrong cross-layer imports)
- Base class usage (aggregate, entity, value object)
- Naming conventions (Cmd, Qry, Exe, ExtPt, etc.)
- Dependency direction (domain → infrastructure = CRITICAL)
- Identity protection (setId must be protected)
- Exception handling mode (阻断 throw vs 分支 ResultDO)
- Anti-patterns (anemic model, service overuse, wrong exception mode)

## File Layout

```
ddd-check/
├── skills/
│   ├── ddd-check-incremental/
│   │   ├── SKILL.md
│   │   ├── evals/
│   │   │   └── evals.json
│   │   ├── references/      → symlink to ../../rules/ + ../../references/
│   │   └── examples/        → symlink to ../../examples/
│   └── ddd-check-full/
│       ├── SKILL.md
│       ├── evals/
│       │   └── evals.json
│       ├── references/      → symlink to ../../rules/ + ../../references/
│       └── examples/        → symlink to ../../examples/
├── rules/                    # Single source of truth — DDD layer rules
│   ├── domain-layer.md
│   ├── application-layer.md
│   ├── adaptor-layer.md
│   ├── infrastructure-layer.md
│   ├── client-layer.md
│   ├── model-layer.md
│   └── anti-patterns.md
├── references/               # Single source of truth — guides and checklists
│   ├── overview.md
│   ├── shared-checks.md
│   ├── base-classes-reference.md
│   ├── exception-handling.md
│   ├── anemic-vs-ddd.md
│   └── quick-start-tutorial.md
├── examples/
│   └── violation-report.md
└── README.md
```

## License

Apache 2.0 — see [LICENSE](LICENSE)
