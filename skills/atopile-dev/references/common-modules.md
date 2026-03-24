# Standard Library Modules & Interfaces

The most commonly used interfaces and modules in atopile's standard library.

## Interfaces (connectable with `~` or `~>`)

| Type | Children / Parameters | Purpose |
| ---- | --------------------- | ------- |
| `Electrical` | _(single node)_ | Raw electrical connection point |
| `ElectricPower` | `.hv`, `.lv` (Electrical); `.voltage`, `.max_current` | Power rails |
| `ElectricLogic` | `.line` (Electrical), `.reference` (ElectricPower) | Digital signals with voltage context |
| `ElectricSignal` | `.line` (Electrical), `.reference` (ElectricPower) | Analog signals |
| `I2C` | `.scl`, `.sda` (ElectricLogic); `.frequency`, `.address` | I2C bus |
| `SPI` | `.sclk`, `.mosi`, `.miso` (ElectricLogic); `.frequency` | SPI bus |
| `UART` / `UART_Base` | `.tx`, `.rx` (ElectricLogic); flow control lines | Serial |
| `I2S` | audio data bus lines | Digital audio |
| `DifferentialPair` | `.p`, `.n` | Differential signals |
| `USB2_0` / `USB2_0_IF` | USB data + power | USB interfaces |
| `CAN_TTL` | CAN bus lines | CAN bus |
| `SWD` / `JTAG` | debug lines | Debug interfaces |
| `Ethernet` / `HDMI` / `RS232` / `PDM` / `XtalIF` | protocol-specific | Other protocols |

## Modules (instantiable with `new`)

| Type | Children / Parameters | Designator |
| ---- | --------------------- | ---------- |
| `Resistor` | `.unnamed[0..1]`; `.resistance`, `.max_power` | R |
| `Capacitor` | `.unnamed[0..1]`, `.power`; `.capacitance`, `.max_voltage`, `.temperature_coefficient` | C |
| `CapacitorPolarized` | polarized variant of Capacitor | C |
| `Inductor` | `.unnamed[0..1]`; `.inductance` | L |
| `Fuse` | `.unnamed[0..1]`; `.trip_current`, `.fuse_type` | F |
| `Diode` | `.anode`, `.cathode`; `.forward_voltage`, `.current` | D |
| `LED` | `.diode`; `.brightness`, `.color` | D |
| `MOSFET` | `.source`, `.gate`, `.drain`; `.channel_type`, `.gate_source_threshold_voltage` | Q |
| `BJT` | `.emitter`, `.base`, `.collector`; `.doping_type` | Q |
| `Regulator` / `AdjustableRegulator` | `.power_in`, `.power_out` | — |
| `Crystal` | `.unnamed[0..1]`, `.gnd`; `.frequency`, `.load_capacitance` | XTAL |
| `Crystal_Oscillator` | oscillator module | — |
| `ResistorVoltageDivider` | voltage divider circuit | — |
| `FilterElectricalRC` | RC filter | — |
| `Net` | `.part_of` (Electrical) | — |
| `TestPoint` | `.contact`; `.pad_size`, `.pad_type` | TP |
| `MountingHole` / `NetTie` | mechanical | — |
| `SPIFlash` | SPI flash memory | — |

## Usage Notes

- **Electrical** — The fundamental electrical connection point. Used for basic pin-to-pin connections.
- **ElectricPower** — A power rail with `hv` (high voltage) and `lv` (low voltage / ground) connections.
- **ElectricLogic** — A logic-level signal with a `line` (the signal) and `reference` (the power domain it operates in). Always set `reference ~ power_rail`.
- **Resistor** — Auto-picked from JLCPCB based on parameter constraints. Connect via `unnamed[0]` and `unnamed[1]`. Always use `+/- N%` tolerance.
- **Capacitor** — Auto-picked from JLCPCB based on parameter constraints. Connect via `unnamed[0]` and `unnamed[1]`. Always use `+/- N%` tolerance.
- **I2C** — Standard I2C bus interface with clock, data, frequency, and address parameters.
- **ResistorVoltageDivider** — Voltage divider with `output` signal for ADC sensing.

## Source Code Snippets

### Electrical

```ato
interface Electrical:
    pass
```

### ElectricPower

```ato
interface ElectricPower:
    hv = new Electrical
    lv = new Electrical
```

### Resistor

```ato
module Resistor:
    resistance: ohm
    max_power: W
    max_voltage: V
    unnamed = new Electrical[2]
```

### Capacitor

```ato
module Capacitor:
    capacitance: F
    max_voltage: V
    unnamed = new Electrical[2]
```

### I2C

```ato
interface I2C:
    scl = new ElectricLogic
    sda = new ElectricLogic
    frequency: Hz
    address: dimensionless
```

### ElectricLogic

```ato
interface ElectricLogic:
    line = new Electrical
    reference = new ElectricPower
```

## Traits (attachable with `trait`)

`has_part_removed`, `is_atomic_part`, `can_bridge`, `can_bridge_by_name`, `has_datasheet`, `has_designator_prefix`, `has_doc_string`, `has_net_name_affix`, `has_net_name_suggestion`, `has_package_requirements`, `has_single_electric_reference`, `is_auto_generated`, `requires_external_usage`

For the full list of available interfaces and modules, use the MCP tools:
- `stdlib_list` — list all interfaces and modules
- `stdlib_get_item` — view source code of any module/interface
