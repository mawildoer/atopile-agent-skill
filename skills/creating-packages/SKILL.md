---
name: creating-packages
description: Use when adding a new IC, component, or subcircuit to an atopile project. Triggers on creating atopile packages, sourcing parts, vendoring an existing package, or deciding whether to make a shared package vs. inline in a board. Also use when you see ato create part or are about to write a new .ato driver module.
---

# Creating Packages

For atopile syntax, interfaces, and design patterns, see the `ato-language` skill. For build workflow and debugging, see `build-and-test`.

## Search Before You Create

Writing a new part/driver is the last resort. Stop as soon as you find something usable.

1. **The atopile registry** — use the `find_packages` / `inspect_package` MCP tools, or browse packages.atopile.io. Install with `ato dependencies add <name>` (e.g. `atopile/usb-connectors`).

2. **Packages you can vendor** — other repos and your own past projects often have a driver for the same IC. Copy it into a local `packages/` directory and reference it via `type: file` in `ato.yaml`. Review and adapt — vendored packages may carry application-specific assumptions.

3. **Local shared packages** — check the project's own `packages/` (or equivalent) for something already vendored.

4. **Create new** — only if nothing above works. Use `ato create part` to generate the footprint/symbol + a `.ato` stub from a JLCPCB search, then write the driver around it.

## Package Conventions

These conventions keep packages predictable and reviewable. Adapt the exact namespace to your project.

- **Naming:** `<vendor>-<device>` (e.g. `ti-bq25756`, `microchip-emc2101`).
- **Identifier:** `<namespace>/<package-name>` in `ato.yaml`'s `package.identifier` (the namespace is your project/org slug).
- **`hide_designators: true`** on build targets — designators are assigned at the consuming board level, not in the package.
- **Two-file pattern:**
  - The **driver** (`<package-name>.ato`) contains the reusable, configurable module that consumers import.
  - A separate **`test.ato`** instantiates it in one specific configuration, connects all interfaces, and adds assertions that verify that configuration. Consumers import from the driver, never from the test.
- **Layout:** `<package-name>/ato.yaml`, `<package-name>.ato` (driver — importable), `test.ato` (build validation + design verification), `parts/` (footprint/symbol/step files).
- **Verification status:** every public module's docstring should declare its lifecycle state, updated as the module progresses:
  - `UNVERIFIED` — design complete, builds clean, not yet fabricated or tested
  - `BUILT` — PCB fabricated, not yet powered or tested
  - `TESTED` — powered and functionally verified on hardware
  - `PRODUCTION` — used in a shipped/deployed product

### Two build targets per package

```yaml
builds:
  package:
    entry: <package-name>.ato:<DriverModule>
    hide_designators: true
  test:
    entry: test.ato:Test
    hide_designators: true
```

- The `package` build compiles the driver module standalone (used for the package-level layout).
- The `test` build compiles the test harness that instantiates and wires the driver (used for a test-level layout with instance-prefixed addresses).

### Consuming a local package

```yaml
dependencies:
  - type: file
    identifier: <namespace>/<package-name>
    path: ../packages/<package-name>
```

Then import the driver:

```ato
from "<namespace>/<package-name>/<package-name>.ato" import DriverModule
```

## Shared Package vs. Inline

**Shared package** (in `packages/`): two or more boards use it, or it's a commodity block (MCU, regulator, current sense, connector).

**Inline in the board:** only one board uses it, and it's tightly coupled to that application.

When in doubt, start inline. Extract to a shared package when a second board needs it.
