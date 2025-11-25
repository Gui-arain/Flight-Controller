# Power Architecture Overview

## Purpose

The power subsystem provides clean, reliable power to the STM32H743ZIT6 microcontroller, sensors (IMU, magnetometer, barometer), and external peripherals while maintaining compatibility with the Pixhawk power module ecosystem.

## Design Philosophy

**First Iteration Goals:**
- Validate hardware architecture with proven standards
- Achieve stable sensor operation with minimal noise
- Demonstrate Pixhawk ecosystem compatibility
- Provide flexible development capability (USB + battery power)
- Minimize complexity and risk

**Key Principles:**
- **Ecosystem Integration:** Leverage Pixhawk power module standard for battery interfacing
- **Power Isolation:** Separate external peripheral power from critical onboard systems
- **Clean Analog Rails:** Ultra-low noise power for precision sensors (IMU, magnetometer)
- **Development Flexibility:** Dual power input (battery + USB) with automatic switching
- **Conservative Margins:** 50-75% power headroom for first iteration validation

---

## System Architecture

### Power Flow Hierarchy

```
Pixhawk Power Module (2-3A @ 5V from LiPo battery)
    |
    +-- EXT_5V (JST GH Pins 1-2) ----+
    |                                 |
    |                                 +-- EXTERNAL 5V PERIPHERALS
    |                                 |   (GPS, telemetry, future)
    |                                 |   Direct connection - Full power module capability
    |                                 |
    |                                 +-- TPS2113A Power MUX (U2)
    |                                     Input IN1 (Priority 1)
    |                                          |
    +-- BAT_VOLTAGE (Pin 4) -----> STM32 ADC   |
    |                                          |
    +-- BAT_CURRENT (Pin 3) -----> STM32 ADC   |
    |                                          |
    +-- GND (Pins 5-6) ----------------+       |
                                               v
                                         TPS2113A (U2)
                                         [EXT_5V > USB priority]
                                               |
                                               v
                                         5V_SYS (2.5A max)
                                         ONBOARD SYSTEMS ONLY
                                               |
                                               v
                                         TPS62130RGTR (U3)
                                         Buck Converter
                                               |
                                               v
                                         3V3_DIG (2A max)
                                         Digital power rail
                                               |
                                               +-- STM32H743 VDD
                                               +-- Communication interfaces
                                               +-- SD card
                                               +-- Ferrite Bead Filter (FB1)
                                                        |
                                                        v
                                                   3V3_ANA (300mA max)
                                                   Filtered sensor power
                                                        |
                                                        +-- ICM-42688-P (IMU)
                                                        +-- MMC5983MA (Magnetometer)
                                                        +-- STM32H743 VDDA
                                                        +-- Barometer

USB_VBUS (0.5-2A, development fallback)
    |
    +-- ESD Protection (D6: TSD05DYFR)
    |
    +-- TPS2113A (U2) Input IN2 (Priority 2)
            |
            v
        5V_SYS (when no external power present)
```

---

## Power Rails Specification

### Input Rails

#### EXT_5V (Pixhawk Power Module)
- **Voltage:** 5.0V (4.75-5.25V tolerance)
- **Current Capability:** 2-3A (module dependent)
- **Source:** Pixhawk power module (regulated from 2S-6S LiPo)
- **Connector:** JST GH 6-pin (SM06B-GHS-TB), Pins 1-2
- **Loads:**
  - External 5V peripherals (direct connection)
  - TPS2113A power MUX input (for onboard systems)
- **Monitoring:** Battery voltage and current sensing via dedicated ADC inputs
- **Protection:** Provided by power module (overcurrent, transient)

#### USB_VBUS (Development Fallback)
- **Voltage:** 5.0V
- **Current Capability:** 0.5-2.0A (USB port dependent)
- **Source:** USB connection (development/debugging)
- **Protection:** ESD protection diode (TSD05DYFR)
- **Priority:** Secondary (TPS2113A automatically switches to EXT_5V when available)
- **Use Cases:** Firmware development, bench testing, configuration without battery

### Regulated Rails

#### 5V_SYS (Post-MUX Onboard Rail)
- **Voltage:** 5.0V (4.75-5.25V)
- **Max Current:** 2.5A (TPS2113A limit)
- **Typical Current:** 1.0-1.5A (first iteration)
- **Source:** TPS2113A output (automatic source selection)
- **Loads:**
  - TPS62130 buck converter input (~750mA)
  - USB PHY (if used)
  - Future onboard 5V devices (~1.0A margin)
- **Reserved For:** Onboard systems only (external peripherals bypass this rail)
- **Characteristics:** <100mV ripple (depends on power module quality)

#### 3V3_DIG (Digital Power Rail)
- **Voltage:** 3.3V (3.20-3.40V, ±3%)
- **Max Current:** 2.0A (TPS62130 capability)
- **Typical Current:** 0.6-1.0A (first iteration with margin)
- **Source:** TPS62130RGTR synchronous buck converter
- **Loads:**
  - STM32H743ZIT6 VDD pins (~200-400mA)
  - Communication interfaces (~20-50mA)
  - SD card interface (~50-100mA during writes)
  - LEDs and indicators (~10-20mA)
  - Input to analog LDO
- **Characteristics:**
  - <50mV p-p ripple (well filtered)
  - Fast transient response (<10µs)
  - 75-85% efficiency from 5V input
- **Filtering:** 32µF bulk + 100nF ceramic + 3.3nF additional

#### 3V3_ANA (Analog Sensor Power)
- **Voltage:** 3.3V (3.20-3.40V, ±3%)
- **Max Current:** 300mA (ferrite bead rating)
- **Typical Current:** 20-50mA (first iteration sensors)
- **Source:** Ferrite bead filter (FB1) from 3V3_DIG
- **Loads:**
  - ICM-42688-P IMU (~1mA)
  - MMC5983MA magnetometer (~3mA)
  - Barometer (~5mA, if included)
  - STM32H743 VDDA pin (~10mA)
  - GPS module (~50-150mA, via ferrite bead)
- **Characteristics:**
  - >20dB noise attenuation @ 1MHz and above
  - <10mV voltage drop @ 50mA typical load
  - <10mV p-p ripple (filtered from 3V3_DIG)
  - Passive LC filter topology
- **Filter Components:**
  - FB1: Ferrite bead (100-600 ohm @ 100MHz)
  - C14: 100nF input capacitor
  - C15: 1uF output capacitor
  - TP1: Test point
- **Purpose:** Filtered power for precision sensor measurements

#### VDD33_USB (USB PHY Power)
- **Voltage:** 3.3V (±3%)
- **Max Current:** 100mA
- **Source:** Derived from 3V3_DIG with filtering (TBD)
- **Load:** STM32H743 VDD33USB pin (~50mA)
- **Criticality:** MANDATORY - Must be powered even if USB not used
- **Requirement:** STM32H743 USB PHY power domain must be active for proper MCU operation

---

## Power Budget Analysis

### First Iteration Current Budget

**Onboard 3.3V Consumption:**
| Component | Current | Notes |
|-----------|---------|-------|
| STM32H743ZIT6 | 200-400mA | Activity dependent |
| ICM-42688-P (IMU) | ~1mA | Low power mode capable |
| MMC5983MA (Mag) | ~3mA | Including SET/RESET cycles |
| Barometer | ~5mA | If included |
| SD Card | 50-100mA | Peak during writes |
| CAN Transceivers | ~20mA | Dual CAN interfaces |
| LEDs/Indicators | ~10-20mA | Status indicators |
| **Total @ 3.3V** | **~550mA** | Conservative estimate |

**Buck Converter Input (5V_SYS):**
| Load | Current | Efficiency |
|------|---------|-----------|
| Buck converter | ~750mA @ 5V | ~75% (550mA @ 3.3V) |
| USB PHY | 0mA | Not used in first iteration |
| **Total @ 5V_SYS** | **~750mA** | ~1.25A margin remaining |

**External Peripherals (EXT_5V Direct):**
| Peripheral | Current | Connection |
|------------|---------|------------|
| GPS Module | 50-150mA | Direct to EXT_5V |
| Telemetry Radio | 100-500mA | Direct to EXT_5V |
| Future Expansion | Variable | Direct to EXT_5V |

**Total Power Module Load:**
- Onboard (via MUX): ~1.0-1.5A
- External (direct): ~0.2-0.7A (first iteration)
- **Total Estimated:** 1.2-2.2A
- **Available:** 2-3A from Pixhawk power module
- **Safety Margin:** 25-50% headroom

---

## Key Design Decisions

### 1. Pixhawk Power Module Compatibility

**Decision:** Use standard JST GH 6-pin connector and Pixhawk power module interface

**Rationale:**
- Leverages established drone ecosystem and available hardware
- Integrated battery voltage/current monitoring (no onboard sensing needed)
- Proven reliability in flight controller applications
- Wide vendor availability ensures supply chain flexibility
- Community familiarity simplifies troubleshooting and support

**Trade-off:** JST GH connector limited to 1A per contact (2A total with dual pins) vs. higher-current alternatives

**Conclusion:** For first iteration validation with minimal external peripherals, Pixhawk standard provides optimal balance of capability and ecosystem integration.

### 2. External Peripheral Power Architecture

**Decision:** External 5V peripherals connect directly to EXT_5V (bypass power MUX)

**Rationale:**
- **Protects critical onboard systems:** External faults cannot affect flight controller core
- **Avoids MUX current limit:** External peripherals access full power module capability (2-3A)
- **Isolates variable loads:** GPS, telemetry, LEDs don't disturb MCU/sensor power
- **Simple implementation:** EXT_5V rail branches before MUX input

**Affected Peripherals:** GPS modules, telemetry radios, LED strips, companion computers

**Architecture:**
```
Power Module → EXT_5V → Splits to:
                         1. External peripherals (direct)
                         2. Onboard systems (via MUX → 5V_SYS)
```

### 3. Dual-Input Power Multiplexer

**Decision:** TPS2113A ideal-diode power MUX with automatic source selection

**Rationale:**
- **Development flexibility:** Enables firmware development and debugging without battery
- **Automatic switching:** Priority logic (EXT_5V > USB) prevents USB overcurrent
- **Seamless transitions:** No brownout during source changes
- **Low voltage drop:** 0.1-0.2V forward drop (vs. 0.3-0.5V for Schottky diodes)
- **Proven topology:** Used in commercial flight controllers (Pixhawk, Cube, etc.)

**Priority Logic:** External power always takes precedence when present (USB cannot back-feed into battery)

### 4. Buck Converter vs. Linear Regulator

**Decision:** TPS62130RGTR synchronous buck converter for 3.3V generation

**Rationale:**
- **Efficiency:** 75-85% efficiency vs. 66% for linear regulator
- **Thermal management:** Lower power dissipation critical in compact flight controller
- **Current capability:** 2A output supports future expansion
- **Heat reduction:** ~0.5-1W dissipation vs. ~2W for linear regulator at 1A load
- **Compact solution:** QFN package with minimal external components

**Trade-off:** Buck converter introduces switching noise vs. linear regulator's clean output

**Mitigation:** Separate 3V3_ANA ferrite bead filter rail for noise-sensitive sensors

### 5. Separate Analog Power Rail

**Decision:** Dedicated 3V3_ANA rail via ferrite bead filter for sensors

**Rationale:**
- **Sensor performance:** ICM-42688-P gyro noise specification (2.8 mdps/√Hz) requires clean power
- **ADC accuracy:** STM32H743 VDDA pin needs stable reference for battery monitoring
- **Magnetometer stability:** MMC5983MA 18-bit resolution benefits from filtered supply
- **Noise isolation:** Ferrite bead filters buck converter switching noise (>20dB @ 1MHz)
- **Simple passive approach:** No active regulation, minimal voltage drop (<10mV)
- **Cost-effective:** Fewer components than LDO approach, high reliability

**Target Specifications:**
- Noise attenuation: >20dB @ 1MHz and above
- Voltage drop: <10mV @ 50mA
- Ripple: <10mV p-p

---

## Grounding Strategy

### Star Grounding Architecture

**Implementation:**
- **Power Ground (PGND):** Buck converter switching currents, high-current return paths
- **Analog Ground (AGND):** Sensor and ADC reference ground, isolated from switching noise
- **Digital Ground (GND):** Digital logic ground plane
- **Star Point:** All grounds converge at or near power input connector

**Best Practices:**
- Minimize ground loops between analog and digital sections
- Route high-current return paths directly to power ground
- Keep sensor grounds quiet (no switching currents)
- Use ground plane stitching vias for low impedance
- Ferrite beads or resistors between ground domains if needed

**Critical for Performance:**
Proper grounding is essential for ICM-42688-P and MMC5983MA noise performance. Multiple ground symbols in schematic indicate separate ground planes that must be carefully routed in PCB layout.

---

## Power Sequencing

### Preferred Startup Sequence

1. **t=0ms: 5V_SYS establishes**
   - TPS2113A power MUX output
   - Source: EXT_5V or USB_VBUS (automatic selection)

2. **t=~10ms: 3V3_DIG ramps up**
   - TPS62130 buck converter soft-start
   - Provides power to STM32H743 and digital systems
   - PG (Power Good) signal available for monitoring

3. **t=~15ms: 3V3_ANA stabilizes**
   - Low-noise LDO soft-start
   - Clean power available for sensors
   - ADC reference stable

4. **t=~15ms: VDD33_USB powered**
   - USB PHY power domain active
   - Required for STM32H743 operation

**Inrush Current Management:**
- Total capacitance: ~100µF (all rails combined)
- Soft-start limiting: TPS62130 internal, power module current limit
- Typical inrush: <2A peak (limited by power module)

---

## Pixhawk Power Module Integration

### Standard Pinout (JST GH 6-pin, SM06B-GHS-TB)

| Pin | Signal | Function | Voltage Range |
|-----|--------|----------|---------------|
| 1 | VCC | +5V output | 4.75-5.25V |
| 2 | VCC | +5V output (parallel) | 4.75-5.25V |
| 3 | CURRENT | Battery current sense | 0-3.3V analog |
| 4 | VOLTAGE | Battery voltage sense | 0-3.3V analog |
| 5 | GND | Ground return | 0V |
| 6 | GND | Ground return (parallel) | 0V |

**Dual Power/Ground Pins:** Allows 2A total current within 1A per contact rating

### Battery Monitoring

**Voltage Sensing:**
- **Signal:** BAT_VOLTAGE (Pin 4)
- **Source:** Power module voltage divider (module dependent)
- **Range:** 0-3.3V analog (represents battery voltage)
- **Scaling:** Typically ~0.5-0.6V per cell, requires calibration
- **Connection:** STM32H743 ADC input
- **Purpose:** Real-time battery voltage monitoring for low-voltage warnings

**Current Sensing:**
- **Signal:** BAT_CURRENT (Pin 3)
- **Source:** Power module current sensor (hall effect or shunt)
- **Range:** 0-3.3V analog (represents battery current)
- **Scaling:** Module dependent (e.g., 18.0V/V, 36.6V/V), requires calibration
- **Connection:** STM32H743 ADC input
- **Purpose:** Battery current monitoring for mAh consumed and power tracking

**Calibration Required:**
Both voltage and current scaling factors are power module specific and must be calibrated in ground station against known reference values.

### Compatible Power Modules

- **Standard Pixhawk Power Module:** Most common, 2.5A output
- **Holybro PM02/PM03:** Higher quality filtering, 3A output
- **Mauch Power Modules:** High-precision current sensing
- **CUAV PM Series:** Various current ratings
- **Generic 6-pin Pixhawk modules:** Wide availability from multiple vendors

**All modules provide:**
- Regulated 5V output from 2S-6S LiPo input
- Battery voltage sensing (scaled analog output)
- Battery current sensing (analog output)
- Overcurrent protection
- Input filtering and transient protection

---

## Thermal Management

### Critical Components

**U3 (TPS62130RGTR) - Buck Converter**
- **Power Dissipation:** 0.5-1W at full load
- **Thermal Solution:** Exposed pad soldered to PCB, thermal vias to ground plane
- **Junction Temperature Max:** 125°C
- **PCB Requirements:** 2oz copper or heavier, adequate ground plane

**U2 (TPS2113A) - Power MUX**
- **Power Dissipation:** 0.2-0.5W (I²R losses in internal FETs)
- **Thermal Solution:** PCB copper pour for heat spreading
- **Current-dependent:** Increases with load current

**L1 (Inductor)**
- **Power Dissipation:** 0.1-0.3W (DCR losses)
- **Thermal Solution:** Adequate copper clearance, airflow if enclosed

### PCB Design Considerations

- **Copper weight:** 2oz or heavier on power planes
- **Thermal vias:** Under QFN packages for heat extraction
- **Trace width:** Keep high-current traces short and wide (minimize resistance)
- **Component spacing:** Adequate spacing for airflow in enclosed designs
- **Ground plane:** Solid pour for thermal dissipation and low impedance

---

## Testing and Validation

### Power-On Validation Sequence

1. **5V_SYS Test**
   - Connect Pixhawk power module
   - Verify 4.9-5.1V with no load
   - Test automatic switchover (remove external power, USB should take over)
   - Success: Seamless transition, no brownout

2. **3V3_DIG Regulation**
   - Measure with no load: 3.25-3.35V
   - Apply resistive load test: 0.5A, 1.0A, 1.5A
   - Verify regulation maintained at all load points
   - Measure ripple: <50mV p-p

3. **3V3_ANA Quality** (if implemented)
   - Measure with oscilloscope, 20MHz bandwidth limit
   - Ripple: <1mV p-p
   - RMS noise: <10µV (requires spectrum analyzer or low-noise scope)

4. **VDD33_USB Presence**
   - Verify 3.25-3.35V present
   - Required for STM32H743 operation

5. **Thermal Performance**
   - Run full load for 1 hour
   - Thermal imaging: All components <80°C ambient, <100°C with airflow
   - Monitor for thermal throttling or instability

### Battery Monitoring Calibration

**Voltage Calibration:**
1. Connect known voltage source to power module input (e.g., bench supply at 12.0V)
2. Read BAT_VOLTAGE signal with STM32 ADC
3. Calculate scaling factor: `V_batt = ADC_reading * scale_factor`
4. Test across battery voltage range (2S: 7.4V, 3S: 11.1V, 4S: 14.8V)
5. Verify linearity and accuracy

**Current Calibration:**
1. Apply known current load (use calibrated electronic load)
2. Read BAT_CURRENT signal with STM32 ADC
3. Calculate scaling factor: `I_batt = ADC_reading * scale_factor`
4. Test linearity across current range (0-3A)
5. Store calibration values in flash memory

### Efficiency Measurement

**Test Conditions:** Typical operating load (~1A total consumption)

**Measurements:**
- Input power from Pixhawk power module (voltage × current)
- Output power on all rails (sum of 5V_SYS, 3V3_DIG loads)
- Calculate system efficiency: `η = P_out / P_in × 100%`

**Target:** >70% overall system efficiency (5V input to 3.3V outputs)

**Typical Results:**
- Buck converter: 75-85% efficient
- Power MUX: 95-98% efficient (low voltage drop)
- System losses: Wiring, connectors, inductor DCR

---

## Known Issues and Future Improvements

### Current Design Status

**Confirmed Implementation:**
- ✅ TPS2113A power MUX (U2)
- ✅ TPS62130RGTR buck converter (U3)
- ✅ Pixhawk power module interface (JST GH connector)
- ✅ Battery voltage/current monitoring (ADC inputs)
- ✅ ESD protection on USB (TSD05DYFR)

**Unclear / TBD:**
- ⚠️ VDD33_USB source and filtering (not clearly defined)
- ⚠️ External peripheral connector details (GPS, telemetry)

### v1.1 Improvements

**Priority: HIGH**
- Confirm VDD33_USB connection to 3V3_DIG
- Add test points on all major power rails for debug visibility

**Priority: MEDIUM**
- Implement power sequencing with enable signals (staged startup)
- Add current sense resistor on 5V_SYS for onboard consumption monitoring
- LED indicators for each power rail (visual debug aid)
- Evaluate ferrite bead ratings for analog rail filtering

**Priority: LOW**
- Consider reverse polarity protection on power module input
- Add switchable power domains for peripherals

### v2.0 Considerations

**If Higher Power Needed:**
- Upgrade to higher-current connector (XT30/XT60) for >3A applications
- Dual power module inputs for redundancy
- Hot-swap capability for power module replacement

**Enhanced Monitoring:**
- I²C-based power monitoring IC (INA226) for real-time telemetry
- Software-controlled power switches for peripheral management
- Per-rail current and voltage monitoring

---

## References

### Schematics
- `hardware/KiCad/FC_v1.0/FC-proto.kicad_sch` - Main schematic
- `hardware/KiCad/FC_v1.0/FC-power-section.kicad_sch` - Power subsystem detail

### Configuration
- `config/power.yaml` - Comprehensive power subsystem specification

### Component Documentation
- `docs/hardware/power-subsystem/power-mux.md` - TPS2113A detailed documentation
- `docs/hardware/power-subsystem/buck-converter.md` - TPS62130RGTR detailed documentation

### Datasheets
- TPS2113A: `resources/datasheets/POWER/TPS2113A.pdf`
- TPS62130: `resources/datasheets/POWER/TPS62130.pdf`
- TSD05DYFR: `resources/datasheets/ESD/TSD05DYFR.pdf` (USB ESD protection)

### External Standards
- [Pixhawk Connector Standard](https://docs.px4.io/main/en/assembly/pixhawk_connector_standard.html)
- [Pixhawk Hardware Standard](https://pixhawk.org/)

---

**Document Version:** 1.0
**Last Updated:** 2025-11-24
**Author:** Flight Controller Development Team
**Status:** First Revision - Validation Phase
