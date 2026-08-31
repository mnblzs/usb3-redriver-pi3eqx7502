# USB 3.0 ReDriver Repeater Board (PI3EQX7502)

> A USB 3.0 signal repeater board built around the PI3EQX7502 — fixes USB 3.0 link training failures and fallback to USB 2.0 over long cables.

[简体中文](README.md) | **English**

![Status](https://img.shields.io/badge/Status-Prototyped%20%26%20Tested-brightgreen)

![Layers](https://img.shields.io/badge/PCB-2%20Layers-blue)

![License](https://img.shields.io/badge/License-CERN--OHL--S%20v2-orange)

![Chip](https://img.shields.io/badge/Chip-PI3EQX7502AI-yellow)

Board size 43.59 × 57.42 mm, 2 layers.

| Item | Path |
| --- | --- |
| Schematic PDF / vector SVG | [`hardware/schematic.pdf`](hardware/schematic.pdf) · [`schematic.svg`](hardware/schematic.svg) |
| Bill of materials (with LCSC part numbers, orderable as-is) | [`hardware/BOM.csv`](hardware/BOM.csv) |
| Gerber fabrication files | [`hardware/gerber/Gerber_PCB2_4_2026-09-01.zip`](hardware/gerber/Gerber_PCB2_4_2026-09-01.zip) |
| EasyEDA project source (.epro2) | [`hardware/source/`](hardware/source) |
| 3D model (STEP) | [`hardware/3d/`](hardware/3d) |

Download the Gerber archive and order the board directly — no EDA software needed.

## Table of Contents

- [1. What Problem Does This Solve](#1-what-problem-does-this-solve)
- [2. Key Specifications](#2-key-specifications)
- [3. Signal Path and Direction](#3-signal-path-and-direction)
- [4. Hardware Design](#4-hardware-design)
  - [4.1 ReDriver Configuration (Tri-Level EQ / DE)](#41-redriver-configuration-tri-level-eq--de)
  - [4.2 Power Supply](#42-power-supply)
  - [4.3 AC Coupling Capacitors](#43-ac-coupling-capacitors)
  - [4.4 USB 2.0 Channel](#44-usb-20-channel)
- [5. PCB Design](#5-pcb-design)
- [6. Bill of Materials](#6-bill-of-materials)
- [7. Connectors and Pinout](#7-connectors-and-pinout)
- [8. Assembly and Soldering](#8-assembly-and-soldering)
- [9. Usage](#9-usage)
- [10. Verification and Measured Results](#10-verification-and-measured-results)
- [11. Replication Guide](#11-replication-guide)
- [12. License](#12-license)
- [13. Version History](#13-version-history)
- [Appendix](#appendix)

---

## 1. What Problem Does This Solve

On robotics projects you often hit this: the Jetson Orin sits inside the chassis while USB 3.0 peripherals such as depth cameras have to mount on the enclosure or at the end of an arm. The user-facing ports are broken out through adapter cables and an interface board. With that many transitions plus a long external cable, the 5 Gbps SuperSpeed signal degrades until link training fails, and the host falls back to USB 2.0 — the device enumerates, but it will not run at USB 3.0 speed.

> The Jetson Orin is just the scenario the author happened to run into. **This board is not host-specific** — any host with a USB 3.0 port works, as long as there is a long cable run between host and peripheral: industrial PCs, x86 machines, other ARM boards, all the same. The rest of this document simply says "host".

**Typical symptom**: the device enumerates, but `lsusb -t` shows it hanging off `480M` instead of `5000M`; the camera drops frames and cannot reach full resolution.

| Approach | Pros | Cons |
| --- | --- | --- |
| Active optical cable (AOC) | Reliable, long reach (10–50 m) | Expensive, stiff |
| Fiber extender | Longest reach | Most expensive, powered at both ends |
| **Passive cable + ReDriver board** | **Cheap, flexible cable** | Limited distance (3 m measured on this project) |

This project takes the third option: put a USB 3.0 ReDriver in the middle of the link, apply **equalization (EQ) + re-shaping + de-emphasis (DE)** to the degraded signal, and re-drive it — essentially giving the signal a second wind.

> **Note**: a ReDriver only does analog signal conditioning; it does not retime at the protocol layer (that is a Retimer's job), so it cannot eliminate accumulated jitter.

> **What this board is**: a single-channel board with an 8-position DIP switch, built as a **characterization board** for exploring EQ / DE gain combinations — not a final product. See the design rationale at the end of section 4.1.

---

## 2. Key Specifications

| Item | Value |
| --- | --- |
| Main IC | PI3EQX7502AIZDEX (Diodes, formerly Pericom) |
| Data rate | USB 3.2 Gen 1 / USB 3.0, 5 Gbps |
| Channels | 2 independent 5 Gbps differential pairs (one TX, one RX, forming one USB 3.0 port) |
| Differential impedance | 100 Ω CML I/O |
| Host-side connector | U11 — HC-USB3.0-L257-P, USB 3.0 Type-A receptacle (edge-mount) |
| Peripheral-side connector | USB5 — GSB412137CHR, USB 3.1 Gen 2 Type-A receptacle, 9P (vertical) |
| Power | From USB VBUS (5 V), regulated to 3.3 V by on-board AMS1117-3.3 |
| Configuration | 8-position DIP switch + 4.7 kΩ resistor arrays, tri-level EQ / DE |
| PCB | 2 layers, 43.59 × 57.42 mm, 1.6 mm FR-4 |
| Mounting | 2 × M3 holes, 24.0 mm pitch |
| Measured distance | 3 m (board ↔ peripheral) |

### ⚠️ Main IC availability risk

**The PI3EQX7502AI is marked NRND (Not Recommended for New Design) by Diodes** and may be discontinued at any time. The front page of the datasheet names a replacement directly:

> Datasheet: *"Not Recommended for New Design, **Use PI3EQX7741AI**"*

- For new designs, look at **PI3EQX7741AI** (verify pin compatibility yourself — not validated on this project);
- Check LCSC (C526708) stock before replicating;
- For personal or learning use, existing 7502 stock is perfectly fine.

All figures in this document come from the official datasheet **DS41855 Rev 1-3 (2019-04)**.

---

## 3. Signal Path and Direction

USB 3.0 adds two SuperSpeed differential pairs on top of the four USB 2.0 wires. The naming is **relative to the device itself**: `SSTX±` is what this end transmits, `SSRX±` is what this end receives. So the host's SSTX connects to the device's SSRX (crossed).

| Channel | ReDriver input | ReDriver output | Direction | Carries |
| --- | --- | --- | --- | --- |
| **Channel A** | `RXA±` (U3 pin 8/9) | `TXA±` (U3 pin 22/23) | Peripheral → host | Upstream; output via C3/C4 to `U11.SSTX±` |
| **Channel B** | `RXB±` (U3 pin 19/20) | `TXB±` (U3 pin 11/12) | Host → peripheral | Downstream; output via C1/C2 to `USB5.SSTX±` |

Tested link:

```
Downstream  Host.SSTX± → [30cm] → U11.SSRX± → U3.RXB± →[Ch B]→ U3.TXB± → C1/C2 → USB5.SSTX± → [3m cable] → Peripheral.SSRX±
Upstream    Peripheral.SSTX± → [3m cable] → USB5.SSRX± → U3.RXA± →[Ch A]→ U3.TXA± → C3/C4 → U11.SSTX± → [30cm] → Host.SSRX±
```

---

## 4. Hardware Design

### 4.1 ReDriver Configuration (Tri-Level EQ / DE)

The PI3EQX7502 has 4 configuration pins, all **tri-level** inputs:

| Pin | Pin # | Function | Level |
| --- | --- | --- | --- |
| `EQ_A` / `EQ_B` | 2 / 17 | Input equalization for channel A / B; compensates cable loss **before** the input | L/M/H |
| `DE_A` / `DE_B` | 3 / 16 | Output de-emphasis for channel A / B; compensates cable loss **after** the output | L/M/H |

```
Preceding channel        ReDriver                 Following channel
───[cable]──► RX± ──[EQ]──►[driver]──►[DE]──► TX± ───[cable]──►
            compensates loss "before"      compensates loss "after"
```

**Rule of thumb: whichever side has the long cable, turn up the setting on that side.**

Tri-level thresholds: `L` < 0.2 V<sub>DD</sub>, `M` 0.4–0.6 V<sub>DD</sub>, `H` > 0.8 V<sub>DD</sub>. The chip internally biases a floating pin to 0.5 V<sub>DD</sub>, so **leaving it unconnected = M**.

#### DIP switch implementation

Each configuration pin gets **2 positions** of SW1 (one pull-up to 3V3, one pull-down to GND, each through 4.7 kΩ), which is far more flexible than soldering fixed resistors:

| Config pin | Pull-up position | Pull-down position |
| --- | --- | --- |
| `EQ_A` | 8 | 7 |
| `DE_A` | 4 | 3 |
| `EQ_B` | 6 | 5 |
| `DE_B` | 2 | 1 |

Level selection: **H** = pull-up ON / pull-down OFF; **M** = both OFF; **L** = pull-up OFF / pull-down ON. (Both ON divides down to 1.65 V, equivalent to M, but wastes 350 µA — not recommended.)

Values per level (from the datasheet "Mode Adjustment" table):

| Level | EQ | DE |
| --- | --- | --- |
| **L** | 3 dB | 0 dB |
| **M** (chip default) | **6 dB** | **−3.5 dB** |
| **H** | 9 dB | −6 dB |

Note: EQ has no 0 dB setting — 3 dB is the minimum. DE values are negative, indicating attenuation of the output swing.

#### Tested configuration (verified)

```
SW1:  [1]=ON  [2]=OFF [3]=OFF [4]=OFF [5]=OFF [6]=OFF [7]=ON  [8]=OFF
      DE_B=L          DE_A=M          EQ_B=M          EQ_A=L
       0 dB            -3.5dB           6dB             3dB
```

That is, **positions 1 and 7 ON, all others OFF**. This matches exactly what the author's 4-channel board derives (internal project, not open-sourced; uses fixed-resistor pinstraps, where designators marked `n'c'c` are not populated) — two independent paths arriving at the same answer.

> ✅ Measured: 30 cm A-male to A-male cable to the host + 3 m A-male to Micro-B cable to a RealSense D450, running stably at 5 Gbps.

#### 💡 Works in practice, though not theoretically optimal

With the long cable on the peripheral side, you would expect `EQ_A` (compensating the upstream input) and `DE_B` (compensating the downstream output) to be turned up. Yet the tested configuration uses the minimum on both (3 dB / 0 dB), leaving a theoretical 3–6 dB of headroom — **and 3 m is still rock solid**.

Why: the ReDriver's own repeater gain is enough on its own. It restores the input to full swing (typically 1000 mV) before driving it out, and that "free" gain already covers the 3 m of loss. EQ/DE are a bonus on top.

To push further (4–5 m+), try this combination first:

```
SW1:  [1]=OFF [2]=ON  [3]=OFF [4]=OFF [5]=OFF [6]=OFF [7]=OFF [8]=ON
      DE_B=H          DE_A=M          EQ_B=M          EQ_A=H
       -6dB            -3.5dB           6dB             9dB
```

> More EQ is not always better — too much amplifies the noise along with the signal. If H performs worse, drop back to M.

#### The other three configuration pins — floating is correct

| Pin | Pin # | Internal resistor | Floating state |
| --- | --- | --- | --- |
| `EN_A#` / `EN_B#` | 10 / 21 | 200 kΩ **pull-down** | Logic 0 → channel enabled |
| `RxD_EN` | 5 | 200 kΩ **pull-up** | Logic 1 → receiver detect active |

Confirmed by the datasheet: **floating is intentional, not an oversight** — it lands exactly on the best state in the configuration table (chip and channel enabled, receiver detect active). Only break these out if you need GPIO control for low-power modes.

The chip also has adaptive power management: it enters low power after the signal detector is idle for > 1.3 ms, and restarts the receiver-detect loop after another > 6 ms of idle in low power. Unplugging the device saves power automatically — no intervention needed.

#### Design rationale: why single-channel + DIP switch

This board is a **gain characterization tool**, not a final product. Both the single channel and the DIP switch serve that purpose.

**Why only one channel**

The author has a 4-channel version (internal project, not open-sourced) intended for installation inside a machine. But it is awkward for exploring gain settings: four channels running at once, four sets of resistors to change for every configuration, plus crosstalk and supply coupling between channels — you cannot tell which channel deserves the credit for a result.

A single channel minimizes the variables: one link, one chip, four configuration pins. Conclusions drawn this way are clean and carry over confidently to your own multi-channel design.

**Why a DIP switch instead of fixed resistors**

That 4-channel board uses fixed-resistor pinstraps (designators marked `n'c'c` in the schematic are not populated). Fine once the design is settled, painful while exploring: **every EQ/DE combination means another component swap**, so one experiment costs one soldering session — comparing a handful of combinations burns an entire afternoon.

The DIP switch removes that friction. Four config pins × two positions each puts all 3⁴ = 81 combinations on the board; no iron required. In practice the pair that matters most is `EQ_A` × `DE_B` (the ones that compensate when the long cable is on the peripheral side), and sweeping the common combinations takes only minutes — check the rate with `lsusb -t`, watch for dropped frames, and you will quickly find the boundary between "enough" and "too much".

The switch can hang directly off these pins because config inputs are **DC levels, not high-speed signals** — contact resistance and the bit of parasitic they add are completely irrelevant.

**Trade-offs and what comes next**

SW1 is a 16P package (11.7 × 5.4 mm) plus eight 4.7 kΩ resistors, so it costs more board area and more money than fixed resistors. Once the design is frozen, switch back to fixed-resistor pinstraps or move to an I²C-configured ReDriver.

> **Replication note**: if your goal is "install it in a machine", freezing this board's settings into fixed resistors is the better route (it drops SW1 and the eight 4.7 kΩ parts). For multiple channels, extend the single-channel design yourself — the author's 4-channel version is not open-sourced and its design files cannot be provided. The value of this board is that it lets you resolve the uncertainty around gain settings *before* committing to a final version.

---

### 4.2 Power Supply

```
U11 VBUS +5V ──┬──► C19 (10µF) ──► GND              input filtering
               └──► U1 (AMS1117-3.3S, SOT-89-3) ──► 3V3
                      ├──► C18 (22µF) ──► R12 ──► GND     damping
                      ├──► C5, C6 (100nF) ──► GND         decoupling
                      └──► U3 pin 1 / pin 13 (Vdd33)
```

- **Source**: 5 V from the host-side VBUS — no external supply needed. Note that this rail is shared with the peripheral; a USB 3.0 port is spec'd for 900 mA, so leave headroom for power-hungry peripherals.
- **C18 + R12 damping**: legacy LDO architectures like the AMS1117 require the output capacitor to have some ESR (0.1–10 Ω); an MLCC's ESR is too low and the regulator can oscillate. A small resistor (R12) in series with the 22 µF ceramic capacitor's ground path raises the ESR artificially — **1 Ω** here (FOJAN FRC0603F1R00TS).
- **U3 has two Vdd33 pins (1 and 13)** — both need decoupling capacitors.

Power consumption (measured values from the datasheet):

| Mode | Condition | Typical | Max |
| --- | --- | --- | --- |
| **Active** | Channels enabled, signal present | **328 mW** | 450 mW |
| Slumber | Channels enabled, no signal | — | 65 mW |
| Device Unplug | Output unterminated | 7.3 mW | — |
| Standby | `EN_x#` = 1 | 0.15 mW | 1.8 mW |

Active current is 125 mA max (99 mA typical). Adding roughly 168 mW of LDO loss gives about **496 mW** total on the 5 V side, drawing about **99 mA** from VBUS. SOT-89-3 thermal resistance is around 100–150 °C/W, so expect a 17–25 °C rise; in a sealed high-temperature enclosure, give U1 extra copper.

---

### 4.3 AC Coupling Capacitors

The USB 3.0 spec requires AC coupling capacitors in series with SuperSpeed differential pairs, **100 nF** recommended. This board uses four 100 nF 0402 parts: C1/C2 on `U3.TXB±`→`USB5.SSTX±`, and C3/C4 on `U3.TXA±`→`U11.SSTX±`.

**Why only on the ReDriver outputs?** Because the spec puts the DC-blocking capacitor at the transmitting end — signals leaving a host or device are already DC-blocked internally, while the ReDriver's own CML output carries a DC bias that it must block itself. Adding capacitors on the input side would form an extra high-pass with the internal 50 Ω termination and degrade low-frequency response.

Layout requirements: close to the ReDriver output pins, small 0402 packages, the two parts placed symmetrically, and do not cut the ground plane directly beneath them.

> **Good news**: the datasheet explicitly states the input and output pins are **pin polarity reversible** — swapping the polarity of a differential pair still works. If routing gets awkward, just cross the traces; no need for vias or layer changes.

---

### 4.4 USB 2.0 Channel

D+ / D− run straight from U11 to USB5, **bypassing the ReDriver**. Reasoning: 480 Mbps attenuates much less and 3 m is entirely fine; also, USB 2.0 is half-duplex bidirectional on a single pair, which a unidirectional ReDriver cannot repeat. Measurement confirms it — with a direct connection the device still enumerates (just at 480M), proving the USB 2.0 path is intact end to end.

---

## 5. PCB Design

| Item | Value | Item | Value |
| --- | --- | --- | --- |
| Layers | 2 (Top / Bottom) | Default signal width | 10 mil |
| Board outline | **43.59 × 57.42 mm** | Power trace width | 20 mil |
| Traces / vias / pads | 242 / 93 / 106 | Clearance | 5.98 mil |
| Copper pour | GND on both top and bottom | Via | 24 mil outer / 12 mil drill |

Mounting holes (M3, tied to GND, 24.0 mm pitch): SCREW4 (16.21, 3.13 mm), SCREW3 (40.21, 3.13 mm), coordinates relative to the lower-left corner of the board outline.

Seven differential pairs are defined on the board: DP45 (USB 2.0 D±), DP60/DP61/DP62 (channel B), DP63/DP64/DP65 (channel A). DP60's polarity is labeled backwards, but since the chip supports polarity inversion this **does not affect function** — correcting the net names is recommended for tidiness only.

**Trace lengths and intra-pair matching** (PCB portion only):

| Pair | Positive | Negative | Intra-pair skew |
| --- | --- | --- | --- |
| TXA± | 4.17 mm | 4.17 mm | **0.00 mil** |
| TXB± | 7.67 mm | 7.64 mm | 1.18 mil |
| RXA± | 17.20 mm | 17.23 mm | 0.84 mil |
| RXB± | 16.89 mm | 16.77 mm | 4.92 mil |
| D± (USB 2.0) | 36.68 mm | 36.53 mm | 5.65 mil |

The worst case, 4.92 mil, is only 0.886 ps in time. At 5 Gbps one UI is 200 ps, so this is **0.004 UI** — negligible.

### On impedance control

**This board does not implement strict 90 Ω differential impedance control.** Achieving 90 Ω differential on 1.6 mm FR-4 with 2 layers needs roughly 12–15 mil trace width with 6–8 mil spacing; this board's differential traces are 5–10 mil wide, so the actual impedance runs high (about 100–120 Ω).

It still works because the on-board traces are very short (17 mm max), so reflections are limited; USB 3.0 link training has tolerance; and the dominant bottleneck is the cable, not the PCB.

If you want better performance: explicitly ask the fabricator for "90 Ω ±10% impedance control on the USB 3.0 differential pairs" when ordering, or move to a 4-layer board.

---

## 6. Bill of Materials

18 placed items in total (including 2 screw holes), 10 distinct line items.

| Designator | Part | Package | Qty | LCSC | Notes |
| --- | --- | --- | --- | --- | --- |
| U3 | PI3EQX7502AIZDEX | TQFN-24 (4×4, EP) | 1 | C526708 | ReDriver, **NRND** |
| U11 | HC-USB3.0-L257-P | USB-3.0-A-TH | 1 | C7501847 | Host-side Type-A receptacle |
| USB5 | GSB412137CHR | USB-TH_9P-P2.00-V | 1 | C464567 | Peripheral-side Type-A receptacle |
| U1 | AMS1117-3.3S | SOT-89-3 | 1 | C347256 | LDO 5 V→3.3 V |
| SW1 | DSHP08TSGER | SW-SMD 16P (P1.27) | 1 | C3293147 | 8-position DIP switch |
| RN1, RN2 | 4D03WGJ0472T5E | RES-ARRAY 0603-8P | 2 | C1980 | 4.7 kΩ × 4 resistor array |
| R12 | FRC0603F1R00TS | R0603 | 1 | C2907004 | 1 Ω damping resistor |
| C1–C6 | CL05B104KO5NNNC | C0402 | 6 | C1525 | 100 nF (4 AC coupling + 2 decoupling) |
| C18 | CL10A226MQ8NRNC | C0603 | 1 | C59461 | 22 µF, LDO output |
| C19 | CL10A106KP8NNNC | C0603 | 1 | C19702 | 10 µF, LDO input |
| SCREW3/4 | M3 hole | — | 2 | — | Mounting hole, tied to GND |

**Sourcing notes**:

- Check U3 stock first (NRND).
- USB5 is an Amphenol Gen 2 receptacle rated for 5000 mating cycles, but it is pricey (¥10–20). On a tight budget, substitute an ordinary USB 3.0 receptacle (mind the 9P / 2.00 mm pitch / through-hole vertical). **A different receptacle will not get you 10 Gbps** — the bottleneck is the PI3EQX7502, which only supports 5 Gbps.
- Use X7R or better for the capacitors. Watch out for the MLCC DC bias effect — a 22 µF/6.3 V 0603 part may retain only half its nominal capacitance at a 3.3 V bias.

---

## 7. Connectors and Pinout

### U11 — Host-side Type-A receptacle (HC-USB3.0-L257-P)

| Pin | Name | Board connection | Pin | Name | Board connection |
| --- | --- | --- | --- | --- | --- |
| 1 | VBUS | +5V (powers the board) | 7 | GND_DRAIN | GND |
| 2 | D− | Straight to USB5 pin 2 | 8 | SSTX− | ← C3 ← U3 pin 23 |
| 3 | D+ | Straight to USB5 pin 3 | 9 | SSTX+ | ← C4 ← U3 pin 22 |
| 4 | GND | GND | 10 | SHELL | GND |
| 5 | SSRX− | → U3 pin 20 | 11 | SHELL | GND |
| 6 | SSRX+ | → U3 pin 19 | | | |

### USB5 — Peripheral-side Type-A receptacle (GSB412137CHR)

| Pin | Name | Board connection | Pin | Name | Board connection |
| --- | --- | --- | --- | --- | --- |
| 1 | VBUS | +5V (powers the peripheral) | 6 | SSRX+ | → U3 pin 9 |
| 2 | D− | Straight to U11 pin 2 | 7 | GND_DRAIN | GND |
| 3 | D+ | Straight to U11 pin 3 | 8 | SSTX− | ← C1 ← U3 pin 11 |
| 4 | GND | GND | 9 | SSTX+ | ← C2 ← U3 pin 12 |
| 5 | SSRX− | → U3 pin 8 | 10 | SHELL | GND |

### U3 — PI3EQX7502AIZDEX (TQFN-24)

| Pin | Name | Board connection | Notes |
| --- | --- | --- | --- |
| 1 | Vdd33 | 3V3 | Decoupled by C5 |
| 2 | EQ_A | SW1 pos 7/8 via RN1 | Tri-level config |
| 3 | DE_A | SW1 pos 3/4 via RN2 | Tri-level config |
| 4, 6, 7 | DNC | Floating | Do Not Connect |
| 5 | RxD_EN | Floating | Internal 200 kΩ pull-up |
| 8 / 9 | RXA− / RXA+ | USB5 pin 5 / 6 | Channel A input (polarity reversible) |
| 10 | EN_A# | Floating | Internal 200 kΩ pull-down |
| 11 / 12 | TXB− / TXB+ | → C1/C2 → USB5 pin 8/9 | Channel B output (polarity reversible) |
| 13 | Vdd33 | 3V3 | Decoupled by C6 |
| 14, 15, 18 | DNC | Floating | Do Not Connect |
| 16 | DE_B | SW1 pos 1/2 via RN2 | Tri-level config |
| 17 | EQ_B | SW1 pos 5/6 via RN1 | Tri-level config |
| 19 / 20 | RXB+ / RXB− | U11 pin 6 / 5 | Channel B input (polarity reversible) |
| 21 | EN_B# | Floating | Internal 200 kΩ pull-down |
| 22 / 23 | TXA+ / TXA− | → C4/C3 → U11 pin 9/8 | Channel A output (polarity reversible) |
| 24 | DNC | Floating | Do Not Connect |
| 25 | EP (exposed pad) | GND | **Must be soldered**, also the thermal path |

> **DNC pins must be left floating** — do not connect them to anything (including GND).  
> **The EP must be soldered**: it is the main thermal path and the internal ground reference. Use 4–9 Ø0.3 mm vias down to the bottom-layer GND, and have them tented or resin-filled so solder paste does not wick away during reflow.

---

## 8. Assembly and Soldering

**Difficulty: moderately hard.** The main challenges are U3's TQFN-24 (0.5 mm pitch plus a large exposed pad) and the two through-hole USB receptacles.

Recommended order:

1. **U3 (TQFN-24)** — solder this first. Stencil plus reflow or a hot plate is best; keep the EP paste coverage at 50–70% (too much lifts the chip and causes opens). A hot air gun also works: 350 °C, heat evenly, and pre-tin or separately add solder to the EP. Afterwards, check that pins 1/13 are not shorted to GND and are continuous to 3V3.
2. **C1–C6, C18, C19, R12** — for 0402 parts use a fine tip: tin one pad first, position with tweezers, solder one end, then the other.
3. **RN1 / RN2 (0603-8P arrays)** — pins are dense; use paste plus hot air, or drag-solder with plenty of solder and clean up with desoldering braid.
4. **SW1 (SMD 16P)** — 1.27 mm pitch is easy; mind the notch orientation against the silkscreen.
5. **U11 / USB5 (through-hole receptacles)** — **solder these last** (they block access to surrounding SMD area). Seat them flush against the board first, tack 1–2 anchor pins, then finish; use plenty of solder on the shell ground pins since they carry both shielding and mechanical retention.
6. **U1 (SOT-89-3)** — make sure the center thermal pad (the extension of pin 2) is well soldered.

Post-solder checks: 3V3-to-GND impedance should not be near zero → plug in USB and confirm U3 pin 1 is 3.3 V ±5% → after one minute powered, U3 should be slightly warm; if it is too hot to touch, something is wrong.

---

## 9. Usage

This section is a generic debug procedure — it applies regardless of your host, cable length, or peripheral. The author's own measured data is in section 10 for comparison.

### Wiring

The board sits in series between host and peripheral; both sides are Type-A receptacles:

```
Host ──[short cable, ideally < 0.5 m]──► U11 ──► ReDriver ──► USB5 ──[long cable: the distance you need]──► Peripheral
```

| Connector | Location | Connects to | Cable advice |
| --- | --- | --- | --- |
| **U11** | Edge-mount | Host | **As short as possible** — this run also attenuates, so keep what you can |
| **USB5** | Vertical, on board | Peripheral | Your target length |

No external power needed; it takes 5 V from the host-side VBUS.

> ⚠️ **The most common pitfall is the host-side A-male to A-male cable.** Both receptacles are Type-A, so the host side requires an A-male to A-male cable — and that cable is not USB-IF compliant, so internal wiring varies between vendors (some do not connect all the SS pairs, some tie VBUS straight through). **If nothing happens, suspect this cable first** — swap it, or try flipping the plugs end for end.

### Confirming link speed

```bash
lsusb -t
```

Look at the last number: **`5000M`** ✅ link is fine | **`480M`** ❌ it has fallen back.

If you see 480M, work through these in order:

1. Swap the host-side A-male to A-male cable (**the most common cause**);
2. Measure U3 pin 1 — it should be 3.3 V, confirming the board is powered;
3. Shorten the peripheral-side cable to under 1 m and retest, to rule out exceeding the distance limit;
4. Adjust the EQ / DE settings on SW1 (see below).

### Adjusting EQ / DE

Switch mapping: `EQ_A` pull-up position 8 / pull-down 7, `DE_A` 4 / 3, `EQ_B` 6 / 5, `DE_B` 2 / 1.  
Levels: **H** = pull-up ON; **M** = both OFF; **L** = pull-down ON.

Recommended procedure:

1. **Start with everything OFF** (all four pins at the chip's default M level) and run a baseline test;
2. Raise the level on whichever side has the long cable:

| Long cable on | Turn up these two | Why |
| --- | --- | --- |
| **Peripheral side** (most common) | `EQ_A` + `DE_B` | Compensates upstream input + downstream output |
| **Host side** | `EQ_B` + `DE_A` | Compensates downstream input + upstream output |
| Both sides | Push all four up | Step by step — do not max them out at once |

3. Working back from symptoms:

- Unstable link, device dropping → raise EQ (3 → 6 → 9 dB)
- Overshoot / EMI issues → lower DE (−6 → −3.5 → 0 dB)

**Change only one configuration pin at a time** and record what you did. The DIP switch needs no power cycle (these are pure DC level settings, effective immediately), but the link will retrain and the peripheral will briefly disconnect and reconnect.

> If nothing helps, remember that a ReDriver only conditions the signal; it does not retime, so jitter accumulates. Beyond the physical limit you need an active optical cable (AOC) or a fiber solution — that is not something tuning can fix.

---

## 10. Verification and Measured Results

### Suggested verification order

Do not start with the long cable. Following this order localizes the problem quickly:

1. **Short direct connection** (< 1 m) between host and peripheral — confirm those two can do 5000M on their own;
2. **Long direct connection** (your target length) — reproduce the fallback; this is your control. If it still shows 5000M here, this distance was never a problem and you do not need this board;
3. **Insert the board** — the rate should return to 5000M;
4. **Stress test** — run the peripheral at full load continuously for a while and watch for drops or errors:

   - General: `dmesg -w | grep -i usb`, looking for reset / disconnect
   - Cameras: continuous capture at full resolution and frame rate, watching for dropped frames (with RealSense, observe in `realsense-viewer`)
   - Storage: continuous large-file reads and writes, watching for dismounts or sudden rate drops

5. **Tune as needed** (see section 9).

> Only when step 2 reproduces the fallback and step 3 recovers it have you shown the board actually did something — that comparison is the sole criterion for judging whether this board is effective.

### The author's measurement

The numbers below come from one measurement session and represent **a single sample, not a distance limit**. Your cables, peripheral, and host will differ, so your results will too.

| Item | Value |
| --- | --- |
| Host | Jetson Orin (any other USB 3.0 host works equally well) |
| Host-side cable | 30 cm, USB A-male to A-male → `U11` |
| Peripheral-side cable | 3 m, USB A-male to Micro-B → `USB5` |
| Peripheral | Intel RealSense Depth Module D450 depth camera |
| Board configuration | `EQ_A`=L(3dB), `DE_A`=M(−3.5dB), `EQ_B`=M(6dB), `DE_B`=L(0dB) — i.e. positions 1 and 7 ON |

| Scenario | Result |
| --- | --- |
| Without the board, 3 m cable direct | ❌ Enumerates, but **falls back to USB 2.0 (480 Mbps)** |
| With the board inserted | ✅ Runs properly at **USB 3.0 (5 Gbps)** |

Same 3 m cable, and inserting the board takes it from 480M to 5000M — roughly a 10× bandwidth increase, which for a depth camera is the difference between hitting full resolution and frame rate or not.

**About the D450**: active IR stereo plus RGB, global shutter, depth up to 1280×720@30fps, FOV 86°×57°, 50 mm baseline, USB 3.1 interface. It streams stereo and RGB simultaneously, so it is demanding on both bandwidth and stability — a fairly "harsh" test subject. If it passes, ordinary peripherals should be no trouble.

### Pushing to longer distances

In order of value for money:

1. **Tune first** (free): move `EQ_A` to M/H (6/9 dB) and `DE_B` to M/H (−3.5/−6 dB);
2. **Use a better cable**: thicker conductors, good shielding, ferrite beads. This step often pays off more than tuning;
3. **Cascade, or switch to an active optical cable (AOC)**: note that a ReDriver does not retime, so jitter accumulates across stages — generally no more than two.

What cable to use and how far you can push it depend on your peripheral and cable quality — measure it yourself using the procedure above rather than copying these numbers.

---

## 11. Replication Guide

**What you need**: the PCB (2 layers, 43.6 × 57.4 mm), a hot air gun or hot plate, a soldering iron, a multimeter, and a magnifier.

**Steps**:

1. Confirm U3 availability (NRND — check C526708 first);
2. Get the Gerbers: use the archive under `hardware/gerber/` to order directly. JLCPCB's default process is fine (if you want better signal quality, add "90 Ω impedance control on the USB 3.0 differential pairs" in the fabrication notes);
3. Source parts from the section 6 BOM → solder per section 8 → wire up and verify per section 9 → run the "direct vs. with board" comparison per section 10.

**Changes worth making** (ordered by value for money):

1. Tune EQ/DE (free — with the long cable on the peripheral side, move `EQ_A` to 6/9 dB and `DE_B` to −3.5/−6 dB);
2. Break `EN_x#` out to a pin header (handy for GPIO low-power control and A/B comparisons);
3. Replace the 3V3 trace with a copper pour;
4. Move to a 4-layer board with impedance control (doubles the cost, biggest signal-quality gain).

---

## 12. License

This project is released under the **CERN-OHL-S v2 (Strongly Reciprocal)** license:

- ✅ Free to use, modify, manufacture, and sell, including commercially;
- ⚠️ If you distribute a modified version (selling products or publishing files), you must release your modified source files under the same license;
- ⚠️ Keep the original author's attribution and the license notice;
- ❌ Provided "as is", without any warranty.

Full text: <https://ohwr.org/project/cernohl/wikis/Documents/CERN-OHL-version-2>

> The strong copyleft variant was chosen so that improvements flow back to the community. Using it yourself without distributing is unrestricted.

---

## 13. Version History

| Version | Description | Availability |
| --- | --- | --- |
| **Single-channel (this repository)** | 1 × PI3EQX7502 with 8-position DIP switch configuration. Internally named `PI3EQX501i` (carried over from an earlier part selection that did not match the actual part fitted; since corrected) | ✅ This repo |
| **V0.7 4-channel** | 4 × PI3EQX7502, EQ/DE set by fixed-resistor pinstraps (`n'c'c` = not populated), plus DP and RJ45. This board's configuration is inherited from it | ❌ Not open-sourced |
| **V0.7.1 4-channel** | Minor revision of V0.7 | ❌ Not open-sourced |

> This repository **only open-sources the single-channel version**. The V0.7 / V0.7.1 4-channel boards are internal projects under the same top-level project and are **not planned for release**; they are mentioned only to show where this board's configuration came from and why it was designed this way.

---

## Appendix

**Project structure** (EasyEDA Pro project `orin接口扩展`):

```
├── PI3EQX501i                          ← the board in this repository (schematic2_4 + PCB2_4)
├── orin接口扩展V0.7（4*PI3EQX7502）        ← internal project, not open-sourced
├── orin接口扩展V0.7.1（4*PI3EQX7502）_1    ← internal project, not open-sourced
├── Board1 / Board1_1 / Board2_1        ← internal project, not open-sourced
└── 4层测试                              ← internal project, not open-sourced
```

> Except for `PI3EQX501i`, everything listed above is an internal project under the same top-level project and is **outside the scope of this repository**. They are listed only so that anyone receiving this repository's source files understands the overall project — design files for these boards cannot be provided, so please do not request them.

**References**:

- PI3EQX7502AI datasheet — Diodes, DS41855 Rev 1-3 (2019-04). All EQ/DE dB values, tri-level thresholds, internal pull-ups/downs, power figures, pin definitions, and the polarity-reversal feature in this document come from it
- CERN-OHL-S v2 license text
- USB 3.2 specification (USB-IF — SuperSpeed differential pair and AC coupling requirements)

**Feedback**: the design certainly has oversights — feedback is welcome.

---

*Last updated: 2026-09-01*
