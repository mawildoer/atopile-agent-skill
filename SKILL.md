---
name: atopile-dev
description: >
  Design electronics with atopile's declarative DSL (ato). Use when working with
  .ato files, ato.yaml configs, PCB design, electronic component selection,
  or when the user mentions atopile, ato, circuits, PCBs, or electronics design.
license: MIT
metadata:
  author: atopile
  version: "1.0"
---

# atopile Development Skill

## What is ato

ato is a declarative DSL for designing electronics (PCBs). It is part of the atopile project. The atopile toolchain is run via the VS Code / Cursor / Windsurf extension, which invokes the CLI to build projects.

## What's NOT in ato

ato is **fully declarative** — no procedural code, no side effects. The following constructs do **not** exist:

- if statements
- while loops
- functions (calls or definitions)
- classes
- objects
- exceptions
- generators

## Key Concepts

### Block types

- **`module`** — the primary building block; can be instantiated (like a class in OOP)
- **`interface`** — describes a connectable interface (e.g. `Electrical`, `ElectricPower`, `I2C`)
- **`component`** — a module variant for code-as-data (footprint pinmaps)

### Parameters

Variables for numeric/physical values that work with constraints.

```ato
resistance: ohm
assert resistance within 10kohm +/- 10%
```

**Important:** Always use toleranced values. Constraining to exactly `10kohm` (0% tolerance) will match zero parts.

### Connections

- **`~`** (wire) — connects two interfaces of the same type: `a ~ b`
- **`~>`** (directed/bridge) — series connection through a bridgeable module: `led.anode ~> resistor ~> power.hv`
- Only interfaces of the **same type** can be connected

### Traits

Mark a module with functionality: `trait has_designator_prefix`

### Experimental features

Enable with `#pragma experiment("FEATURE_NAME")`:

| Feature | Enables |
|---|---|
| `BRIDGE_CONNECT` | `~>` directed connect syntax |
| `FOR_LOOP` | `for item in container:` syntax |
| `TRAITS` | `trait` syntax |
| `MODULE_TEMPLATING` | `new MyModule<param=literal>` syntax |

## Syntax Quick-Reference

### Imports

```ato
import ModuleName
import Module1, Module2.Submodule
from "path/to/source.ato" import SpecificModule
```

### Block definitions

```ato
module MyModule:
    pass

module ChildModule from ParentModule:
    pass

interface MyInterface:
    pin io

component MyComponent:
    pin 1
    pin "GND"
```

### Pins and signals

```ato
pin my_pin          # by name
pin 1               # by number
pin "GND"           # by string
signal my_signal
```

### Instantiation

```ato
instance = new MyModule
array = new MyModule[10]
templated = new MyModule<param=1, other="hello">
```

### Assignments and declarations

```ato
field: TypeName                    # declaration with type
value = 123                        # assignment
voltage: V = 5V                   # typed assignment
resistance = 10kohm +/- 10%       # bilateral tolerance
voltage_range = 3V to 3.6V        # bounded range
value += 1                         # cumulative
```

### Connections

```ato
pin_a ~ pin_b                     # wire connection
mif ~> bridge ~> mif              # directed (bridge) connection
```

### Assertions

```ato
assert voltage within 3V to 3.6V
assert resistance is 10kohm +/- 10%
assert current within 1A +/- 10mA
assert x > 5V
assert 5V < x < 10V
```

### For loops

```ato
#pragma experiment("FOR_LOOP")

for item in container:
    item ~ target

for item in container[0:4]:
    item.value = 1
```

### Retyping

```ato
instance.field -> NewType
```

## Standard Library

### Electrical (interface)

```ato
interface Electrical:
    pass
```

The fundamental electrical connection point.

### ElectricPower (interface)

```ato
interface ElectricPower:
    hv = new Electrical
    lv = new Electrical
```

A power rail with high-voltage and low-voltage (ground) connections.

### Resistor (module)

```ato
module Resistor:
    resistance: ohm
    max_power: W
    max_voltage: V
    unnamed = new Electrical[2]
```

### Capacitor (module)

```ato
module Capacitor:
    capacitance: F
    max_voltage: V
    unnamed = new Electrical[2]
```

### I2C (interface)

```ato
interface I2C:
    scl = new ElectricLogic
    sda = new ElectricLogic
    frequency: Hz
    address: dimensionless
```

### ElectricLogic (interface)

```ato
interface ElectricLogic:
    line = new Electrical
    reference = new ElectricPower
```

For additional interfaces and modules, use the atopile MCP server tools.

## MCP Tools

The atopile MCP server provides tools for working with projects:

| Tool | Purpose |
|---|---|
| `get_library_interfaces` | List available interfaces |
| `get_library_modules` | List available modules |
| `inspect_library_module_or_interface` | View source code of a module/interface |
| `search_and_install_jlcpcb_part` | Search JLCPCB and install a part |
| `find_packages` | Search for atopile packages |
| `inspect_package` | View package details |
| `install_package` | Install a package |
| `build_project` | Build the project and update PCB |

## Footprints & Part Picking

- Use `ato create part` to auto-generate part code
- The `pin` keyword builds footprint pinmaps — use only inside `component` blocks
- Use `Electrical` for electrical interfaces, `ElectricLogic` for GPIOs, `ElectricPower` for power
- Passive components (Resistors, Capacitors) are auto-picked from parameter constraints
- Set package: `package = "0402"`
- Explicit part selection: `lcsc = "<LCSC_PART_NUMBER>"`

## CLI & Packages

Run ato commands through the MCP tool. Install packages with `ato add <PACKAGE_NAME>`:

```ato
from "atopile/addressable-leds/sk6805-ec20.ato" import SK6805_EC20_driver

module MyModule:
    led = new SK6805_EC20_driver
```

## Reference Files

For deeper detail, consult the reference files in `references/`:

| File | Content |
|---|---|
| [syntax.md](references/syntax.md) | Complete ato syntax examples |
| [grammar.md](references/grammar.md) | Full ANTLR4 parser grammar |
| [common-modules.md](references/common-modules.md) | Standard library module/interface APIs |
| [language.md](references/language.md) | Language features deep-dive (experimental features, connecting, CLI, packages) |
| [packages.md](references/packages.md) | Step-by-step package creation guide |
| [vibe-coding.md](references/vibe-coding.md) | End-to-end electronics design workflow for AI agents |
