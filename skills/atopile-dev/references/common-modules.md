# Standard Library Modules & Interfaces

The most commonly used interfaces and modules in atopile's standard library.

```ato
interface Electrical:
    pass

interface ElectricPower:
    hv = new Electrical
    lv = new Electrical

module Resistor:
    resistance: ohm
    max_power: W
    max_voltage: V
    unnamed = new Electrical[2]

module Capacitor:
    capacitance: F
    max_voltage: V
    unnamed = new Electrical[2]

interface I2C:
    scl = new ElectricLogic
    sda = new ElectricLogic
    frequency: Hz
    address: dimensionless

interface ElectricLogic:
    line = new Electrical
    reference = new ElectricPower
```

## Usage Notes

- **Electrical** — The fundamental electrical connection point. Used for basic pin-to-pin connections.
- **ElectricPower** — A power rail with `hv` (high voltage) and `lv` (low voltage / ground) connections.
- **Resistor** — Auto-picked from JLCPCB based on parameter constraints. Connect via `unnamed[0]` and `unnamed[1]`.
- **Capacitor** — Auto-picked from JLCPCB based on parameter constraints. Connect via `unnamed[0]` and `unnamed[1]`.
- **I2C** — Standard I2C bus interface with clock, data, frequency, and address parameters.
- **ElectricLogic** — A logic-level signal with a `line` (the signal) and `reference` (the power domain it operates in).

For the full list of available interfaces and modules, use the MCP tools:
- `get_library_interfaces` — list all interfaces
- `get_library_modules` — list all modules
- `inspect_library_module_or_interface` — view source code of any module/interface
