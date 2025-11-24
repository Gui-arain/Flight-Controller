# Power Architecture

## Purpose
Provide clean, reliable power to STM32H743, IMU, MAG, BAR, and external peripherals.

## Rails
- +5V : direct output of the power connector
- 5V_SYS (post-MUX): up to 2.5 A
- 3V3_DIG (buck): 2 A

## Regulators
- Buck: TPS54202DDC
- LDO: TPS7A2033

## Power MUX
Ideal-diode MUX TPS2113A
Priority: EXT_5V > USB 5V

## Notes
- VDD33_USB must be fed 3.3V even if USB not used.
- GPS delivered with 3.3V rail through ferrite bead.

## Related Files
- KiCad sheet: FC-power-section.kicad_sch
- Config: ../../config/power.yaml
