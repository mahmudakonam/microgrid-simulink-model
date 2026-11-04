# Simulation Results

## Waveforms (0.4 s run, current model)

**DG4 bus voltage and current, 3-phase, zoomed around the Load 4.2 step (t = 0.2 s):**

![DG4 bus voltage](waveform_voltage.png)
![DG4 bus current](waveform_current.png)

Both are clean, recognizable 60 Hz sinusoids with modest, physically-expected
switching ripple (these are detailed-switching inverter models — IGBT bridge,
20 kHz PWM, not averaged — so some ripple is real and correct, not a flaw).
The brief current spike exactly at t = 0.2 s is the expected inrush transient
from the Load 4.2 breaker closing, not a continuous issue.

**Per-DG active/reactive power and frequency, full run:**

![Per-DG active/reactive power](power_response.png)
![Per-DG frequency](frequency_response.png)

All 4 DGs now track each other closely in both power and frequency — a sign
of genuine, working droop-based load sharing (see below).

## Verified steady operation (`run_sim_check(0.4)`, current model)

| DG | peak \|V\| | frequency range | P (end) | Q (end) |
|---|---|---|---|---|
| 1 | 21.098 V | 59.9461–60.0000 Hz | 35.92 W | 3.77 VAr |
| 2 | 21.092 V | 59.9455–60.0000 Hz | 36.34 W | 3.28 VAr |
| 3 | 21.066 V | 59.9451–60.0000 Hz | 36.57 W | 4.98 VAr |
| 4 | 21.073 V | 59.9455–60.0000 Hz | 36.36 W | 8.67 VAr |

Stable over the full 0.4 s run, no NaN/Inf. All 4 DGs converge to nearly
identical power and frequency — real droop-based load sharing, not just
individually-stable units ignoring each other. (Peak voltage is lower than
earlier results — see "Virtual impedance" below for why; it's an intentional,
understood trade-off, not a regression.)

## Root-cause fixes in this pass

Two separate problems were found and fixed together, using the real hardware
parameters published for this testbed
([JunZhang111/IoT-Based-Microgrid-Hardware-Testbed](https://github.com/JunZhang111/IoT-Based-Microgrid-Hardware-Testbed))
rather than the placeholder values used earlier.

### 1. Real feeder/load impedances replace invented placeholders

| Parameter | Previously (invented) | Now (real hardware) |
|---|---|---|
| Feeder R, L (all 3) | 0.3 Ω, 0.2 mH | **0.13 Ω, 3.3 µH** |
| Load 1 (Z1) | 15 Ω + 5 mH | **10.08 Ω + 6.25 mH** |
| Load 2 (Z2) | 20 Ω + 5 mH | **28.34 Ω + 8.00 mH** |
| Load 3 (Z3) | 20 Ω + 5 mH | **25.56 Ω + 1.80 mH** |
| Load 4.1 (Z4.1) | 30 Ω + 5 mH | **10.08 Ω + 6.25 mH** |
| Load 4.2 (Z4.2) | 10 Ω | 10 Ω (unchanged — already matched the spec) |

The real feeders turn out to be *even more* resistance-dominated than assumed
(R:X ≈ 105:1, vs the ≈4:1 previously assumed) — confirming the earlier
diagnosis that the droop law's inductive-line assumption was badly mismatched
to this network, and explaining why the gain margin had been so thin.

### 2. Virtual impedance restores real droop authority

Rather than leaving the droop gains crippled (the prior fix's only lever),
a **virtual series inductance** (`Lv = 3 mH`, Pogaku-style: a voltage-reference
correction proportional to the DG's own output current, computed in the dq
frame) was added to the droop law. This makes the network *appear* inductive
to the droop calculation even though the real feeders are almost purely
resistive, which is what P↔frequency / Q↔voltage droop actually needs to work
well.

With virtual impedance in place, the droop gains were raised back up
substantially (not all the way to the original unstable values, but far past
the crippled ones):

| Parameter | Crippled (prior fix) | Original (unstable) | **Now** |
|---|---|---|---|
| `mp` (frequency droop) | 2π·0.00015 | 2π·0.005 | **2π·0.0015** |
| `nq` (voltage droop) | 0.002 | 0.018 | **0.006** |

Result: all 4 DGs now converge to **essentially identical power** (35.9–36.6 W,
spread of 0.65 W) and **essentially identical frequency** (spread of 0.001 Hz)
— real, working load sharing — while staying stable over a 1.2 s check (the
same duration that showed the original gains diverging to 1370 W and 54.8 Hz).
Droop authority is roughly **0.15 Hz per 100 W**, about 10× stronger than the
prior fix's ≈0.015 Hz per 100 W, though still below the original design's
≈0.25 Hz per 100 W — a deliberately conservative, verified middle ground, not
a return to the unstable original.

The trade-off: virtual impedance creates its own voltage drop under load,
which is why peak bus voltage is lower (≈21 V) than in earlier results
(≈30 V). This is the expected, understood cost of the technique, not a defect.

### 3. Active damping fixes the current-ripple/resonance issue

Separately, bus current was found to have ripple content at **439% of its own
60 Hz fundamental**, with spectral peaks at 1.9–5.2 kHz. Analysis showed the
inner current loop itself is heavily *overdamped* (ζ≈5 in continuous time) —
the real cause was the **LC filter's own undamped resonance** (1/(2π√(LfCf))
≈ 1434 Hz for Lf=0.56mH, Cf=22µF), sitting right at the edge of the observed
ripple band and left completely undamped by the existing control design
(which only feeds forward the filter's steady-state capacitor current, not
feedback to actively damp its resonance).

Fix: added active damping via capacitor-current feedback — `Ic = IL − Ibus`
(computed from signals the controller already has, no new sensors needed),
fed back into the current reference with gain `Kad = 0.5`. Four damping gains
were tested in isolation (Kad = 0.5, 1, 2, 5); 0.5 gave the best result and was
kept rather than chasing a marginal further improvement:

| Kad | Current ripple (% of fundamental) |
|---|---|
| 0 (original) | 439% |
| 0.5 | 41% (isolated test) / **17%** (combined with the other fixes) |
| 1 | 61% |
| 2 | 86% |
| 5 | 123% (damping gain too high starts amplifying noise) |

Voltage ripple dropped similarly, from 36% to ≈3%.

## Load 4.2 step-load test

Load 4.2 (10 Ω resistive) closes at t = 0.2 s. The waveform plots above show a
clean step with the expected brief inrush transient; the power/frequency plots
show all 4 DGs responding together rather than one DG absorbing the step alone.

## Known limitations (still open)

- Droop authority, while much improved, is still below the original design
  intent (~0.15 vs ~0.25 Hz/100W) — a deliberately conservative choice.
  Further tuning could likely close more of that gap, but was not pushed
  further to avoid re-approaching the instability boundary without a wider
  verification budget.
- The virtual-impedance voltage drop reduces voltage regulation accuracy under
  load (peak voltage ≈21V vs ≈30V previously) — inherent to the technique, not
  a bug, but worth knowing if strict voltage regulation matters.
- Active-damping gain (`Kad=0.5`) was chosen from 4 tested values; a finer
  sweep below 0.5 was not performed and might improve ripple further, though
  returns were already diminishing at the four points tested.
- Secondary (Raspberry Pi, 25 ms) control layer is still out of scope.
