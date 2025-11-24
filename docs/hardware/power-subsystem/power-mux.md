# TPS2113A Power Multiplexer (U2)

## Component Overview

**Part Number:** TPS2113ADRBR
**Manufacturer:** Texas Instruments
**Package:** DRB8 (SON-8, 2.4mm � 1.65mm)
**Function:** Autoswitching Power Multiplexer (Ideal-Diode Controller)
**Schematic Reference:** U2
**Datasheet:** https://www.ti.com/lit/gpn/tps2113a

---

## Purpose

The TPS2113A automatically selects between two power sources (Pixhawk power module and USB) based on priority configuration, providing seamless power switching for the flight controller without user intervention or external diodes.

**Key Features:**
- Automatic source switching with configurable priority
- Low forward voltage drop (0.1-0.2V typical via integrated MOSFETs)
- Reverse current blocking (prevents USB back-feeding into battery)
- 2.5A continuous current capability per input
- Status output for source indication
- No external Schottky diodes required

---

## Functional Description

### Ideal-Diode Operation

The TPS2113A uses **integrated N-channel MOSFETs** to implement an "ideal diode" function:

**Traditional Schottky Diode:**
- Forward voltage drop: 0.3-0.5V
- Power loss: V_drop � I_load (e.g., 0.4V � 1.5A = 0.6W wasted)
- No control over switching

**TPS2113A Ideal-Diode FETs:**
- Forward voltage drop: ~0.1-0.2V (R_DS(on) � I_load)
- Power loss: Reduced by 50-75%
- Active switching control with priority logic
- Reverse current blocking

### Priority Logic

**Configuration:** IN1 (EXT_5V) has priority over IN2 (USB_VBUS)

**Operation:**
1. **Both inputs present:** IN1 (external power) supplies output, IN2 is disconnected
2. **IN1 only:** IN1 supplies output
3. **IN2 only:** IN2 supplies output
4. **Neither input:** Output is disconnected

**Switchover Behavior:**
- **IN1 appears:** Immediate switch from IN2 to IN1 (no glitch)
- **IN1 disappears:** Seamless switch to IN2 if available (minimal brownout)
- **Voltage threshold:** Programmable via resistor divider (R6)

---

## Pin Configuration

### Pin Assignments

| Pin | Name | Function | Connection |
|-----|------|----------|------------|
| 1 | STAT | Status output | R5 pull-up � LED indicator |
| 2 | ~EN | Enable input (active low) | Pulled low (always enabled) |
| 3 | VSNS | Voltage sense for priority | R6 resistor divider |
| 4 | ILIM | Current limit setting | R7 = 820� |
| 5 | GND | Ground | System ground |
| 6 | IN1 | Input 1 (Priority) | EXT_5V (Pixhawk power module) |
| 7 | IN2 | Input 2 (Secondary) | USB_VBUS (via ESD protection) |
| 8 | OUT | Output | 5V_SYS rail (onboard systems) |

### Detailed Pin Functions

#### Pin 1: STAT (Status Output)
- **Function:** Indicates active input source
- **Logic:**
  - HIGH: IN1 active
  - LOW: IN2 active
  - Float/Tri-state: No valid input
- **Connection:** Pull-up resistor R5 (1.5k�) to drive LED indicator D2
- **Use:** Visual indication of power source (battery vs. USB)

#### Pin 2: ~EN (Enable, Active Low)
- **Function:** Enables/disables power MUX operation
- **Logic:**
  - LOW: Enabled (normal operation)
  - HIGH: Disabled (all outputs off)
- **Connection:** Tied to GND (always enabled)
- **Future Use:** Could be connected to MCU GPIO for software control

#### Pin 3: VSNS (Voltage Sense)
- **Function:** Sets switching threshold for IN1 vs. IN2
- **Connection:** Resistor divider R6 (4.3k�)
- **Threshold:** Programmable via R6 value
- **Typical:** IN1 must be >4.5V to be considered valid

#### Pin 4: ILIM (Current Limit)
- **Function:** Sets overcurrent protection threshold
- **Connection:** R7 = 820� to ground
- **Formula:** I_limit H 500 / R_ILIM (k�)
- **Calculated:** 500 / 0.82 H 610 � � ~2.5A limit
- **Protection:** Prevents damage to inputs during fault conditions

#### Pin 5: GND (Ground)
- **Connection:** System ground plane
- **Current:** Return path for control circuits and gate drive

#### Pin 6: IN1 (Primary Input)
- **Source:** EXT_5V from Pixhawk power module
- **Priority:** 1 (highest)
- **Voltage Range:** 2.8-5.5V (TPS2113A spec)
- **Nominal:** 5.0V (4.75-5.25V)
- **Bypass:** C41 (1�F) + C42 (100nF) for local filtering

#### Pin 7: IN2 (Secondary Input)
- **Source:** USB_VBUS (via ESD diode D6)
- **Priority:** 2 (fallback only)
- **Voltage Range:** 2.8-5.5V
- **Nominal:** 5.0V (USB standard)
- **Use Case:** Development/debugging without battery

#### Pin 8: OUT (Power Output)
- **Output:** 5V_SYS rail (onboard systems only)
- **Max Current:** 2.5A continuous
- **Load:** TPS62130 buck converter (~750mA typical)
- **Reserved:** Onboard systems (external peripherals bypass this)

---

## Support Components

### Input Filtering

**C41 (1�F) - Bulk Input Capacitor**
- **Type:** Ceramic (X7R preferred)
- **Voltage Rating:** 10V minimum (16V recommended for margin)
- **Placement:** Close to IN1/IN2 pins
- **Function:** Provides charge reservoir for switching transients

**C42 (100nF) - High-Frequency Bypass**
- **Type:** Ceramic (X7R or X5R)
- **Voltage Rating:** 10V minimum
- **Placement:** Very close to VCC pins (shortest path)
- **Function:** Filters high-frequency switching noise

### Configuration Resistors

**R5 (1.5k�) - Status Pull-up**
- **Connection:** STAT pin to LED indicator
- **Function:** Provides pull-up current for status output
- **LED Drive:** Sources current through D2 (PWR_LED)
- **Calculation:** I_LED H (5V - V_LED) / 1.5k� H 2-3mA

**R6 (4.3k�) - Voltage Sense Divider**
- **Connection:** VSNS pin configuration
- **Function:** Sets valid input voltage threshold
- **Threshold:** IN1 must exceed ~4.5V to be active
- **Tuning:** Lower R6 = higher threshold, Higher R6 = lower threshold

**R7 (820�) - Current Limit Setting**
- **Connection:** ILIM pin to GND
- **Function:** Programs overcurrent protection threshold
- **Formula:** I_limit(A) H 500 / R_ILIM(k�)
- **Result:** 500 / 0.82 H 610� � ~2.5A limit
- **Protection:** Prevents damage during short circuit or overload

**R_ILIM (250�) - External Input Current Limit**
- **Connection:** Series with EXT_5V input
- **Function:** Provides additional current limiting for external power
- **Note:** This is upstream of U2, separate from R7 internal limit

---

## Electrical Characteristics

### Input Specifications

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Input Voltage Range | 2.8 | 5.0 | 5.5 | V | Per TPS2113A datasheet |
| IN1 (EXT_5V) Nominal | 4.75 | 5.0 | 5.25 | V | Pixhawk power module spec |
| IN2 (USB_VBUS) Nominal | 4.75 | 5.0 | 5.25 | V | USB 2.0 specification |
| Continuous Current | - | - | 2.5 | A | Per input, TPS2113A rating |
| Typical Load (First Iteration) | - | 1.0-1.5 | - | A | Onboard systems only |

### Output Specifications

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Output Voltage (5V_SYS) | 4.65 | 5.0 | 5.35 | V | Input minus drop |
| Voltage Drop (FET R_DS(on)) | 0.1 | 0.15 | 0.2 | V | At 1.5A load |
| Output Ripple | - | - | 100 | mV p-p | Depends on input quality |
| Max Output Current | - | - | 2.5 | A | Limited by ILIM setting |

### Switching Characteristics

| Parameter | Min | Typ | Max | Unit | Notes |
|-----------|-----|-----|-----|------|-------|
| Switchover Time (IN1 � IN2) | - | 10 | 50 | �s | Depends on load capacitance |
| Switchover Time (IN2 � IN1) | - | 5 | 20 | �s | Priority switch is faster |
| Reverse Current Blocking | - | <1 | - | �A | Prevents back-feeding |
| IN1 Priority Threshold | - | 4.5 | - | V | Set by R6 resistor |

### Efficiency

| Parameter | Value | Notes |
|-----------|-------|-------|
| Forward Voltage Drop | 0.1-0.2V @ 1.5A | Much lower than Schottky (0.3-0.5V) |
| Power Dissipation | 0.15-0.3W @ 1.5A | I� � R_DS(on) |
| Efficiency | 95-98% | (V_out / V_in) � 100% |

---

## Power Architecture Role

### System Integration

```
Pixhawk Power Module (2-3A @ 5V)
    |
    +-- EXT_5V ----+
                   |
                   +-- EXTERNAL PERIPHERALS (direct connection)
                   |   (GPS, telemetry, future)
                   |   Bypass MUX entirely
                   |
                   +-- TPS2113A U2 Pin 6 (IN1, Priority 1)
                            |
USB_VBUS (0.5-2A)           |
    |                       |
    +-- ESD Protection      |
        (D6: TSD05DYFR)     |
            |               |
            +-- TPS2113A U2 Pin 7 (IN2, Priority 2)
                    |
                    v
              TPS2113A Logic
              [IN1 > IN2 priority]
                    |
                    v
              5V_SYS (U2 Pin 8)
              ONBOARD SYSTEMS ONLY
                    |
                    +-- TPS62130RGTR Buck Converter (U3)
                    +-- USB PHY (if used)
                    +-- Future onboard 5V devices
```

### Design Rationale

**Why External Peripherals Bypass the MUX:**

1. **Current Capacity:** MUX limited to 2.5A, external peripherals can draw 0.5-1A+
2. **Fault Isolation:** External faults (short circuit, overcurrent) cannot affect flight controller core
3. **Full Power Access:** External peripherals access full power module capability (2-3A)
4. **Clean Core Rail:** 5V_SYS remains stable for buck converter and MCU

**Why Dual-Input MUX is Critical:**

1. **Development Workflow:** Enables firmware development without battery (USB power)
2. **Safe Connection:** Automatic priority prevents USB overcurrent when external power connected
3. **No User Action:** Seamless switching, no switches or jumpers required
4. **Back-Feed Protection:** USB cannot charge battery (reverse blocking)

---

## Operational Modes

### Mode 1: Normal Flight (External Power Only)

**Condition:** Pixhawk power module connected, USB disconnected

**Operation:**
- IN1 (EXT_5V) = 5.0V (active)
- IN2 (USB_VBUS) = 0V (disconnected)
- OUT (5V_SYS) = 4.85V (IN1 minus 0.15V drop)
- STAT = HIGH (IN1 active indicator)

**Current Flow:**
- Onboard systems (via MUX): 1.0-1.5A
- External peripherals (direct): 0.2-0.7A
- Total from power module: 1.2-2.2A

**Status:** Normal flight operations, battery-powered

---

### Mode 2: Development (USB Only)

**Condition:** USB connected, no external power

**Operation:**
- IN1 (EXT_5V) = 0V (disconnected)
- IN2 (USB_VBUS) = 5.0V (active)
- OUT (5V_SYS) = 4.85V (IN2 minus 0.15V drop)
- STAT = LOW (IN2 active indicator)

**Current Flow:**
- Onboard systems: 0.5-1.0A (limited by USB port capability)
- External peripherals: None (would be unpowered)

**Status:** Development/bench testing mode

**Limitations:**
- USB current limited to 0.5-2A (port dependent)
- Not suitable for full flight operations
- External 5V peripherals unpowered (direct connection to EXT_5V)

---

### Mode 3: Bench Testing (Both Connected)

**Condition:** Both Pixhawk module AND USB connected

**Operation:**
- IN1 (EXT_5V) = 5.0V (priority)
- IN2 (USB_VBUS) = 5.0V (available but not used)
- OUT (5V_SYS) = 4.85V (IN1 minus 0.15V drop)
- STAT = HIGH (IN1 active, IN2 ignored)

**Priority Logic:** IN1 always wins when both present

**Benefit:** USB can remain connected for data/debugging while external power supplies system

**Safety:** USB cannot back-feed into battery (reverse blocking)

---

### Mode 4: Switchover (Power Source Transition)

**Scenario A: External Power Applied (IN2 � IN1)**

**Sequence:**
1. System running on USB (IN2 active, STAT = LOW)
2. External power module connected (IN1 voltage rises)
3. TPS2113A detects IN1 > threshold (4.5V)
4. Switchover: IN2 FET turns off, IN1 FET turns on
5. System now on external power (IN1 active, STAT = HIGH)
6. **Switchover time:** ~5-20�s (minimal brownout)

**Result:** Seamless transition, no user intervention, no system reset

---

**Scenario B: External Power Removed (IN1 � IN2)**

**Sequence:**
1. System running on external power (IN1 active, STAT = HIGH)
2. External power disconnected (IN1 voltage drops)
3. TPS2113A detects IN1 < threshold
4. Switchover: IN1 FET turns off, IN2 FET turns on (if available)
5. System now on USB (IN2 active, STAT = LOW)
6. **Switchover time:** ~10-50�s (brief brownout possible)

**Result:** If USB present, system continues operation. If no USB, system powers down cleanly.

**Note:** Downstream capacitance (C25, C28 on 3.3V rail) provides holdup during switchover

---

## Fault Conditions and Protection

### Overcurrent Protection

**Mechanism:** Current limit set by R7 (820�) = ~2.5A

**Fault Response:**
1. Output current exceeds 2.5A threshold
2. TPS2113A limits current by increasing FET resistance
3. Output voltage sags (I � R_increased)
4. If sustained, thermal shutdown may occur

**Recovery:** Automatic when fault clears (latchoff not used)

**Design Consideration:** Onboard typical load (1.0-1.5A) well below limit, provides margin for transients

### Reverse Current Blocking

**Protection:** Integrated body diode blocking + active FET control

**Scenario:** IN2 (USB) attempts to back-feed into IN1 (battery)

**Response:**
- FET remains off when output voltage > input voltage
- Reverse current <1�A (body diode leakage)
- Battery cannot be charged via USB (safety)

### Short Circuit Protection

**Input Short Circuit:**
- Power module provides overcurrent protection upstream
- R_ILIM (250�) provides additional limiting

**Output Short Circuit:**
- TPS2113A current limit engages (2.5A)
- FET resistance increases, limiting fault current
- Thermal protection may shut down if sustained

**Recovery:** Automatic when short removed

### Undervoltage Lockout (UVLO)

**Threshold:** IN1 or IN2 must exceed ~2.8V to activate

**Purpose:** Prevents operation at invalid voltages

**Behavior:**
- Below UVLO: Output FET off, no power delivered
- Above UVLO: Normal operation resumes

### Thermal Protection

**Junction Temperature Limit:** 125�C typical

**Mechanism:** Thermal shutdown disables outputs

**Recovery:** Automatic when junction cools below threshold

**Design Mitigation:**
- PCB copper pour for heat spreading
- 2oz copper or heavier recommended
- Typical load (1.5A) generates 0.2-0.3W (manageable)

---

## PCB Layout Considerations

### Component Placement

**Critical:**
- Place C41 (1�F) and C42 (100nF) immediately adjacent to IN1/IN2 pins
- Short, wide traces from power connector to IN1 (minimize resistance)
- TPS2113A should be near power input connector (J_PWR)

**Thermal:**
- U2 exposed pad (if present) should connect to ground plane with thermal vias
- 2oz copper or heavier on power planes for thermal dissipation
- Adequate copper pour around U2 for heat spreading

### Trace Routing

**Power Traces (IN1, IN2, OUT):**
- Width: Minimum 40 mils (1mm) for 2.5A current
- Preferred: 60-80 mils for lower resistance and better thermal performance
- Length: Keep as short as practical
- Avoid 90� corners (use 45� or curved)

**Ground Return:**
- Solid ground plane underneath TPS2113A
- Multiple vias connecting GND pin to plane
- Low-impedance return path for switching currents

**Sensitive Signals (STAT, VSNS, ILIM):**
- Keep away from switching power traces
- Route on separate layer if possible
- Shorter traces for configuration resistors

### Decoupling Strategy

**Input Decoupling (C41, C42):**
```
Power Connector � IN1/IN2 � [C42 100nF very close] � [C41 1�F nearby] � GND plane
```

**Ground Connection:**
- C41, C42 negative terminals should connect to ground plane with short traces
- Multiple vias to ground plane for low impedance

### Example Layout

```
       EXT_5V
         |
    [C42 100nF] [C41 1�F]
         |
    +----+----+
    |   U2    |  TPS2113A
    | TPS2113 |  (Top View)
    |         |
    +----+----+
         |
      5V_SYS (to U3 buck converter)
         |
    [Bulk caps on 5V_SYS rail]
```

---

## Testing and Validation

### Functional Tests

**1. Input Voltage Test**
- Connect EXT_5V to 5.0V bench supply
- Measure 5V_SYS output: Should be 4.8-4.9V (expect 0.1-0.2V drop)
- Vary input 4.75-5.25V, verify output tracks

**2. Priority Logic Test**
- Connect USB only � STAT should be LOW, 5V_SYS = 4.85V
- Connect EXT_5V (5.0V) � STAT should go HIGH, 5V_SYS = 4.85V
- Disconnect EXT_5V � STAT goes LOW, system switches to USB seamlessly

**3. Switchover Test**
- System running on USB (1A load via electronic load)
- Apply EXT_5V (5.0V)
- Monitor 5V_SYS with oscilloscope: Should see <50�s transition, no brownout
- Verify STAT transitions HIGH

**4. Current Limit Test**
- Apply 5V to IN1
- Gradually increase load on 5V_SYS (use electronic load)
- At ~2.5A, output voltage should sag (current limiting)
- Verify no damage, recovers when load reduced

**5. Reverse Current Test**
- Connect EXT_5V to 5.0V
- Connect USB to 5.0V (both active)
- Disconnect EXT_5V (unplug)
- Verify USB cannot back-feed into EXT_5V connector
- Measure reverse current: <1�A expected

### Performance Measurements

**Voltage Drop Measurement:**
- Load: 1.0A (typical), 1.5A (max expected), 2.0A (stress test)
- Measure: V_IN1 - V_OUT
- Expected: 0.1-0.2V @ 1.5A (indicates healthy FET R_DS(on))
- Failure: >0.3V indicates high resistance or thermal issue

**Efficiency Calculation:**
```
� = (V_OUT � I_OUT) / (V_IN � I_IN) � 100%
Expected: 95-98% at 1.5A load
```

**Power Dissipation:**
```
P_diss = I_load� � R_DS(on)
Example: (1.5A)� � 0.1� = 0.225W
Verify thermal: U2 should be warm but not hot (<60�C ambient)
```

### Troubleshooting

| Symptom | Possible Cause | Test / Fix |
|---------|---------------|------------|
| No output voltage | Neither input present | Verify EXT_5V or USB_VBUS present |
| Output low (<4.5V) | High current, voltage drop | Check load current, verify not >2.5A |
| STAT always LOW | IN1 below threshold | Check EXT_5V voltage, R6 configuration |
| Switchover causes reset | Insufficient holdup capacitance | Add bulk capacitance on 5V_SYS or 3V3 rails |
| Hot U2 component | Excessive current or short | Check load current, inspect for shorts |
| No switchover to IN1 | IN1 voltage too low or R6 wrong | Verify EXT_5V >4.5V, check R6 = 4.3k� |

---

## Design Validation Checklist

**Schematic Review:**
- [ ] IN1 connected to EXT_5V (Pixhawk power module)
- [ ] IN2 connected to USB_VBUS via ESD protection
- [ ] OUT connected to 5V_SYS rail (onboard systems)
- [ ] C41 (1�F) and C42 (100nF) present on inputs
- [ ] R5 (1.5k�), R6 (4.3k�), R7 (820�) values correct
- [ ] STAT connected to LED indicator or MCU GPIO
- [ ] ~EN pulled low (enabled)
- [ ] GND connected to system ground

**PCB Layout Review:**
- [ ] C41, C42 placed close to IN1/IN2 pins (<5mm)
- [ ] Power traces e40 mils (1mm) width for 2.5A
- [ ] Solid ground plane underneath U2
- [ ] Thermal vias on exposed pad (if applicable)
- [ ] Short traces for R5, R6, R7 configuration
- [ ] No high-frequency noise sources near VSNS, ILIM pins

**Bring-Up Testing:**
- [ ] Power-on with USB: Verify 5V_SYS present, STAT = LOW
- [ ] Power-on with EXT_5V: Verify 5V_SYS present, STAT = HIGH
- [ ] Switchover test (both � remove EXT_5V): Seamless transition
- [ ] Voltage drop: <0.2V @ 1.5A load
- [ ] Thermal: U2 temperature <60�C @ 1.5A, ambient 25�C
- [ ] Priority logic: IN1 always takes precedence over IN2

---

## Related Documentation

### References
- `power-architecture.md` - Overall power system design
- `buck-converter.md` - TPS62130RGTR downstream converter
- `config/power.yaml` - Complete power subsystem configuration

### Datasheets
- `resources/datasheets/POWER/TPS2113A.pdf` - TPS2113A datasheet
- `resources/datasheets/ESD/TSD05DYFR.pdf` - USB ESD protection

### External Resources
- [TPS2113A Product Page](https://www.ti.com/product/TPS2113A)
- [TI Power Management Reference Designs](https://www.ti.com/power-management/overview.html)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-24
**Component:** U2 (TPS2113ADRBR)
**Status:** First Revision Documentation
