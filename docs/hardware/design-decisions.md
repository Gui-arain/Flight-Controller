
# Why We Chose ICM-42688-P + MMC5983MA for the First Flight Controller Design

For our first custom flight controller prototype, we selected the **ICM-42688-P** IMU and the **MMC5983MA** magnetometer.
This combination provides the ideal balance of **performance**, **stability**, and **practical integration** for an early hardware revision.

## 🧭 ICM-42688-P — Primary IMU

* **Very low noise**
  Gyro noise: ~2.8 mdps/√Hz
  Accel noise: ~65–70 µg/√Hz
  → Clean inertial data, easier filtering and tuning.

* **High max ODR (up to 32 kHz)**
  Enables low-latency control loops and oversampling.

* **Low power (~0.88 mA)**
  Ideal for embedded systems and efficient power budgeting.

* **Compact (2.5 × 3 mm) and easy SPI integration**
  Simple routing and small footprint.

* **Good software support**
  Supported in many flight stacks and with community drivers.

**Why it’s ideal for a prototype:**
It gives the highest-quality inertial data with minimal integration effort, reducing debugging time on early hardware.

---

## 🧲 MMC5983MA — High-Precision Magnetometer

* **Extremely low noise (~0.4 mG)**
  Much better than typical consumer magnetometers.

* **18-bit resolution**
  Offers highly stable heading estimation.

* **Built-in SET/RESET (degaussing)**
  Removes magnetic offset and clears residual magnetization.
  Reduces drift and improves operation near motors/power traces.

* **Robust to temperature drift**
  Maintains accuracy in varying environments.

**Why it’s ideal for a prototype:**
Gives reliable yaw/heading information without susceptibility to drift or hard-iron effects, simplifying early fusion development.

---

## 🛠️ Why This Combination Is Perfect for the First Revision

* Maximizes **signal quality**, minimizing noise-related debugging.
* Reduces risk — both chips are **well-characterized and widely used**.
* Keeps integration straightforward (simple SPI + I²C/SPI lines).
* Leaves room for future upgrades:

  * Add **BMI088** as a second, vibration-resistant IMU.
  * Add a second magnetometer (e.g., BMM350) for redundancy.
* Allows us to focus on **core PCB, power architecture, and firmware**, not sensor issues.

---

**Summary:**
The ICM-42688-P + MMC5983MA pairing provides a **high-performance, low-noise, and low-risk sensor foundation** to ensure the first flight controller prototype is stable, debuggable, and future-proof.

---

## ⚡ Power Architecture — Pixhawk Compatibility & Design Rationale

For the first flight controller revision, the power architecture was designed around **Pixhawk power module compatibility** to leverage the established drone ecosystem while providing flexible development capability.

### Power Input Design

* **JST GH 6-pin connector (SM06B-GHS-TB)**
  Industry-standard Pixhawk power connector (1A per contact rating)
  → Compatible with off-the-shelf power modules from multiple vendors

* **Pixhawk power module interface**
  Standard pinout: 5V (pins 1-2), GND (pins 5-6), BAT_VOLTAGE (pin 4), BAT_CURRENT (pin 3)
  → Integrated battery monitoring without additional onboard sensing circuitry

* **Dual-input power MUX (TPS2113A)**
  Automatic switching between external power module and USB
  Priority: External 5V > USB (prevents USB overcurrent)
  → Enables development and debugging without battery connection

**Why this design is ideal for first revision:**
- Reduces design complexity by using proven power module ecosystem
- Integrated voltage/current sensing eliminates custom sensor circuits
- USB fallback allows bench testing and firmware development
- Wide availability of compatible modules ensures supply chain flexibility
- Community-familiar interface simplifies troubleshooting and support

---

### Power Regulation Strategy

* **Synchronous buck converter (TPS62130RGTR)**
  Converts 5V_SYS → 3.3V_DIG at 75-85% efficiency
  2A capability with <50mV ripple
  → Minimal heat generation, adequate margin for expansion

* **Separate analog filtered rail (ferrite bead filter)**
  High-frequency noise attenuation >20dB @ 1MHz for 3V3_ANA
  Dedicated filtered power for ICM-42688-P, MMC5983MA, and STM32 VDDA
  → Critical for maintaining sensor noise specifications with minimal voltage drop

* **Ideal-diode power multiplexer (TPS2113A)**
  Automatic source selection with 0.1-0.2V forward drop
  2.5A current capability exceeds first-iteration requirements
  → Seamless power transitions during development

**Why these components:**
- **Buck converter over LDO:** High efficiency reduces thermal load (critical in compact design)
- **Ferrite bead filter for analog rail:** Passive filtering provides adequate noise isolation with minimal voltage drop and cost
- **Separate analog rail:** Power supply noise would degrade ICM-42688-P gyro performance (2.8 mdps/√Hz specification)
- **Ideal-diode MUX over Schottky diodes:** Lower voltage drop improves efficiency, automatic switching eliminates manual selection
- **Well-characterized parts:** TI power components with extensive application notes and proven flight controller heritage

---

### Sensor Power Architecture

**Decision:** All onboard sensors (ICM-42688-P, MMC5983MA, barometer, STM32 VDDA) powered from single 3V3_ANA rail

**Architecture:**
```
3V3_DIG → FB1 (Ferrite Bead) → [C14 100nF, C15 1µF] → 3V3_ANA
                                                           |
                                                           +-- ICM-42688-P (IMU): ~1mA
                                                           +-- MMC5983MA (Mag): ~3mA
                                                           +-- Barometer: ~5mA
                                                           +-- STM32 VDDA: ~10mA
                                                           +-- Future GPS: ~50-150mA
```

#### Rationale for v1.0 (First Iteration)

**Why single rail for all sensors:**

1. **Simplicity and Debug-ability**
   - Single power domain reduces complexity during bring-up
   - One test point (TP1) for monitoring all sensor power
   - Easier troubleshooting (fewer variables)
   - Lower BOM cost (one ferrite bead vs. multiple)

2. **Adequate for Low Current Sensors**
   - Total sensor current: ~20mA (excluding GPS)
   - Ferrite bead voltage drop: ~2mV @ 20mA (negligible)
   - Well within tolerance (3.20-3.40V spec)

3. **Common Reference Benefits**
   - All sensors share same voltage reference
   - Simplifies sensor fusion algorithms (no rail-to-rail drift)
   - Consistent calibration across sensor suite

4. **Good Noise Isolation**
   - Ferrite bead provides >20dB attenuation @ 1MHz
   - 1µF output cap provides transient filtering
   - Adequate for first-iteration validation

5. **First-Iteration Focus**
   - Goal: Validate hardware architecture and firmware
   - Not optimizing for maximum reliability yet
   - Allows testing before committing to complex power scheme

#### Known Limitations and Trade-offs

**⚠️ Critical Limitation: Single Point of Failure**
- If ANY sensor fails (short, latchup, excessive current), ALL sensors lose power
- No fault isolation between different sensor types
- Could cause complete sensor suite failure in flight

**Example Failure Scenarios:**
- Barometer develops short → IMU fails → loss of attitude → flight failure
- GPS ESD event → all sensors brown out → complete system failure
- Sensor I/O latchup during development → debugging difficulty

**⚠️ Voltage Drop Scaling**
- Voltage drop increases with total current
- Adding GPS (50-150mA) would increase drop to 7-20mV
- Still acceptable but reduces margin

**⚠️ No Power Sequencing**
- All sensors power up simultaneously
- No control over startup order
- Cannot disable individual sensors for testing

**⚠️ Mixed-Signal Noise Coupling**
- Sensors have both analog and digital circuits
- Digital I/O switching creates current spikes on shared rail
- Noise couples between sensors through common power
- Particularly concerning for 18-bit magnetometer (0.4mG resolution)

**⚠️ Limited VDDA Isolation**
- STM32 VDDA shares rail with sensors
- Sensor current pulses → voltage drop → ADC reference error
- ~0.15% error acceptable for battery monitoring but limits precision measurements

#### Why This Is Acceptable for v1.0

**✅ Risk Assessment:**
- Low sensor current (20mA) minimizes voltage drop impact
- First iteration is for validation, not production deployment
- Simplicity aids debugging and reduces time-to-first-flight
- Can validate sensor noise performance before committing to complex architecture
- Failure modes are understood and documented

**✅ Test Plan:**
- Measure actual voltage drop under load
- Validate IMU noise floor (2.8 mdps/√Hz spec)
- Verify magnetometer performance (0.4mG noise)
- Document ripple on 3V3_ANA during operation
- Test fault scenarios to understand cascading failures

#### Evolution Path: v1.1 and Beyond

**v1.1 Improvements (HIGH PRIORITY):**

If GPS is added, implement separate GPS power domain:
```
3V3_DIG → FB1 → 3V3_ANA_SENSORS (IMU, mag, baro, VDDA)
       → FB2 → 3V3_GPS (isolated from sensors)
```

**Rationale:**
- GPS draws 7-10x more current than all sensors
- GPS has noisy RF circuits (switching PA, LNA)
- GPS is more fault-prone (ESD, antenna issues)
- **Critical:** Prevents GPS faults from killing sensor suite
- Simple implementation (one additional ferrite bead + caps)

**Additional v1.1 Enhancements:**
- Per-sensor 10µF bulk decoupling for improved isolation
- Test points on each sensor VDD pin
- Document and specify ferrite bead impedance/DCR
- Optional: Add 0Ω resistor jumpers for current measurement

**v2.0 Production Architecture (MEDIUM PRIORITY):**

For aerospace/commercial applications, implement per-sensor isolation:
```
3V3_DIG → FB_IMU → 3V3_IMU (ICM-42688-P only, ultra-clean)
       → FB_MAG → 3V3_MAG (MMC5983MA + VDDA)
       → Direct → Barometer (can use 3V3_DIG, less noise-sensitive)
       → FB_GPS → 3V3_GPS (completely isolated)
```

**Benefits:**
- Complete fault isolation (any sensor failure contained)
- Maximum noise performance per sensor
- Individual power monitoring/control
- Meets commercial flight controller standards

**Trade-offs:**
- 3-4 additional ferrite beads + capacitors (~$2-3 BOM increase)
- More complex PCB routing
- More board space required
- Additional test points and validation effort

**Optional Advanced Features:**
- Load switches with current limiting (e.g., TPS22965)
- I²C power monitors per rail (INA226)
- Configurable power sequencing
- Over-current protection per sensor

#### Comparison Table

| Approach | Fault Isolation | Noise Isolation | Voltage Drop | Complexity | Cost | Best For |
|----------|----------------|-----------------|--------------|------------|------|----------|
| **v1.0: Single 3V3_ANA** | ❌ None | ✅ Good | ✅ Low (2mV) | ✅ Simple | $ | Prototype ✅ |
| **v1.1: GPS Separate** | ⚠️ Partial | ✅ Good | ✅ Low | ⚠️ Medium | $$ | Development |
| **v2.0: Per-Sensor** | ✅ Complete | ✅✅ Excellent | ✅ Minimal | ❌ Complex | $$$ | Production 🏆 |

#### Design Philosophy

**First iteration principle:**
> "Optimize for learning, not for production. Choose simplicity over robustness when validating architecture."

The single 3V3_ANA rail for all sensors embodies this philosophy:
- ✅ Simple enough to debug quickly
- ✅ Good enough to validate sensor performance
- ✅ Flexible enough to evolve based on test results
- ⚠️ Documented limitations guide future improvements

**Evolution based on data:**
Once v1.0 testing reveals actual noise performance, voltage drop, and failure modes, we can make informed decisions about v1.1/v2.0 power architecture investments.

---

### Current Budget and Margins

**First iteration power consumption:**
- STM32H743ZIT6: ~400mA @ 3.3V
- IMU + Magnetometer + Sensors: ~10-20mA @ 3.3V
- Communication interfaces: ~20-50mA @ 3.3V
- SD card (write operations): ~100mA @ 3.3V
- **Total estimated: ~550mA @ 3.3V** (~750mA from 5V rail)

**Available capacity:**
- Pixhawk power module: 2-3A @ 5V
- Onboard consumption: ~1-1.5A
- **Remaining margin: ~1-1.5A (50-75% headroom)**

**Why conservative budget for first iteration:**
Focuses validation on core systems (MCU, sensors, firmware) without external peripheral loading. Adequate margin allows future expansion (additional IMUs, companion computer, telemetry radios) without redesigning power architecture.

---

### Design Trade-offs and Rationale

| Design Choice | Alternative Considered | Rationale |
|--------------|------------------------|-----------|
| Pixhawk connector | XT30 or custom high-current connector | Ecosystem compatibility > raw current capacity for first iteration |
| Buck converter | Linear 3.3V regulator | Efficiency (75-85% vs 66%) reduces heat in compact design |
| Dual-input MUX | Single power source | USB fallback essential for development workflow |
| Ferrite bead filter | LDO or single 3.3V rail | Sensor noise isolation with minimal voltage drop and cost |
| Single 3V3_ANA for all sensors | Per-sensor power domains | Simplicity > fault isolation for first iteration prototyping |
| 2.5A power MUX | Higher-current device | Pixhawk modules typically provide 2-3A, matching MUX capability |

**Key insight:**
The JST GH connector is **not a limitation** — it's a deliberate design choice that prioritizes ecosystem integration over maximum current capability. First-iteration focus is validating hardware/firmware foundation, not supporting high-power peripherals.

---

### Future Expansion Considerations

**v1.1 improvements:**

*Power Architecture (HIGH PRIORITY):*
- Separate GPS power domain (FB2 from 3V3_DIG to 3V3_GPS)
- Per-sensor 10µF bulk decoupling capacitors
- Test points on each sensor VDD pin
- Document ferrite bead part number and specifications (impedance, DCR)

*General Improvements:*
- Add test points on all power rails for debug visibility
- Implement power sequencing with enable signals
- Current sense resistor on 5V_SYS for onboard monitoring
- 0Ω resistor jumpers for per-sensor current measurement

**v2.0 considerations:**

*Sensor Power (MEDIUM PRIORITY - Production):*
- Per-sensor power domains with individual ferrite beads
- Load switches with current limiting (TPS22965 or similar)
- I²C power monitors per sensor rail (INA226)
- Over-current protection per sensor

*System Power (if higher power needed):*
- Evaluate XT30/XT60 connector for >3A applications
- Dual power module inputs for redundancy
- Software-controlled power domains for peripheral management

---

**Summary:**
The power architecture balances **proven standards** (Pixhawk compatibility), **efficiency** (synchronous buck conversion), and **sensor performance** (clean analog rail) to create a low-risk first revision that integrates seamlessly with the drone ecosystem while leaving room for future expansion.

---


