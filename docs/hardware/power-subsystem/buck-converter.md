# TPS62130RGTR Buck Converter (U3)

## Component Overview

**Part Number:** TPS62130RGTR
**Manufacturer:** Texas Instruments
**Package:** RGT16 (VQFN-16, 3.0mm × 3.0mm)
**Function:** 3A Synchronous Step-Down Converter
**Schematic Reference:** U3
**Datasheet:** https://www.ti.com/lit/gpn/tps62130

---

## Purpose

The TPS62130RGTR converts the 5V_SYS rail (from power MUX) to a regulated 3.3V_DIG rail that powers the STM32H743 microcontroller and all digital peripherals. It provides high efficiency (75-85%), low ripple, and adequate current capability for the flight controller's core systems.

**Key Features:**
- Input voltage range: 3.0-17.0V (nominal 5.0V for this application)
- Output current: Up to 3A continuous (2A typical for first iteration)
- Synchronous rectification: High efficiency, minimal heat
- Fixed switching frequency: ~1MHz (configurable)
- Integrated power MOSFETs: Minimal external components
- Power Good output: System monitoring capability
- Soft-start: Controlled inrush current

---

## Functional Description

### Buck Converter Operation

A buck (step-down) converter uses **switching** and an **inductor** to efficiently reduce voltage:

**Basic Principle:**
1. **High-side FET ON:** Current flows through inductor, storing energy
2. **High-side FET OFF:** Inductor releases energy, maintaining current flow
3. **Low-side FET ON (synchronous):** Provides low-resistance return path
4. **Cycle repeats** at switching frequency (~1MHz)

**Synchronous Rectification:**
- Traditional buck: Uses Schottky diode for low-side (0.3-0.5V drop, inefficient)
- TPS62130: Uses N-channel MOSFET for low-side (~0.05-0.1V drop, efficient)
- **Benefit:** 75-85% efficiency vs. 60-70% for non-synchronous

### Efficiency vs. Linear Regulator

**Linear Regulator (LDO):**
```
Efficiency = V_OUT / V_IN = 3.3V / 5.0V = 66%
Power dissipation @ 1A: (5.0V - 3.3V) × 1A = 1.7W (lost as heat)
```

**Buck Converter (TPS62130):**
```
Efficiency = ~80% (typical at 1A load)
Power dissipation @ 1A: (1 - 0.80) × (3.3V × 1A) / 0.80 = ~0.8W
```

**Result:** Buck converter wastes **half the heat** of a linear regulator, critical for compact flight controller designs.

---

## Pin Configuration

### Pin Assignments

| Pin | Name | Function | Connection |
|-----|------|----------|------------|
| 1-3 | SW | Switch node outputs | To inductor L1 (all tied together) |
| 4 | PG | Power Good output | Available for monitoring |
| 5 | FB | Feedback input | Resistor divider (R8, R9) |
| 6 | AGND | Analog ground | Control circuit ground |
| 7 | FSW | Switching frequency | Configuration resistor |
| 8 | DEF | Default mode | Configuration |
| 9 | SS/TR | Soft-start/Tracking | C18 (100nF) for soft-start time |
| 10 | AVIN | Auxiliary input | 5V_SYS for control circuits |
| 11-12 | PVIN | Primary power input | 5V_SYS (dual pins for current sharing) |
| 13 | EN | Enable input | High = enabled (default) |
| 14 | VOS | Output voltage sense | Connected to 3V3_DIG output |
| 15-16 | PGND | Power ground | Switching current return (dual pins) |
| 17 | EPAD | Exposed pad | Thermal connection to PCB ground |

### Detailed Pin Functions

#### Pin 1-3: SW (Switch Node)
- **Function:** High-voltage switching node, connects to inductor
- **Why 3 pins?:** Current sharing and thermal distribution
- **Connection:** All three tied together, connected to inductor L1
- **Voltage:** Switches between 0V (low-side FET) and 5V (high-side FET)
- **Frequency:** ~1MHz switching (creates AC waveform)
- **Caution:** High dV/dt (fast voltage transitions) - minimize PCB trace length, keep away from sensitive signals

#### Pin 4: PG (Power Good)
- **Function:** Open-drain output indicating regulation status
- **Logic:**
  - LOW (pulled to ground): Output voltage out of regulation
  - HIGH-Z (open): Output voltage within ±8% of target
- **Connection:** Can be pulled up to 3.3V via resistor for MCU monitoring
- **Use Cases:**
  - MCU reset control (don't start MCU until power stable)
  - System health monitoring
  - Power sequencing (enable downstream regulators)
- **Current Status:** Not used in first iteration (available for future)

#### Pin 5: FB (Feedback)
- **Function:** Regulation feedback input for closed-loop control
- **Connection:** Resistor divider from 3V3_DIG output
- **Target:** 0.8V reference (internal to TPS62130)
- **Formula:** `V_OUT = 0.8V × (1 + R8/R9)`
- **For 3.3V output:**
  ```
  3.3V = 0.8V × (1 + R8/R9)
  R8/R9 = (3.3V / 0.8V) - 1 = 3.125
  Example: R8 = 7.5k©, R9 = 2.4k© ’ Ratio = 3.125 
  ```
- **Compensation:** C18 (100nF) provides loop stability

#### Pin 6: AGND (Analog Ground)
- **Function:** Ground reference for control circuits
- **Connection:** Connect to system ground plane at star point
- **Separation:** Keep separate from PGND (high-current switching ground) at chip level
- **Recommendation:** Connect AGND and PGND together at single point near IC or on ground plane

#### Pin 7: FSW (Switching Frequency)
- **Function:** Sets switching frequency
- **Connection:** Resistor to ground or specific configuration
- **Default:** ~1MHz (provides balance between size and efficiency)
- **Trade-offs:**
  - Higher frequency (>1MHz): Smaller inductor, higher switching losses
  - Lower frequency (<500kHz): Larger inductor, better efficiency
- **Current Design:** Configured for ~1MHz (typical)

#### Pin 8: DEF (Default Mode)
- **Function:** Configures operating mode
- **Modes:**
  - Power Save Mode (PSM): High efficiency at light loads
  - Forced PWM Mode: Constant frequency, predictable noise spectrum
- **Connection:** Configuration resistor or tied to ground/AVIN
- **Trade-off:** PSM improves light-load efficiency but may introduce frequency variation

#### Pin 9: SS/TR (Soft-Start / Tracking)
- **Function:** Controls startup ramp rate and output tracking
- **Connection:** C18 (100nF) capacitor to ground
- **Soft-Start Time:** T_ss H C_ss × 1.2V / 2µA
  ```
  T_ss = 100nF × 1.2V / 2µA = 60ms (approximate)
  ```
- **Purpose:** Limits inrush current during startup, prevents voltage overshoot
- **Tracking:** Can be used to follow another rail's ramp (not used in this design)

#### Pin 10: AVIN (Auxiliary Input)
- **Function:** Powers internal control circuits
- **Connection:** 5V_SYS rail (same as PVIN)
- **Current:** ~1-2mA for control logic
- **Decoupling:** Shares input capacitors (C24)
- **Note:** Can be powered from different source than PVIN in special applications

#### Pin 11-12: PVIN (Primary Power Input)
- **Function:** Main power input for buck converter
- **Connection:** 5V_SYS rail (both pins tied together)
- **Dual Pins:** Current sharing and thermal distribution
- **Voltage:** 5.0V nominal (4.75-5.25V range)
- **Max Current:** Input current = (V_OUT × I_OUT) / (V_IN × ·)
  ```
  Example: (3.3V × 2A) / (5V × 0.80) = 1.65A input current
  ```
- **Decoupling:** C24 (100nF) very close to pins

#### Pin 13: EN (Enable)
- **Function:** Enables/disables converter operation
- **Logic:**
  - HIGH (>1.2V): Converter enabled
  - LOW (<0.4V): Converter disabled, output off
- **Connection:** Tied high (enabled) or via RC delay for sequencing
- **Current Design:** Always enabled (or pulled high)
- **Future Use:** Could be connected to MCU GPIO for software power control

#### Pin 14: VOS (Output Voltage Sense)
- **Function:** Remote sense for output voltage regulation
- **Connection:** Connected directly to 3V3_DIG output (at load)
- **Purpose:** Compensates for trace resistance, improves regulation at load
- **Kelvin Connection:** Separate from power output trace (sense trace)

#### Pin 15-16: PGND (Power Ground)
- **Function:** High-current switching return path
- **Connection:** System ground plane (dual pins tied together)
- **Current:** Full inductor current flows through PGND
- **PCB Design:** Requires wide traces, solid ground plane, thermal vias
- **Noise:** High-frequency switching currents - keep away from analog circuits

#### Pin 17: EPAD (Exposed Pad)
- **Function:** Thermal and electrical ground connection
- **Connection:** Soldered to PCB ground plane with thermal vias
- **Thermal Resistance:** Critical for heat dissipation (0.5-1W typical)
- **PCB Requirements:**
  - 2oz copper or heavier
  - Multiple thermal vias (9-16 vias recommended)
  - Solid ground plane pour underneath
  - Via diameter: 10-12 mils, pad clearance minimal

---

## Support Components

### Inductor (L1)

**Specification:**
- **Value:** 2.2µH
- **Current Rating:** >2.5A (saturation current)
- **DC Resistance (DCR):** <50m© (for efficiency)
- **Type:** Shielded or semi-shielded (reduces EMI)
- **Package:** Power inductor (e.g., Bourns SRP series, Coilcraft XAL series)

**Selection Criteria:**
```
L = (V_IN - V_OUT) × V_OUT / (V_IN × ”I_L × f_SW)

Where:
- V_IN = 5.0V
- V_OUT = 3.3V
- ”I_L = 30% of I_OUT (ripple current)
- f_SW = 1MHz

Example:
L = (5.0 - 3.3) × 3.3 / (5.0 × 0.6A × 1MHz) = 1.87µH
’ Choose 2.2µH (standard value, provides margin)
```

**Function:**
- Stores energy during high-side FET on-time
- Releases energy during low-side FET on-time
- Smooths output current (reduces ripple)

**PCB Considerations:**
- Short, wide traces to/from inductor (minimize resistance)
- Place close to SW pins (minimize switching node length)
- Keep away from sensitive analog circuits (magnetic field)

### Input Capacitors

**C24 (100nF) - High-Frequency Bypass**
- **Type:** Ceramic X7R or X5R
- **Voltage Rating:** 10V minimum (16V recommended)
- **Placement:** IMMEDIATELY adjacent to PVIN pins (<2mm)
- **Function:**
  - Provides high-frequency current during switching
  - Filters input voltage ripple
  - Low ESR/ESL critical for performance

**Additional Bulk Capacitance (if needed):**
- Typical: 10-22µF ceramic or electrolytic
- Placement: Near input connector and U3
- Function: Provides charge reservoir for load transients

### Output Capacitors

**C25 (10µF) - Bulk Output Capacitor**
- **Type:** Ceramic X7R or low-ESR electrolytic
- **Voltage Rating:** 6.3V minimum (10V recommended for derating)
- **Function:**
  - Provides bulk charge storage
  - Filters output voltage ripple
  - Supplies current during load transients

**C26 (100nF) - High-Frequency Bypass**
- **Type:** Ceramic X7R
- **Voltage Rating:** 6.3V minimum
- **Placement:** Close to 3V3_DIG loads (MCU, sensors)
- **Function:**
  - High-frequency decoupling
  - Filters buck converter switching noise
  - Low impedance at MHz frequencies

**C27 (3.3nF) - Additional Output Filter**
- **Type:** Ceramic
- **Voltage Rating:** 6.3V minimum
- **Function:**
  - Reduces output ripple
  - Filters intermediate frequencies
  - Complements bulk and HF caps

**C28 (22µF) - Additional Bulk Capacitor**
- **Type:** Ceramic X7R or electrolytic
- **Voltage Rating:** 6.3V minimum
- **Function:**
  - Increases total output capacitance
  - Improves load transient response
  - Provides holdup during power switchover

**Total Output Capacitance:** 32µF (C25 + C28) + 100nF + 3.3nF H 32.1µF

**Ripple Voltage Calculation:**
```
V_ripple = ”I_L / (8 × f_SW × C_OUT)

Where:
- ”I_L H 0.6A (inductor ripple current)
- f_SW = 1MHz
- C_OUT = 32µF

V_ripple = 0.6A / (8 × 1MHz × 32µF) H 2.3mV (negligible)

Actual ripple dominated by ESR: V_ripple_ESR = ”I_L × ESR
With low-ESR ceramics (ESR <10m©): V_ripple_ESR H 6mV
```

### Feedback Network

**R4 (100k©) - Feedback Divider Upper Resistor**
- **Function:** Sets output voltage (upper leg of divider)
- **Tolerance:** 1% recommended for accuracy
- **Power Rating:** 1/10W minimum (low current)

**R8 (7.5k©) - Output Voltage Setting**
- **Function:** Primary output voltage setting resistor
- **Tolerance:** 1% recommended
- **Power Rating:** 1/10W minimum

**R9 (2.4k©) - Feedback Divider Lower Resistor**
- **Function:** Completes feedback divider
- **Tolerance:** 1% recommended
- **Power Rating:** 1/10W minimum

**Output Voltage Calculation:**
```
V_OUT = V_REF × (1 + R8/R9)
Where V_REF = 0.8V (TPS62130 internal)

With R8 = 7.5k©, R9 = 2.4k©:
V_OUT = 0.8V × (1 + 7.5k©/2.4k©)
V_OUT = 0.8V × (1 + 3.125)
V_OUT = 0.8V × 4.125 = 3.3V 
```

**Feedback Current:**
```
I_FB = V_OUT / (R8 + R9) = 3.3V / 9.9k© = 333µA
Power dissipation: P = V_OUT × I_FB = 1.1mW (negligible)
```

**C18 (100nF) - Feedback Compensation Capacitor**
- **Connection:** Parallel with R9 (FB pin to ground)
- **Function:** Loop compensation, stabilizes feedback control
- **Type:** Ceramic X7R
- **Effect:** Slows feedback response, reduces overshoot, improves stability

---

## Electrical Characteristics

### Input Specifications

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Input Voltage Range (Absolute) | 3.0 | 5.0 | 17.0 | V | TPS62130 datasheet spec |
| Input Voltage (Nominal) | 4.75 | 5.0 | 5.25 | V | 5V_SYS from power MUX |
| Input Current (Typical Load) | - | 1.0-1.5 | - | A | At 1.5A output, ~80% efficiency |
| Input Ripple | - | - | 100 | mV p-p | From 5V_SYS (depends on power module) |
| UVLO Threshold (Rising) | - | 2.9 | - | V | Converter starts when V_IN > UVLO |

### Output Specifications

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Output Voltage (3V3_DIG) | 3.20 | 3.30 | 3.40 | V | ±3% tolerance |
| Output Current (Max) | - | - | 3.0 | A | TPS62130 capability |
| Output Current (Typical, First Iteration) | - | 0.6-1.0 | - | A | STM32 + peripherals |
| Output Voltage Ripple | - | 10-30 | 50 | mV p-p | With 32µF output capacitance |
| Line Regulation | - | - | ±50 | mV | V_IN = 4.75-5.25V |
| Load Regulation | - | - | ±50 | mV | I_OUT = 0.1-2.0A |

### Switching Characteristics

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Switching Frequency | 0.9 | 1.0 | 1.1 | MHz | Configurable via FSW pin |
| Minimum On-Time | - | 100 | - | ns | Limits maximum V_IN / V_OUT ratio |
| Soft-Start Time | - | 60 | - | ms | With C_SS = 100nF |
| Power Good Threshold | - | 92 | - | % | PG asserts when V_OUT > 92% target |
| Power Good Delay | - | 10 | - | µs | Delay after V_OUT reaches threshold |

### Efficiency

| Load Current | Input Power | Output Power | Efficiency | Notes |
|--------------|-------------|--------------|------------|-------|
| 0.1A | 0.5W | 0.33W | ~70% | Light load, PSM mode |
| 0.5A | 2.2W | 1.65W | ~75% | Moderate load |
| 1.0A | 4.3W | 3.3W | ~77% | Typical load |
| 1.5A | 6.5W | 4.95W | ~76% | High typical load |
| 2.0A | 8.8W | 6.6W | ~75% | Max expected load |

**Note:** Efficiency varies with V_IN, V_OUT, load current, and operating mode

### Thermal Characteristics

| Parameter | Value | Unit | Notes |
|-----------|-------|------|-------|
| Junction Temperature (Max) | 125 | °C | Absolute maximum |
| Thermal Resistance (¸_JA) | 35 | °C/W | With exposed pad, 4-layer PCB, thermal vias |
| Power Dissipation (Typical @ 1A) | 0.8 | W | At ~77% efficiency |
| Power Dissipation (Max @ 2A) | 1.65 | W | At ~75% efficiency |
| Expected ”T (@ 1A, 25°C ambient) | 28 | °C | T_J H 53°C (safe) |

---

## Operating Modes

### Mode 1: Normal Operation (Continuous Conduction Mode, CCM)

**Condition:** Load current > critical current (~0.3A for this design)

**Operation:**
- Inductor current never reaches zero during switching cycle
- Constant switching frequency (~1MHz)
- High efficiency (75-80% typical)
- Predictable output ripple

**Waveforms:**
- Inductor current: Triangular waveform, continuous
- Switch node (SW): Square wave at f_SW
- Output voltage: DC + small ripple (~20mV)

**Best For:** Typical flight controller operation (0.5-2A load)

---

### Mode 2: Power Save Mode (PSM, Light Load)

**Condition:** Load current < critical current (~0.3A), DEF pin configured for PSM

**Operation:**
- Converter skips cycles when output voltage sufficient
- Variable switching frequency (burst mode)
- Improves light-load efficiency (e.g., MCU in sleep mode)
- Slightly increased output ripple

**Trade-off:**
- **Advantage:** Better efficiency at light loads (important for battery life)
- **Disadvantage:** Variable frequency may cause unpredictable EMI spectrum

**Configuration:** Typically enabled by default on TPS62130

---

### Mode 3: Forced PWM Mode

**Condition:** DEF pin configured for continuous PWM operation

**Operation:**
- Constant switching frequency regardless of load
- Inductor current may reverse (Discontinuous Conduction Mode at light loads)
- Predictable EMI spectrum
- Lower efficiency at very light loads

**Best For:** Applications requiring constant frequency (e.g., to avoid specific interference bands)

**Configuration:** Requires specific DEF pin resistor value

---

## Load Transient Response

### Fast Load Step (0.1A ’ 1.5A)

**Scenario:** STM32 wakes from sleep, activates peripherals, SD card write

**Response:**
1. **t=0:** Load current suddenly increases
2. **t=0-10µs:** Output voltage sags (capacitors discharge)
3. **t=10µs:** Feedback loop detects voltage drop
4. **t=10-50µs:** Buck converter increases duty cycle, more energy to output
5. **t=50-100µs:** Output voltage recovers to 3.3V ±50mV

**Capacitor Requirement:**
```
Energy deficit = C_OUT × (V_nominal - V_min) × ”I_load / f_SW

With C_OUT = 32µF, voltage droop budget = 100mV, ”I = 1.4A:
Energy supplied by caps during transient = 32µF × 0.1V = 3.2µJ

Droop time until feedback responds = ~20µs
Current deficit = 1.4A × 20µs = 28µC
Voltage drop = 28µC / 32µF = 875mV (excessive if underdamped)

This is why bulk capacitance (32µF) and fast control loop are important!
```

**Mitigation:**
- Large output capacitance (32µF total)
- Low-ESR ceramics (minimize I × ESR drop)
- Fast feedback loop (C18 compensation)

---

### Slow Load Change (0.5A ’ 0.7A)

**Scenario:** Gradual increase in MCU activity, sensor polling

**Response:**
- Output voltage remains within ±1% (excellent regulation)
- Feedback loop adjusts duty cycle smoothly
- No visible transient on oscilloscope

**This is normal operation:** TPS62130 regulation is excellent for slow changes

---

## PCB Layout Considerations

### Critical Layout Rules

**1. Minimize Switching Node Length**
- **Concern:** SW pins (1-3) carry high dV/dt, high-frequency AC current
- **Solution:** Place inductor L1 immediately adjacent to SW pins (<5mm trace)
- **Trace Width:** Wide enough for current (30-40 mils), but SHORT
- **Avoid:** Long traces, vias, routing near sensitive signals

**2. Input Loop Minimization**
- **Loop:** PVIN ’ C24 ’ PGND ’ GND plane ’ back to power source
- **Solution:** Place C24 (100nF) within 2-3mm of PVIN pins
- **Multiple Vias:** Connect C24 ground to plane with 2-4 vias (low impedance)
- **Why:** High-frequency input current (f_SW = 1MHz) creates EMI if loop is large

**3. Output Capacitor Placement**
- **Place C25, C26, C27, C28 near** 3V3_DIG load (STM32, sensors)
- **Distribute:** Don't cluster all caps at U3, spread across PCB at loads
- **Bulk caps (C25, C28):** Can be slightly further from U3
- **HF caps (C26):** Must be very close to MCU VDD pins

**4. Feedback Trace Routing**
- **FB pin (5):** Keep trace short, direct to R8/R9 divider
- **VOS pin (14):** Kelvin sense trace to 3V3_DIG output (separate from power trace)
- **Sensitive:** Route away from SW node, shield with ground if possible

**5. Ground Plane Strategy**
- **Solid ground plane** underneath U3 for thermal dissipation
- **PGND (15-16):** Connect directly to ground plane with multiple vias
- **AGND (6):** Connect to ground plane at single point or near IC
- **Star point:** AGND and PGND should meet at or near U3

**6. Thermal Management**
- **EPAD (17):** Solder to ground plane with thermal vias
- **Via Array:** 9-16 vias, 10-12 mil diameter, drilled through to bottom layer
- **Copper Pour:** 2oz copper or heavier on power planes
- **Clearance:** Minimize thermal vias' mask clearance for better thermal conduction

### Example Layout (Top View)

```
      5V_SYS
         |
    [C24 100nF]  <-- VERY close to PVIN
         |
    +----+----+
    | PVIN  SW|--- [L1 2.2µH] ---+
    |         |                   |
    |   U3    |                   |
    | TPS62130|                [C25]
    |         |                   |
    | PGND OUT|-------------------+---> 3V3_DIG
    +----+----+                   |
         |                      [C26]
        GND                       |
      (thermal vias)             GND
```

**Layer Stackup (4-layer PCB recommended):**
- Layer 1 (Top): Signal, components, short power traces
- Layer 2: Ground plane (solid)
- Layer 3: Power plane (3V3_DIG, 5V_SYS pours)
- Layer 4 (Bottom): Signal, ground

---

## Testing and Validation

### Bring-Up Procedure

**1. Visual Inspection**
- Verify U3 orientation (Pin 1 marker)
- Check for solder bridges (especially QFN package)
- Inspect L1, C24, C25, C26, C27, C28 placement
- Verify R8, R9 feedback resistors (7.5k©, 2.4k©)

**2. Resistance Check (Power OFF)**
- Measure 5V_SYS to GND: Should be >1k© (not shorted)
- Measure 3V3_DIG to GND: Should be >10k© (no load, feedback divider only)
- Measure SW node to GND: Should be >100© (inductor DCR)

**3. Initial Power-On (No Load)**
- Apply 5.0V to 5V_SYS input
- Measure 3V3_DIG output: Should ramp up smoothly to 3.25-3.35V
- Check soft-start time: ~60ms ramp (oscilloscope)
- Measure input current: ~10-30mA (no load, quiescent current)

**4. Load Testing**
- Apply 0.1A load (electronic load): V_OUT = 3.30V ±20mV
- Apply 0.5A load: V_OUT = 3.30V ±30mV
- Apply 1.0A load: V_OUT = 3.30V ±40mV
- Apply 1.5A load: V_OUT = 3.30V ±50mV
- Apply 2.0A load (stress test): V_OUT = 3.30V ±60mV

**5. Efficiency Measurement**
```
At 1.0A load:
- V_IN = 5.0V, I_IN = measure (e.g., 0.85A)
- V_OUT = 3.30V, I_OUT = 1.0A
- P_IN = 5.0V × 0.85A = 4.25W
- P_OUT = 3.30V × 1.0A = 3.30W
- Efficiency · = 3.30W / 4.25W = 77.6% 
```

**6. Ripple and Noise Measurement**
- **Equipment:** Oscilloscope with 20MHz bandwidth limit
- **Probe:** 1:1 probe or coaxial cable (minimize ground loop)
- **Measurement:** Probe 3V3_DIG at load (MCU VDD pin)
- **Expected:** 10-50mV p-p ripple at 1MHz fundamental
- **AC Coupling:** Use AC coupling to see ripple clearly
- **Pass Criteria:** <50mV p-p ripple, no spikes >100mV

**7. Transient Response Test**
- **Equipment:** Electronic load with pulse mode, oscilloscope
- **Test:** Step load 0.1A ’ 1.5A, rise time <10µs
- **Measure:** V_OUT droop and recovery time
- **Expected:**
  - Voltage droop: <100mV
  - Recovery time: <100µs
  - No ringing or oscillation

**8. Thermal Test**
- **Load:** 1.5A continuous (typical max)
- **Duration:** 1 hour
- **Measurement:** Thermal camera or thermocouple on U3
- **Pass Criteria:**
  - U3 temperature <80°C (ambient 25°C)
  - Inductor L1 temperature <70°C
  - No thermal shutdown events

### Troubleshooting

| Symptom | Possible Cause | Test / Fix |
|---------|----------------|------------|
| No output voltage | EN pin low or V_IN < UVLO | Check EN pin high, verify V_IN >3V |
| Output voltage low (<3.2V) | Incorrect feedback resistors | Verify R8=7.5k©, R9=2.4k© |
| Output voltage high (>3.4V) | R8/R9 ratio wrong or FB pin open | Check feedback divider continuity |
| High output ripple (>100mV) | Insufficient output capacitance | Add more bulk caps, check ESR |
| Converter not switching | EN low, V_IN too low, or IC damaged | Check EN pin, input voltage, replace IC |
| Thermal shutdown | Excessive load, poor PCB thermal | Reduce load, add thermal vias, check EPAD solder |
| Oscillation / instability | Poor layout, wrong compensation | Check C18 present, review PCB layout |
| Low efficiency (<70%) | High DCR inductor, excessive trace resistance | Verify L1 DCR <50m©, widen power traces |

---

## Design Validation Checklist

**Schematic Review:**
- [ ] PVIN (11-12) connected to 5V_SYS
- [ ] AVIN (10) connected to 5V_SYS
- [ ] SW (1-3) connected to inductor L1
- [ ] Inductor L1 connected to 3V3_DIG output
- [ ] Output caps C25 (10µF), C26 (100nF), C27 (3.3nF), C28 (22µF) present
- [ ] Feedback divider R8 (7.5k©), R9 (2.4k©) correct values
- [ ] C18 (100nF) on SS/TR pin for soft-start
- [ ] C24 (100nF) input bypass capacitor present
- [ ] PGND (15-16) and AGND (6) connected to ground
- [ ] EN pin pulled high (enabled)
- [ ] VOS pin connected to 3V3_DIG output

**PCB Layout Review:**
- [ ] C24 placed <3mm from PVIN pins
- [ ] Inductor L1 placed <5mm from SW pins
- [ ] SW node trace short, wide, no vias
- [ ] Output caps distributed across PCB at load points
- [ ] EPAD connected to ground plane with 9+ thermal vias
- [ ] Solid ground plane underneath U3
- [ ] Feedback trace (FB) routed away from SW node
- [ ] VOS Kelvin sense trace separate from power output
- [ ] 2oz copper or heavier on power planes

**Bring-Up Testing:**
- [ ] Output voltage = 3.25-3.35V (no load)
- [ ] Load regulation: 3.20-3.40V (0.1A - 2A)
- [ ] Line regulation: ±50mV (V_IN = 4.75-5.25V)
- [ ] Efficiency: >75% @ 1A load
- [ ] Output ripple: <50mV p-p @ 1A load
- [ ] Transient response: <100mV droop, <100µs recovery
- [ ] Thermal: U3 <80°C @ 1.5A, 1 hour, ambient 25°C
- [ ] No audible noise (from inductor or caps)

---

## Power Budget and Loads

### 3V3_DIG Rail Loads (First Iteration)

| Load | Current (mA) | Duty Cycle | Avg Current (mA) | Notes |
|------|--------------|------------|------------------|-------|
| STM32H743 Core | 200-400 | 100% | 300 | Activity dependent |
| STM32H743 Peripherals | 50-100 | 50% | 75 | UART, SPI, I2C, CAN |
| ICM-42688-P (IMU) | 1 | 100% | 1 | Low power |
| MMC5983MA (Mag) | 3 | 100% | 3 | Including SET/RESET |
| Barometer | 5 | 100% | 5 | If included |
| SD Card | 50-100 | 10% | 10 | Burst writes |
| CAN Transceivers | 20 | 100% | 20 | Dual CAN |
| LEDs / Indicators | 10-20 | 50% | 15 | Status LEDs |
| **Total Typical** | | | **429 mA** | Nominal operation |
| **Total Peak** | | | **650 mA** | All active simultaneously |

**Buck Converter Input Current (5V_SYS):**
```
I_IN = (V_OUT × I_OUT) / (V_IN × ·)
I_IN = (3.3V × 0.65A) / (5.0V × 0.77) H 0.56A

At peak load (1A output):
I_IN = (3.3V × 1.0A) / (5.0V × 0.77) H 0.86A
```

**Margin Available:**
- Buck converter capability: 3A
- First iteration typical: 0.65A (22% of capability)
- First iteration peak: 1.0A (33% of capability)
- **Available margin: 2.0A (67%)** for future expansion

---

## Related Documentation

### References
- `power-architecture.md` - Overall power system design
- `power-mux.md` - TPS2113A upstream power MUX documentation
- `config/power.yaml` - Complete power subsystem configuration

### Datasheets
- `resources/datasheets/POWER/TPS62130.pdf` - TPS62130 datasheet

### External Resources
- [TPS62130 Product Page](https://www.ti.com/product/TPS62130)
- [TI Buck Converter Design Guide](https://www.ti.com/lit/an/slva477b/slva477b.pdf)
- [Power Supply Layout Considerations](https://www.ti.com/lit/an/slva787/slva787.pdf)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-24
**Component:** U3 (TPS62130RGTR)
**Status:** First Revision Documentation
