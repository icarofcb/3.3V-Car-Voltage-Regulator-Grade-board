# 3V3-REGULATOR — Design Notes

**Project:** Automotive-grade 9 V–15 V to 3.3 V / 500 mA power supply
**Author:** Icaro (github.com/icarofcb)
**Tool:** KiCad 10.0.4 · 2-layer PCB
**Rev:** 1 · 2026-06-28

---

## 1. Overview

This board converts an unregulated automotive DC bus (9 V–15 V) into a clean, regulated **3.3 V rail at up to 500 mA**, with the input, the converter, and the load protected end to end. The signal path is:

```
J2 (9–15 V) → [Protected Input: fuse + ferrite + flat-clamp TVS + bulk caps]
            → [Synchronous Buck: LMR33630BDDA @ 1.4 MHz]
            → [Smart Fuse: TPS25200 eFuse, current limit + OV clamp + fault flag]
            → J9 (3.3 V protected output)
```

Visual diagnostics (power-good and fault LEDs) and four mounting holes are provided for integration and bring-up.

---

## 2. Electrical Specifications

| Parameter | Specification | As-designed |
|---|---|---|
| Input voltage | 9 V – 15 V DC | 9 V – 15 V (LMR33630: 3.8 V–36 V) |
| Output voltage | 3.3 V DC | 3.315 V (FB divider 100 k / 43.2 k) |
| Output current | 500 mA max | eFuse limit ≈ 1.43 A typ (margin) |
| Output power | 1.65 W | 1.65 W |
| Topology | Candidate's choice | **Synchronous buck** (justified §3) |
| Switching frequency | — | 1.4 MHz (LMR33630**B**DDA) |
| Estimated efficiency | State | **≈ 86–88 %** @ 12 V, 0.5 A (§5) |
| Target ripple | State | **< 20 mV pk-pk** (est. ~5 mV, §6) |

---

## 3. Topology Selection & Justification

**Chosen: synchronous step-down (buck) converter.**

A linear regulator (LDO) was rejected on thermal grounds. Dropping 12 V → 3.3 V at 0.5 A in a linear pass element dissipates `(12 − 3.3) × 0.5 = 4.35 W`, and at the 15 V corner `(15 − 3.3) × 0.5 = 5.85 W`. That is an efficiency of only 22–28 % and several watts to remove from a small 2-layer board with no dedicated copper plane — not viable.

A synchronous buck reaches ~85–90 % at this operating point, dissipating only a few hundred mW. The **LMR33630BDDA** was selected because it:

- Covers the full input range with wide margin (3.8 V–36 V vs. 9–15 V), tolerating load-dump ride-through above 15 V.
- Integrates both MOSFETs (synchronous), removing the external catch diode and its loss.
- Switches at **1.4 MHz**, which shrinks the inductor (2.2 µH) and output caps and places the switching fundamental above the AM band, easing conducted-EMI compliance.
- Is marketed as an "ultra-low EMI" SIMPLE SWITCHER in an SO PowerPAD package with low parasitic loop inductance.
- Has an internally compensated, peak-current-mode loop — minimal external part count, robust across line/load.

---

## 4. Functional Block Description

### 4.1 Protected Input Stage

| Ref | Part | Role |
|---|---|---|
| F1 | SF-1206HHA1000R-2 | Board input fuse — hard-fault / short protection. Note on schematic: can be replaced by an automotive blade fuse. |
| L1 | Ferrite 26 Ω @ 100 MHz, 10 A | Series element of the input EMI filter (conducted-emissions / di-dt damping). |
| IC1 | TVS1800DRVR | Flat-clamp surge protection (load-dump / ISO 7637-style transients), 18 V standoff. |
| C1 | 47 µF | Bulk input hold-up. |
| C2, C21 | 10 µF 50 V | Buck input decoupling (high-frequency hot-loop caps). |
| C3, C4, C22, C24 | 0.1 µF | Local HF bypass. |
| C29 | 1 µF | Protected-rail decoupling. |

The fuse, ferrite, and bulk/HF caps form a **pi (C–L–C) input filter**: the ferrite blocks the switcher's di/dt from the harness while the caps supply the pulsed input current locally. The TVS1800 shunts surge energy and clamps flat, with an 18 V standoff that sits comfortably above the 15 V input corner.

### 4.2 Buck Regulator

| Ref | Part | Role |
|---|---|---|
| U2 | LMR33630BDDA | 1.4 MHz synchronous buck, 3.8–36 V in, 3 A. |
| L4 | 2.2 µH | Power inductor. |
| R8 / R9 | 100 k / 43.2 k | Feedback divider → V_OUT = 1.0 V × (1 + 100/43.2) = **3.315 V**. |
| C30 | 0.1 µF | Bootstrap (BOOT–SW). |
| C5 / C7 | 1 µF | VCC / local decoupling. |
| C34, C35, C37, C38, C42 | 10 µF 50 V | Output capacitor bank (3.3 V rail). |

The 1.0 V internal reference and the 100 k / 43.2 k divider set 3.315 V (+0.5 %, within tolerance). The output bank is deliberately conservative for low ripple and good transient response; effective capacitance after DC-bias derating is well below the 50 µF nominal but still ample for this load.

### 4.3 Output Smart Fuse (eFuse)

| Ref | Part | Role |
|---|---|---|
| IC2 | TPS25200DRVR | eFuse: adjustable current limit, OV clamp, OVLO, thermal, reverse blocking, fault flag. |
| R11 | 68 k | R_ILIM → current-limit threshold. |
| D8, D9 (blue) | Power-good indicators | Lit on healthy 3.3 V. |
| D10 (red) | Fault indicator | Driven by FAULT (open-drain). |
| R15, R17 (220 Ω), R16 (1 k) | LED / pull-up resistors | |
| J9 | 282834-2 | Protected 3.3 V output. |

`R_ILIM = 68 kΩ` sets a current-limit threshold of **≈ 1.43 A typ** (≈ 1.30 A min / 1.55 A max over tolerance from the datasheet I_OS equations). That sits comfortably above the 500 mA spec and even above a 1 A transient peak, while still cutting off hard on a downstream short. The TPS25200 also provides an output OV **clamp at ~5.4 V** and **OVLO shutoff at 7.6 V V_IN**, so a failed buck high-side FET dumping the raw input onto the 3.3 V rail is disconnected from the load rather than passed through. The open-drain FAULT asserts on over-current, over-temperature, or over-voltage and drives the red LED — no microcontroller required.

---

## 5. Efficiency Estimate

At 12 V → 3.3 V, 0.5 A, the LMR33630 (1.4 MHz variant) operates around **87 %** per its characteristic curves. The series eFuse adds `60 mΩ × 0.5 A = 30 mV` drop → `~0.9 %` loss. Net **≈ 86 %** at nominal.

Trend across the input range (switching loss scales with V_IN):

| V_IN | Est. converter η | Net (incl. eFuse) |
|---|---|---|
| 9 V | ~89 % | ~88 % |
| 12 V | ~87 % | ~86 % |
| 15 V | ~85 % | ~84 % |

Total dissipation at the worst corner is on the order of a few hundred mW, spread across U2's PowerPAD and L4 — manageable on 2 layers with a stitched ground pour and a thermal-via array under the IC.

---

## 6. Ripple Budget

**Inductor ripple current** (`ΔI_L = V_OUT·(V_IN−V_OUT)/(V_IN·L·f_sw)`):

- 12 V: `3.3 × 8.7 / (12 × 2.2µ × 1.4M) ≈ 0.78 A` pk-pk
- 15 V (worst): `≈ 0.84 A` pk-pk
- Peak inductor current at 0.5 A load: `0.5 + 0.42 ≈ 0.92 A` — far below L4 / U2 limits.

**Output voltage ripple** (ceramic bank, ESR negligible, `ΔV ≈ ΔI_L/(8·f_sw·C_OUT)`):

- With ~40 µF effective: `0.84 / (8 × 1.4M × 40µ) ≈ 2 mV` pk-pk.

**Target: < 20 mV pk-pk**; estimated **~5 mV** including ESL/ESR margin. Input ripple is held down by C1/C2/C21 plus the L1 ferrite (pi filter).

---

## 7. Automotive Best Practices

**Protection (layered):**
1. Input hard-fault / short → F1 (replaceable by a blade fuse, coordinated with the vehicle supply).
2. Surge / load-dump transients → TVS1800 flat clamp (18 V standoff, clamps surge below ~25 V — well under the LMR33630 36 V abs-max and the 50 V cap rating).
3. Output over-current / short / OV / reverse → TPS25200 eFuse with auto-recovery and fault flag.
4. Visual go/no-go → power-good (blue) and fault (red) LEDs.

**EMI / EMC:**
- 1.4 MHz fundamental keeps key harmonics out of the AM band.
- Pi input filter (C–ferrite–C) attenuates conducted emissions on the harness.
- Tight switching hot loop and small SW-node copper minimize radiated emissions.
- TVS + bulk caps address surge immunity (IEC 61000-4-5 class) and EFT/ESD on the input.
- Exposed gold perimeter ring bonds PCB ground to the aluminum enclosure (360° chassis bond), extending the shield to the case.

**Power / Signal Integrity:**
- Output cap bank close to the load connector for low rail impedance and good step-load response.
- Feedback divider routed away from the SW node; FB tap taken at the point of regulation.
- Unbroken bottom-layer ground plane as the reference and lowest-impedance return for every net.

**DFM / DFT / DFA:**
- DFM: assembly-friendly 1206/0603 passives, standard footprints, thermal-via arrays under U2 and IC2, generous clearances for a 2-layer process.
- DFT: test points on V_IN, 3.3 V, FAULT, and GND; FAULT net brought out; LEDs give instant powered diagnostics; output on a keyed connector for in-line current measurement.
- DFA: polarized 282834-2 connectors (no reverse mating), consistent passive orientation, four mounting holes (H1–H4) for mechanical fixturing and vibration robustness.

---

## 8. 2-Layer PCB Layout Strategy

The stackup is deliberately simple and EMC-driven: **the top layer carries all signals and power; the bottom layer is an unbroken ground plane.** On a 2-layer board this is the single biggest lever for EMC and integrity — every trace on top runs over continuous reference copper, so each signal's return current flows in the image directly beneath it, giving the smallest possible loop area and a clean, low-impedance return for every net.

- **Solid ground plane (bottom):** the entire second layer is ground, with no splits or cutouts. Nothing signal or power is routed on the bottom, because any trace there would slot the plane and force return current to detour around it (added loop area → radiated emissions and ground bounce).
- **Input hot loop** (C2/C21 → VIN → GND) kept physically tiny on top — it carries the highest di/dt — with its ground return dropped straight into the plane through vias at the cap pads.
- **SW node** copper minimized in area (radiator) but wide enough for current; inductor placed immediately at SW.
- **Returns / stitching:** a ground via at every ground pad ties each device to the plane with minimal inductance; vias along the high-current return path keep impedance low.
- **Chassis-bond shield ring:** an exposed (soldermask-free), gold-finished (ENIG) ground ring runs around the board perimeter and is via-stitched to the bottom plane, so it bonds PCB ground to the aluminum enclosure around the full edge. Gold keeps the contact low-resistance and corrosion-free under compression and vibration (bare copper would oxidize into an unreliable joint). The 360° bond extends the Faraday shield onto the case, improving radiated emissions and immunity, and gives a short chassis return path for ESD/surge.
- **Thermal:** U2 PowerPAD and IC2 EP drop into the bottom ground copper through via arrays, using the full plane as the heat-spreader.
- **Output caps** hugging the inductor output and the eFuse input; protected-rail caps near J9.
- **Sensitive nets** (FB, ILIM) kept short over solid ground, away from SW and the inductor field.
- **Input protection** (F1, L1, TVS1800) placed at the connector edge so transient energy is shunted before it propagates onto the board.

---

## 9. Component Selection Notes

- **Decoupling / bulk MLCCs:** specified as Murata **GCM** series (AEC-Q200, X7R, −55…+125 °C). For positions in 0805 and larger, or anywhere board flex/vibration is a concern, the **GCJ** soft-termination equivalents are the preferred upgrade (conductive-polymer layer that absorbs board-bending stress and prevents flex cracks).
- **10 µF / 50 V / 1206:** only available as **X7T** in this size/voltage (e.g. GCM31CD71H106KE36L) — note the wider temperature/DC-bias characteristic vs. X7R and the heavy DC-bias derating on the 9–15 V input cap (C2).
- **Voltage derating:** 50 V ceramics on a 3.3 V rail and on the 12 V bus keep stress well inside ECSS/IPC-style derating; the 1 µF/0603 parts are 25 V (fine on 3.3 V/5 V nodes, re-evaluate if any sits on the raw 9–15 V bus).
- **eFuse R_ILIM = 68 kΩ** verified against the TPS25200 datasheet I_OS equations (not just the table): min ≈ 1.30 A stays clear of the load; max ≈ 1.55 A gives a defined cut-off.

---

## 10. Submitted Files

| File | Description |
|---|---|
| `3v3-regulator.kicad_pro` | KiCad project |
| `3v3-regulator.kicad_sch` | Schematic |
| `3v3-regulator.kicad_pcb` | 2-layer PCB layout |
| `README` / Design Notes | This document |
