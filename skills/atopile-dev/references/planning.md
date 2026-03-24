# Spec-Driven Planning for Complex Designs

## When to Plan

**Simple tasks — just do it:**
- Single component add/remove/change, value change, rename
- Read/explain code or design
- Fix a specific build error
- Any task with a single clear action

**Complex tasks — always plan first:**
- Multi-component system design (2+ ICs interacting)
- New board or subsystem from scratch
- Unclear or function-level requirements ("I need a motor driver", "design me a sensor board")
- Tasks where you need to make multiple architectural choices

Do not ask whether to plan. For complex tasks, go straight into planning.

## The Spec IS the Design

The spec and the design are **one and the same** `.ato` file. A spec is just the design at a high level of abstraction — skeleton modules, interfaces, constraints, and requirements. As you implement, you fill in real components, pin mappings, and values. The file grows; the structure stays.

**Do not create separate spec files.** The main `.ato` file IS the spec.

**Do not suffix module names with "Spec".** `PowerSupply`, not `PowerSupplySpec`. These names persist into the final design.

## Project Structure

Every project with ICs should follow this structure. IC wrapper packages are separate from the main design.

```
my-project/
├── ato.yaml                        # Project-level builds defined here
├── main.ato                        # Top-level design — imports packages, not raw parts
├── packages/
│   ├── stm32g474/
│   │   └── stm32g474.ato           # Wrapper: raw pins → standard interfaces
│   ├── drv8317/
│   │   └── drv8317.ato
│   └── tcan3414/
│       └── tcan3414.ato
├── parts/                          # All raw parts (ICs + connectors)
└── layouts/
```

### What goes where

| Item | Location | Why |
|------|----------|-----|
| IC wrapper modules | `packages/<name>/<name>.ato` | Complex pin mapping, reusable |
| All raw parts | `parts/` (project root) | Installed by `parts_install` |
| Simple self-contained parts | Used directly in `main.ato` | No supporting components needed (connectors, LEDs, test points) |
| Generic passives | stdlib (`import Resistor`) | No package needed |
| Top-level design | `main.ato` | Imports wrappers, never raw `_package` |

### Key rules

- **ICs always get wrapper packages** — MCU, gate driver, transceiver, anything with complex pin mapping
- **Wrapper modules expose standard interfaces** — `ElectricPower`, `I2C`, `SPI`, `CAN`, `UART`, `SWD`, `USB2_0_IF`, `ElectricLogic`, `ElectricSignal`, not raw pins
- **Check stdlib before defining new interfaces** — if stdlib already has the right interface, or the boundary can be modeled as arrays/composition of stdlib interfaces, use that instead
- **Wrapper packages are generic** — expose the chip's reusable capabilities, not one project's exact naming or decomposition
- **No `ato.yaml` inside package directories** — package targets are discovered automatically

### `ato.yaml` format

```yaml
requires-atopile: ^0.14.0

paths:
  src: ./
  layout: ./layouts

builds:
  default:
    entry: main.ato:DualBLDCController
```

## Package Wrapper Pattern

```ato
#pragma experiment("BRIDGE_CONNECT")

import ElectricPower
import CAN
import ElectricLogic
import Capacitor

from "parts/STMicroelectronics_STM32G474CBT6/STMicroelectronics_STM32G474CBT6.ato" import STMicroelectronics_STM32G474CBT6_package

module STM32G474:
    """STM32G474 MCU with decoupling and standard interfaces."""

    # External interfaces
    power = new ElectricPower
    can = new CAN
    pwm_a = new ElectricLogic[3]
    pwm_b = new ElectricLogic[3]

    # Package
    package = new STMicroelectronics_STM32G474CBT6_package

    # Power
    power.hv ~ package.VDD
    power.hv ~ package.VDDA
    power.lv ~ package.VSS
    power.lv ~ package.VSSA
    assert power.voltage within 3.3V +/- 10%

    # Decoupling
    decoupling = new Capacitor[3]
    for cap in decoupling:
        cap.capacitance = 100nF +/- 10%
        cap.package = "C0402"
        power ~> cap ~> power.lv

    # CAN
    can.tx.line ~ package.PA11
    can.rx.line ~ package.PA12
    can.tx.reference ~ power
    can.rx.reference ~ power

    # PWM
    pwm_a[0].line ~ package.PA8
    pwm_a[1].line ~ package.PA9
    pwm_a[2].line ~ package.PA10
```

## Clean `main.ato`

```ato
#pragma experiment("BRIDGE_CONNECT")

import ElectricPower

from "packages/stm32g474/stm32g474.ato" import STM32G474
from "packages/drv8317/drv8317.ato" import DRV8317

module DualBLDCController:
    """Dual BLDC motor controller for robot drivetrain."""

    mcu = new STM32G474
    motor_a = new DRV8317
    motor_b = new DRV8317

    power = new ElectricPower
    power ~ mcu.power
    power ~ motor_a.motor_supply
    power ~ motor_b.motor_supply

    mcu.pwm_a ~ motor_a.pwm
    mcu.pwm_b ~ motor_b.pwm
```

## Requirements in Docstrings

Capture natural-language requirements directly in the module's docstring under a `Requirements:` section.

```ato
module PowerStage:
    """Three-phase MOSFET bridge sized for continuous motor current.

    Requirements:
    - R1: 20A continuous — FET stage rated for 20A with thermal margin
    """
```

Format: `- R<id>: <short text> — <criteria>`

## Spec Format

The spec is the skeleton of the design — skeleton modules, interfaces, constraints, and requirements. Implementation details (pin mappings, support circuits) get filled in during implementation.

```ato
#pragma experiment("BRIDGE_CONNECT")

import ElectricPower
import CAN
import ElectricLogic

module BLDCController:
    """
    # BLDC Motor Controller

    Dual-motor BLDC controller using STM32G474 and two DRV8317 drivers.

    ## Requirements
    - R1: MCU platform — Uses STM32G474
    - R2: 5-18V input — Operating voltage range
    - R3: Dual motor — 2x DRV8317 in 3-PWM mode

    ## Open Questions
    - Current sensing: phase shunt vs low-side?
    """

    power = new PowerSupply
    control = new MCU
    motor_a = new MotorDrive
    motor_b = new MotorDrive

    power.rail_3v3 ~ control.power
    power.motor_supply ~ motor_a.supply
    power.motor_supply ~ motor_b.supply
    control.pwm_a ~ motor_a.pwm
    control.pwm_b ~ motor_b.pwm

    assert power.vin.voltage within 5V to 18V

module PowerSupply:
    """Power input and regulation."""
    vin = new ElectricPower
    rail_3v3 = new ElectricPower
    motor_supply = new ElectricPower

module MCU:
    """STM32G474 with timers and comms peripherals."""
    power = new ElectricPower
    can = new CAN
    pwm_a = new ElectricLogic[3]
    pwm_b = new ElectricLogic[3]

module MotorDrive:
    """DRV8317 3-phase gate driver."""
    supply = new ElectricPower
    pwm = new ElectricLogic[3]
```

## Planning Flow

### Phase 1: Spec & Ask (end turn after this)

Do steps 1-4 in a SINGLE turn — do not end your turn after announcing you will plan.

1. **Read** existing project files to understand current state.
2. **Set up project structure** — create the project-level `ato.yaml` and `packages/` directories.
3. **Write the spec** as `main.ato` — architecture with sub-modules, requirements in docstrings, interface connections, and formal constraints. Use standard library interfaces (CAN, I2C, SPI, SWD, USB2_0, ElectricPower, ElectricLogic, ElectricSignal) in the spec instead of inventing local interfaces unless there is a real reusable boundary not covered by stdlib.
4. **Gather all open questions at once.** Use a single batched call for ALL open questions. Include suggested options and recommended defaults where possible.

### Phase 2: Lock decisions (brief)

5. **Wait for user answers.** Incorporate all decisions into the spec in one pass.

### Phase 3: Implement end-to-end (do not stop)

6. **Create package wrappers** — one per IC. Install parts, inspect vendor datasheets/design guides with `web_search`, map pins to interfaces.
   - Keep wrappers reusable across projects.
   - Start each wrapper as a basic reusable boundary with the minimum standard interfaces needed to validate the package target.
   - Once a package project exists and the work is independent, work on packages in parallel where practical.
7. **Validate package targets first** — use `workspace_list_targets` to discover package targets, then build/fix wrappers before attempting the full design.
8. **Wire up `main.ato`** — connect packages through their interfaces. No raw `_package` imports here.
9. **Build and verify the full design last** — once package targets are green, run the top-level build to catch integration issues.

### Phase 4: Report results

10. **Return results** with a concise summary of what changed, build status, and any assumptions you made.

## Rules

- **Do not ask whether to plan** — for complex tasks, just do it.
- The spec IS the design file — same modules, same names, same structure. It just starts abstract and gets filled in.
- **Do not rename modules** when transitioning from spec to implementation. `PowerSupply` stays `PowerSupply`.
- Place requirements in the docstring of whichever module owns them, not all on the top-level.
- **IC wrappers go in `packages/`**, not in `main.ato`. Raw `_package` components are never imported in `main.ato`.
- Update the spec as you learn things (it's a living document).
- If a build fails during implementation, check if the fix still meets requirements before moving on.
- For simple tasks, skip all of this — just implement directly.
- Keep requirements verifiable, not vague.
