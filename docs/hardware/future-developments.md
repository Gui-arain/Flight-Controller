# Future developments

This flight controller is a first version and we want to add some hardware upgrades in the future

## Power Architecture Improvements

### Current Design (v1.0)
All onboard sensors (ICM-42688-P, MMC5983MA, barometer, VDDA) are powered from a single 3V3_ANA rail via ferrite bead filter (FB1).

**Status:** ✅ Acceptable for first iteration prototyping
**Advantages:**
- Simple architecture, easy to debug
- Good noise isolation from 3V3_DIG switching noise (>20dB @ 1MHz)
- Low voltage drop at current consumption (~2mV @ 20mA)
- Common reference for sensor fusion

**Limitations:**
- ⚠️ Single point of failure (one sensor fault affects entire sensor suite)
- ⚠️ No fault isolation between sensors
- ⚠️ No individual power control/monitoring
- ⚠️ Voltage drop scales with current (GPS would add significant load)

---

### v1.1 Improvements (Recommended)

**Priority: HIGH** - Separate GPS Power Domain

If GPS is added, it should have its own isolated power rail:

```
3V3_DIG → FB1 → 3V3_ANA_SENSORS (IMU, mag, baro, VDDA)
       → FB2 → 3V3_GPS (isolated from sensors)
```

**Rationale:**
- GPS draws 50-150mA (vs. 20mA for all sensors combined)
- GPS has switching RF circuits that generate noise
- GPS is more fault-prone (ESD, antenna issues)
- Prevents GPS faults from affecting critical sensor suite
- Reduces voltage drop on sensor rail

**Implementation:**
- Add second ferrite bead (FB2) from 3V3_DIG
- Add capacitors C_GPS_IN (100nF) + C_GPS_OUT (10µF)
- Route GPS power separately

**Additional v1.1 Improvements:**
- Add per-sensor 10µF bulk decoupling capacitors for isolation
- Add test points on each sensor power pin for debugging
- Document actual ferrite bead part number and DCR specification
- Measure and validate ripple/noise under full load

---

### v2.0 Production Architecture (Best Practice)

**Priority: MEDIUM** - Per-Sensor Power Domains

For production/commercial applications, implement tiered power isolation:

```
3V3_DIG → FB_IMU → 3V3_IMU (ICM-42688-P only, ultra-clean)
       → FB_MAG → 3V3_MAG (MMC5983MA + VDDA, magnetometer-grade)
       → Direct → Barometer (less noise-sensitive, can use 3V3_DIG)
       → FB_GPS → 3V3_GPS (completely isolated)
```

**Benefits:**
- ✅ Complete fault isolation (one sensor failure doesn't cascade)
- ✅ Maximum noise isolation for each critical sensor
- ✅ Individual power monitoring and control capability
- ✅ Meets aerospace/commercial flight controller standards
- ✅ Easier debugging (can power down individual sensors)

**Trade-offs:**
- More complex PCB routing
- Higher BOM cost (~3-4 additional ferrite beads + capacitors)
- Requires more board space
- More test points needed

**Optional Enhancements:**
- Load switches with current limiting per sensor
- I²C-addressable power monitor (e.g., INA226) per rail
- Configurable power sequencing
- Over-current protection per sensor

---

### Testing and Validation Requirements

**v1.0 Validation (Current Design):**
- [ ] Measure 3V3_ANA voltage drop under full load (should be >3.25V)
- [ ] Oscilloscope ripple measurement (<10mV p-p target)
- [ ] IMU noise floor verification (compare to datasheet: 2.8 mdps/√Hz)
- [ ] Magnetometer noise measurement (should be ~0.4mG)
- [ ] Fault testing (verify single-point failure behavior)

**v1.1 Validation (GPS Isolated):**
- [ ] Verify GPS power domain isolation (ripple doesn't couple to sensors)
- [ ] Measure crosstalk between 3V3_ANA_SENSORS and 3V3_GPS
- [ ] GPS fault test (short GPS rail, verify sensors unaffected)

**v2.0 Validation (Per-Sensor Rails):**
- [ ] Individual sensor power-up sequencing verification
- [ ] Fault isolation testing (short each rail independently)
- [ ] Per-sensor current monitoring calibration
- [ ] EMI/EMC testing with all sensors active

---

## Sensor Suite

We in the future might want to add other sensors and integrate them into the PCB design. We detail here the different sensors we want to add by order of priority

### 🧭 IMU

Another IMU could help improve the inertial odometry precision by using sensor fusion techniques

We think the *BMI088* would be best suited and complementary with our current *ICM-42688-P* IMU
It indeed has:
- a robust and reliable design
- good mechanical filtering
- a higher maximum acceleration limit (24g)

**Power Consideration:** With dual IMUs, consider separate power domains for each IMU to prevent cross-coupling and improve fault tolerance.

---

### 🧲 Magnetometer

---

# DShot Protocol on STM32H743ZIT6 - Compatibility with TIM1
## Can You Use the Same Timer and Pins?

---

## 🎯 Short Answer: **YES, but with important differences!**

You **CAN** use TIM1 with the same pins (PE9, PE11, PE13, PE14) for DShot, **BUT**:
- ❌ You **CANNOT** use the standard PWM mode
- ✅ You **MUST** use **DMA** (Direct Memory Access)
- ✅ You'll configure the timer in a **completely different way**

---

## 📊 PWM vs DShot Comparison

| Feature | Standard PWM | DShot |
|---------|-------------|-------|
| **Signal Type** | Analog pulse width | Digital serial protocol |
| **Frequency** | 50Hz (20ms period) | 150kHz, 300kHz, 600kHz, 1200kHz |
| **Resolution** | ~1000 steps (1000-2000μs) | 2048 steps (11-bit) |
| **Latency** | Up to 20ms | < 1ms |
| **Bidirectional** | No | Yes (telemetry) |
| **Timer Mode** | PWM Generation | Output Compare + DMA |
| **Complexity** | Simple | Moderate |
| **ESC Support** | All ESCs | Modern ESCs only (BLHeli_S, BLHeli_32) |

---

## 🔧 Hardware Compatibility: TIM1 + Pins

### ✅ What Stays the Same:

**Pins:**
- PE9 (TIM1_CH1) ✓
- PE11 (TIM1_CH2) ✓
- PE13 (TIM1_CH3) ✓
- PE14 (TIM1_CH4) ✓

**Timer:**
- TIM1 ✓ (Excellent choice - has DMA support on all channels)

**Why TIM1 is PERFECT for DShot:**
- Advanced Control Timer
- All 4 channels have **independent DMA channels**
- High frequency capable (400MHz+ on STM32H7)
- DMA burst mode support

### ❌ What Changes Completely:

| Configuration | PWM Mode | DShot Mode |
|--------------|----------|------------|
| **Timer Frequency** | 1MHz (1μs ticks) | 12.5-37.5MHz (depends on DShot speed) |
| **Prescaler** | 199 | 0-15 (much higher freq) |
| **Period (ARR)** | 19999 (20ms) | 20-80 (one DShot bit) |
| **Output Method** | Hardware PWM | DMA to CCR register |
| **Data Format** | Pulse width | 16-bit frames |

---

## 🚀 DShot Protocol Basics

### What is DShot?

DShot (Digital Shot) is a **digital protocol** that sends throttle commands as serial data:

**DShot Frame Structure:**
```
16 bits total per frame:
├─ 11 bits: Throttle value (0-2047)
├─  1 bit:  Telemetry request
└─  4 bits: CRC checksum

Sent as GCR (Grouped Code Recording) encoding
```

### DShot Speeds:

| DShot Version | Bit Rate | Bit Period | ESC Update Rate |
|--------------|----------|------------|-----------------|
| **DShot150** | 150 kbit/s | 6.67 μs | ~9.4 kHz |
| **DShot300** | 300 kbit/s | 3.33 μs | ~18.75 kHz |
| **DShot600** | 600 kbit/s | 1.67 μs | ~37.5 kHz |
| **DShot1200** | 1200 kbit/s | 0.83 μs | ~75 kHz |

**Most Common: DShot600** (good balance of speed and reliability)

### DShot Bit Encoding:

Each bit is represented by a pulse:
```
Bit '1': |----|____|   (75% duty cycle)
Bit '0': |--|______|   (37.5% duty cycle)

Example for DShot600 (1.67μs per bit):
'1' = 1.25μs HIGH, 0.42μs LOW
'0' = 0.625μs HIGH, 1.045μs LOW
```

---

## ⚙️ TIM1 Configuration for DShot600

### Timer Settings:

**For DShot600 on STM32H743 @ 200MHz APB2 timer clock:**

```c
// Timer frequency calculation for DShot600
// Bit period = 1.67μs
// Need 20 timer ticks per bit (for resolution)
// Timer freq = 20 / 1.67μs ≈ 12 MHz

Prescaler = (200MHz / 12MHz) - 1 = 15
ARR = 20 - 1 = 19  // 20 ticks per DShot bit

// For DShot '1' bit (75% duty cycle):
CCR = 15  // (20 × 0.75 = 15)

// For DShot '0' bit (37.5% duty cycle):
CCR = 7   // (20 × 0.375 = 7.5 ≈ 7)
```

### DMA Configuration:

**This is KEY - DShot REQUIRES DMA!**

For TIM1 on STM32H743:
- **TIM1_CH1**: DMA2 Stream 1 or Stream 6
- **TIM1_CH2**: DMA2 Stream 2 or Stream 7
- **TIM1_CH3**: DMA2 Stream 3 or Stream 6
- **TIM1_CH4**: DMA2 Stream 4 or Stream 7

**Why DMA?**
- You need to update CCR register 16 times per frame (one per bit)
- At DShot600: 16 bits × 1.67μs = ~26μs per frame
- Too fast for CPU to handle in software
- DMA automatically writes the bit patterns to CCR

---

## 🔄 Migration Path: PWM → DShot

### Option 1: Keep Both (Recommended for Testing)

You can switch between PWM and DShot **in software**:

```c
// Use PWM initially
void Setup_PWM_Mode(void) {
    // Configure as shown in your previous setup
    // Prescaler = 199, ARR = 19999
    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
}

// Switch to DShot later
void Setup_DShot_Mode(void) {
    // Stop PWM first
    HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
    
    // Reconfigure timer for DShot
    // Prescaler = 15, ARR = 19
    // Setup DMA
    // Start DMA-based transmission
}
```

### Option 2: Conditional Compilation

```c
#define USE_DSHOT  // Comment out to use PWM

#ifdef USE_DSHOT
    // DShot configuration
#else
    // PWM configuration
#endif
```

---

## 📝 STM32CubeIDE Configuration for DShot

### If You Want to Configure DShot in CubeMX:

**1. Timer Configuration:**
- Timer: TIM1
- Channels: Keep as Output Compare (NOT PWM mode)
- Prescaler: 15 (for DShot600 @ 200MHz)
- ARR: 19
- Channels: CH1-4 as "Output Compare CHx"

**2. DMA Configuration:**

For each channel, add DMA request:

```
TIM1_CH1 → DMA2 Stream 1 or 6
├─ Direction: Memory to Peripheral
├─ Mode: Normal (or Circular for continuous)
├─ Increment: Memory increment enabled
├─ Data Width: Half Word (16-bit)
└─ Priority: High
```

Repeat for CH2, CH3, CH4 with their respective streams.

**3. GPIO Configuration:**
- Pins stay the same (PE9, PE11, PE13, PE14)
- Still configured as TIM1 alternate function

---

## 💻 DShot Implementation Overview

### Basic DShot Code Structure:

```c
// DShot frame buffer (16 bits per motor)
uint16_t dshot_buffer[16];  // For one motor

// Prepare DShot frame
void DShot_Prepare_Frame(uint16_t throttle, uint16_t *buffer) {
    // 1. Build 16-bit frame
    uint16_t frame = (throttle & 0x7FF) << 5;  // 11 bits throttle
    frame |= (0 << 4);  // Telemetry bit
    
    // 2. Calculate CRC
    uint16_t crc = calculate_dshot_crc(frame);
    frame |= (crc & 0x0F);
    
    // 3. Convert to bit patterns
    for (int i = 0; i < 16; i++) {
        if (frame & (0x8000 >> i)) {
            buffer[i] = 15;  // '1' bit = 75% duty
        } else {
            buffer[i] = 7;   // '0' bit = 37.5% duty
        }
    }
}

// Send DShot frame via DMA
void DShot_Send(uint16_t *buffer) {
    HAL_TIM_PWM_Start_DMA(&htim1, TIM_CHANNEL_1, 
                          (uint32_t*)buffer, 16);
}
```

---

## 🎓 When to Use What?

### Use Standard PWM When:
- ✅ Using older/budget ESCs
- ✅ Learning/prototyping
- ✅ Simple applications
- ✅ Want maximum compatibility
- ✅ Don't need fast updates
- ✅ Simpler code is priority

### Use DShot When:
- ✅ Using modern ESCs (BLHeli_S, BLHeli_32)
- ✅ Need fast response (racing drones)
- ✅ Want bidirectional telemetry
- ✅ Need better noise immunity
- ✅ Digital protocol advantages
- ✅ Precision control required

---

## 🔌 ESC Compatibility Check

### How to Know if Your ESC Supports DShot:

**Check ESC specifications:**
- BLHeli_S firmware: Supports DShot ✓
- BLHeli_32 firmware: Supports DShot ✓
- SimonK (old): No DShot support ✗
- Generic ESCs: Usually PWM only ✗

**Common DShot-compatible ESCs:**
- Most modern racing drone ESCs (2018+)
- Holybro Tekko32
- T-Motor F45A/F55A
- Aikon AK32PIN
- iFlight SucceX-E series
- Basically any ESC advertising "DShot support"

**Firmware flash:**
- Many ESCs can be flashed with BLHeli_S/32 to add DShot support

---

## 🚧 Implementation Difficulty

### Complexity Rating:

**Standard PWM: ⭐ (Very Easy)**
```c
// Just 3 lines of code!
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);
// Done!
```

**DShot: ⭐⭐⭐ (Moderate)**
```c
// Need to implement:
- DMA configuration (10+ lines)
- Bit encoding (20+ lines)
- CRC calculation (10+ lines)
- Frame building (20+ lines)
- DMA callbacks (10+ lines)
- Buffer management (10+ lines)
= ~100+ lines of code minimum
```

---

## 📚 Resources for DShot Implementation

### Open Source Implementations:

1. **Betaflight DShot Code** (Reference)
   - GitHub: betaflight/betaflight
   - Very optimized, production-tested
   - STM32F4/F7/H7 support

2. **Cleanflight/iNav**
   - Similar implementations
   - Good learning resource

3. **STM32 DShot Examples**
   - Search GitHub: "STM32 DShot"
   - Many hobbyist implementations

### Recommended Approach:

**Phase 1: Get PWM Working First** ← YOU ARE HERE
- Learn timer basics
- Test with your motors
- Understand ESC behavior
- Build your control algorithms

**Phase 2: Add DShot Later** (if needed)
- Same hardware, different firmware
- Incremental upgrade
- Can keep PWM as fallback

---

## 🎯 Quick Decision Guide

```
Do you need DShot RIGHT NOW?
│
├─ No → Use PWM ✓
│      • Simpler
│      • Works with all ESCs
│      • Get flying faster
│
└─ Yes → Consider if you have:
       ├─ DShot-compatible ESCs? 
       │  ├─ Yes → Proceed with DShot
       │  └─ No → Stick with PWM for now
       │
       ├─ Experience with DMA?
       │  ├─ Yes → Good foundation
       │  └─ No → Steep learning curve
       │
       └─ Need <1ms latency?
          ├─ Yes → DShot beneficial
          └─ No → PWM is fine
```

---

## 🔧 Practical Recommendation

### For Your Project:

**Step 1:** Start with PWM (your current path) ✓
- Get your motors spinning
- Test your control system
- Learn the basics

**Step 2:** Keep hardware compatible (TIM1 + those pins) ✓
- You've already chosen the right timer!
- Pins support both protocols

**Step 3:** Add DShot later if needed
- Same hardware
- Firmware update only
- No rewiring required

**Your current hardware choice is DShot-ready! 🎉**

---

## 📊 Pin Summary: TIM1 on STM32H743ZIT6

### Available TIM1 Pins (You're Using ✓):

| Channel | Your Pins | Other Options | DShot Ready? |
|---------|-----------|---------------|--------------|
| CH1 | **PE9** ✓ | PA8 | Yes ✓ |
| CH2 | **PE11** ✓ | PA9 | Yes ✓ |
| CH3 | **PE13** ✓ | PA10 | Yes ✓ |
| CH4 | **PE14** ✓ | PA11 | Yes ✓ |

All your chosen pins support both PWM and DShot! Perfect choice! 👍

---

## 🎯 Final Answer to Your Question

**Q: Can I use TIM1 with the same pins for DShot?**

**A: YES! ✓**

- ✅ Same timer (TIM1) works great
- ✅ Same pins (PE9, PE11, PE13, PE14) work perfectly
- ✅ Just need different timer configuration
- ✅ Must add DMA (not needed for PWM)
- ✅ Software-only change (no hardware rewiring)

**Your current hardware setup is DShot-ready!**

You made excellent hardware choices that give you flexibility for the future. Start with PWM now, and when/if you need DShot, you can upgrade with just a firmware change.

---

## 💡 Next Steps

**Recommended Path:**

1. **Now**: Finish your PWM implementation ✓
   - Get motors working
   - Test thoroughly
   - Build confidence

2. **Later**: Learn DMA (useful for many things)
   - UART DMA
   - ADC DMA
   - Understanding for DShot

3. **Future**: Implement DShot (if needed)
   - You'll have the foundation
   - Hardware already compatible
   - Incremental upgrade

**No need to worry about DShot now - your hardware choice was perfect!** 🚀
