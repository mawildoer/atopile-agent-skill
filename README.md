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

This plugin provides a collection of skills covering ato language reference, electronics design workflows, package creation, and more. See the `skills/` directory for the full list — skills are extracted from the [atopile/atopile](https://github.com/atopile/atopile) repository and may change over time.

## Contributing

Contributions to this skill are welcome via pull requests on the [GitHub repository](https://github.com/mawildoer/atopile-agent-skill).

If you find inaccuracies, missing patterns, or have suggestions for improving agent behavior with atopile, please open an issue or PR.

## License

This agent skill is provided under the [MIT License](LICENSE).
