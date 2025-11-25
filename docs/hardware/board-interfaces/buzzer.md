# External Buzzer Interface

## Overview

The flight controller provides an external buzzer interface for audio feedback (arming status, battery warnings, GPS lock, failsafe alerts, etc.). The interface uses a MOSFET driver circuit to safely switch buzzer loads that exceed the STM32 GPIO current capability.

## Design Requirements

- **Load current**: Support buzzers drawing 20-100mA (exceeds STM32 GPIO limit of ~25mA)
- **Logic compatibility**: 3.3V GPIO control from STM32H743
- **Buzzer type**: Active buzzer (built-in oscillator) - standard for FPV flight controllers
- **Supply voltage**: 5V or 3.3V depending on buzzer model
- **Simple control**: GPIO on/off switching (no PWM tone generation required)

## Buzzer Type Selection

### Active Buzzer (Recommended)
- **Description**: Self-contained buzzer with built-in oscillator circuit
- **Control**: Simple DC on/off (GPIO high/low)
- **Advantages**:
  - Standard for FPV flight controllers
  - Simple firmware implementation (no timer/PWM required)
  - Loud output (85-100 dB typical)
  - Single fixed frequency (adequate for status alerts)
- **Recommended models**:
  - KEPO KPEG110 (5V, common in flight controllers)
  - Mallory Sonalert series (3.3V-5V compatible)

### Passive Buzzer (Not Recommended)
- **Description**: Electromagnetic coil with diaphragm, no internal oscillator
- **Control**: Requires PWM signal at audio frequency (2-4 kHz)
- **Why avoided**:
  - Requires timer peripheral and PWM tone generation
  - More complex firmware
  - No practical benefit over active buzzers for FPV applications
  - Incompatible with standard flight controller firmware (Betaflight, iNAV, ArduPilot)

**Note**: FPV flight controller ecosystem universally uses active buzzers.

## Circuit Design

### Schematic

```
STM32 GPIO ──[R1: 1kΩ]──┐
                        ├── Gate (Q1: N-MOSFET)
              [R2: 10kΩ]─┤
                    └─── GND

+5V_SYS ─────────┬
           Buzzer+ (Active buzzer)
           Buzzer- ────┤
                  Drain (Q1)
                  Source ── GND

Optional: [D1: 1N4148] across buzzer (cathode to +5V)
```

### Component Selection

#### Q1: N-Channel MOSFET
**Critical parameters**:
- **V_GS(th) < 1.5V**: Logic-level compatible with 3.3V GPIO drive
- **V_DS ≥ 20V**: Adequate margin for 5V supply (4x safety factor)
- **I_D ≥ 200mA**: 2x margin for typical buzzer current
- **R_DS(on) < 1Ω @ V_GS=3.3V**: Minimal power dissipation
- **Package**: SOT-23 (compact, easy assembly)

**Recommended MOSFETs**:

| Part Number | V_GS(th) | V_DS | I_D | R_DS(on) @ 3.3V | Notes |
|-------------|----------|------|-----|-----------------|-------|
| **AO3400A** | 0.9V | 30V | 5.8A | 0.05Ω | Excellent, common |
| **IRLML2502** | 0.9V | 20V | 4.2A | 0.07Ω | Excellent, common |
| BSS138 | 1.5V | 50V | 220mA | ~3.5Ω | Adequate |

**Parts to avoid**:
- **2N7002**: V_GS(th) = 2.1V (marginal at 3.3V drive)
- **IRF540, IRFZ44N**: Standard-level MOSFETs (require 10V gate drive)

#### Resistors
- **R1 (Gate resistor)**: 1kΩ
  - Limits current from GPIO to MOSFET gate
  - Standard value for logic-level MOSFET drive

- **R2 (Pull-down)**: 10kΩ
  - Ensures MOSFET off state during MCU boot/reset
  - Prevents buzzer from sounding during power-up sequence
  - Standard pull-down value for 3.3V logic

#### D1: Flyback Diode (Optional)
- **Part**: 1N4148 or similar fast switching diode
- **Purpose**: Suppresses inductive kickback if buzzer has inductive component
- **Orientation**: Cathode to +5V, anode to buzzer-
- **Note**: Most active buzzers have low inductance; may be omitted if space-constrained

### Power Supply Selection

**Option 1: 5V_SYS (Recommended)**
- Most active buzzers rated for 5V
- Louder output than 3.3V operation
- Sufficient current available from 5V_SYS rail (2.5A capacity)

**Option 2: 3V3_DIG**
- Use if buzzer is 3.3V-rated
- Quieter output but adequate for most applications
- Reduces power consumption slightly

## Pin Assignment

**Recommended GPIO**: Any available GPIO pin (timer capability not required)

**Suggested pins** (not currently allocated in `config/pinout.yaml`):
- **PE5** (TIM15_CH1) - available, can use as GPIO
- **PE6** (TIM15_CH2) - available, can use as GPIO
- **PA15** - available
- **PB11** - available

**Rationale**: Timer functionality not needed for active buzzer (simple on/off), but using timer-capable pin provides flexibility.

**Firmware control**:
```c
// Buzzer on
HAL_GPIO_WritePin(BUZZER_GPIO_Port, BUZZER_Pin, GPIO_PIN_SET);

// Buzzer off
HAL_GPIO_WritePin(BUZZER_GPIO_Port, BUZZER_Pin, GPIO_PIN_RESET);
```

## External Connector

**Connector type**: 2-pin JST-GH or 0.1" header

**Pinout**:
1. **BUZ+**: Connect to +5V_SYS (or +3V3_DIG)
2. **BUZ-**: Connect to MOSFET drain (switched ground)

**Wire gauge**: 26-28 AWG adequate (low current)

**Connector options**:
- **JST-GH 2-pin** (SM02B-GHS-TB): Matches common FPV connectors
- **0.1" header**: Simple, universal, easy prototyping

## Design Rationale

### Why MOSFET Driver?
- STM32H743 GPIO absolute maximum: 25mA source/sink
- Typical active buzzers: 20-100mA current draw
- Direct GPIO drive would violate absolute maximum ratings
- MOSFET provides clean isolation and high current capability

### Why N-Channel Low-Side Switching?
- Simpler than high-side switching (no charge pump needed)
- Logic-level N-channel MOSFETs widely available
- Ground-referenced gate drive from 3.3V GPIO
- Common ground between STM32 and buzzer

### Why Active Buzzer?
- Industry standard for FPV flight controllers
- Firmware simplicity (GPIO bit manipulation only)
- Compatible with existing flight controller software
- No need to allocate timer peripheral for tone generation

## PCB Layout Considerations

- Place MOSFET close to connector to minimize trace length
- Gate resistor (R1) should be close to MOSFET gate pin
- Pull-down resistor (R2) can be placed anywhere on gate trace
- Route buzzer supply trace with adequate width for current (10-20 mil minimum)
- Ground connection should be solid (use ground plane)

## Power Budget

**Typical current draw**:
- Active buzzer @ 5V: 30-60mA typical
- MOSFET quiescent: <1µA
- Gate resistor: ~3mA during switching (transient)

**Worst-case power dissipation**:
- Buzzer: 5V × 60mA = 300mW (buzzer itself)
- MOSFET: I² × R_DS(on) = (60mA)² × 0.05Ω = 0.18mW (negligible)
- Gate resistor: negligible
- Total system impact: ~300mW (minimal)

## Testing and Validation

**Bring-up checklist**:
1. Verify MOSFET pinout (gate/drain/source) before soldering
2. Measure gate voltage with GPIO low: should be <0.1V (pull-down working)
3. Measure gate voltage with GPIO high: should be ~3.3V
4. Connect buzzer and verify switching with GPIO toggle
5. Check for excessive MOSFET heating (should be cool to touch)

**Common issues**:
- **Buzzer always on**: Check pull-down resistor, verify GPIO configured as output
- **Buzzer doesn't work**: Verify MOSFET pinout, check for cold solder joints
- **Buzzer too quiet**: Use 5V supply instead of 3.3V, verify buzzer current rating

## Future Enhancements

- **Passive buzzer support**: Allocate timer channel if tone generation desired
- **PWM volume control**: Use PWM duty cycle to adjust active buzzer volume
- **Dual buzzer**: Add second buzzer for redundancy or different tones
- **Current sensing**: Add inline current sense for buzzer health monitoring

## References

- STM32H743ZI Datasheet: GPIO electrical characteristics (Table 84)
- AO3400A Datasheet: Logic-level MOSFET specifications
- FPV drone buzzer standards: Active buzzer preference in flight controller community
