# Electronics Design Workflow for AI Agents

This is the canonical sequence for designing a board in atopile. Move quickly, keep the structure clean, and avoid spreading planning state across multiple rounds unless the design genuinely requires it.

## #1 Rule

**USE THE TOOLS.** If the tools don't work, don't freak out — you are probably using them wrong. Ask for help if you get stuck.

---

## Step 1: Draft The Architecture

Capture user intent as ato code immediately. Start with a clean high-level architecture and only stop to ask batched design questions when there are real unresolved decisions.

Focus on:
- What the system is supposed to do
- The main functional blocks
- The important interfaces and voltage domains
- Key constraints on size, cost, power, or manufacturing
- Any parts or protocols that are already fixed by the user

Use `web_search` if you need to research unfamiliar domains or components before locking the architecture.

## Step 2: Write The Spec

The spec IS the design file at a high level of abstraction. As you implement, you fill in real components and wiring. The file grows; the structure stays.

**Key principles:**
- **Good naming** — name modules by their role in the system, not implementation topology (e.g. `PowerSupply`, not `PowerSubsystem`).
- **Module boundaries** should encapsulate common functionality to avoid duplication at the top level.
- **Use high-level interfaces** (`ElectricPower`, `I2C`, `SPI`, `UART`, `ElectricLogic`) instead of low-level electrical connections where possible.
- **Custom interfaces are rare** — before defining a new `interface`, check stdlib first with `stdlib_list` / `stdlib_get_item`. If an existing stdlib interface or a simple composition/array of stdlib interfaces works, use that instead.
- **Capture requirements** in the module docstring under a `Requirements:` section on the module that owns them.
- **Add formal constraints** with `assert` for voltage, current, frequency bounds.
- **Wire modules together** at the interface level (`~`). Do NOT wire pins yet.

**Step-by-step:**
1. Break the request into subsystems. Each functional block becomes a `module` — power, MCU, sensors, comms, IO, etc.
2. Define interfaces at module boundaries. Use stdlib interfaces to declare how modules connect.
3. Capture requirements in docstrings. Add a `Requirements:` section to the docstring of the module that owns each requirement.
4. Add formal constraints with `assert` for voltage, current, frequency bounds.
5. Wire modules together at the interface level (`~`).

**Example spec:**

```ato
import ElectricPower
import I2C
import SPI
import ElectricLogic

module SensorBoard:
    """
    # Environmental Sensor Board

    Battery-powered sensor node with temperature, humidity, and
    pressure sensing, BLE comms, and USB-C charging.

    ## Requirements
    - R1: BLE connectivity — nRF52840 with BLE 5.0
    - R2: Environmental sensing — BME280 for temp/humidity/pressure
    - R3: USB-C charging — 5V USB-C input with charge IC
    - R4: Board size — 25mm x 30mm max
    """

    power = new PowerSupply
    mcu = new MCU
    sensors = new EnvironmentalSensor

    power.rail_3v3 ~ mcu.power
    power.rail_3v3 ~ sensors.power
    mcu.i2c ~ sensors.i2c

    assert power.usb_in.voltage within 4.5V to 5.5V
    assert power.rail_3v3.voltage within 3.3V +/- 5%

module PowerSupply:
    """USB-C input, charge controller, LDO regulation."""
    usb_in = new ElectricPower
    rail_3v3 = new ElectricPower

module MCU:
    """nRF52840 with crystal, decoupling, and debug header."""
    power = new ElectricPower
    i2c = new I2C
    spi = new SPI

module EnvironmentalSensor:
    """BME280 environmental sensor."""
    power = new ElectricPower
    i2c = new I2C
```

## Step 3: Resolve Open Decisions

Present the user with the current architecture and any real unresolved decisions:
- List the modules and their responsibilities.
- Show the interface connections between modules.
- Highlight any key decisions or trade-offs made.
- Call out any assumptions or areas where alternatives exist.

Batch unresolved decisions instead of trickling follow-up questions across multiple turns. Then incorporate the answers directly into the spec and continue implementation.

## Step 4: Implement Detailed Design

Now fill in the spec with real components, wiring, and constraints.

### 4a: Find existing packages

Search the atopile package registry before building from scratch.

Use `packages_search` → `packages_install` → `package_ato_read` to inspect public interface. Also check `stdlib_list` for built-in modules.

Prefer reusing a well-tested package over writing a new driver module.

### 4b: Create local packages when none exist

When `packages_search` returns no match for a needed IC, connector, or module, **create a local driver package** instead of giving up or asking the user to find one.

Step-by-step recipe:
1. **Find the part**: Use `parts_search` to find the LCSC component.
2. **Research the part family when needed**: Use `web_search` before locking the part if you need application notes, common reference circuits, or family comparisons.
3. **Install as a local package**: Use `parts_install` with the LCSC ID and `create_package=true`. This installs the raw part and generates the canonical reusable wrapper package under `packages/`.
4. **Inspect the vendor docs with web search**: Use `web_search` with the part number and terms like `datasheet`, `hardware design`, `application circuit`, `decoupling`, `pinout`.
5. **Read the generated files**: Inspect the generated wrapper under `packages/<PartName>/<PartName>.ato` and the installed raw part it imports to see available interfaces and exact pin names.
6. **Refine the wrapper package** if needed:
   - Treat `packages/<PartName>/<PartName>.ato` as the canonical wrapper module for that part.
   - Edit that generated package file in place rather than creating another wrapper layer.
   - Keep the raw installed part file unchanged.
   - Start with a basic reusable wrapper first.
   - Expose standard interfaces: `ElectricPower`, `I2C`, `SPI`, `UART`, `CAN`, `SWD`, `USB2_0_IF`, `ElectricLogic`, or `ElectricSignal`.
   - Before writing any custom `interface`, check `stdlib_list` / `stdlib_get_item` for an existing stdlib interface.
   - Prefer capability-oriented names: `uart`, `spi`, `adc_inputs`, `gpio`, `usb`, `swd`, `power` — not design-specific roles like `sbus`, `weapon_pwm`.
   - Map the internal `_package` component pins to those interfaces.
   - Add decoupling capacitors and required passives.
   - Set voltage/current constraints from the datasheet.
7. **Discover targets**: Run `workspace_list_targets` after package creation to inspect and build the package targets automatically.
8. **Import and use** the local package in your top-level design directly from `packages/<PartName>/<PartName>.ato`.

**Example: local I2C mux wrapper**

```ato
#pragma experiment("BRIDGE_CONNECT")

import ElectricPower
import ElectricLogic
import I2C
import Capacitor
import Resistor

from "parts/Texas_Instruments_TCA9548APWR/Texas_Instruments_TCA9548APWR.ato" import Texas_Instruments_TCA9548APWR_package

module TI_TCA9548A:
    # Public interfaces
    power = new ElectricPower
    assert power.voltage within 1.65V to 5.5V

    i2c = new I2C
    reset = new ElectricLogic

    # Instantiate the auto-generated package component
    package = new Texas_Instruments_TCA9548APWR_package

    # Power connections
    power.hv ~ package.VCC
    power.lv ~ package.GND

    # I2C — connect via .line and .reference
    i2c.sda.line ~ package.SDA
    i2c.scl.line ~ package.SCL
    i2c.sda.reference ~ power
    i2c.scl.reference ~ power

    # Decoupling — use bridge connect (~>) for series path
    decoup_100n = new Capacitor
    decoup_100n.capacitance = 100nF +/- 20%
    decoup_100n.package = "0402"
    power.hv ~> decoup_100n ~> power.lv

    # Reset with pullup
    reset.line ~ package.nRESET
    reset.reference ~ power
    reset_pullup = new Resistor
    reset_pullup.resistance = 10kohm +/- 1%
    reset_pullup.package = "0402"
    reset.line ~> reset_pullup ~> reset.reference.hv
```

**Key rules:**
- Always `parts_install` first — never reference a part that hasn't been installed.
- Always **read the generated package and raw part `.ato` files** to see the exact signal names. Do NOT guess pin names.
- Always use `web_search` to inspect the vendor datasheet for correct pin mapping, constraints, and recommended decoupling.
- The raw installed file is a `component` — never edit it.
- Connect interfaces via `.line` and `.reference` (e.g., `i2c.sda.line ~ package.SDA`; `i2c.sda.reference ~ power`).
- Use bridge connect `~>` for decoupling caps in series (e.g., `power.hv ~> cap ~> power.lv`).
- Use `.capacitance` for Capacitor values, `.resistance` for Resistor values (NOT `.value`).
- Do NOT skip this step and tell the user to create the package themselves.

### 4c: Part selection

Choose components using generics + constraints wherever possible.

- Use stdlib generics (`Resistor`, `Capacitor`, `Inductor`, `Diode`, `LED`, `Fuse`) with value + package constraints for auto-picking.
- Use `parts_search` only when a specific part is needed (IC, connector, specialized component).
- Use `web_search` before locking a part when you need to compare candidate families or find a solid reference circuit.
- Lock only high-risk parts (MCU, PMIC, RF, connectors). Leave commodity passives auto-picked.

### 4d: Detailed wiring and constraints

Wire connectivity, add constraints and equations, complete the design.

- Wire modules through interfaces using `~` (or `~>` for bridge/series paths).
- Add parameter constraints (`assert ... within ...`) for all key electrical properties.
- Add decoupling, pullups, and protection per datasheet recommendations.

## Step 5: Power Architecture

1. Review each package's required voltage and current.
2. Determine the power rails that need to be generated (typically ~3-5% tolerance is acceptable).
3. Determine the input power source (battery, USB connector, XT30, etc.) and install a relevant package.
4. Find suitable regulators:
   - If input voltage > required voltage and current is low: use an **LDO** package
   - If input voltage > required voltage and current is high: use a **buck converter**
   - If input voltage < required voltage: use a **boost converter**
   - If input voltage can be both less than or greater than required: use **buck-boost** (e.g. battery powered device that needs 3v3)
5. If battery powered, add a charger package.

### Typical Power Architecture Example (LDO)

USB input power with low current output:

```ato
from "atopile/ti-tlv75901/ti-tlv75901.ato" import TLV75901_driver
from "atopile/usb-connectors/usb-connectors.ato" import USBCConn

module App:
    power_5v = new ElectricPower
    power_3v3 = new ElectricPower

    ldo = new TLV75901_driver
    usb_connector = new USBCConn

    usb_connector.power ~ power_5v
    power_5v ~> ldo ~> power_3v3
```

## Step 6: Build

Run builds and fix issues iteratively until everything passes. **Build submodules first** — it is much easier to get small chunks working before running the full build.

### Build + fix loop

Use `workspace_list_targets` → `build_run` → `build_logs_search` (filter by `log_levels`/`stage`) → `design_diagnostics` for silent failures. Use `report_variables` to inspect constraint state and `report_bom` to verify part selection.

- Run `workspace_list_targets` first after creating/installing local packages so you know which package targets already exist automatically.
- Split the design into sensible submodules and build those smaller targets first.
- Build wrapper/package targets first, and do so in parallel where practical.
- Fix submodule/package failures before running the top-level design.
- Use the full top-level build after submodules are green; it should mainly be an integration check.
- Check `build_logs_search` for errors/warnings.
- Use `design_diagnostics` for silent failures.

## Step 7: Summary

When the build finishes, give the user a summary:
- **What was built** — list the modules, key components, and interfaces.
- **Blockers or issues** — note any problems encountered and how they were resolved.
- **Suggestions for next steps** — what the user might want to do next (e.g., review placement, order boards, add features, run DRC).

After making changes, use `build_run` to update the PCB. Builds will often generate errors/warnings — these should be reviewed and fixed. Prioritize packages from the `atopile` registry over other packages.
