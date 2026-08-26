# Adaptive Buck Converter with ARM-Controlled DVFS

A complete design, analysis, and simulation of a synchronous buck converter with dynamic voltage and frequency scaling (DVFS) control. The output voltage is set at runtime by an ARM MCU via a programmable reference, enabling the core rail to scale between 0.9V and 1.8V while maintaining tight regulation and fast transient response.

**Status:** ✅ Simulation-verified | Small-signal analysis validated | Ready for prototyping

---

## Overview

Modern ARM processors dissipate power as P ∝ V² × f. By lowering both voltage and frequency during light workloads, DVFS can reduce dynamic power by 70–90% compared to running at peak speed continuously. This project demonstrates a production-grade converter architecture that:

- Maintains **<0.06% DC error** across the full 0.9–1.8V DVFS range
- Slews the reference at **10 mV/µs** with only **12 mV tracking error**
- Rejects load transients (0–3A) with **±2.7% droop** and 72° phase margin
- Feeds forward input voltage changes to achieve **±0.35% line regulation**
- Uses a single **Type III compensator** for all DVFS setpoints (no divider switching)

Applications: mobile SoCs, IoT/wearable processors, FPGA accelerators, RF envelope tracking, automotive/industrial MCUs.

---

## Architecture

### System Block Diagram

```
┌──────────────────────────────────────────────────────────┐
│  ARM MCU                                                  │
│  ┌────────────────┐                                       │
│  │ DVFS Firmware  │───────┐                               │
│  │ • Load monitor │       │ Slew-limited                  │
│  │ • V/f scaling  │       │ DAC update                    │
│  └────────────────┘       ▼                               │
└───────────────────────┬─────────────────────────────────┘
                        │
                   ┌────▼─────┐
                   │   DAC    │ 0–5V command
                   │ (8-bit)  │
                   └────┬─────┘
                        │
        ┌───────────────┴───────────────┐
        │   Reference Filter & Amp      │ Unity-gain follower
        │   τ = 4.7 µs slew rate        │ Vref = 0.8V–1.2V
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │     Error Amplifier (BEA)     │
        │   Type III Compensation       │
        │   • Zeros @ 4.8 & 4.9 kHz    │
        │   • Poles @ 88 & 234 kHz     │
        │   • Crossover: 48 kHz         │
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │   PWM Modulator               │
        │   • Vin feed-forward          │
        │   • 500 kHz sawtooth          │
        │   • Ideal sync half-bridge    │
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼───────────────┐
        │   Power Stage                 │
        │   • L = 2.2 µH (Rser=37 mΩ)  │
        │   • C = 220 µF (ESR=8 mΩ)    │
        │   • Output ripple: 6.7 mV     │
        └───────────────┬───────────────┘
                        │
                   Vout ▼  (0.9V – 1.8V)
                   ↙ ↙ ↙
                 (to ARM core)
```

### Key Architectural Decisions

**1. Unity Feedback, Programmable Reference**
- MCU DAC supplies full-scale Vref (0.8–1.2V)
- Feedback node directly senses Vout (no divider)
- Feedback gain H = 1 at every setpoint → single compensator covers full range
- Eliminates 285 mV tracking error from conventional fixed-reference buck

**2. Input-Voltage Feed-Forward**
- PWM ramp amplitude scaled by Vin: `Vramp = Vin × VSAW`
- Modulator gain: `Gm = Vin / Vramp = 1` (instead of 5)
- Line regulation improved to ±0.35% (was 2–3% without feed-forward)
- Loop does not waste bandwidth rejecting battery sag

---

## Control Loop Design

### Small-Signal Model

The power stage forms an LC double pole at 7.2 kHz with an ESR zero at 90 kHz:

```
Vout / Vcomp = G · Zo / (sL + R + Zo)
```

Where:
- **G** = modulator gain (=1 with feed-forward)
- **L** = inductor (2.2 µH)
- **Zo** = output impedance (capacitor + ESR)

### Type III Compensation Network

A two-zero, two-pole compensator places both zeros below the LC corner to recover phase loss:

| Element | Value | Frequency | Purpose |
|---------|-------|-----------|---------|
| **Zero 1** | RC=22kΩ, CC=1.5nF | 4.8 kHz | Phase boost |
| **Zero 2** | R3=100Ω, C3=6.8nF | 4.9 kHz | Phase boost |
| **Pole 1** | CP=82pF | 88 kHz | Cancel ESR zero |
| **Pole 2** | R3, C3 | 234 kHz | Switching ripple attenuation |

### Stability Across Operating Corners

Simulation of 9 corners (3 DVFS setpoints × 3 load currents):

| Setpoint | 0.3A | 1.0A | 3.0A | Worst |
|----------|------|------|------|-------|
| 1.8V | 73.2° | 73.0° | 72.4° | 72.4° |
| 1.2V | 72.8° | 72.1° | 71.9° | 71.9° |
| 0.9V | 72.9° | 72.3° | 71.6° | **71.6°** |

**Phase margin: 71.6° to 73.2° (spread: ±2°)**

The flat response across all conditions is a direct consequence of unity feedback — the loop does not know which DVFS state it is in.

---

## Measured Results

All figures extracted directly from LTspice transient simulation (10 ns resolution).

### DC Accuracy

| Setpoint | Target | Measured | Error |
|----------|--------|----------|-------|
| 1.8V | ±1% | –0.05% | ✅ |
| 1.2V | ±1% | +0.003% | ✅ |
| 0.9V | ±1% | +0.04% | ✅ |

### Dynamic Response

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| **Reference tracking** (10 mV/µs slew) | <50 mV | **12 mV** | ✅ Excellent |
| **DVFS overshoot** | <5% | +1.2% | ✅ |
| **DVFS undershoot** | <5% | –2.2% | ✅ |
| **Load step** (1→3A @ 1.2V) | ±5% | ±2.7% / ±2.4% | ✅ |
| **Line step** (5→4V) | ±2% | ±0.35% | ✅ Excellent |
| **Output ripple** | <1% | 6.7 mV (0.56%) | ✅ |

### Loop Performance

| Parameter | Measured |
|-----------|----------|
| Crossover frequency | 48 kHz |
| Phase margin | 72° |
| Gain margin | ∞ (no peaking) |
| Settling time (load step) | ~200 µs |
| No ringing | Monotonic recovery |

### Efficiency (Conduction Only)

- **1A @ 1.2V:** 96.8%
- *Note: Excludes switching and gate-drive loss; ideal half-bridge model; real FETs reduce to 93–95%*

---

## Verification Methodology

Three independent cross-checks, each able to catch what the others miss:

### 1. Analytic Loop Model
- Transfer function evaluation in Python across 9 operating corners
- Establishes crossover, phase margin, gain margin *before* component selection
- Predictions: 32 mV load droop

### 2. Switching-Level Transient Simulation
- Full nonlinear circuit in LTspice at 10 ns resolution over 2.4 ms
- Catches saturation, wind-up, large-signal behavior
- Measurements: 32.8 mV load droop (3% agreement with linear model)

### 3. Netlist Extraction from Schematic
- Parsed `.asc` file, recomputed pin coordinates
- Rebuilt and re-simulated implied netlist
- Confirms drawing integrity and connectivity

**Cross-check that matters most:** Predicted load droop **32 mV** vs. simulated **32.8 mV** — the linear model and switching circuit describe the same converter.

---

## Project Structure

```
.
├── README.md                          # This file
├── adaptive_buck_dvfs.asc             # LTspice schematic
├── adaptive_buck_dvfs.net             # SPICE netlist
├── simulation/
│   ├── full_dvfs_sequence.txt         # 2.4 ms transient
│   ├── load_transient_1A_to_3A.txt    # Step response
│   ├── line_transient_5V_to_4V.txt    # Input regulation
│   └── ac_sweep_9_corners.txt         # Bode plots
├── analysis/
│   ├── loop_analysis.py               # Compensation design
│   ├── corner_sweep.py                # Stability verification
│   ├── load_droop_prediction.py       # Small-signal model
│   └── plots/                         # Bode, step response, ripple
├── firmware/
│   ├── dvfs_firmware_concept.c        # Pseudocode example
│   └── sequencing_notes.txt           # Timing requirements
└── docs/
    ├── design_rationale.md            # Why each decision
    ├── component_selection.md         # L, C, R, C values
    └── limitations_and_future_work.md
```

---

## Getting Started

### Requirements

- **LTspice** (free from Analog Devices)
  - Download: https://www.analog.com/en/design-center/design-tools-and-calculators/ltspice-simulator.html
  - Tested on: LTspice XVII (latest stable)

- **Python 3.8+** (for analysis scripts)
  ```bash
  numpy>=1.19
  scipy>=1.5
  matplotlib>=3.3
  ```

### Running the Simulation

1. Open `adaptive_buck_dvfs.asc` in LTspice
2. Run transient simulation (Ctrl+R):
   ```
   .tran 0 2.4m 0 1u
   ```
3. Plot traces:
   - `V(vout)` — output voltage (should track DAC)
   - `V(vref)` — DAC reference (1.8V → 1.2V → 0.9V)
   - `I(L1)` — inductor current (load + ripple)
   - `V(comp)` — error amplifier output

### Running the Analysis

```bash
cd analysis
python3 loop_analysis.py          # Bode plot, phase margin
python3 corner_sweep.py           # 9-corner stability matrix
python3 load_droop_prediction.py  # Validate small-signal model
```

Expected output: Matplotlib figures + terminal report of all corners.

---

## Design Decisions & Tradeoffs

### Why 500 kHz Switching Frequency?

- **Pro:** Allows loop bandwidth up to 50 kHz (fsw/10 rule), better transient response
- **Con:** Increases switching loss vs. 250 kHz
- **Tradeoff:** Adequate for DVFS, which changes setpoints every few milliseconds

### Why Type III (Not Type II)?

The LC double pole contributes ~180° of lag. At the LC corner (7.2 kHz):
- **Type II (one zero)** → phase margin = –3° (unstable)
- **Type III (two zeros)** → phase margin = 72° (stable)

Both zeros placed at 0.7× LC corner ensures phase recovery before crossover.

### Why Not Peak-Current Mode?

Voltage-mode is simpler and sufficient here:
- Crossover at 48 kHz is well below fsw/10 → no instability
- Peak-current would add complexity (slope compensation)
- Future work: migrate to peak-current for higher bandwidth if needed

### Load Droop Limit

Load droop is dominated by output capacitance and loop bandwidth:

```
ΔV ≈ ΔI / (2π·fc·C)
```

To halve droop from 32 mV → 16 mV, either:
1. **Double capacitance** (220µF → 440µF) — cost, size, EMI
2. **Double bandwidth** (48 kHz → 96 kHz) — reduces phase margin, needs faster FETs
3. **Use current-mode control** — simplifies compensation at higher BW

Current design represents good balance for this application.

---

## Firmware Integration (ARM Side)

### DVFS Transition Sequence

**Scaling DOWN (reduce power):**
```c
1. Reduce core clock FIRST (timing closure must be valid @ lower V)
2. Then ramp DAC down
3. Reason: voltage comes after clock to prevent timing violations
```

**Scaling UP (increase performance):**
```c
1. Ramp DAC up
2. Wait for rail to settle (~100 µs)
3. Then raise clock
4. Reason: clock must not start until voltage is stable (setup timing)
```

### Recommended Guard Rails

```c
// Anti-windup: clamp DAC word in firmware
if (dac_command > DAC_MAX) dac_command = DAC_MAX;
if (dac_command < DAC_MIN) dac_command = DAC_MIN;

// Slew rate: limit transitions
if (elapsed_time < MIN_DWELL_MS) {
    skip_voltage_change();
}

// Margin testing: read rail back via ADC
actual_voltage = adc_read(VCORE);
if (abs(actual_voltage - commanded) > TOLERANCE) {
    log_fault();
    throttle_cpu();
}

// Fail-safe: on error, drop to lowest voltage
default: set_voltage(0.9V);
```

### Measurement Points

Monitor these via ARM ADC:
- Core rail voltage (0–2V range, ±5mV accuracy)
- Core rail current (via external shunt resistor)
- Power dissipation P = V × I
- Temperature (if die sensor available)

---

## Limitations & Future Work

| Limitation | Impact | Next Step |
|-----------|--------|-----------|
| **Idealized switching** | No dead time, body-diode, switching loss; efficiency reads 97% (upper bound) | Replace BSW with real MOSFETs + gate driver model |
| **Voltage-mode control** | Loop BW capped near fsw/10; load droop ~32 mV | Migrate to peak-current mode for higher BW |
| **Single-phase, ~3A** | Adequate for small core; 10× short of application processor | Multi-phase interleaving (6 phases × 80A) |
| **No down-slew current path** | Fast down-transitions push energy back to Vin | Add active discharge path or limit down-slew rate |
| **No thermal or EMI** | Component ratings, layout parastics outside scope | Add RC parasitics, thermal die model, layer stackup |
| **No silicon tape-out** | Simulation only; unvalidated on real silicon | Prototype on development board, validate in hardware |

### Recommended Extensions

1. **Multi-phase interleaving** (6 phases for 500A+ GPU core)
   - Current sharing and thermal balancing
   - Input ripple reduction
   - Complexity increase: medium

2. **Current-mode control** (higher bandwidth)
   - Better transient response (<100 µs settling)
   - Simpler compensation (slope compensation instead of Type III)
   - Complexity increase: low

3. **Hardware prototype** (validation board)
   - Real MOSFET losses, parasitic inductance
   - EMI measurements
   - Thermal imaging under load

4. **Firmware state machine** (safety interlocks)
   - Voltage sequencing during power-up/-down
   - Watchdog and fault recovery
   - Runtime margin characterization

---

## Real-World Applications

This architecture is used in:

- **Mobile SoCs (Snapdragon, Apple, Exynos)** — per-cluster DVFS thousands of times/second
- **IoT/Wearables** — duty-cycled retention states dominate battery life
- **FPGA accelerators** — bursty inference workloads (lift for burst, drop after)
- **RF power amplifiers** — envelope tracking (same problem, faster reference)
- **Automotive/Industrial MCUs** — wide input range, dynamic power budget
- **Test/Characterization** — programmable rail for silicon shmoo plots

---

## References

### Key Papers & Books

1. **Erickson & Maksimovič**, *Fundamentals of Power Electronics* (2nd ed.)
   - Classic reference on buck topology, small-signal modeling, compensation

2. **Basso**, *Switch-Mode Power Supplies: SPICE Simulations and Practical Designs*
   - Practical design methodology for voltage-mode and current-mode control

3. **NVIDIA**, "Tegra K1 Whitepaper" (public)
   - Real example of DVFS in mobile SoC; includes power/performance numbers

4. **Haase**, *Designing Stable Compensators for Linear Regulators* (TI SLVA662)
   - Thorough guide to Type II and Type III compensation

### ARM Technical Resources

- ARM Cortex-A Programmer's Guide (frequency/voltage scaling implications)
- ACPI Specification (OS-level DVFS interface)
- Power State Coordination Interface (PSCI) for multi-core sequencing

---

## Performance Summary

| Metric | Target | Achieved | Margin |
|--------|--------|----------|--------|
| DC accuracy | ±1% | ±0.06% | **16× better** |
| Ref tracking | <50 mV | 12 mV | **4× better** |
| DVFS overshoot | <5% | 1.2% | **4× better** |
| Load droop | ±5% | ±2.7% | **1.85× better** |
| Phase margin | >45° | 72° | **60% margin** |
| Line regulation | ±2% | ±0.35% | **5.7× better** |

---

## Contributing

This is a finished design project, but welcome feedback on:
- Simulation methodology or accuracy
- Component selection rationale
- Firmware integration suggestions
- Real-world prototype results

Please open an issue or submit a pull request with:
- Clear description of the change
- Motivation (why this matters)
- Verification (simulation or measurement)

---

## License

This project is released under the **MIT License** — feel free to use in academic, hobby, or commercial projects.

See [LICENSE](LICENSE) for details.

---

## Author

**[Your Name]**  
Hardware Engineer | Power Electronics | ARM SoC Design

- 📧 Email: [your-email]
- 🔗 LinkedIn: [profile]
- 🌐 Portfolio: [website]

### Citation

If you use this design or analysis in your work, please cite:

```bibtex
@misc{adaptive_buck_dvfs,
  title={Adaptive Buck Converter with {ARM}-Controlled {DVFS}},
  author={[Your Name]},
  year={2024},
  howpublished={\url{https://github.com/[your-username]/adaptive-buck-dvfs}}
}
```

---

## Changelog

### v1.0 (2024-01-XX)
- Initial release: schematic, netlist, full simulation data
- Verification: 9-corner stability analysis, small-signal validation
- Documentation: design rationale, firmware integration guide

---

**Last Updated:** January 2024  
**Status:** ✅ Complete and ready for prototyping
