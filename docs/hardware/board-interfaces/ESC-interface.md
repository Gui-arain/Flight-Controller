# STM32CubeIDE - Configuring TIM for ESC PWM Control
## Step-by-Step Guide for STM32H743ZIT6

This guide shows you how to configure a timer in STM32CubeIDE to generate 4 PWM signals for ESC motor control.

---

## 📋 Part 1: Opening STM32CubeMX Configuration

### Step 1: Open Your Project
1. Open your project in **STM32CubeIDE**
2. In the Project Explorer, double-click on the **`.ioc`** file
   - Usually named something like `YourProjectName.ioc`
   - This opens the STM32CubeMX graphical configurator

---

## 🎯 Part 2: Selecting and Configuring GPIO Pins

### Step 2: Select Timer Pins in Pinout View

1. You should now see the **Pinout & Configuration** tab
2. Look at the chip diagram in the center

3. **Find and configure the pins for TIM1:**

   **Option A - Click pins directly on chip diagram:**
   - Click on pin **PE9** → Select **TIM1_CH1**
   - Click on pin **PE11** → Select **TIM1_CH2**
   - Click on pin **PE13** → Select **TIM1_CH3**
   - Click on pin **PE14** → Select **TIM1_CH4**

   **Option B - Use the pin assignment tool:**
   - Go to **Pinout view** (left panel)
   - Expand **Timers** → **TIM1**
   - For each channel, click the dropdown:
     - **Channel1** → Select pin **PE9**
     - **Channel2** → Select pin **PE11**
     - **Channel3** → Select pin **PE13**
     - **Channel4** → Select pin **PE14**

4. The pins should now be **green** on the chip diagram, indicating they're configured

### Visual Guide - What You Should See:
```
STM32H743ZIT6 Chip Diagram:
┌─────────────────────────┐
│                         │
│  PE9  (green) ← TIM1_CH1│  Motor 1
│  PE11 (green) ← TIM1_CH2│  Motor 2
│  PE13 (green) ← TIM1_CH3│  Motor 3
│  PE14 (green) ← TIM1_CH4│  Motor 4
│                         │
└─────────────────────────┘
```

---

## ⚙️ Part 3: Timer Configuration

### Step 3: Configure TIM1 in PWM Mode

1. In the **Categories** panel (left side), find and click **Timers** → **TIM1**

2. In the **TIM1 Mode and Configuration** panel, configure each channel:

   **Set Channel Modes:**
   - **Channel 1**: Select **PWM Generation CH1**
   - **Channel 2**: Select **PWM Generation CH2**
   - **Channel 3**: Select **PWM Generation CH3**
   - **Channel 4**: Select **PWM Generation CH4**

   *(Click the dropdown next to each channel and select "PWM Generation CHx")*

### Step 4: Configure Timer Parameters

Now click on the **Parameter Settings** tab (still in TIM1 configuration):

#### Counter Settings:
```
┌─────────────────────────────────────────────┐
│ Parameter              │ Value              │
├─────────────────────────────────────────────┤
│ Prescaler (PSC)        │ 199                │
│ Counter Mode           │ Up                 │
│ Counter Period (ARR)   │ 19999              │
│ Internal Clock Div     │ No Division        │
│ Auto-reload preload    │ Enable             │
└─────────────────────────────────────────────┘
```

**Enter these values:**
- **Prescaler**: `199`
  - *Why?* 200MHz / (199+1) = 1MHz = 1μs tick resolution
- **Counter Period (AutoReload Register)**: `19999`
  - *Why?* 20,000 ticks × 1μs = 20ms = 50Hz (standard for ESC)
- **Counter Mode**: `Up`
- **auto-reload preload**: `Enable`

#### PWM Generation Channel Settings:

For **each channel** (CH1, CH2, CH3, CH4), configure:

```
┌─────────────────────────────────────────────┐
│ Parameter              │ Value              │
├─────────────────────────────────────────────┤
│ Mode                   │ PWM mode 1         │
│ Pulse (CCRx)          │ 1000               │
│ Output Compare Preload │ Enable             │
│ Fast Mode              │ Disable            │
│ CH Polarity            │ High               │
└─────────────────────────────────────────────┘
```

**For Channel 1 (repeat for CH2, CH3, CH4):**
- **Mode**: `PWM mode 1`
- **Pulse (CCR1)**: `1000`
  - *Why?* Starting value = 1000μs = minimum throttle (safe)
- **Output Compare Preload**: `Enable`
- **Fast Mode**: `Disable`
- **CH Polarity**: `High`

### Step 5: Configure Clock Source

1. Still in TIM1 settings, click on **Clock Source** tab
2. Select: **Internal Clock**

### Step 6: Advanced Settings (Important for TIM1!)

TIM1 is an **Advanced Control Timer** and requires additional configuration:

1. Click on **Advanced settings** section (or look for it in the Parameter Settings)

2. Configure **Break and Dead Time**:
   ```
   ┌─────────────────────────────────────────────┐
   │ Off-State Selection for Run    │ Disable   │
   │ Off-State Selection for Idle   │ Disable   │
   │ Lock Configuration             │ OFF       │
   │ Break Input State              │ Disable   │
   │ Automatic Output Enable        │ Disable   │
   └─────────────────────────────────────────────┘
   ```

   **Important**: Make sure **Break Input** is **Disabled** (we don't need it for ESC control)

---

## 🔧 Part 4: Clock Configuration

### Step 7: Verify/Configure System Clock

1. Click on the **Clock Configuration** tab (top of the window)

2. Ensure your **APB2 Timer Clock** is set correctly:
   - Typical for STM32H7: **200 MHz**
   - This is what we used in our prescaler calculation

3. If you need to change it:
   - Adjust the **PLL** settings
   - Adjust **APB2 prescaler**
   - The tool will show you the resulting frequencies

**Target Clock Tree:**
```
HSE (8MHz or 25MHz)
  └→ PLL
      └→ SYSCLK (400MHz typical)
          └→ AHB
              └→ APB2 (200MHz)
                  └→ TIM1 Clock = 200MHz ✓
```

---

## 💾 Part 5: Generate Code

### Step 8: Generate Initialization Code

1. **Save** your configuration: `Ctrl+S` or File → Save

2. Generate code:
   - Click **Project** → **Generate Code**
   - Or press `Alt+K`
   - Or click the **gear icon** ⚙️ in the toolbar

3. A popup will ask if you want to generate code:
   - Click **Yes** or **OK**

4. STM32CubeMX will generate the initialization code in:
   - `main.c` - MX_TIM1_Init() function
   - `stm32h7xx_hal_msp.c` - GPIO and clock initialization

---

## 📝 Part 6: Using the Generated Code

### Step 9: Start PWM in Your Code

The generated code only **initializes** the timer. You need to **start** it!

In your `main.c`, after `MX_TIM1_Init();`, add:

```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_TIM1_Init();        // ← Generated by CubeMX

  /* USER CODE BEGIN 2 */
  
  // START PWM OUTPUT (ADD THIS!)
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);
  HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_4);
  
  /* USER CODE END 2 */

  while (1)
  {
    /* USER CODE BEGIN 3 */
    
    // Control PWM duty cycle here
    
    /* USER CODE END 3 */
  }
}
```

### Step 10: Control Motor Throttle

To change motor throttle, modify the **CCR (Capture Compare Register)**:

```c
// Set Motor 1 to 1500μs (50% throttle)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 1500);

// Set Motor 2 to 1200μs (20% throttle)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, 1200);

// Set Motor 3 to 2000μs (100% throttle)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, 2000);

// Set Motor 4 to 1000μs (0% throttle - idle)
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, 1000);
```

---

## 🎓 Understanding the Configuration

### Why These Values?

**Timer Frequency Calculation:**
```
Timer Input Clock = 200 MHz (from APB2)
Prescaler = 199

Timer Frequency = 200,000,000 / (199 + 1) = 1,000,000 Hz = 1 MHz
Timer Period = 1 / 1,000,000 = 1 μs per tick ✓
```

**PWM Frequency Calculation:**
```
ARR (Period) = 19999
PWM Frequency = Timer Frequency / (ARR + 1)
              = 1,000,000 / 20,000
              = 50 Hz ✓
```

**Pulse Width:**
```
CCR Value = Pulse width in microseconds

Examples:
CCR = 1000 → 1000 μs = 1.0 ms → Minimum throttle
CCR = 1500 → 1500 μs = 1.5 ms → 50% throttle
CCR = 2000 → 2000 μs = 2.0 ms → Maximum throttle
```

---

## 🔍 Verification Checklist

Before testing with motors:

- [ ] All 4 pins (PE9, PE11, PE13, PE14) are green in pinout view
- [ ] TIM1 channels 1-4 set to "PWM Generation CHx"
- [ ] Prescaler = 199
- [ ] Counter Period (ARR) = 19999
- [ ] Initial Pulse (CCR) = 1000 for all channels
- [ ] Break input disabled (for TIM1)
- [ ] Code generated successfully
- [ ] HAL_TIM_PWM_Start() called for all channels
- [ ] Project compiles without errors

---

## 🎯 Quick Reference Table

| Setting | Value | Purpose |
|---------|-------|---------|
| Timer | TIM1 | Advanced timer with 4 channels |
| Pins | PE9, PE11, PE13, PE14 | PWM outputs |
| Prescaler | 199 | Get 1μs resolution |
| ARR | 19999 | 20ms period = 50Hz |
| CCR Initial | 1000 | Start at min throttle (safe) |
| CCR Range | 1000-2000 | Standard ESC range |
| PWM Mode | Mode 1 | Active when CNT < CCR |
| Polarity | High | Active high output |

---

## 🐛 Common Issues & Solutions

### Issue 1: No PWM Output
**Check:**
- Did you call `HAL_TIM_PWM_Start()`?
- Are GPIO clocks enabled? (Should be automatic)
- Is TIM1 clock enabled? (Should be automatic)
- Check with oscilloscope or logic analyzer

### Issue 2: Wrong Frequency
**Check:**
- Verify APB2 timer clock in Clock Configuration tab
- Recalculate prescaler if clock is different from 200MHz
- Formula: `Prescaler = (Timer_Clock / 1,000,000) - 1`

### Issue 3: ESC Not Responding
**Check:**
- Ground connection between STM32 and ESC
- CCR value is in range 1000-2000
- ESC is powered from battery
- Some ESCs need calibration first

### Issue 4: CubeMX Won't Generate Code
**Check:**
- Resolve any warnings (yellow triangles)
- Check for pin conflicts (red pins)
- Save the .ioc file first

---

## 🔧 Alternative Timer Options

If TIM1 pins are not available, you can use other timers:

### TIM2, TIM3, TIM4, TIM5, TIM8
- **TIM2**: General-purpose, 32-bit (more resolution if needed)
- **TIM3, TIM4, TIM5**: General-purpose, simpler than TIM1
- **TIM8**: Another advanced timer (similar to TIM1)

**To use TIM3 instead:**
1. Select TIM3 in Categories
2. Choose 4 pins that support TIM3_CHx
3. Use same prescaler/ARR values
4. No "Break" configuration needed (simpler!)

---

## 📸 Visual Guide Screenshots Description

When configuring, you should see:

**1. Pinout View:**
- Chip diagram with PE9, PE11, PE13, PE14 highlighted in green
- Each pin labeled with TIM1_CH1, TIM1_CH2, etc.

**2. TIM1 Configuration:**
- Left panel: TIM1 selected under Timers
- Right panel: Mode and Configuration
- All 4 channels showing "PWM Generation CHx"

**3. Parameter Settings:**
- Counter Settings section with PSC=199, ARR=19999
- Four channel configurations with CCR=1000

**4. Clock Configuration:**
- Clock tree showing path to TIM1
- APB2 timer clock = 200MHz

## 📚 Additional Resources

**STM32 Documentation:**
- RM0433: STM32H743 Reference Manual (Timer chapter)
- AN4776: General-purpose timer cookbook
- STM32CubeIDE User Guide

**ESC Resources:**
- Most ESCs expect 50Hz (some support 400Hz)
- 1000μs = min, 2000μs = max (standard)
- Always test without propellers first!

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