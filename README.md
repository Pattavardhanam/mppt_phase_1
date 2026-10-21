[![GDS](../../actions/workflows/gds.yaml/badge.svg)](../../actions/workflows/gds.yaml)
[![DRC](../../actions/workflows/drc.yaml/badge.svg)](../../actions/workflows/drc.yaml)
[![LVS](../../actions/workflows/lvs.yaml/badge.svg)](../../actions/workflows/lvs.yaml)
[![DOCS](../../actions/workflows/docs.yaml/badge.svg)](../../actions/workflows/docs.yaml)

# Who

Aditya Pattavardhanam (pattavardhanam)

# Why

Phase 1 of a fully analog perturb-and-observe maximum power point tracker (MPPT) for a photovoltaic source, aimed at small-satellite power systems. The full tracker has no ADC and no digital control loop. It dithers the operating voltage at 5 kHz, samples the PV power in each half of the dither, and integrates the sign of the difference, delta P = P_H - P_L.

Delta P at the maximum power point is only 0.15 to 0.3 % of full-scale power, so the tracker's accuracy is set by offsets, charge injection and hold-capacitor leakage, none of which simulation predicts well. This chip puts just that detection chain on silicon so the real numbers can set the specs of the full controller (multiplier, instrumentation amplifiers, PWM, dither oscillator, integrator) in a later tape-out.

Target shuttle: Tiny Tapeout SKYTT26d, 1x2 analog tile, SKY130A.

# How

The chip takes the two power samples as two static DC inputs, `VIN_H` and `VIN_L`, so a measurement is a static transfer curve rather than a fast waveform. The chain is:

1. **Dual sample-and-hold.** Two matched NMOS-switch and 5 pF paths. External clocks select which input each path samples and when it holds.
2. **Unity-gain buffers** from each hold node, so nothing loads the hold capacitor (a resistive subtractor would draw microamps against a picoampere droop budget).
3. **Subtractor**, gain 10, forming 10 x (P_H - P_L) around a 0.9 V common mode.
4. **StrongARM comparator**, strobed once per decision, giving the sign of delta P.

For measurability the chip adds:
- a test mux that routes internal nodes to one monitor pin, always through a buffer, never directly from a hold node;
- bypass modes that reuse the two input pins to drive the subtractor or the comparator directly, so each block's offset can be separated from the chain;
- three single-path 2 pF leakage structures (minimum switch, 4x width switch, stacked switch), each read through a small source follower. Leakage is independent of capacitance (dV = I t / C), so the switch geometry, not the capacitor, is what these structures vary.

Droop is measured as a slow decay of the held value with the clocks held static. 1 mV/ms on 1 pF is 1 pA.

# What

| What            |        Cell/Name |
| :----           |  :----:       |
| Schematic       | design/MPPT_PHASE_1_SKY130A/MPPT_PHASE_1.sch |
| Layout          | design/MPPT_PHASE_1_SKY130A/MPPT_PHASE_1.mag |

Blocks (status updated as each is finished):

| Block | Cell | Status |
| :---- | :---: | :---- |
| One S&H path | `sh_unit` | in design |
| S&H pair | `sh_pair` | planned |
| Opamp (buffers, subtractor) | `opamp` | planned |
| Subtractor | planned | planned |
| StrongARM comparator | reused from earlier work | planned |
| Test mux and mode logic | planned | planned |

# Signal interface

Block-level signals. The mapping to Tiny Tapeout pins is not final: the proposal is `VIN_H`, `VIN_L`, `VMON` and `RBIAS` on `ua[0]` to `ua[3]`, the eight digital controls on `ui_in`, and `CMP_OUT` on `uo_out[0]`.

| Signal       | Direction | Domain  | Description                               |
| :---         | :---:     | :---:   | :---                                      |
| VDD_1V8      | Input     | VDD_1V8 | Main supply                               |
| VSS          | Input     | Ground  |                                           |
| VIN_H        | Input     | Analog  | Power sample for the high half of the dither, 0.7 to 1.1 V |
| VIN_L        | Input     | Analog  | Power sample for the low half, 0.7 to 1.1 V |
| VMON         | Output    | Analog  | Monitor: one internal node, selected by SEL |
| RBIAS        | Input     | Analog  | External bias resistor to VSS             |
| CLK_H        | Input     | VDD_1V8 | Track clock, high path (high = track)     |
| CLK_L        | Input     | VDD_1V8 | Track clock, low path (high = track)      |
| STROBE       | Input     | VDD_1V8 | Comparator strobe                         |
| MODE[1:0]    | Input     | VDD_1V8 | Chain, bypass S&H, or bypass subtractor   |
| SEL[2:0]     | Input     | VDD_1V8 | Monitor tap select                        |
| CMP_OUT      | Output    | VDD_1V8 | Sign of delta P                           |

# Key parameters

Targets, not measurements. Values marked (assumed) are starting points to be confirmed in simulation and on silicon.

| Parameter           | Min     | Typ             | Max     | Unit  |
| :---                | :---:   | :---:           | :---:   | :---: |
| Technology          |         | Skywater 130 nm |         |       |
| AVDD                | 1.7     | 1.8             | 1.9     | V     |
| Temperature (specified) | -40 | 27              | 85      | C     |
| Temperature (simulated) | -40 | 27              | 125     | C     |
| Input window, VIN_H and VIN_L (assumed) | 0.7 | 0.9 | 1.1 | V |
| Hold capacitor, main pair (assumed) |  | 5              |         | pF    |
| Subtractor gain     |         | 10              |         | V/V   |
| Chain offset, referred to the input (goal / hard limit) |  | 0.5 / 1.5 |  | mV |
| Hold-edge step, per path |   | | 0.5     | mV    |
| Droop rate, 5 pF, 85 C (about 6 pA) |  |      | 1.25    | mV/ms |
| Comparator input offset |         |                 | 5       | mV    |
| Resistor ratio mismatch, subtractor (3 sigma) |  |  | 0.3     | %     |

The 0.5 mV goal is 0.1 % of an assumed 0.5 V full-scale power signal. At that offset the tracker settles about 0.33 % of the maximum-power-point voltage away from the peak, costing about 0.006 % of power. The 1.5 mV limit is the point where the offset reaches the tracker's dead-zone edge.

# Test

To be written with the measurement plan: stimulus values for a delta V sweep, clock phase sweeps, and temperature points for droop.