# Building and Refining Reusable Package Wrappers

## Purpose

A package wrapper takes a raw IC/connector component (with its physical pin layout) and exposes it through clean, standard interfaces. This makes the component reusable across designs, buildable and testable in isolation, and publishable to the package store.

## Scope

When building a package wrapper, own exactly one package project. Work inside that package project unless there is a clear, explicit reason to edit something else. The goal is a package that is:

- Generic and reusable across designs
- Self-contained
- Minimal but complete
- Validated through its own package build target(s)

## Wrapper Rules

- Expose the chip's general capabilities, not one board's role names.
- Prefer standard library interfaces and simple compositions of them.
- Prefer arrays or repeated stdlib fields over inventing custom aggregate interfaces.
- Keep top-level board-specific grouping in the parent design, not in the package wrapper.
- Start with a minimal viable wrapper first. Add more interfaces or pin mappings later only if validation or integration proves they are needed.

## Supporting Parts

If the package needs supporting passives, crystals, connectors, or regulators that belong to the package itself, install them inside the package project. Use `parts_install(project_path="packages/<name>")` so any new supporting parts land inside that package project instead of the top-level project.

Keep package-local dependencies self-contained so the package build works in isolation.

## Build Workflow

- Work step by step: make one coherent package change, run that package build target, fix the result, then continue.
- Build package targets early and often.
- Prefer fixing one concrete package build error at a time.
- Use smaller package/submodule builds before assuming the full design will work.
- Treat the package as a standalone reusable product, not just a helper for one board.
- Keep the package buildable in isolation because that preserves layout reuse in larger assemblies and makes later publishing to the package store straightforward.
- Stop when the package is coherent, builds, and is minimally complete.

## Interface Exposure

Before writing any custom `interface`, check whether the wrapper boundary can be represented as:
- A stdlib interface (`SPI`, `UART`, `SWD`, `USB2_0_IF`, `CAN_TTL`, etc.)
- An array of stdlib signals/interfaces (`new ElectricLogic[3]`, `new ElectricPower[3]`)
- A few named stdlib fields directly on the module

Only define a custom interface when it represents a real reusable protocol/boundary that stdlib or simple composition does not already cover.

## Good Examples

Good package APIs:
- MCU wrapper exposing `power`, `swd`, `uart`, `spi`, `i2c`, `usb`, `gpio`, `adc`
- Regulator wrapper exposing `power_in`, `power_out`, `enable`, `pgood`
- Motor-driver wrapper exposing `power`, `logic_power`, `phase_outputs`, `fault`, `current_sense`
- Sensor wrapper exposing `power`, `i2c` or `spi`, interrupt pins, reset pins

## Avoid

- Board-specific names like `weapon_motor`, `radio_input`, `battlebot_interfaces`
- Creating extra wrapper aggregation layers instead of refining the package wrapper in place
- Waiting for broad design approval loops
- Treating an incomplete ideal wrapper as blocked work when a minimal generic wrapper can be built now

## Example Wrapper

```ato
#pragma experiment("BRIDGE_CONNECT")

import ElectricPower
import I2C
import ElectricLogic
import Capacitor

from "parts/Bosch_Sensortec_BME280/Bosch_Sensortec_BME280.ato" import Bosch_Sensortec_BME280_package

module BME280:
    """BME280 environmental sensor — temperature, humidity, pressure.

    Exposes:
    - power: 3.3V rail (1.71V to 3.6V operating range)
    - i2c: I2C interface (SDO tied low → address 0x76)
    - csb: chip-select for SPI mode disable
    """

    # External interfaces
    power = new ElectricPower
    i2c = new I2C

    # Package
    package = new Bosch_Sensortec_BME280_package

    # Power
    power.hv ~ package.VDD
    power.hv ~ package.VDDIO
    power.lv ~ package.GND
    assert power.voltage within 1.71V to 3.6V

    # Decoupling
    decoup = new Capacitor
    decoup.capacitance = 100nF +/- 20%
    decoup.package = "0402"
    power.hv ~> decoup ~> power.lv

    # I2C — connect via .line and .reference
    i2c.sda.line ~ package.SDI
    i2c.scl.line ~ package.SCK
    i2c.sda.reference ~ power
    i2c.scl.reference ~ power

    # SDO low → I2C address 0x76
    package.SDO ~ power.lv

    # CSB high → I2C mode (not SPI)
    csb_pullup = new Resistor
    csb_pullup.resistance = 10kohm +/- 5%
    csb_pullup.package = "0402"
    package.CSB ~> csb_pullup ~> power.hv
```

## Imports

- Import package-local dependencies using the package project's own dependency/import structure.
- Do not depend on the top-level design to make your package build pass.

## Publishing

When a package is complete and builds in isolation, it can be published to the atopile registry with `ato publish`. The package can then be installed in other projects with `ato add <package-name>` / `packages_install`.
