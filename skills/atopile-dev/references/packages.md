# Package Creation Guide

Step-by-step guide for creating atopile packages.

## Process

Review structure of other packages.

### 1. Create Directory

Create new directory in `packages/packages` with naming convention `<vendor>-<device>` e.g. `adi-adau145x`.

### 2. Create ato.yaml

```yaml
requires-atopile: '^0.9.0'

paths:
    src: '.'
    layout: ./layouts

builds:
    default:
        entry: <device>.ato:<device>_driver
    example:
        entry: <device>.ato:Example
```

### 3. Create Part

Create part using tool call `search_and_install_jlcpcb_part`.

### 4. Import Part

Import the part into the `<device>.ato` file.

### 5. Read Datasheet

Read the datasheet for the device.

### 6. Find Common Interfaces

Find common interfaces in the part e.g. I2C, I2S, SPI, Power.

### 7. Create Interfaces and Connect Them

**Power interfaces:**

```ato
power_<name> = new ElectricPower
power_<name>.required = True  # If critical to the device
assert power_<name>.voltage within <min_voltage>V to <max_voltage>V
power_<name>.hv ~ <device>.<vcc_pin>
power_<name>.lv ~ <device>.<gnd_pin>
```

**I2C interfaces:**

```ato
i2c = new I2C
i2c.scl.line ~ <device>.<i2c_scl_pin>
i2c.sda.line ~ <device>.<i2c_sda_pin>
```

**SPI interfaces:**

```ato
spi = new SPI
spi.sclk.line ~ <device>.<spi_sclk_pin>
spi.mosi.line ~ <device>.<spi_mosi_pin>
spi.miso.line ~ <device>.<spi_miso_pin>
```

### 8. Add Decoupling Capacitors

Looking at the datasheet, determine the required decoupling capacitors.

Example: 2x 100nF 0402:

```ato
power_3v3 = new ElectricPower

# Decoupling power_3v3
power_3v3_caps = new Capacitor[2]
for capacitor in power_3v3_caps:
    capacitor.capacitance = 100nF +/- 20%
    capacitor.package = "0402"
    power_3v3.hv ~> capacitor ~> power_3v3.lv
```

### 9. Pin-Configurable I2C Addresses

If the device has pin-configurable I2C addresses with format `<n fixed bits><m configurable bits>`:

- Use `Addressor<address_bits=N>` where **N = number of address pins**.
- Connect each `address_lines[i].line` to the corresponding pin, and its `.reference` to a local power rail.
- Set `addressor.base` to the lowest possible address and `assert addressor.address is i2c.address`.

### 10. Create README.md

```markdown
# <Manufacturer> <Part Number> <Short Description>

## Usage

\```ato
<copy in example>
\```

## Contributing

Contributions to this package are welcome via pull requests on the GitHub repository.

## License

This atopile package is provided under the [MIT License](https://opensource.org/license/mit/).
```

### 11. Connect High-Level Interfaces in Example

```ato
i2c = new I2C
power = new ElectricPower
sensor = new Sensor

i2c ~ sensor.i2c
power ~ sensor.power_3v3
```

## Additional Notes & Gotchas

### Multi-rail devices (VDD / VDDIO, AVDD / DVDD, etc.)

- Model separate `ElectricPower` interfaces for each rail (e.g. `power_core`, `power_io`).
- Mark each `.required = True` if the device cannot function without it, and add voltage assertions per datasheet.

### Optional interfaces (SPI vs I2C)

- If the device supports multiple buses, pick one for the initial driver. Leave unused bus pins as `ElectricLogic` lines or expose a second interface module later.

### Decoupling guidance

- If the datasheet shows multiple caps, model the **minimum required** set so the build passes; you can refine values/packages later.

### File / directory layout

- `<vendor>-<device>/` — package root
- `ato.yaml` — build manifest (include `default` **and** `example` targets)
- `<device>.ato` — driver + optional example module
- `parts/<MANUFACTURER_PARTNO>/` — atomic part + footprint/symbol/step files

These tips should prevent common "footprint not found", "pin X missing", and build-time path errors when you add new devices.
