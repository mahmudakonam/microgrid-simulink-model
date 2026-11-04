# IoT-Based Microgrid — Simulink Model (Electrical Part Only)

This is a small personal project. I tried to build the **electrical part** of
an IoT-based microgrid testbed in MATLAB/Simulink, just to learn how this kind
of system works and to practice building and tuning it myself.

It is based on a real hardware project built by Jun Zhang:
**[IoT-Based Microgrid Hardware Testbed](https://github.com/JunZhang111/IoT-Based-Microgrid-Hardware-Testbed)**.
That project is real hardware with Raspberry Pis, DSP boards, and an IoT
network (MQTT) connecting everything together, plus a SCADA system for
monitoring and control.

**This project only covers the electrical/power side of that system — the
inverters, filters, cables, and loads, simulated in Simulink.** It does
**not** include the IoT communication part, the Raspberry Pi boards, or the
SCADA software from the original project.

![Microgrid diagram](image.png)

## What's in the model

Four small power sources ("DG1" to "DG4"), each one built the same way:

```
DC source → Inverter → Filter → Connection point → On/off switch (relay)
```

The four sources are connected to each other through cables (called
"feeders"), and each one also powers its own local load. One of the loads
(Load 4.2) can be switched on partway through the simulation, so you can see
how the system reacts when the load suddenly changes.

Each inverter has its own simple controller. There is no single "boss"
controller telling everyone what to do — instead, every inverter looks at how
much power it is supplying and adjusts its own voltage and frequency a tiny
bit in response. This is called **droop control**, and it's the standard way
to let several power sources share a load fairly without needing to talk to
each other constantly.

## The model file

- **`model/microgrid_4dg.slx`** — the Simulink model. Built in MATLAB R2025a
  with the Specialized Power Systems library (`sps_lib`).
- This file is edited by hand directly in Simulink (it isn't auto-generated
  by a script), so it's the one real copy to open and look at.

## Results

See **[`results/RESULTS.md`](results/RESULTS.md)** for the actual simulation
output — voltage, current, power, and frequency plots — plus a plain
explanation of a couple of problems I ran into and how I fixed them.

## What you need to run it

- MATLAB R2025a (or close to it) with Simulink and the Specialized Power
  Systems library (`sps_lib`).
- Just open `model/microgrid_4dg.slx` in Simulink and run it.

## Reference

This project is based on, and uses real hardware parameters published by:

**Jun Zhang** — [IoT-Based Microgrid Hardware Testbed](https://github.com/JunZhang111/IoT-Based-Microgrid-Hardware-Testbed)
(real hardware testbed with IoT communication and SCADA — this project only
reproduces its electrical/power side as a Simulink simulation)
