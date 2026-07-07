# Bandgap Reference (BGR) + LDO — Full Chain Design

A complete analog reference-and-regulation chain in a TSMC 018 (0.18µm) process, built and verified in three stages: a standalone PNP-based bandgap reference, a standalone LDO regulator (tested against an ideal reference), and finally the two connected together as a full chain and re-verified end-to-end.

Every sub-block was independently characterized before being integrated, and the integrated chain was then stress-tested against the same test suite to see what changes (and why) once the ideal reference is replaced with a real, temperature- and supply-dependent one.

## Why Build It In Three Stages

A bandgap reference and an LDO are almost always designed and tested separately before integration, because each has its own failure modes:
- The BGR is judged on **temperature stability** and **startup behavior** (a bandgap has a degenerate zero-current state it must be forced out of).
- The LDO is judged on **line/load regulation, dropout, and transient response**, and is normally characterized against an ideal, noise-free reference so its own loop dynamics aren't muddied by reference imperfections.
- Only after both work independently does it make sense to connect them and ask the harder question: *what does the reference's own temperature drift, startup timing, and finite output impedance do to the regulated output once it's no longer ideal?*

That's exactly the structure of this project: `BGR.asc` → `LDO.asc` → `BGR+LDO_FINAL.asc`.

---

## 1. Standalone Bandgap Reference (`Schematics/BGR.asc`)

### Topology
- Classic PTAT/CTAT PNP bandgap core using a ratioed PNP array (`Q1`–`Q10`, with unit PNPs combined in 8:1 groupings to set the ΔV_BE ratio)
- Self-biased PMOS current mirror: `M1`, `M2`, `M3` (all l=2µ, w=50µ)
- Startup circuit: `M4`, `M5` (NMOS, l=4µ, w=20µ) and `M6`, `M7` (NMOS, l=1µ, w=20µ) — kicks the core out of its degenerate zero-current state on power-up
- Auxiliary PMOS mirror stage: `M8`, `M9`, `M10` (l=1µ, w=80µ)
- Resistors: `R1` = 500k, `R2` = 5.4k, `R3` = 50k

### Tests Conducted (`BGR_Tests/`)

| # | Test | Command | Result | Screenshot |
|---|---|---|---|---|
| 1 | Operating point | `.op` | Baseline bias point for all downstream tests | `Operating Point (.op)` |
| 2 | Temperature coefficient | `.step temp -40 125 5` | V_ref = 1.1529V at −40°C, 1.1512V at 125°C → ΔV ≈ 1.72mV over 165°C → **TC ≈ 10.42 ppm/°C** | `Temperature Coefficient.png` |
| 3 | Line regulation | `.dc V1 1.2 3 0.01` | V_ref: 1.139V (at 2V) → 1.165V (at 3V) → **≈25.88 mV/V** (decent, not great) | `Line Regulation.png` |
| 4 | Worst-case startup (degenerate state) | `.ic V(N004)=0 V(N006)=0 ...` + `.tran 200u`, V1 = PWL(0 0 5u 2.5 200u 2.5) | V_ref rises from 0 and settles to ≈1.15V by ~5µs — startup circuit works as intended | `Worst Case Startup.png` |
| 5 | Normal-ramp startup | V1 = PWL(0 0 1m 2.5), `.tran 1.2m` | Smooth monotonic rise to final V_ref, no overshoot, no oscillation, no glitch through the 2V knee | `Normal Ramp Startup.png` |
| 6 | Power-cycling restart | V1 = PWL(0 0 5u 2.5 80u 2.5 85u 0 90u 0 95u 2.5 200u 2.5), `.tran 200u` | V_ref recovers cleanly after each supply dip, not just first power-up | `Power Cycling Restart.png` |
| 7 | Startup across temperature corners | `.ic` + `.temp` at −40°C / 27°C / 125°C, `.tran 200u` | Settles correctly in all three corners | `-40 C Startup.png`, `27 C Startup.png`, `125 C Startup.png` |
| 8 | PSRR | V1 = PWL ramp + AC 1, `.ac dec 10 1 1e6` | Near 0 dB across the sweep instead of the -30 to -40 dB expected from a well-designed core. The bandgap's current sources (`M8`–`M10`) are simple, uncascoded PMOS mirrors referenced directly off Vdd, so they have low output impedance — supply ripple directly modulates the branch currents, and since V_ref is just those currents times a resistor, V_ref follows Vdd almost 1:1 | `PSRR.png` |

---

## 2. Standalone LDO (`Schematics/LDO.asc`)

Tested against an **ideal 1V DC reference** (`V1`), independent of the BGR above, so the loop's own dynamics can be characterized cleanly.

### Topology
- PMOS pass device `MP` (l=360n, w=2.02m — deliberately huge width to minimize dropout)
- Feedback divider: `R1` = 10k, `R2` = 20k
- Error amplifier: block-level ideal op-amp (`X1`) — this was intentional, to isolate LDO loop behavior from any op-amp non-idealities, with a note left in the schematic questioning how EA gain affects loop speed
- Output capacitor `C1` = 10p, load modeled with a variable current source `I_load`

### Tests Conducted (`LDO_Tests/`)

| # | Test | Command | Result | Screenshot |
|---|---|---|---|---|
| 1 | Line regulation | `.dc V2 1.3 2 0.01` | V_out flat at ≈1.5V once Vdd clears the dropout knee (~1.5V + MP headroom) | `Line_regulation_LDO.png` |
| 2 | Load regulation | `.dc I_load 0 20m 0.1m` | V_out: 1.514V (no load) → 1.500V (20mA) — a few mV shift, good | `Load_regulation_LDO.png` |
| 3 | Line transient | `PWL(0 1.8 100u 1.8 101u 2 ... )`, `.tran 0 25m 0 5u` | V_out shifts only a few hundred µV–mV for Vdd steps between 1.6V–2V | `Line_Transient_LDO.png` |
| 4 | Load transient | `PWL` current steps between 10µA and 20mA, `.tran 0 25m 0 5u` | V_out shifts only a few hundred mV despite ~1000× current swings | `Load_transient_LDO.png` |
| 5 | Dropout voltage | `.dc V2 1.5 1.8 0.005` | **Dropout ≈ 94mV** (regulation lost around V2 ≈ 1.59V) | `Dropout_voltage_LDO.png` |
| 6 | Startup transient | V2 = PWL(0 0 100u 1.8) | Clean startup, no overshoot | `Startup_transient_LDO.png` |
| 7 | PSRR | V2 = 1.8 DC + AC 1, `.ac dec 20 1 10Meg` | See screenshot | `PSRR_LDO.png` |

---

## 3. Full Chain (`Schematics/BGR+LDO_FINAL.asc`)

The real BGR now drives the LDO's error amplifier directly (replacing the ideal 1V source), with the LDO's feedback ratio changed slightly (`R4` = 7k, `R5` = 23k) to hit a 1.5V output from the BGR's ≈1.15V reference. Two independent supply rails are used — `V1` for the BGR, `V2` for the LDO — specifically so startup and line-regulation tests can stress each rail independently.

### Tests Conducted (`BGR_LDO_Combo_Tests/`)

| # | Test | Command | Result | Screenshot |
|---|---|---|---|---|
| 1 | Op-point cross-check | `.op` | V_ref ≈1.15V, V_out1 ≈1.5V — matches both standalone results, confirming correct integration | `Operating point (.op)` |
| 2a | Full-chain startup — simultaneous ramp | `V1 PWL(0 0 1m 2.5)`, `V2 PWL(0 0 1m 1.8)` | V_ref settles well before V_out1; no race condition | `Full_Chain_Startup_Baseline.png` |
| 2b | Full-chain startup — staggered ramp (stress test) | `V1 PWL(0 0 500u 2.5)`, `V2 PWL(0 0 50u 1.8)` (LDO rail 10× faster) | V_out1 correctly tracks V_ref × (R4+R5)/R5 as it rises — stays low while V_ref is still low (correct behavior, not a fault) — no overshoot/ringing/latch-up once V_ref settles | `Full_Chain_Startup_Staggered.png` |
| 3a | Full-chain line regulation — sweep BGR rail | `.dc V1 1.5 3 0.01` | V_out1 slope ≈0.03–0.034 V/V, consistent with V_ref's own 25.9mV/V slope amplified by the feedback divider gain (1.304×) — confirms the LDO faithfully passes through the BGR's residual line sensitivity | `Full_Chain_BGR_Rail_Sweep.png` |
| 3b | Full-chain line regulation — sweep LDO rail | `.dc V2 1.3 2 0.01` | Matches standalone LDO result (same loop, same effective fixed reference) | `Full_Chain_LDO_Rail_Sweep.png` |
| 4 | Full-chain load regulation | `.dc I_Load 0 20m 0.1m` | V_out1 changes ≈−0.7mV/mA (matches standalone LDO); **V_ref stays essentially flat** — the BGR has no direct path to sense LDO load current, only the divider current through R4/R5, which doesn't change with I_load | `Full_Chain_Load_Regulation.png` |
| 5 | Full-chain temperature sweep | `.step temp -40 125 5`, `.op` | V_out1: 1.5038V (−40°C), 1.5055V (27°C, max), 1.5009V (125°C, min) → **combined TC ≈ 18.53 ppm/°C**, noticeably worse than the standalone BGR's 10.42 ppm/°C — the LDO adds its own temperature dependence (likely MP's V_th drift and/or EA input-referred offset drift) on top of the BGR's | `Full_Chain_Temp_Sweep.png` |
| 6 | Full-chain load transient | PWL current profile, `.tran 0 25m 0 5u` | V_out1 transient matches standalone LDO; V_ref is *not* perfectly flat during fast load edges — traced to gate-capacitance coupling from the EA's internal node back onto V_ref through M2's gate capacitance | `Full_Chain_Load_Transient.png` |

---

## Repository Structure

```
Bandgap-Reference-BGR-Low-Dropout-Regulator-LDO-Circuit-/
├── README.md
│
├── Schematics/
│   ├── BGR.asc                          # Standalone bandgap reference
│   ├── LDO.asc                          # Standalone LDO (driven by ideal 1V ref)
│   └── BGR+LDO_FINAL.asc                # Full chain: real BGR driving the LDO
│
├── BGR_Tests/
│   ├── Operating Point (.op)
│   ├── Temperature Coefficient.png
│   ├── Line Regulation.png
│   ├── Worst Case Startup.png
│   ├── Normal Ramp Startup.png
│   ├── Power Cycling Restart.png
│   ├── -40 C Startup.png
│   ├── 27 C Startup.png
│   ├── 125 C Startup.png
│   └── PSRR.png
│
├── LDO_Tests/
│   ├── Line_regulation_LDO.png
│   ├── Load_regulation_LDO.png
│   ├── Line_Transient_LDO.png
│   ├── Load_transient_LDO.png
│   ├── Dropout_voltage_LDO.png
│   ├── Startup_transient_LDO.png
│   └── PSRR_LDO.png
│
└── BGR_LDO_Combo_Tests/
    ├── Operating point (.op)
    ├── Full_Chain_Startup_Baseline.png
    ├── Full_Chain_Startup_Staggered.png
    ├── Full_Chain_BGR_Rail_Sweep.png
    ├── Full_Chain_LDO_Rail_Sweep.png
    ├── Full_Chain_Load_Regulation.png
    ├── Full_Chain_Temp_Sweep.png
    └── Full_Chain_Load_Transient.png
```

> Note: This design uses `tsmc018.lib` (TSMC 0.18µm PDK), included via `.include tsmc018.lib` in all three schematics. Since this is a licensed foundry PDK, not committing the library file itself to the public repo.
