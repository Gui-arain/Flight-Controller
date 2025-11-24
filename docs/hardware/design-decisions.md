
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

* **Separate analog LDO rail (planned: TPS7A2033)**
  Ultra-low noise <10µV RMS for 3V3_ANA
  Dedicated power for ICM-42688-P, MMC5983MA, and STM32 VDDA
  → Critical for maintaining sensor noise specifications

* **Ideal-diode power multiplexer (TPS2113A)**
  Automatic source selection with 0.1-0.2V forward drop
  2.5A current capability exceeds first-iteration requirements
  → Seamless power transitions during development

**Why these components:**
- **Buck converter over LDO:** High efficiency reduces thermal load (critical in compact design)
- **Separate analog rail:** Power supply noise would degrade ICM-42688-P gyro performance (2.8 mdps/√Hz specification)
- **Ideal-diode MUX over Schottky diodes:** Lower voltage drop improves efficiency, automatic switching eliminates manual selection
- **Well-characterized parts:** TI components with extensive application notes and proven flight controller heritage

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
| Separate analog LDO | Single 3.3V rail | Sensor noise performance justifies additional component |
| 2.5A power MUX | Higher-current device | Pixhawk modules typically provide 2-3A, matching MUX capability |

**Key insight:**
The JST GH connector is **not a limitation** — it's a deliberate design choice that prioritizes ecosystem integration over maximum current capability. First-iteration focus is validating hardware/firmware foundation, not supporting high-power peripherals.

---

### Future Expansion Considerations

**v1.1 improvements:**
- Add test points on all power rails for debug visibility
- Implement power sequencing with enable signals
- Current sense resistor on 5V_SYS for onboard monitoring
- Verify 3V3_ANA LDO implementation (currently unclear in schematics)

**v2.0 considerations (if higher power needed):**
- Evaluate XT30/XT60 connector for >3A applications
- Dual power module inputs for redundancy
- I²C power monitoring IC (INA226) for real-time telemetry
- Software-controlled power domains for peripheral management

---

**Summary:**
The power architecture balances **proven standards** (Pixhawk compatibility), **efficiency** (synchronous buck conversion), and **sensor performance** (clean analog rail) to create a low-risk first revision that integrates seamlessly with the drone ecosystem while leaving room for future expansion.

---


