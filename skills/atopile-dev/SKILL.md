---
name: atopile-dev
description: >
  Design electronics with atopile's declarative DSL (ato). Use when working with
  .ato files, ato.yaml configs, PCB design, electronic component selection,
  or when the user mentions atopile, ato, circuits, PCBs, or electronics design.
license: MIT
metadata:
  author: mawildoer
  version: "2.0"
---

# atopile Development Skill

## What is ato

ato is a **declarative, constraint-based DSL** for describing electronic circuits. There is no control flow, no mutation, and no execution order — you declare _what_ a circuit is, and the compiler + solver resolve it into a valid design. It is part of the atopile project. The atopile toolchain is run via the VS Code / Cursor / Windsurf extension, which invokes the CLI to build projects.

## Quick Start

A minimal complete `.ato` file:

```ato
#pragma experiment("BRIDGE_CONNECT")

import Resistor
import ElectricPower
import Capacitor

module PowerFilter:
    """A simple decoupled power input with a pull-down resistor."""
    power = new ElectricPower
    decoupling_capacitor = new Capacitor
    pulldown_resistor = new Resistor

    power.hv ~> decoupling_capacitor ~> power.lv
    power.hv ~> pulldown_resistor ~> power.lv

    decoupling_capacitor.capacitance = 100nF +/- 20%
    pulldown_resistor.resistance = 100kohm +/- 5%
    assert power.voltage within 3.0V to 3.6V
```

Validate with `ato build` from the package directory.

## What's NOT in ato

ato is **fully declarative** — no procedural code, no side effects. The following constructs do **not** exist:

- if statements
- while loops
- functions (calls or definitions)
- classes / objects
- exceptions / generators

## Key Concepts

### Block types

- **`module`** — the primary building block; can be instantiated (like a class in OOP)
- **`interface`** — describes a connectable interface (e.g. `Electrical`, `ElectricPower`, `I2C`)
- **`component`** — a module variant for code-as-data (footprint pinmaps); usually auto-generated

### Parameters and Constraints

Variables for numeric/physical values that work with constraints.

```ato
resistance: ohm
assert resistance within 10kohm +/- 10%
```

**Important:** Always use toleranced values. Constraining to exactly `10kohm` (0% tolerance) will match zero parts.

### Connections

- **`~`** (wire) — connects two interfaces of the same type: `a ~ b`
- **`~>`** (directed/bridge) — series connection through a bridgeable module: `power.hv ~> cap ~> power.lv`
- Only interfaces of the **same type** can be connected
- Requires `#pragma experiment("BRIDGE_CONNECT")` for `~>`

### Traits

Mark a module with functionality: `trait has_designator_prefix`

### Experimental features

Enable with `#pragma experiment("FEATURE_NAME")` at the top of the file, before imports:

| Feature | Enables |
|---|---|
| `BRIDGE_CONNECT` | `~>` directed connect syntax |
| `FOR_LOOP` | `for item in container:` syntax |
| `TRAITS` | `trait` syntax |
| `MODULE_TEMPLATING` | `new MyModule<param=literal>` syntax |
| `INSTANCE_TRAITS` | traits on instances |

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

## Statement Reference

Every statement inside a block body is one of:

| Statement | Syntax | Purpose |
| --------- | ------- | ------- |
| `assign` | `name = value` or `name = new Type` | Bind a value or instantiate a child |
| `connect` | `a ~ b` | Wire two interfaces together |
| `bridge` | `a ~> b ~> c` | Insert bridgeable components in series |
| `assert` | `assert expr <op> expr` | Declare a constraint |
| `retype` | `name -> NewType` | Replace an inherited child's type |
| `pin` | `pin VCC` | Declare a physical pin |
| `signal` | `signal reset` | Declare an electrical signal |
| `trait` | `trait TraitName` | Attach a trait |
| `import` | `import Type` | Import a type |
| `for` | `for x in arr:` | Iterate over an array (pragma-gated) |
| `string` | `"""..."""` | Documentation string |
| `pass` | `pass` | Empty placeholder |

Statements within a block are **order-independent** — the compiler resolves the full graph, not a sequence of operations.

## Standard Library

### Key Interfaces

| Type | Children / Parameters | Purpose |
| ---- | --------------------- | ------- |
| `Electrical` | _(single node)_ | Raw electrical connection point |
| `ElectricPower` | `.hv`, `.lv` (Electrical); `.voltage`, `.max_current` | Power rails |
| `ElectricLogic` | `.line` (Electrical), `.reference` (ElectricPower) | Digital signals with voltage context |
| `ElectricSignal` | `.line` (Electrical), `.reference` (ElectricPower) | Analog signals |
| `I2C` | `.scl`, `.sda` (ElectricLogic); `.frequency`, `.address` | I2C bus |
| `SPI` | `.sclk`, `.mosi`, `.miso` (ElectricLogic); `.frequency` | SPI bus |
| `UART` / `UART_Base` | `.tx`, `.rx` (ElectricLogic); flow control lines | Serial |
| `CAN_TTL` | CAN bus lines | CAN bus |
| `USB2_0` / `USB2_0_IF` | USB data + power | USB interfaces |
| `SWD` / `JTAG` | debug lines | Debug interfaces |
| `DifferentialPair` | `.p`, `.n` | Differential signals |

### Key Modules

| Type | Parameters | Designator |
| ---- | ---------- | ---------- |
| `Resistor` | `.resistance`, `.max_power` | R |
| `Capacitor` | `.capacitance`, `.max_voltage`, `.temperature_coefficient` | C |
| `Inductor` | `.inductance` | L |
| `Diode` | `.forward_voltage`, `.current` | D |
| `LED` | `.brightness`, `.color` | D |
| `MOSFET` | `.channel_type`, `.gate_source_threshold_voltage` | Q |
| `Fuse` | `.trip_current`, `.fuse_type` | F |
| `Crystal` | `.frequency`, `.load_capacitance` | XTAL |
| `TestPoint` | `.pad_size`, `.pad_type` | TP |
| `ResistorVoltageDivider` | voltage divider circuit | — |

### Traits

`has_part_removed`, `is_atomic_part`, `can_bridge`, `has_datasheet`, `has_designator_prefix`, `has_net_name_suggestion`, `has_package_requirements`

## Units and Literals

**SI-prefixed units**: `V`, `mV` | `A`, `mA` | `ohm`, `kohm`, `Mohm` | `F`, `uF`, `nF`, `pF` | `Hz`, `kHz`, `MHz`, `GHz` | `s`, `ms` | `W`, `mW`

**Number formats**: decimal (`3.3`), scientific (`1e-6`), hex (`0x48`), binary (`0b1010`), underscore-separated (`1_000_000`)

**Booleans**: `True`, `False`

## Common Invariants

1. **Type-safe connections**: `~` and `~>` connect matching interface types. `ElectricPower ~ I2C` is a type mismatch.
2. **Pragma gates syntax**: using `~>`, `for`, `trait`, or `<>` without the matching pragma is a compile error.
3. **Tolerances on passives**: `resistance = 10kohm` (zero tolerance) matches no real parts. Always use `+/- N%`.
4. **ElectricLogic needs a reference**: logic signals require a power reference for voltage context. Set `signal.reference ~ power_rail`.
5. **Order independence**: statements within a block are not sequentially executed. The solver resolves the full graph.

## MCP Tools

The atopile MCP server provides tools for working with projects:

| Tool | Purpose |
|---|---|
| `stdlib_list` | List available interfaces and modules |
| `stdlib_get_item` | View source code of a module/interface |
| `parts_search` / `parts_install` | Search JLCPCB and install a part |
| `packages_search` / `packages_install` | Search for and install atopile registry packages |
| `package_ato_list` / `package_ato_read` | Inspect installed package `.ato` sources |
| `examples_list` / `examples_search` / `examples_read_ato` | Browse curated reference designs |
| `build_run` | Build the project and update PCB |
| `build_logs_search` | Search build logs with filters |
| `workspace_list_targets` | Discover all buildable targets |
| `design_diagnostics` | Diagnose silent failures |
| `report_bom` | Generate bill of materials |
| `report_variables` | Inspect constraint state |
| `layout_set_component_position` | Set component placement |
| `layout_run_drc` | Run design rule check |
| `web_search` | Research datasheets and application notes |

## Footprints & Part Picking

- Use `parts_install` to install parts from JLCPCB/LCSC
- Use `parts_install(create_package=true)` to generate a reusable wrapper package under `packages/`
- The `pin` keyword builds footprint pinmaps — use only inside `component` blocks
- Use `Electrical` for electrical interfaces, `ElectricLogic` for GPIOs, `ElectricPower` for power
- Passive components (Resistors, Capacitors) are auto-picked from parameter constraints
- Set package: `package = "0402"`
- Explicit part selection: `lcsc = "<LCSC_PART_NUMBER>"`

## Project Structure

Every project with ICs should follow this structure:

```
my-project/
├── ato.yaml                        # Project-level builds defined here
├── main.ato                        # Top-level design — imports packages, not raw parts
├── packages/
│   ├── stm32g474/
│   │   └── stm32g474.ato           # Wrapper: raw pins → standard interfaces
│   └── drv8317/
│       └── drv8317.ato
├── parts/                          # All raw parts installed by parts_install
└── layouts/
```

Key rules:
- **ICs always get wrapper packages** — MCU, gate driver, transceiver, anything with complex pin mapping
- **Wrapper modules expose standard interfaces** — `ElectricPower`, `I2C`, `SPI`, `CAN`, `UART`, not raw pins
- **`main.ato` imports wrapper packages**, never raw `_package` components
- **No `ato.yaml` inside package directories** — package targets are discovered automatically

### `ato.yaml` format

```yaml
requires-atopile: ^0.14.0

paths:
  src: ./
  layout: ./layouts

builds:
  default:
    entry: main.ato:MyBoard
```

## Module Naming

Name modules by their **role in the system**, not their implementation topology.

Good names: `PowerSupply`, `BatteryCharger`, `GateDriver`, `CANTransceiver`, `USBPort`, `IMU`, `Debug`

Avoid: `CANSubsystem`, `PowerUnit`, `MotorBlock` — too generic. Avoid shadowing stdlib types (`CAN`, `USB`).

IC wrapper packages use the **part name** directly: `STM32G474`, `DRV8317`, `TCAN3414`.

## CLI & Packages

Install packages with `ato add <PACKAGE_NAME>`:

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
| [planning.md](references/planning.md) | Spec-driven planning for complex designs |
| [package-development.md](references/package-development.md) | How to build and refine reusable package wrappers |
