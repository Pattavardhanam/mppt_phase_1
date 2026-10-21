[![GDS](../../actions/workflows/gds.yaml/badge.svg)](../../actions/workflows/gds.yaml)
[![DRC](../../actions/workflows/drc.yaml/badge.svg)](../../actions/workflows/drc.yaml)
[![LVS](../../actions/workflows/lvs.yaml/badge.svg)](../../actions/workflows/lvs.yaml)
[![DOCS](../../actions/workflows/docs.yaml/badge.svg)](../../actions/workflows/docs.yaml)

# Who
pattavardhanam

# Why

First tapeout of a fully analog perturb-and-observe maximum power point tracker (MPPT) for small-satellite (CanSat/CubeSat class) solar panels, with no ADC and no digital control loop.

The tracker dithers the panel voltage at 5 kHz, samples the power in the high and low halves of the dither, and decides which way to move from the sign of the difference, ΔP. At the maximum power point ΔP is under 0.3 % of full-scale power, so the accuracy of the tracker is set by offset, charge injection and leakage in this detection path, not by the multiplier or the converter. Those are the quantities simulation is least able to predict, so this chip builds only the ΔP detection chain and measures them on silicon before the full controller is committed.

# How

Test chip for one 1x2 Tiny Tapeout analog tile (SKYTT26d, SKY130A, 1.8 V). Two DC inputs stand in for the multiplier output in the high and low halves of the dither, so offsets are read from a static transfer curve rather than a dynamic stimulus.

```
PH_IN, PL_IN -> [S&H H, S&H L] -> buffers -> subtractor (G = 10) -> StrongARM -> CMP_OUT
                                      |
                                   test mux -> VMON
```

- **Sample-and-hold pair:** NMOS switches and 5 pF MIM hold capacitors, with identical layout of the two paths. Only the mismatch between them matters, because a step common to both cancels in ΔP.
- **Leakage test structures:** single-path 2 pF holds with different switch geometries (minimum, 4x width, stacked), read through small source followers. Droop rate gives the leakage and shows which part of the switch dominates.
- **Buffers and subtractor:** one reused two-stage opamp as unity-gain hold buffers and as the amplifier of a resistive difference stage. Buffers keep the monitor path and the subtractor from discharging the hold nodes.
- **Comparator:** StrongARM latch, strobed once per decision.
- **Mode and clocks:** all timing is supplied externally on digital pins, so track windows, non-overlap and strobe phase can be swept on the bench. Modes bypass the S&H, or the S&H and the subtractor, to separate the error sources.

Built with the aicex flow (xschem, ngspice via cicsim, magic, netgen).

# What

| What            |        Cell/Name |
| :----           |  :----:       |
| Schematic       | design/MPPT_PHASE_1_SKY130A/MPPT_PHASE_1.sch |
| Layout          | design/MPPT_PHASE_1_SKY130A/MPPT_PHASE_1.mag |
| S&H, single path | sh_unit (planned) |
| S&H, matched pair | sh_pair (planned) |
| Opamp           | opamp (planned) |
| Subtractor      | subtractor (planned) |
| Comparator      | strongarm (planned) |

# Signal interface

Provisional until the pin map is frozen. Analog pins are in order from `ua[0]`.

| Signal       | Direction | Domain  | Description                               |
| :---         | :---:     | :---:   | :---                                      |
| VDD_1V8      | Input     | VDD_1V8 | Main supply                               |
| VSS          | Input     | Ground  |                                           |
| PH_IN (`ua[0]`) | Input  | Analog  | Held in the high half of the dither, 0.7 to 1.1 V |
| PL_IN (`ua[1]`) | Input  | Analog  | Held in the low half of the dither, 0.7 to 1.1 V |
| VMON (`ua[2]`)  | Output | Analog  | Monitor: mux of input, held values, subtractor output, comparator input |
| IBIAS (`ua[3]`) | Input  | Analog  | External bias resistor to ground          |
| CLK_H, CLK_L | Input     | VDD_1V8 | Track windows of the two S&H paths (high = track) |
| STROBE       | Input     | VDD_1V8 | Comparator decision strobe                |
| MODE[1:0]    | Input     | VDD_1V8 | Chain, bypass S&H, bypass subtractor      |
| MUX_SEL[2:0] | Input     | VDD_1V8 | Selects the monitored node                |
| CMP_OUT      | Output    | VDD_1V8 | Comparator decision: sign of ΔP           |


# Key parameters

Targets. Measured values are added to this table as the simulations and the silicon come back.

| Parameter           | Min     | Typ             | Max     | Unit  |
| :---                | :---:   | :---:           | :---:   | :---: |
| Technology          |         | Skywater 130 nm |         |       |
| AVDD                | 1.7     | 1.8             | 1.9     | V     |
| Temperature         | -40     | 27              | 125     | C     |
| Input window (PH_IN, PL_IN) | 0.7 |             | 1.1     | V     |
| Subtractor gain     |         | 10              |         | V/V   |
| Hold capacitor (main pair) |  | 5               |         | pF    |
| S&H hold-edge step  | -0.5    |                 | 0.5     | mV    |
| S&H droop rate (5 pF, 85 C) |  |                 | 1.25    | mV/ms |
| S&H mismatch, H minus L | -0.5 |                 | 0.5     | mV    |
| Chain offset, referred to input (goal) |  |      | 0.5     | mV    |
| Chain offset, hard limit |    |                  | 1.5     | mV    |
| Comparator input offset |     |                  | 5       | mV    |