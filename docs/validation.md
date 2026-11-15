# Hardware validation — Surface Laptop Go 1 (i5-1035G1)

All numbers below were recorded on 2026-11-14 on the machine thermaltune was
developed on, using the exact measurement method thermaltune automates:

- **Machine**: Microsoft Surface Laptop Go (model 1943), Intel Core i5-1035G1
  (4C/8T, Ice Lake, 1.0 GHz base / 3.6 GHz boost), 15 W-class thin chassis
- **System**: CachyOS, kernel 7.3.1-1-cachyos, intel_pstate (active mode),
  governor `powersave`, EPP `balance_performance`, platform profile `balanced`
- **Load**: `stress-ng --cpu 8 --cpu-method matrixprod`, 360 s per config,
  on AC, idle system (browser/Waydroid closed)
- **Recording**: 5 s samples of package temperature, per-CPU `scaling_cur_freq`
  (avg + min), RAPL package power; steady state = last 60% of samples

## Round 1 — coarse matrix

| Config | max_perf_pct | Steady clock | Throughput | Avg temp | Peak temp | Power |
|---|---|---|---|---|---|---|
| uncapped | 100 | 1075 MHz | 2863 ops/s | 74.9 °C | 100 °C | 12.4 W |
| cap 2.8 GHz | 78 | 1013 MHz | 2575 ops/s | 73.3 °C | 100 °C | 11.3 W |
| cap 2.4 GHz | 66 | 1223 MHz | 3089 ops/s | 71.8 °C | 89 °C | 11.6 W |
| cap 1.0 GHz | 28 | 1100 MHz (flat) | 2403 ops/s | 61.5 °C | 63 °C | 7.2 W |

Key observations:

- Uncapped, the CPU spent **~56% of samples at the 200–400 MHz throttle
  floor** (thermal sawtooth: burst → 100 °C → deep throttle → cool → burst).
- Capping at 2.8 GHz did not prevent the overshoot and scored *worse* than
  uncapped — capping only helps once it is low enough to stop the sawtooth.
- The 1.0 GHz cap was completely stable but left performance unharvested.

## Round 2 — refinement

| Config | max_perf_pct | Steady clock | Throughput | Avg temp | Peak temp | Power |
|---|---|---|---|---|---|---|
| cap 2.2 GHz | 61 | 1549 MHz (still dips 19% of samples) | 3916 ops/s | 75.9 °C | 84 °C | 13.0 W |
| cap 2.0 GHz | 55 | 2000 MHz — **flat, 0 dips** | 4424 ops/s | 78.8 °C | 80 °C | 14.7 W |

## Conclusion (the ground truth thermaltune must reproduce)

The flat point on this machine is **max_perf_pct = 55 (2.0 GHz all-core)**:

- **+54% sustained throughput** vs the best sawtoothing config (2.4 GHz cap)
- **+84% vs the blunt 1.0 GHz lock**
- **+55% vs uncapped**, with peak temperature 20 °C below Tjmax (80 °C vs 100 °C)
- Draws *more* average power (14.7 W) while running cooler than the
  oscillating configs — constant efficient work beats burst-slam-burst cycling

The package temperature sensor (`x86_pkg_temp`) reported 100.0 °C excursions
on unstable configs and never exceeded 80 °C at the flat point — peak
temperature is the fastest reliable sawtooth indicator, which is why
`thermaltune check` judges primarily by peak-vs-baseline.

## Selftest

`thermaltune --selftest` re-analyzes recorded calibration CSVs and must
reproduce exactly these stability verdicts: unstable for uncapped, 2.8 GHz,
2.4 GHz, 2.2 GHz; stable for 1.0 GHz and 2.0 GHz. At validation time it
validated both fixture sets against the numbers above: **12/12 passing**
(steady temps, clocks and power matched to the displayed precision).

The six manual-harness CSVs lived in /tmp and were lost in a reboot wipe;
their numbers are preserved in the tables above. The six root-recorded CSVs
from the first real end-to-end run now live in the repo as committed
regression fixtures (`tests/data/`) and are the selftest's default data set.

## First real end-to-end run (battery, 2026-11-14 23:10)

The first full `thermaltune calibrate` run on this machine (on battery,
target 85 °C, thorough profile) — 5-column CSVs with RAPL power:

| Step | Cap | Peak | Steady clock | Power | Verdict (post-fix analyzer) |
|---|---|---|---|---|---|
| baseline-100 | 3600 MHz | 100 °C | 1796 MHz, 38% floor dips | 17.2 W | unstable |
| sweep-85 | 3060 MHz | 100 °C | 1152 MHz, 64% dips | 13.1 W | unstable |
| sweep-73 | 2628 MHz | 98 °C | 1200 MHz, 60% dips | 12.7 W | unstable |
| sweep-63 | 2268 MHz | 87 °C | 2300 MHz flat | 18.4 W | unstable (temp only) |
| **sweep-55** | **1980 MHz** | **77 °C** | **2000 MHz flat, 0 dips** | **14.4 W** | **stable** |
| sweep-47 | 1692 MHz | 71 °C | 1700 MHz flat | 11.3 W | stable |

Two results from this run:

1. **A real bug was caught** (the reason acceptance runs exist): the cap
   reference was computed in kHz while the sampler records MHz, so every
   step looked 100% dipped / 0% ratio and the sweep degenerated to the
   lowest cap (47%, "degraded"). The measurement data itself was correct.
   Fixed in 0.2.1; the analyzer now classifies all six steps exactly as the
   table above.
2. The sweep independently rediscovered the manually-found flat point:
   **55% ≈ 2.0 GHz, flat, peak 77 °C on battery** — matching the AC ground
   truth (peak 80 °C) within measurement noise.
