
This file reports the tests done on the board and the different issues encountered. 
(Version tested: **0.9.5**)

![FC manufactured](imgs/FC-Shirley-manufactured.png)

Three boards of this version were produced and almost fully assembled by the manufacturer JLCPCB.

Parts not assembled:
- GH connectors
- USB-C socket
- SD card socket (back of the board)
- STM32 BOOT Slide switch (back of the board)
- O Ohms jumper resistors (connects the power stage to the rest of the board)

In this document we reffer to each board produced as board A, B and C.
## Power section

### Hardware operations

**Operations done on board A:**
- Soldering of every components (hot air blower + soldering paste)
- Board edges removed
- USB-C not properly soldered -> failed when plugging -> damages on USB power traces
- Power wires soldering
- Voltage test -> ❌  Voltage MUX not working
- Desoldering of jumper resistors
- Voltage test -> ❌  Voltage MUX still not working

**Operations done on board B:**
- Power wires soldering
- Voltage test ->  ✅ Every voltages nominal
- Soldering of jumper resistors (soldering iron)
- Board edges removed
- Voltage test ->  ✅ Every voltages nominal, 🚨 Power led working

**Operations done on board C:**
- Board edges removed
- Power wires soldering
- Voltage test -> ❌  Voltage MUX not working
- USB-C soldered -> power through USB -> ❌  Voltage MUX still not working

### MUX checks

Visual inspection (Microscope):
- No apparent damages on the IC and nearby passives
- Looks properly soldered

![[Capture d’écran 2026-05-28 à 21.54.42.png]]

Measures on board C MUX when powered (through USB):
Pin voltages:
- STAT = 0V
- EN_ = 0V
- VSNS = 0V
- ILIM = 0V
- IN1 = 0V
- OUT = 0V
- IN2 = 5.2V
- GND = 0V

Measures on board C MUX unpowered:
- Output <-> GND = 2 MOhms
- Output <-> IN1 = 2.5 MOhms
- Output <-> IN2 = 2.4 MOhms
=> No unexpected shorts noticed

Output voltage monitoring (osciloscope trigger) when IN2 is powered up:
=> When the rising edge of IN2 is detected -> No voltage change detected on OUT

*Note: 
- The MUX on Board A seems to have the same behavior
- These electrical tests have been conducted without load (power stage only)*

### Conclusion

We still don't know what caused the MUX A and C to fail and B to work. Here are the hypotheses we have aproximatly ordered from the most likely:
- Bad soldering of the IC bottom pad caused by too many vias
- Damages done the the IC during operations (overheating when soldering, board edges removal)
- The power supplied used showed to be faulty when conducting recent tests on board C (2V Pk-Pk, 160Hz instead of 5V DC) -> could have damaged the MUX of board C (can't explain board A)
- Unreliable MUX implementation -> might work or not depending on other factors (we know some capacitors might be undersized)
- Damaged or faulty components from the start

Unfortunaly, no satisfactory explanation has been identified yet.

## Sensor section
## References

[Voltage MUX Datasheet](resources/datasheets/POWER/TPS2113a.pdf)