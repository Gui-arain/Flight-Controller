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


