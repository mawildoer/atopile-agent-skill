# Electronics Design Workflow for AI Agents

## #1 Rule

**USE THE TOOLS.** If the tools don't work, don't freak out — you are probably using them wrong. Ask for help if you get stuck.

## Top-Level Design

1. Research available packages relevant to the user requests using `find_packages`
2. Inspect promising packages using `inspect_package`
3. Propose packages to use for project and architecture to user, revise if needed
4. Install needed packages using `install_package`
5. Import packages into main file
6. Create instances of packages in main module

## Power

1. Review for each package the required voltage and current (current may not be provided, use judgment if necessary)
2. Determine the power rails that need to be generated and a suitable tolerance (typically ~3-5% is acceptable)
3. Determine the input power source, typically a battery, USB connector or other power connector (e.g. XT30) and install relevant package
4. Find suitable regulators:
   - If input voltage > required voltage and current is low, use an **LDO** package
   - If input voltage > required voltage and current is high, use a **buck converter**
   - If input voltage < required voltage, use a **boost converter**
   - If input voltage can be both less than or greater than required voltage, use **buck-boost** (e.g. battery powered device that needs 3v3)
5. If battery powered, add charger package

### Typical Power Architecture Example (LDO)

USB input power with low current output (e.g. microcontroller):

```ato
from "atopile/ti-tlv75901/ti-tlv75901.ato" import TLV75901_driver
from "atopile/usb-connectors/usb-connectors.ato" import USBCConn

module App:

    # Rails
    power_5v = new Power
    power_3v3 = new Power

    # Components
    ldo = new TLV75901_driver
    usb_connector = new USBCConn

    # Connections
    usb_connector.power ~ power_vbus
    power_vbus ~> ldo ~> power_3v3
```

## Communications

1. Review packages' required interfaces, typically I2C, SPI or ElectricLogics
2. Find suitable pins on the controller, typically a microcontroller or Linux SoC
3. Connect interfaces, e.g. `micro.i2c[0] ~ sensor.i2c`

## Development Process Notes

- After making changes, be sure to use `build_project` to update the PCB
- Builds will often generate errors/warnings — these should be reviewed and fixed
- Prioritize packages from `atopile` over other packages
