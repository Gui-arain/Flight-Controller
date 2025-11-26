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

---

## 💡 Next Steps

After configuration:
1. Generate code
2. Add the HAL_TIM_PWM_Start() calls
3. Test with one motor (no propeller!)
4. Gradually increase CCR value to verify motor response
5. Implement your control algorithm

---

## 📚 Additional Resources

**STM32 Documentation:**
- RM0433: STM32H743 Reference Manual (Timer chapter)
- AN4776: General-purpose timer cookbook
- STM32CubeIDE User Guide

**ESC Resources:**
- Most ESCs expect 50Hz (some support 400Hz)
- 1000μs = min, 2000μs = max (standard)
- Always test without propellers first!

---

**Good luck with your motor control project! 🚁**

*Remember: Safety first - always start at minimum throttle and remove propellers during testing!*