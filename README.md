# ddd-check

DDD compliance check skill for Claude Code. Auto-detects incremental vs full mode from trigger phrase.

[![agentskills.io](https://img.shields.io/badge/agentskills.io-compliant-6e3bf0)](https://agentskills.io)

## Install

### Via plugin marketplace (recommended)

```bash
claude plugin marketplace add bookiosk/bookiosk-marketplace
claude plugin install ddd-check@bookiosk-marketplace
```

Or in Claude Code: `/plugin` → Discover → search `ddd-check`.

### Manual

```bash
git clone https://github.com/bookiosk/ddd-check.git
```

Add to `.claude/settings.json`:

```json
{
  "skills": {
    "ddd-check": {
      "path": "ddd-check/skills/ddd-check/SKILL.md"
    }
  }
}
```

## Usage

```
/ddd-check     # Auto-detects mode from your prompt
```

| Say this | Does this |
|---|---|
| "check my changes" / "pre-commit check" | **Incremental** — git diff only |
| "full DDD audit" / "architecture review" | **Full** — every Java file |

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
├── .claude-plugin/
│   └── plugin.json                  # Plugin metadata
├── skills/
│   └── ddd-check/
│       ├── SKILL.md                 # Skill entry point
│       ├── references/              # DDD rules + guides (loaded on demand)
│       │   ├── domain-layer.md
│       │   ├── application-layer.md
│       │   ├── adaptor-layer.md
│       │   ├── infrastructure-layer.md
│       │   ├── client-layer.md
│       │   ├── model-layer.md
│       │   ├── anti-patterns.md
│       │   ├── shared-checks.md
│       │   ├── base-classes-reference.md
│       │   ├── exception-handling.md
│       │   ├── overview.md
│       │   ├── anemic-vs-ddd.md
│       │   └── quick-start-tutorial.md
│       └── examples/
│           └── violation-report.md
├── evals/
│   └── evals.json
├── README.md
└── LICENSE
```

## License

Apache 2.0 — see [LICENSE](LICENSE)
