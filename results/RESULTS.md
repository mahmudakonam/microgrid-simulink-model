# Simulation Results

This project is based on the real hardware testbed by Jun Zhang:
**[IoT-Based Microgrid Hardware Testbed](https://github.com/JunZhang111/IoT-Based-Microgrid-Hardware-Testbed)**.
The real feeder and load values used below come from that project's published
specs (see "Using the real hardware numbers" below).

## Waveforms (0.4 second run)

**DG4's voltage and current, zoomed in around the moment Load 4.2 switches on
(t = 0.2 s):**

![DG4 bus voltage](waveform_voltage.png)
![DG4 bus current](waveform_current.png)

Both look like clean, normal 60 Hz AC sine waves, with a small amount of
expected high-frequency "fuzz" on top. That fuzz is real and normal — the
inverters are simulated as actual switching devices (not simplified/averaged),
so a small amount of switching noise is expected, the same way a real
oscilloscope trace would look. The small spike right at t = 0.2 s is just the
load switching on, not a problem.

**Power and frequency for all 4 inverters, across the whole run:**

![Per-DG active/reactive power](power_response.png)
![Per-DG frequency](frequency_response.png)

All four inverters end up supplying almost the same amount of power and
running at almost the same frequency — meaning they're genuinely sharing the
load between them, not just each doing their own thing.

## Numbers at the end of the short (0.4 second) run

| Inverter | Peak voltage | Frequency range | Power at t=0.4s | Reactive power at t=0.4s |
|---|---|---|---|---|
| DG1 | 21.1 V | 59.946–60.000 Hz | 35.9 W | 3.8 VAr |
| DG2 | 21.1 V | 59.946–60.000 Hz | 36.3 W | 3.3 VAr |
| DG3 | 21.1 V | 59.945–60.000 Hz | 36.6 W | 5.0 VAr |
| DG4 | 21.1 V | 59.946–60.000 Hz | 36.4 W | 8.7 VAr |

Stable the whole time, no errors or blow-ups, and all four inverters land
within about half a watt of each other. **But as the longer run below shows,
0.4 seconds isn't actually long enough for the power to finish settling** —
these numbers are still partway up the climb, not the true final value.

## Checking real steady state with a much longer run (20 seconds)

A 20-second run (sped up using Simulink's Accelerator mode — this compiles
the model to run faster, it does **not** change the simulation itself or its
accuracy, just how quickly MATLAB computes it) shows what actually happens
once everything has fully settled:

![Per-DG active/reactive power, 20 seconds](power_longrun.png)
![Per-DG frequency, 20 seconds](frequency_longrun.png)

Power actually keeps climbing past the 0.4s mark and doesn't fully settle
until around **t = 1.5–2 seconds**, landing at roughly **42 W per inverter**
(not the ~36 W seen at t=0.4s) — and then stays completely flat for the
remaining 18+ seconds. Frequency behaves the same way, settling to about
**59.937 Hz** and holding there. The bottom panel of the frequency plot zooms
into just the last 2 seconds (t=18–20s): it looks "noisy," but look at the
scale on the left — that's a window of only **0.0001 Hz**. That's just tiny
background measurement noise, not any real instability; the actual value is
rock-steady.

**In the last 1 second of the run, every inverter is this close to identical:**

| Inverter | Power range | Reactive power | Frequency range |
|---|---|---|---|
| DG1 | 41.99–42.04 W | 3.52–3.55 VAr | 59.93695–59.93702 Hz |
| DG2 | 41.98–42.04 W | 3.28–3.32 VAr | 59.93695–59.93703 Hz |
| DG3 | 41.99–42.03 W | 5.56–5.59 VAr | 59.93695–59.93702 Hz |
| DG4 | 41.99–42.04 W | 10.09–10.12 VAr | 59.93694–59.93702 Hz |

All four inverters land on essentially the same power and the exact same
frequency — this is the clearest confirmation that the load-sharing is
genuinely working, not just stable in the short term.

## Using the real hardware numbers

I originally guessed some of the cable and load values, since I didn't have
them. After finding Jun Zhang's project above, I swapped in the real numbers
from the hardware testbed:

| Part | My guess | Real hardware value |
|---|---|---|
| Cable between inverters (all 3) | 0.3 Ω, 0.2 mH | **0.13 Ω, 3.3 µH** |
| Load 1 | 15 Ω + 5 mH | **10.08 Ω + 6.25 mH** |
| Load 2 | 20 Ω + 5 mH | **28.34 Ω + 8.00 mH** |
| Load 3 | 20 Ω + 5 mH | **25.56 Ω + 1.80 mH** |
| Load 4 (fixed part) | 30 Ω + 5 mH | **10.08 Ω + 6.25 mH** |
| Load 4 (switchable part) | 10 Ω | 10 Ω (I had this one right already) |

## Two problems I ran into, and how I fixed them

### 1. The inverters fought each other instead of sharing the load

At first, when each inverter tried to adjust its own voltage/frequency based
on how much power it was supplying (droop control), the whole system became
unstable — power kept growing out of control instead of settling down.

The reason: the cables connecting the inverters are mostly resistive (more
like a simple resistor than a coil), but droop control normally assumes the
opposite — that cables behave like coils. On a network shaped the "wrong way"
like this, the inverters' attempts to correct themselves made things worse
instead of better.

**The fix:** I added something called a *virtual impedance* — basically, each
inverter pretends there's a bit of extra "coil-like" cable between it and the
rest of the network, even though there isn't one physically. This small trick
makes the network behave the way droop control expects, so the inverters can
cooperate properly instead of fighting each other.

### 2. The current waveform looked noisy/jittery

Even once the system was stable, the current waveform looked much noisier
than expected — far more than the normal switching noise mentioned above.

The reason: every inverter has a small filter (an inductor + capacitor) that
smooths out its output. That filter has its own natural "ringing" frequency,
and nothing in the original design was damping that ringing down.

**The fix:** I fed a small signal (based on the filter capacitor's current)
back into the controller to actively damp that ringing — the same idea as
feedback used in audio or vibration damping.

**Result:** the noisy part of the current waveform dropped from being over 4
times the size of the normal signal, down to about 1/6th the size.

## Load step test

Load 4.2 (a 10 Ω resistor) switches on at t = 0.2 s. The plots above show a
clean, smooth response — the power and frequency plots show all four
inverters reacting together, not just one inverter struggling alone.

## What's still not perfect

- The inverters' ability to automatically adjust to share load (droop
  response) is weaker than the original design aimed for. It's enough to work
  correctly, but there's room to push it stronger in a future pass.
- The virtual impedance trick lowers the output voltage a bit under load
  (about 21 V instead of around 30 V) — a known side effect of the fix, not a
  mistake.
- The IoT communication, Raspberry Pi secondary control, and SCADA monitoring
  from the original hardware project are not part of this model — this
  project only covers the electrical/power side in simulation.
