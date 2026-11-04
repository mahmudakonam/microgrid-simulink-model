# IoT-Based Microgrid — 4-Inverter Droop-Controlled Simulink Model

A MATLAB/Simulink model of a 4-inverter AC microgrid testbed: four
grid-forming inverters (DG1–DG4) in a line topology, each on its own 45 V DC
bus, interconnected by feeders, each feeding its own local load, running
primary droop control with no central coordination.

![Microgrid testbed — grid-level diagram](image.png)

## Model

**`model/microgrid_4dg.slx`** — built in MATLAB R2025a using the Specialized
Power Systems library (`sps_lib`). This file is **hand-maintained directly in
Simulink** (not regenerated from a script), so it is the single source of
truth for the model.

### Structure
Each DG is a detailed-switching power stage (IGBT bridge, 20 kHz natural-sampled
PWM) visible at the top level as a clear one-line:

```
DC Source → Inverter → LC Filter → Bus (V/I measurement) → Relay
```

connected to its neighbors through `Feeder 1/2/3` and to a local `Load
1/2/3/4.1/4.2` (Load 4.2 is a switchable 10 Ω step load, closed at t = 0.2 s to
test the droop response). Each DG's control electronics — the droop law, the
inner voltage/current loops, and the PWM modulator — are tucked into a
ported-free `Controller i` subsystem that communicates with the power stage
purely through Simulink Goto/From signal tags, keeping the top-level diagram
uncluttered. See `results/model_structure.png` for the current layout.

### Control
Primary droop only (no secondary/centralized layer): each inverter derives its
own frequency and voltage reference from a local power/reactive-power droop
law, running a standard voltage-outer / current-inner cascaded control loop in
the synchronous dq frame — no PLL, each unit free-runs on its own
droop-integrated angle.

## Results

See **[`results/RESULTS.md`](results/RESULTS.md)** for verified simulation
output, the droop-gain tuning record, and known limitations.

| | |
|---|---|
| Current model structure (top level) | `results/model_structure.png` |
| Controller subsystem (per DG) | `results/controller_subsystem.png` |
| LC filter subsystem (per DG) | `results/lc_filter_subsystem.png` |

## Requirements

- MATLAB R2025a (or compatible) with Simulink and the Specialized Power
  Systems library (`sps_lib`).
- Open `model/microgrid_4dg.slx` directly in Simulink — no setup script is
  required or should be run against it.
