# ddd-check

DDD compliance check skills for Claude Code. Two modes — fast incremental and thorough full audit.

## Skills

### `ddd-check-incremental`

Scans **only changed/new Java files** (via `git diff`) against DDD rules. Fast, pre-commit focused.

Trigger: "check my changes", "DDD check", "review this code"

### `ddd-check-full`

Audits **every Java file** in the project against all DDD rules. For architecture reviews and pre-release checks.

Trigger: "full DDD audit", "architecture review", "check entire project"

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
│   │   └── SKILL.md
│   └── ddd-check-full/
│       └── SKILL.md
├── rules/                    # DDD layer rules (checked against code)
│   ├── domain-layer.md
│   ├── application-layer.md
│   ├── adaptor-layer.md
│   ├── infrastructure-layer.md
│   ├── client-layer.md
│   ├── model-layer.md
│   └── anti-patterns.md
├── references/               # Education and onboarding
│   ├── overview.md
│   ├── base-classes-reference.md
│   ├── exception-handling.md
│   ├── anemic-vs-ddd.md
│   └── quick-start-tutorial.md
└── README.md
```

## License

Apache 2.0 — see [LICENSE](LICENSE)
