---
name: board-design
description: Use when designing a new board or modifying board-level architecture in atopile. Triggers on new board creation, connector wiring, power rail routing, MCU peripheral assignment, or composing packages into a board.
---

# Board Design

For atopile syntax and patterns, see the `ato-language` skill. For build workflow, see `build-and-test`. For creating new shared packages, see `creating-packages`.

A board is a composition: you instantiate packages (MCU, regulators, connectors, application ICs) and wire them together at the board level. The board module's body should read like a block diagram — major blocks first, then the connections between them.

## File Structure

```
<board-name>/
    <board-name>.ato         # Main board module
    ato.yaml                 # Build manifest
    layouts/                 # KiCad layout files (created by ato build)
    parts/                   # Board-specific parts (from ato create part)
```

### ato.yaml template

```yaml
requires-atopile: ^0.15.7
paths:
  src: ./
  layout: ./layouts
builds:
  <board-name>:
    entry: <board-name>.ato:<BoardModule>
    hide_designators: true
dependencies:
  # Local packages:
  - type: file
    identifier: <namespace>/<package-name>
    path: ../packages/<package-name>
  # Registry packages:
  - type: registry
    identifier: atopile/<package-name>
    release: <version>
```

### Board module template

```ato
#pragma experiment("BRIDGE_CONNECT")
#pragma experiment("TRAITS")

import Resistor
import Capacitor

from "<namespace>/<mcu-package>/<mcu-package>.ato" import MyMCU
from "atopile/usb-connectors/usb-connectors.ato" import USBCConn

module MyBoard:
    """One-line description of what this board does."""

    # --- Major blocks ---
    mcu = new MyMCU
    usb = new USBCConn

    # --- Power ---
    # power_in ~> regulator ~> power_3v3
    # power_3v3 ~ mcu.power

    # --- Communication / control ---
    # usb.usb2 ~ mcu.usb
    # app_ic.i2c ~ mcu.i2c[0]

    # --- Ground domains tied together ---
    # power_hv.lv ~ power_3v3.lv
```

## Composition Methodology

1. **Instantiate the major blocks first** — MCU, power source/regulators, connectors, application ICs. Give them clear names.
2. **Establish power rails** — identify each voltage rail, route the input source through regulators to each rail (`power_in ~> regulator ~> power_3v3`), and connect every block's power interface to the right rail.
3. **Wire the standard MCU peripherals consistently** — pick the same peripheral assignment convention across boards (e.g. always `mcu.i2c[0]` for the primary on-board I2C bus, a fixed UART for debug). Document which MCU pins each peripheral consumes so you don't double-assign a pin that's already used by a peripheral.
4. **Connect application buses** — wire each application IC's communication interface to an MCU peripheral (`app_ic.i2c ~ mcu.i2c[0]`, `app_ic.spi ~ mcu.spi[0]`). For single control lines use a GPIO: connect `.line` for the signal and set `.reference` to the rail that defines its logic level.
5. **Use bridge syntax for inline series elements** — `~>` for things like current-sense modules in a power path (requires `BRIDGE_CONNECT`).
6. **Tie ground domains together explicitly** at the board level — don't rely on implicit shared ground.

> **Watch for shared peripheral pins.** On most MCU packages, connecting a high-level peripheral (`mcu.i2c[0]`) also consumes the underlying GPIO pins. Keep a list of which pins each peripheral uses and which remain free, and never double-assign.

## Power Architecture Notes

- Decide where each rail is generated. A board may generate its own rails from an input source, or receive pre-regulated rails from a backplane/host — be explicit about which.
- **Current sensing:** prefer high-side sensing when multiple boards share a common ground. Low-side shunts create ground offsets between boards on a shared ground bus. Choose a current-sense amp whose common-mode range covers the bus voltage.
- **Single ground domain:** tie all grounds together unless you have a deliberate reason for isolated domains.

## External Interface Protection

Boards used in lab / test / field environments expose interfaces that need protection:

- **TVS diodes** for ESD/surge on power and exposed connectors
- **Current-limiting resistors** on high-impedance sense lines (e.g. in series before an ADC divider)
- **ADC clamping diodes** — Schottky to the rail and to GND on ADC inputs
- **Anti-alias capacitors** at divider taps
- **Reverse-polarity protection** where connectors are not keyed
- **PTC fuses** for overcurrent on unprotected connectors

## Self-Test Design

Design every board so it can be brought up and verified automatically — firmware flashed, every output exercised and confirmed without human interaction. This means every output action needs a feedback path:

- **Voltage output → ADC measurement** (through a divider if HV)
- **Relay/switch → sense line** confirming the state changed
- **I2C/SPI command → read-back** confirming the register was written
- **Current drive → current sense** confirming the expected current flows

Budget MCU ADC channels and GPIO for self-test. If an output can't be self-tested, document why in the module docstring.

## Composition Checklist

1. Create the `<board-name>/` directory
2. Create `ato.yaml` with build target and dependencies
3. Create `<board-name>.ato` with pragmas and imports
4. Instantiate the MCU and power source/regulators
5. Establish and wire each power rail
6. Wire the standard MCU peripherals (debug UART, host comms, power) consistently
7. Instantiate application-specific packages
8. Wire application IC power rails to the correct rail
9. Wire application IC communication buses to MCU peripherals (I2C, SPI, GPIO)
10. Add protection on external-facing interfaces (TVS, current-limit resistors, clamp diodes)
11. Tie all ground domains together explicitly
12. Run `ato build` to verify
