---
name: build-and-test
description: Use when building, validating, or debugging atopile projects. Triggers on build errors, "ato build", solver failures, "no parts found", package management, creating parts from JLCPCB, or debugging connectivity issues.
---

# Build & Test Workflow

## Typical Order

1. **`ato validate <file.ato>`** -- quick syntax check. Catches parse errors before a full build. No solver, no parts, no PCB. Use after editing .ato files to confirm they parse cleanly.
2. **`ato build`** -- the main command. Runs from the project directory (wherever `ato.yaml` lives). Stages in order: resolve dependencies, build context, instantiate, verify design, pick parts from JLCPCB, update PCB, generate reports. A successful build means the design is electrically valid and all parts are available.
3. **`ato build --open`** -- same as above, then launches KiCad layout editor so you can inspect/route the PCB.
4. **`ato build -b <name>`** -- build a specific target from `ato.yaml` (e.g. `ato build -b test`). Useful when a project has multiple build targets.
5. **`ato build --verbose`** -- runs sequentially without the live display. Better for reading error messages.
6. **`ato inspect`** -- debug connectivity. Shows what is connected to a module at a given boundary. Use `--inspect <module> --context <parent>` to see connections from a specific viewpoint.
7. **`ato create part -s "<search>"`** -- search JLCPCB and generate a component package (.ato + KiCad footprint/symbol) in `parts/`. Interactive: presents matches and lets you pick.

Run any command with `--help` for full flag details.

## Build Outputs

After a successful build, outputs land in `build/builds/<target>/`:

| File | Contents |
|------|----------|
| `<target>.bom.csv` | Bill of materials -- designators, footprints, manufacturer, LCSC part numbers |
| `<target>.bom.json` | Same BOM in JSON format |
| `<target>.variables.ato.json` | All solved parameter values -- spec vs actual, whether constraints are met |
| `<target>.power_tree.ato.json` | Power tree structure (JSON) |
| `power_tree.md` | Power tree as Mermaid diagram |
| `<target>.data_interface_tree.ato.json` | Data interface topology |
| `pinout/` | Per-component pinout JSON files |
| `backups/` | PCB backup snapshots |

The `build/cache/` directory stores downloaded datasheets (PDFs) for picked parts.

## Debugging Build Errors

### "No parts found matching constraints"

The most common error. The JLCPCB part solver cannot find a real part matching your parameter assertions. Fix in this order:

1. **Add tolerances** -- `resistance = 10kohm` matches nothing (0% tolerance). Use `resistance = 10kohm +/- 5%` or `assert resistance within 10kohm +/- 10%`.
2. **Widen package constraint** -- `package = "0201"` severely limits options. Try `"0402"` or remove the package constraint entirely.
3. **Check for conflicting assertions** -- two assertions on the same parameter can create an impossible range. Use `get_hover` from the LSP on the parameter to see the effective constraint.
4. **Check max_voltage / max_current** -- if you assert `max_voltage within 100V +/- 10%` on a 0402 cap, nothing exists. Relax or remove secondary constraints.
5. **Inspect the variables report** -- after a partial build, check `build/builds/<target>/<target>.variables.ato.json` to see what the solver computed for each parameter.

### "Assertion failed"

A design rule check (assert statement) evaluated to false after part picking. The error message names the specific assertion and the values it compared.

1. Read the assertion in the .ato source -- it tells you what relationship was expected.
2. Use `get_hover` on the parameters involved to see their solved values.
3. Either loosen the assertion (widen the tolerance) or tighten the input (pick a more precise part, constrain upstream parameters).

### Connection / wiring errors

Symptoms: "cannot connect X to Y", type mismatch on `~`, or missing pins.

1. You can only connect interfaces of the same type. `ElectricPower ~ Electrical` will fail -- you need to connect to the correct sub-interface (e.g. `power.hv ~ some_electrical`).
2. Use `ato inspect --inspect <module> --context <parent>` to see what is actually connected where.
3. Check `~` vs `~>` usage. `~>` is the bridge-connect operator and requires `#pragma experiment("BRIDGE_CONNECT")` and the module must have the `can_bridge` trait.
4. Make sure pin names match. Use `get_hover` or `get_definition` on the component to verify pin names.

### Import / dependency errors

Symptoms: "module not found", "cannot resolve import", file path errors.

1. Check `ato.yaml` -- is the dependency listed? Is the path correct for `type: file` deps?
2. Run `ato dependencies sync` to install/update dependencies.
3. For registry packages: `ato dependencies add <package-name>` to add and install.
4. For local packages: use `type: file` with a relative path from the project root.
5. Verify the import path matches the actual file path and module name.

### Build hangs or is very slow

The part picker queries JLCPCB and can be slow. If it hangs:

1. Check your internet connection -- part picking requires network access.
2. Use `--verbose` to see which stage is stalling.
3. If stuck on "Picking parts", you may have too many unconstrained components. Add package constraints to narrow the search space.
4. Use `--keep-picked-parts` to reuse previously picked parts and skip re-solving.

### PCB-related errors

1. `--frozen` mode (CI) fails if the PCB needs changes. Remove `--frozen` locally to let it update, then commit the result.
2. "Backing up unsaved pcb changes" is informational, not an error -- it saves your KiCad session before overwriting.
3. If KiCad is open and locked, close it or remove the `.lck` file before building.

## Package Management

```yaml
# ato.yaml dependency types:
dependencies:
  - type: registry          # From packages.atopile.io
    identifier: atopile/netties
    release: 0.2.0

  - type: file              # Local/vendored package
    identifier: <namespace>/<package-name>
    path: ../packages/<package-name>
```

| Command | What it does |
|---------|-------------|
| `ato dependencies add <pkg>` | Add a registry package (e.g. `atopile/netties`) |
| `ato dependencies add file://<path>` | Add a local package |
| `ato dependencies remove <pkg>` | Remove a dependency |
| `ato dependencies sync` | Install/update all deps to match ato.yaml |
| `ato dependencies sync -U` | Upgrade deps, ignoring pinned versions |
| `ato dependencies list` | Show current dependencies |

## LSP Integration for Debugging

Use the atopile LSP MCP tools (see the `lsp` skill) alongside builds:

- **`get_diagnostics`** -- faster than a full build for catching syntax errors after edits.
- **`get_hover`** -- inspect solved parameter constraints, interface types, and member lists. Essential for debugging "no parts found" and "assertion failed".
- **`get_definition`** -- jump to where an interface or module is defined when you need to understand its structure.
- **`get_references`** -- find everywhere a symbol is used when tracing connectivity issues.

## Build Flags Quick Reference

| Flag | Use case |
|------|----------|
| `--verbose` / `-v` | See full sequential output instead of live display |
| `--open` | Launch KiCad after build |
| `--frozen` | CI mode -- fail if PCB would change |
| `--keep-picked-parts` | Reuse previously picked parts |
| `--keep-net-names` | Preserve net names from PCB |
| `--keep-designators` | Preserve designators from PCB |
| `--all` | Build all projects found recursively |
| `-j N` | Set max concurrent builds (default: 14) |
