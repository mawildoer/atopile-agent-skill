---
name: ato-language
description: Use when writing, reading, or reviewing .ato files. Triggers on "what interface for X", "how to connect Y", choosing between interface types, parameter constraints, tolerance syntax, or atopile design patterns.
---

# Atopile Language

## How to Find Information
- **Hover** (`mcp__atopile-lsp__get_hover`): pass file path + line + character to get full member lists, usage examples, and docs for any symbol. Always hover on an import or type before writing code that uses it.
- **Definition** (`mcp__atopile-lsp__get_definition`): jump to where a module/interface is defined -- use to read library source.
- **References** (`mcp__atopile-lsp__get_references`): find all usages of a symbol across the project.
- **Diagnostics** (`mcp__atopile-lsp__get_diagnostics`): parse errors and type mismatches for a file.
- **CLI**: run `ato <command> --help` for full flags. Use `ato --help` for the command list.
- **Packages**: use `find_packages` / `inspect_package` MCP tools or the ato registry at packages.atopile.io.

See the `lsp` skill for full LSP-over-MCP usage.

## Choosing the Right Interface
| Need | Use | Not |
|---|---|---|
| Power rail (VCC/GND pair) | `ElectricPower` | `Electrical` |
| Digital logic (GPIO, CS, EN, INT) | `ElectricLogic` | `Electrical` |
| Raw copper (single net) | `Electrical` | -- |
| I2C bus | `I2C` | two `ElectricLogic` |
| SPI bus | `SPI` | individual `ElectricLogic` |
| Analog signal (continuous, not binary) | `ElectricSignal` | `ElectricLogic` |

**Key judgment calls:**
- `ElectricLogic` has a `.reference` (power rail) that MUST be connected -- this sets the logic level. Forgetting it causes silent build failures.
- `Capacitor` has a `.power` member (ElectricPower) for direct power rail connection, AND `.unnamed[0]`/`.unnamed[1]` for pin-level wiring. Use `.power` when decoupling a rail; use `.unnamed[]` when wiring between arbitrary nets (e.g. bootstrap cap between BTST and SW).
- `Resistor` only has `.unnamed[0]`/`.unnamed[1]` -- no named pins.

## CLI Workflow
1. `ato build` -- solver: picks parts, evaluates assertions, generates BOM, updates layout
2. `ato build --open` -- launches KiCad after build
3. `ato create part` -- generate component .ato from JLCPCB
4. `ato dependencies add` / `ato dependencies remove` -- manage package deps

Run `ato <command> --help` for full flags. See the `build-and-test` skill for the full build/debug workflow.

## Key Gotchas
- **Tolerances REQUIRED**: `10kohm` finds zero parts (0% tolerance). Always use `10kohm +/- 5%` or a range like `1kohm to 1.1kohm`.
- **ElectricLogic.reference**: MUST connect to the correct power rail. Open-drain outputs (nCE, nINT, I2C) need pull-ups to `.reference.hv`.
- **Passive pin names**: `Resistor` and `Inductor` use `unnamed[0]`/`unnamed[1]`, not named pins. `Capacitor` also has `.power` as an ElectricPower shortcut.
- **`package = "0402"`**: constrains footprint only, not voltage or power rating. Always add `assert X.max_voltage >= ...` separately.
- **Assertion syntax**: `within` for bilateral ranges, `is` for exact match, `>=`/`<=` for bounds. Use `assert X within 10V +/- 5%` not `assert X is 10V`.
- **Bridge syntax** (`~>`): requires `#pragma experiment("BRIDGE_CONNECT")`. The module must have the `can_bridge` trait (Resistor, Capacitor, Inductor have it).
- **For loops**: require `#pragma experiment("FOR_LOOP")`.
- **Leading underscore = private**: prefix internal submodules and components with `_` (e.g. `_nt_acp`, `_cregn`). External code accessing `_`-prefixed members is a code smell — the module's public interface should be sufficient.
- **Avoid EOL / NRND parts**: when selecting ICs, check the manufacturer's product page for lifecycle status. Do not use parts marked "End of Life", "Not Recommended for New Designs", or "Last Time Buy". These will become unavailable mid-project. Always pick the current recommended part in the family.

## Datasheet-Driven Design
- **Download datasheets FIRST** — before writing any code. Use `curl` to grab the PDF from the manufacturer's website (ti.com, analog.com, st.com, microchip.com — prefer manufacturer over distributors) and save to a local `datasheets/` directory (e.g. `<manufacturer>_<part>_datasheet.pdf`). Also grab the application note if one exists (`_appnote.pdf`).
- **All agents read the local copy** — never search the internet for datasheets during design or review. The local copy is the single source of truth.
- **Encode every datasheet formula as an `assert`** — this makes designs reconfigurable (change one parameter, solver checks everything downstream) and self-documenting (the assertion IS the design rationale).
- **Use the datasheet's own constants** in assertions (e.g. `48kV` for a K-factor) so formulas are recognizable alongside the datasheet.

## Documentation
Every module must have clear documentation written while the design context is fresh:
- **Module-level docstring** — what it does, design-point parameters, how to re-rate. Written as a triple-quoted string at the top of the module.
- **Usage example** — show how to instantiate, connect interfaces, and constrain. This is the first thing a consumer reads — it must be complete and correct.
- **Section comments** — group related circuitry with `# --- Section Name ---` headers.
- **Assertion comments** — every non-obvious `assert` should explain what it checks and where the threshold comes from (datasheet section, calculation, etc.).

Documentation quality is a review gate — if a future agent can't use the module from its docs alone, the docs aren't good enough.

## Design Patterns
See `design-patterns.md` for idiomatic patterns extracted from production driver code — decoupling, for-loops, voltage dividers, pull-ups, Kelvin sense taps, component locking, datasheet formula encoding, submodule decomposition, config pins, reset handling, and I2C addressing.
