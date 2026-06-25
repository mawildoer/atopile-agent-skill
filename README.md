# `atopile` Agent Skill

An unofficial, community-maintained [Agent Skills](https://agentskills.io)-compatible plugin that teaches AI agents how to develop with [atopile](https://atopile.io)'s declarative DSL for PCB and electronics design.

## Install

### Claude Code (plugin marketplace)

Add the marketplace and install the plugin:

```
/plugin marketplace add https://github.com/mawildoer/atopile-agent-skill.git
/plugin install atopile-dev@atopile
```

Or add it from a local checkout:

```
/plugin marketplace add ./agent-skill
/plugin install atopile-dev@atopile
```

### Claude Code (manual)

Load directly during development:

```bash
claude --plugin-dir ./agent-skill
```

### Cursor

Copy the relevant `skills/<name>/SKILL.md` files (and their reference files, e.g. `skills/ato-language/design-patterns.md`) into `.cursor/rules/`.

### Other agents

Point your agent at the `skills/` directory and let it load the relevant `SKILL.md` per task.

## What's included

These skills teach an agent how to *use* atopile to design boards — language reference, build/debug workflow, package creation, board composition, design review, and supporting workflows. They were distilled from real production atopile projects and generalized for any project.

| Skill | Use it for |
|---|---|
| `ato-language` | Writing/reading `.ato`: interfaces, constraints, tolerances, idiomatic design patterns (see `ato-language/design-patterns.md`) |
| `build-and-test` | Building, validating, and debugging — `ato build`, solver failures, "no parts found", dependency management |
| `lsp` | Semantic code intelligence over `.ato` via the atopile LSP (hover, diagnostics, definition, references) |
| `creating-packages` | Sourcing/vendoring parts and authoring new driver packages; shared-vs-inline decisions |
| `board-design` | Composing a board from packages: power rails, MCU peripherals, protection, self-test |
| `design-review` | Four-agent parallel hardware design review with confidence scoring and remediation flow |
| `layout-relinking` | Preserving PCB placements when renaming/refactoring components (`atopile_address` relinking) |
| `trade-study` | Structured comparison of viable options for hard-to-reverse design decisions |

Skills may change over time as atopile and the patterns evolve.

## Contributing

Contributions to this skill are welcome via pull requests on the [GitHub repository](https://github.com/mawildoer/atopile-agent-skill).

If you find inaccuracies, missing patterns, or have suggestions for improving agent behavior with atopile, please open an issue or PR.

## License

This agent skill is provided under the [MIT License](LICENSE).
