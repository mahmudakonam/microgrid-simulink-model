# Simulation Results

## Verified steady operation (`run_sim_check(0.4)`, current gains)

| DG | peak \|V\| | frequency range | P (end) | Q (end) |
|---|---|---|---|---|
| 1 | 32.489 V | 59.9957–60.0000 Hz | 28.40 W | 1.46 VAr |
| 2 | 30.756 V | 59.9961–60.0000 Hz | 26.13 W | −8.79 VAr |
| 3 | 31.069 V | 59.9959–60.0000 Hz | 27.63 W | −7.54 VAr |
| 4 | 29.966 V | 59.9929–60.0000 Hz | 47.35 W | −2.47 VAr |

Stable over the full 0.4 s run, no NaN/Inf. Peak phase voltages are consistent
with the 45 V DC bus on each inverter.

## Load 4.2 step-load test

Load 4.2 (a 10 Ω resistive step load) closes at t = 0.2 s. DG4's power response
is a smooth ramp (≈11 → 18 → 31 → 40 → 47 W across the trace straddling the
step), with frequency staying within 0.007 Hz of 60 Hz throughout the run.

## Droop gain tuning

The droop gains were retuned after the original values produced a **genuine
growing instability** (not just a slow transient): on a 1.2 s run, DG3's
filtered power reached 1370 W on a network whose total load is on the order of
150 W, with frequency drooping to 54.8–58.4 Hz.

| Parameter | Original | Tuned | Change |
|---|---|---|---|
| `mp` (frequency droop) | 2π·0.005 | 2π·0.00015 | 33× smaller |
| `nq` (voltage droop) | 0.018 | 0.002 | 9× smaller |

The inner voltage/current loop gains (`kpv, kiv, kpi, kii`) and the power
measurement filter cutoff (`wc`) were left unchanged — the instability was
isolated to the droop/network interaction, confirmed by testing three
different reduction ratios before settling on the final values above.

**Likely root cause:** the implemented droop law (P↔frequency, Q↔voltage)
assumes inductive feeder lines, but the feeders in this model are
resistance-dominated (R:X ≈ 4:1). On a resistive line those roles partially
swap, which is almost certainly why the stable gain margin was so thin.

## Known limitations

- **Droop authority is now very weak.** At the tuned gains, a 50 W power
  deviation produces only ≈0.0075 Hz of frequency droop, versus the ≈0.25 Hz
  originally intended for the same deviation — the real price of stabilizing
  by shrinking the gain alone. Restoring meaningful load-sharing authority
  would need a structural fix (virtual impedance / P-Q decoupling for the
  resistive feeder), not further gain tuning.
- A 3.0 s long-horizon check stayed finite with no NaN/Inf and most DGs showed
  clearly decreasing drift, but DG1's power showed a small, slow late-run
  uptick (bounded, two orders of magnitude below the original instability, but
  not proven to be perfectly flat at steady state).
- Feeder and load impedances are assumed/placeholder values, not from the
  original spec (only Load 4.2's 10 Ω is specified). If real impedances are
  supplied later, the gains here should be re-validated, not assumed to still
  be stable.
