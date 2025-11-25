# Voltage & Current Measurement (Pixhawk Power Connector → STM32H7)

## 1. Overview

The Pixhawk-style **POWER** connector exposes two **analog** signals:

- **VOLTAGE**: battery voltage via a resistor divider (max ≈ 3.3 V)
- **CURRENT**: load current via a current-sense circuit (max ≈ 3.3 V)

These are read by the STM32H7’s ADC as **single-ended analog inputs**.

> ⚠ The exact scaling (divider ratio & A/V gain) depends on the power module.
> Check its datasheet for numerical values.

---

## 2. Hardware Connections

Assuming use of **ADC1 Channel 2 & 3** (example):

- `POWER.VOLTAGE` → MCU pin mapped to **ADC1_IN2** (Rank 1)
- `POWER.CURRENT` → MCU pin mapped to **ADC1_IN3** (Rank 2)
- GND of the power module tied to MCU analog GND
- Optional small RC filters at the MCU pins (e.g. 1–4.7 kΩ + 100 nF to GND)

All signals stay within **0 – 3.3 V** (ADC reference range).

---

## 3. STM32CubeIDE ADC Configuration (ADC1)

**Global settings**

- Resolution: **12 bits**
- Scan Conversion Mode: **Enabled**
- Continuous Conversion Mode: **Enabled**
- Discontinuous Mode: **Disabled**
- End of Conversion Selection: **End of single conversion**
- Overrun behaviour: **Overrun data preserved**
- Conversion Data Management: **Regular Conversion data stored in DR**

**Regular Conversion Mode**

- Enable Regular Conversions: **Enabled**
- Number Of Conversions: **2**
- External Trigger Source: **Regular Conversion launched by software**
- External Trigger Edge: **None**

**Ranks**

- Rank 1  
  - Channel: **Channel 2** (ADC1_IN2 → VOLTAGE)  
  - Sampling Time: **64.5 Cycles**  
  - Offset: **None**
- Rank 2  
  - Channel: **Channel 3** (ADC1_IN3 → CURRENT)  
  - Sampling Time: **64.5 Cycles**  
  - Offset: **None**

(Other channels: **Disabled**, Single-ended mode.)

---

## 4. Minimal Firmware Flow (Polling Version)

```c
uint16_t adc_voltage_raw = 0;
uint16_t adc_current_raw = 0;

void ADC1_Start(void)
{
    HAL_ADC_Start(&hadc1);               // continuous + scan mode
}

void ADC1_Read(void)
{
    // Rank 1: VOLTAGE
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
    adc_voltage_raw = HAL_ADC_GetValue(&hadc1);

    // Rank 2: CURRENT
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
    adc_current_raw = HAL_ADC_GetValue(&hadc1);
}
````

For higher performance / less CPU load, use **DMA circular mode** with a 2-element buffer instead of polling.

---

## 5. Converting ADC Values

Let:

* `ADC_MAX = 4095` (12-bit)
* `VREF = 3.3f` (ADC reference)
* `V_div_ratio` = battery divider ratio (e.g. 1:10 → 10.0)
* `I_sense_scale` = amps per volt at CURRENT pin (e.g. 36.7 A/V)

Then:

```c
float v_adc  = (adc_voltage_raw / (float)ADC_MAX) * VREF;
float i_adc  = (adc_current_raw / (float)ADC_MAX) * VREF;

float v_batt = v_adc * V_div_ratio;
float i_batt = i_adc * I_sense_scale;
```

Tune `V_div_ratio` and `I_sense_scale` from the power-module documentation
or via calibration (measured voltage/current vs ADC readings).

---

