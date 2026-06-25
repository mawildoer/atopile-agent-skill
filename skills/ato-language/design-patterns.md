# Atopile Design Patterns

Idiomatic patterns for writing `.ato` drivers and boards. Examples use real parts (e.g. the TI BQ25756 charger) to make the patterns concrete. Each pattern shows what it solves, the code, and the non-obvious reasoning.

---

## 1. Decoupling a Power Rail

**Problem:** Power rails need bulk + ceramic capacitors for stability.

```ato
# Bulk electrolytic/ceramic for energy storage
cin_bulk = new Capacitor
cin_bulk.capacitance = 10uF +/- 20%
cin_bulk.package = "1210"
power_adapter ~ cin_bulk.power       # .power connects hv/lv automatically

# Small ceramic for high-frequency filtering
cin_cer = new Capacitor
cin_cer.capacitance = 100nF +/- 20%
cin_cer.package = "0805"
power_adapter ~ cin_cer.power

# ALWAYS assert voltage rating separately from package
assert cin_bulk.max_voltage >= 100V
assert cin_cer.max_voltage >= 100V
```

**Non-obvious:** Use `cap.power ~ rail` (not `unnamed[]`) when decoupling a power rail -- the `.power` member is an `ElectricPower` that connects hv and lv in one statement. Use `unnamed[0]`/`unnamed[1]` only when wiring between arbitrary nets (bootstrap caps, AC coupling, etc.).

---

## 2. Array Instantiation + For Loop

**Problem:** Multiple identical components with the same constraints (decoupling caps, parallel FETs, LED strings, connector pin arrays).

```ato
#pragma experiment("FOR_LOOP")
#pragma experiment("BRIDGE_CONNECT")

power_3v3_caps = new Capacitor[4]
for cap in power_3v3_caps:
    cap.capacitance = 100nF +/- 20%
    cap.package = "0402"
    power_3v3.hv ~> cap ~> power_3v3.lv
```

**Non-obvious:** The `~>` bridge syntax through a Capacitor requires BRIDGE_CONNECT pragma. For loops require FOR_LOOP pragma. Both pragmas go at file top level. Inside the loop body you can set per-element parameters and connections.

---

## 3. Voltage Divider for Feedback/Sensing

**Problem:** IC feedback pins (FB, OVP, UVP) need precision resistor dividers.

```ato
fb_div = new ResistorVoltageDivider
power_battery ~ fb_div.ref_in            # Input voltage across the divider
fb_div.ref_in.lv ~ ic.FBG                # Ground reference for the divider
fb_div.chain.taps[0] ~ ic.FB             # Tap point connects to IC sense pin
fb_div.ratio = 0.028 +/- 5%              # V_out / V_in ratio
fb_div.total_resistance = 257kohm +/- 20%

# High-side resistor sees full input voltage -- assert its rating
assert fb_div.chain.resistors[0].max_voltage >= 75V
# Cross-check: ratio * max_voltage must reach the IC reference window
assert fb_div.ratio * 70V >= 1.566V
```

**Non-obvious:** `chain.resistors[0]` is the TOP resistor (high side, sees full voltage). `chain.resistors[1]` is the BOTTOM resistor (low side, sees only the tap voltage). `chain.taps[0]` is the midpoint. Always assert `max_voltage` on the top resistor separately -- `package = "0603"` alone does NOT guarantee voltage rating. Use a larger package (0603) for the high-voltage top leg and 0402 for the bottom leg.

---

## 4. Pull-ups on Open-Drain Signals

**Problem:** I2C, active-low chip enable, interrupts -- all need pull-ups to the correct logic rail.

```ato
#pragma experiment("BRIDGE_CONNECT")

# I2C pull-ups
ic.SCL ~ i2c.scl.line
ic.SDA ~ i2c.sda.line
i2c.scl.reference ~ power_3v3           # Sets the logic level
i2c.sda.reference ~ power_3v3

rscl = new Resistor
rscl.resistance = 4.7kohm +/- 5%
power_3v3.hv ~> rscl ~> ic.SCL          # Pull-up from rail to line

# Active-low chip enable with default-off (pull HIGH = disabled)
ic.nCE ~ nce.line
nce.reference ~ power_3v3
rnce = new Resistor
rnce.resistance = 100kohm +/- 5%
power_3v3.hv ~> rnce ~> ic.nCE
```

**Non-obvious:** The `ElectricLogic.reference` connection is CRITICAL and easy to forget. It tells the system what power domain the signal belongs to. Without it, the logic level is undefined and the build may silently produce wrong netlists. The bridge syntax `power.hv ~> resistor ~> pin` reads naturally as "pull up from rail through resistor to pin."

---

## 5. Kelvin Sense Taps (0-ohm NetTie Pattern)

**Problem:** Current-sense shunts need Kelvin connections -- the sense traces must be separate from the power traces right up to the shunt pads.

```ato
# The sense shunt in the power path
rac_sns = new Resistor
rac_sns.resistance = 5mohm +/- 1%
rac_sns.max_power = 1W to 5W

# Power path connection
power_adapter.hv ~ rac_sns.unnamed[0]

# Kelvin tap: 0-ohm jumper creates a distinct net for the sense trace
nt_acp = new Resistor
nt_acp.resistance = 0ohm to 50mohm
nt_acp.package = "0402"
rac_sns.unnamed[0] ~ nt_acp.unnamed[0]  # Shunt pad
nt_acp.unnamed[1] ~ ic.ACP              # IC sense input

# Same pattern on the other side
nt_acn = new Resistor
nt_acn.resistance = 0ohm to 50mohm
nt_acn.package = "0402"
rac_sns.unnamed[1] ~ nt_acn.unnamed[0]
nt_acn.unnamed[1] ~ ic.ACN
```

**Non-obvious:** The 0-ohm resistor acts as a "NetTie" -- it creates two distinct nets in the PCB layout so the router keeps the high-current power trace and the thin sense trace physically separate. The `0ohm to 50mohm` range lets the part picker select any zero-ohm jumper. Without NetTies, the sense and power traces merge into one net and the layout tool cannot enforce Kelvin routing.

---

## 6. Locking Components: Package + Rated Parameters

**Problem:** The part picker can change components between builds — different package, different voltage rating, different power rating. This causes unnecessary layout rework and can silently select parts that don't meet electrical requirements.

**Solution:** Lock down both the package (for layout stability) AND the rated parameters (for electrical correctness). Ideally, derive the rated parameter assertions from the module's design parameters so they stay correct when the design is reconfigured.

```ato
# A module with a configurable design point
v_max = 48V

# --- Voltage divider top-leg resistor ---
# Package lock: prevents layout churn between builds
acov_div.chain.resistors[0].package = "0603"

# Rated param assertion: DERIVED from v_max, not a magic number.
# The top leg sees the full input voltage, so it must be rated above it.
assert acov_div.chain.resistors[0].max_voltage >= v_max + 25V  # margin for transients

# --- Sense shunt in the power path ---
shunt = new Resistor
shunt.resistance = 5mohm +/- 1%
shunt.package = "2512"                    # layout lock

# Rated params derived from design-point current (15A):
# P = I^2 * R * derating = 15^2 * 5m * 2 = 2.25W
shunt.max_power = 1W to 5W               # range for picker flexibility
assert shunt.max_power >= 2.25W           # derived minimum

# --- Bulk capacitor on a power rail ---
cin_bulk = new Capacitor
cin_bulk.capacitance = 10uF +/- 20%
cin_bulk.package = "1210"                 # layout lock

# Voltage rating derived from the rail it decouples:
# IC abs max is 70V, ceramic derating ~50% at rated → need ≥100V part
assert cin_bulk.max_voltage >= v_max * 2  # covers derating + transients
```

**Non-obvious:** There are two distinct concerns that both need locking:

1. **Package lock** (`.package = "XXXX"`) — prevents the picker from switching footprints between builds. Without this, a rebuild might pick a different-sized part and invalidate your PCB layout. Lock every component that has a placed footprint.

2. **Rated parameter assertions** (`assert max_voltage >= ...`, `assert max_power >= ...`) — ensures the picked part meets electrical requirements. The package alone says nothing about ratings — a 0402 resistor might be rated 50V or 200V.

The key insight: **derive rated assertions from the module's design parameters** (like `v_max`, current targets, etc.) rather than hardcoding magic numbers. This way, when someone changes `v_max = 48V` to `v_max = 60V`, the solver immediately flags any component whose rating can't handle the new operating point. The assertions become self-updating design checks, not stale comments.

---

## 7. Bridge Syntax for Series Components

**Problem:** Connecting components in series with `unnamed[]` is verbose and error-prone.

```ato
#pragma experiment("BRIDGE_CONNECT")

# WITHOUT bridge syntax (verbose):
rdrv = new Resistor
rdrv.resistance = 2.2ohm +/- 5%
ic.REGN ~ rdrv.unnamed[0]
rdrv.unnamed[1] ~ ic.DRV_SUP

# WITH bridge syntax (clear intent):
ic.REGN ~> rdrv ~> ic.DRV_SUP

# Chaining multiple: gate drive through series resistor to FET gate
ic.HIDRV1 ~> rg_hs1 ~> q12.G1
```

**Non-obvious:** Bridge syntax only works with modules that have the `can_bridge` trait (Resistor, Capacitor, Inductor have it by default). It connects `unnamed[0]` to the left side and `unnamed[1]` to the right. You can chain multiple bridges: `a ~> r1 ~> r2 ~> b`. The direction arrow `~>` vs `<~` matters for directed interfaces but for passives either direction works. Always enable the pragma at file top level.

---

## 8. Encoding Datasheet Formulas as Assertions

**Problem:** IC datasheets define relationships between component values and operating parameters (current limits, switching frequency, voltage thresholds). These formulas need to be captured in the design so they're verified automatically and can be reconfigured by changing a single parameter.

```ato
# BQ25756 ICHG programming: I_BAT_MAX = K_ICHG / R_ICHG
# where K_ICHG = 48 A*kOhm = 48000 V (datasheet Table 8-5)
richg = new Resistor
richg.resistance = 3.0kohm +/- 1%
ic.ICHG ~ richg.unnamed[0]
richg.unnamed[1] ~ power_adapter.lv

# Cross-check: hardware ceiling covers the design point
assert 48kV / richg.resistance >= 15A   # must reach i_battery target
assert 48kV / richg.resistance <= 20A   # must stay below IC absolute max

# FB divider: ratio sets CV regulation ceiling
# VFB_NOM = 1.536V (datasheet), so V_BAT_REG = VFB / ratio
fb_div = new ResistorVoltageDivider
fb_div.ratio = 0.028 +/- 5%
assert fb_div.ratio * v_max <= 1.40V    # CV ceiling below rated max
assert fb_div.ratio * 70V >= 1.566V     # OVP can trip in operating range
```

**Non-obvious:** Every datasheet formula should become an `assert` statement. This serves two purposes: (1) if you change a design parameter (e.g. raise `v_max` from 48V to 60V), the solver immediately flags any component that can't handle the new operating point, and (2) it documents the design intent — future readers can trace why a resistor value was chosen by reading the assertion, not hunting through the datasheet. Use the datasheet's own K-factor constants directly in the assertion (e.g. `48kV` for K_ICHG) so the formula is recognizable to anyone reading the datasheet alongside the code.

---

## 9. Datasheet-Driven Design Workflow

**Problem:** Component drivers need to faithfully implement the datasheet's typical application circuit, but datasheets are dense and easy to misread.

**Workflow:**

1. **Download the datasheet and app note FIRST** — before writing any code. Save to a local `datasheets/` directory so they're available to all agents and future sessions:

   ```bash
   # Find the PDF on the manufacturer's website (prefer manufacturer over distributors)
   # TI: ti.com, Analog Devices: analog.com, ST: st.com, Microchip: microchip.com, etc.
   curl -L -o datasheets/ti_bq25756_datasheet.pdf "https://www.ti.com/lit/ds/symlink/bq25756.pdf"
   curl -L -o datasheets/ti_bq25756_appnote.pdf "https://www.ti.com/lit/an/..."
   ```

   Naming convention: `<manufacturer>_<part>_datasheet.pdf`, `<manufacturer>_<part>_appnote.pdf`.
   Bias towards the manufacturer's website — distributor copies may be outdated.

2. **Read the datasheet locally** — all references should use the local copy:
   ```
   Read datasheets/ti_bq25756_datasheet.pdf
   ```

3. **Find the typical application circuit** — this is your schematic template. Every component in it should appear in your driver module.

4. **Find the application note** — many ICs have a separate app note with detailed design procedures, worked examples, and component selection tables. These are more useful than the datasheet for sizing components.

5. **Encode every formula** — for each component the datasheet says to calculate (programming resistors, feedback dividers, compensation networks, sense resistors), write the formula as an `assert` using the datasheet's constants.

6. **Cross-check operating limits** — assert that every component's voltage/current/power rating covers the operating envelope with margin.

**Non-obvious:** Download datasheets at the START of the design process, not when you first need to look something up. Having them local means review agents can read them without internet access, and the reference persists across sessions. The application note is often more valuable than the datasheet itself — datasheets give specs; app notes give step-by-step component selection procedures with worked examples. Always search for `<part number> application note` or `<part number> design guide` on the manufacturer's site.

---

## 10. Submodule Decomposition

**Problem:** Large driver modules become hard to read and reason about. A 500-line module mixing adapter port circuitry, battery port circuitry, gate drive, and I2C control is harder to review than four focused submodules.

**When to extract a submodule:**
- The group of components has a clear function name ("adapter sense path", "gate drive network", "TS bias divider")
- It has a meaningful interface (power in, sense output, control signals)
- The parent module gets noticeably easier to read
- The submodule could plausibly be reused or independently verified

**When NOT to extract:**
- Two components with no meaningful abstraction ("just a cap and resistor")
- The interface would be more complex than the implementation
- It's only done for line-count reduction, not clarity

```ato
# GOOD: clear function, meaningful interface, parent reads better
module AdapterSensePath:
    """Kelvin-sensed shunt with NetTies for current measurement."""
    power_in = new ElectricPower     # adapter-side power
    power_out = new ElectricPower    # IC-side power (through shunt)
    sense_p = new Electrical         # to IC ACP pin
    sense_n = new Electrical         # to IC ACN pin

    shunt = new Resistor
    shunt.resistance = 5mohm +/- 1%
    shunt.max_power = 1W to 5W

    nt_p = new Resistor              # Kelvin NetTie
    nt_p.resistance = 0ohm to 50mohm
    nt_n = new Resistor
    nt_n.resistance = 0ohm to 50mohm

    power_in.hv ~ shunt.unnamed[0]
    shunt.unnamed[1] ~ power_out.hv
    power_in.lv ~ power_out.lv

    shunt.unnamed[0] ~ nt_p.unnamed[0]
    nt_p.unnamed[1] ~ sense_p
    shunt.unnamed[1] ~ nt_n.unnamed[0]
    nt_n.unnamed[1] ~ sense_n

# Parent module is now clean:
module BQ25756Charger:
    adapter_sense = new AdapterSensePath
    power_adapter ~ adapter_sense.power_in
    adapter_sense.sense_p ~ ic.ACP
    adapter_sense.sense_n ~ ic.ACN
```

**Non-obvious:** The goal is not minimal line count — it's making the parent module read like a block diagram. Each submodule should correspond to a recognizable functional block from the datasheet. If you can point to a section of the datasheet and say "this submodule implements that section," the decomposition is right. Keep submodules in the same file unless they're reused across packages.

---

## 11. Configuration Pins via Resistors (Not Direct Ties)

**Problem:** Tying IC configuration pins (address, mode select, enable) directly to VCC or GND is permanent. If the design is wrong — wrong I2C address, wrong mode, wrong default state — fixing it requires a board respin.

```ato
#pragma experiment("BRIDGE_CONNECT")

# BAD: direct tie — can't change without rework
ic.ADDR0 ~ power_3v3.lv   # address bit 0 = 0

# GOOD: 0-ohm jumper — swap to pull-up to change address
_r_addr0 = new Resistor
_r_addr0.resistance = 0ohm to 50mohm
_r_addr0.package = "0402"
power_3v3.lv ~> _r_addr0 ~> ic.ADDR0

# ALSO GOOD: pull-down resistor — remove or replace to change
_r_addr0 = new Resistor
_r_addr0.resistance = 10kohm +/- 5%
_r_addr0.package = "0402"
power_3v3.lv ~> _r_addr0 ~> ic.ADDR0
```

**Non-obvious:** During early development, the flexibility of a single 0402 part swap far outweighs the cost of the extra resistor. Once the configuration is proven across multiple board revisions, these can be optimized out and replaced with direct copper ties. But for rev 1, always use resistors — every config pin that's tied wrong is a potential board respin.

---

## 12. Reset Pin Handling

**Problem:** Reset pins with wrong polarity or missing decoupling cause intermittent failures that are extremely hard to debug — the IC works sometimes, fails randomly, or doesn't start reliably.

```ato
#pragma experiment("BRIDGE_CONNECT")

# Active-low reset (most common): pull-up + decoupling cap
_r_reset = new Resistor
_r_reset.resistance = 10kohm +/- 5%
_r_reset.package = "0402"
power_3v3.hv ~> _r_reset ~> ic.nRESET

_c_reset = new Capacitor
_c_reset.capacitance = 100nF +/- 20%
_c_reset.package = "0402"
ic.nRESET ~ _c_reset.unnamed[0]
_c_reset.unnamed[1] ~ power_3v3.lv

# Connect to the reset ElectricLogic interface for host control
reset.line ~ ic.nRESET
reset.reference ~ power_3v3
```

**Non-obvious:** The RC time constant (R_pull * C_decouple) sets the minimum reset pulse width and the power-on reset delay. 10kohm * 100nF = 1ms, which is sufficient for most ICs. Check the datasheet for minimum reset pulse width — if it's longer than 1ms, increase the cap. The pull-up ensures the IC runs by default; the host can pull low to reset. Always verify the polarity: `nRESET` / `RESET_N` / `RST#` = active-low (pull HIGH for normal operation); `RESET` / `RST` = active-high (pull LOW for normal operation).

---

## 13. I2C Address Configuration

**Problem:** Multiple I2C devices on the same bus need unique addresses. Most ICs have hardware address pins — getting the wiring wrong gives silent bus collisions.

```ato
#pragma experiment("MODULE_TEMPLATING")
#pragma experiment("FOR_LOOP")

import Addressor

module MyI2CDevice_driver:
    i2c = new I2C
    power = new ElectricPower

    addressor = new Addressor<address_bits=2>
    assert addressor.address is i2c.address
    addressor.base = 0x48

    for line in addressor.address_lines:
        line.reference ~ power

    addressor.address_lines[0].line ~ ic.A0
    addressor.address_lines[1].line ~ ic.A1
```

Consumer constrains the address:

```ato
sensor = new MyI2CDevice_driver
assert sensor.i2c.address within 0x4A
```

For multi-state address pins (e.g. TI's GND/VS/SDA/SCL scheme), use `Addressor<address_bits=2, states_per_pin=4>` and connect `state_options[0..3]` to the corresponding nets.

**Non-obvious:**
- `assert addressor.address is i2c.address` uses `is` for bidirectional param equality — this is NOT the deprecated `assert x is <literal>` form. At the consumer level, use `within` for literals: `assert dev.i2c.address within 0x4A`.
- **Duplicate address detection is not yet implemented** in the solver — builds succeed silently. This is a review-time check only.
- Fixed-address ICs (no address pins): set `i2c.address.default = 0x3C` directly, no Addressor needed.
- `address_lines[].reference` MUST be connected — it's the power rail the Addressor ties pins high/low against.
