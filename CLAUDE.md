# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a custom flight controller hardware design project centered around the STM32H743ZIT6 microcontroller. The repository contains hardware schematics, PCB layouts, firmware configuration, and comprehensive documentation for building a high-performance flight controller.

## Repository Structure

```
├── hardware/
│   ├── KiCad/FC_v1.0/          # KiCad 9.0 schematic and PCB files
│   │   ├── FC-proto.kicad_sch  # Main schematic (hierarchical)
│   │   ├── FC-power-section.kicad_sch
│   │   ├── FC-interfaces_sch.kicad_sch
│   │   ├── FC-Sensors.kicad_sch
│   │   └── FC-proto.kicad_pcb
│   └── cad-layout/             # Custom footprints for specific components
├── firmware/
│   └── STM32 Cube IDE/         # STM32CubeMX .ioc configuration
├── config/
│   ├── pinout.yaml             # STM32H743 peripheral pinout 
│   └── power.yaml              # Power rail specifications
├── docs/
│   ├── hardware/               # Hardware subsystem documentation
│   │   ├── overview.md
│   │   ├── power-architecture.md
│   │   ├── design-decisions.md
│   │   └── stm32-subsystem/
│   └── README.md               # Atomic documentation philosophy
└── resources/
    ├── datasheets/             # Organized by component type (BAR, CAN, IMU, etc.)
    └── manufacturer-notes/
```

## Hardware Architecture

### Core MCU
- **STM32H743ZIT6** (480 MHz Cortex-M7, FPU, DSP)
- All peripheral assignments defined in `config/pinout.yaml`
- Hierarchical schematic design with separate sheets for power, interfaces, and sensors

### Key Sensors
- **ICM-42688-P** (Primary IMU): Low-noise 6-axis IMU with up to 32 kHz ODR, connected via SPI
- **MMC5983MA** (Magnetometer): 18-bit high-precision magnetometer with built-in degaussing
- See `docs/hardware/design-decisions.md` for sensor selection rationale

### Power Architecture
The power system uses a hierarchical rail structure:
- **5V_SYS**: 2.5A (via TPS2113A ideal-diode MUX with priority: EXT_5V > USB)
- **3V3_DIG**: 2A (buck regulator TPS54202DDC for digital logic)
- **3V3_ANA**: 300 mA (LDO TPS7A2033 for analog/sensor clean power)

Critical note: VDD33_USB must be supplied 3.3V even if USB is not actively used.

### Communication Interfaces
(from `config/pinout.yaml`)
- **UART4, UART5**: GPS and telemetry
- **SPI1, SPI2**: IMU and sensor communication
- **I2C1, I2C2**: Peripheral communication
- **FDCAN1, FDCAN2**: CAN bus interfaces (PD0/PD1, PB5/PB6)
- **SDMMC1**: SD card interface (PC8-PC12, PD2)
- **USB_OTG_FS**: USB interface (PA11/PA12)

### Debug Interface
- **SWD**: PA13 (SWDIO), PA14 (SWCLK)
- **SWO**: PB3 (JTDO/TRACESWO)
- **External oscillator**: PH0-OSC_IN, PH1-OSC_OUT

## Working with Hardware Files

### KiCad Schematics
- Project uses **KiCad 9.0** format
- Main schematic: `hardware/KiCad/FC_v1.0/FC-proto.kicad_sch`
- Hierarchical design with subsystem sheets:
  - Power section
  - Interface subsystem (UART, CAN, USB, SD card)
  - Sensor subsystem
- Custom footprints in `hardware/cad-layout/` for components like:
  - STM32H743ZIT6, ICM-42688-P, MMC5983MA
  - Power ICs (TPS2113A, TPS62130RGTR, TPS54202DDC)
  - Connectors (SM03B-GHS-TB, FTSH-105-01-F-DV-K)

### Firmware Configuration
- STM32CubeMX project: `firmware/STM32 Cube IDE/FC-proto-El.ioc`
- Open this file with STM32CubeMX to regenerate initialization code
- Pin configuration should match `config/pinout.yaml`

## Configuration Management

All configurations are stored in YAML format in `config/`:

### pinout.yaml
Defines all STM32H743 peripheral pin assignments. When modifying hardware:
1. The user update source files (KiCad, STM32CubeIDE)
2. this pinout file is updated to match source files


### power.yaml
Power rail specifications (currently empty template). Should contain:
- Voltage rail definitions
- Current budgets
- Regulator specifications

## Workflow and File Modification Guidelines

This project follows a strict workflow hierarchy to maintain consistency across hardware, configuration, and documentation:

### Source Files (User-Modified Only)
The following files should **only be modified by the user** using their dedicated software tools:
- **KiCad files** (`hardware/KiCad/FC_v1.0/*.kicad_sch`, `*.kicad_pcb`): Must be edited in KiCad
- **STM32 Cube IDE files** (`firmware/STM32 Cube IDE/*.ioc`): Must be edited in STM32CubeMX/CubeIDE

**Claude should never directly modify these files.**

### Configuration Files (Claude-Modifiable)
Files in `/config` can be modified and should be updated to reflect changes made in the source files:
- **`config/pinout.yaml`**: Update this after making pin assignment changes in STM32CubeMX or KiCad schematics
- **`config/power.yaml`**: Update this after modifying power architecture in KiCad schematics

### Documentation Files (Claude-Modifiable)
Files in `/docs` should be updated according to changes in `/config`:
- Documentation should reflect the current state defined in YAML configuration files
- When configurations change, corresponding documentation must be updated to match

### Update Flow
```
[User modifies KiCad/STM32CubeMX] → [Update /config/*.yaml] → [Update /docs/*.md]
```

This ensures a single source of truth: the user-controlled hardware design files, with derived configuration and documentation that can be maintained programmatically.

## Documentation Philosophy

From `docs/README.md`:
- **Atomic documentation**: Each subsystem documented independently
- **YAML configs**: All configurations in YAML format
- **LLM-optimized**: Documentation structured for AI consumption

When adding new subsystems:
1. Create separate markdown files for each hardware block
2. Store component specifications in YAML where possible
3. Add datasheets to `resources/datasheets/` in the appropriate category

## Component Datasheets

Organized by category in `resources/datasheets/`:
- BAR (barometer), CAN, CONNECTOR, CRYSTAL, ESD, FUSE
- IMU, LED, MAG (magnetometer), POWER, SD_CARD
- STM32, SWITCH, USB-C

When referencing components, datasheets are always available in these subdirectories.

## Version Control

From `.gitignore`:
- Reference designs excluded: `resources/reference-designs/`
- KiCad lock files (e.g., `~FC-proto.kicad_sch.lck`) should not be committed

## Design Intent

This is a **first revision prototype** focused on:
- High-quality sensor data with minimal noise
- Robust power architecture with clean analog rails
- Low-risk component selection (well-characterized parts)
- Future expansion capability (room for additional IMUs, magnetometers)

The goal is to establish a stable hardware foundation for firmware and control algorithm development, not to maximize features in the first iteration.
