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

Copy `skills/atopile-dev/SKILL.md` and `skills/atopile-dev/references/` into `.cursor/rules/`.

### Other agents

Point your agent at `skills/atopile-dev/SKILL.md`.

## What's included

| File | Description |
|---|---|
| `skills/atopile-dev/SKILL.md` | Main skill -- overview of ato syntax, key concepts, standard library, and MCP tools |
| `skills/atopile-dev/references/syntax.md` | Complete ato syntax examples |
| `skills/atopile-dev/references/grammar.md` | Full ANTLR4 parser grammar |
| `skills/atopile-dev/references/common-modules.md` | Standard library module and interface APIs |
| `skills/atopile-dev/references/language.md` | Language features deep-dive |
| `skills/atopile-dev/references/packages.md` | Step-by-step package creation guide |
| `skills/atopile-dev/references/vibe-coding.md` | End-to-end electronics design workflow for AI agents |

## Contributing

Contributions to this skill are welcome via pull requests on the [GitHub repository](https://github.com/mawildoer/atopile-agent-skill).

If you find inaccuracies, missing patterns, or have suggestions for improving agent behavior with atopile, please open an issue or PR.

## License

This agent skill is provided under the [MIT License](LICENSE).
