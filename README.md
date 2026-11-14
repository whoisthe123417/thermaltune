# thermaltune

**Calibrate the highest stable CPU frequency cap on thermally-limited Linux laptops.**

```
thermaltune calibrate     # ~40 min, finds + applies + persists the sweet spot
thermaltune check         # 4 min, verifies the saved cap is still valid
thermaltune status        # what's currently applied
thermaltune doctor        # full system audit
thermaltune off / apply   # temporary full speed / re-apply
thermaltune uninstall     # remove everything, restore full speed
```

> ## Made by AI
>
> This tool was **authored by an AI** (GLM, via [OpenCode](https://opencode.ai)),
> directed and reviewed by a human, and — the important part — **validated
> against real hardware measurements** on a thermally-limited laptop
> (Surface Laptop Go 1, i5-1035G1, CachyOS). Every number in
> [`docs/validation.md`](docs/validation.md) comes from runs on that machine.
> The AMD (`amd_pstate`) and legacy `acpi-cpufreq` code paths are implemented
> per specification but **not yet hardware-validated**. Treat "universal" as
> "designed universal, proven on Intel pstate" until more test reports arrive.

## Why (the physics, measured)

Thin laptops can't reject enough heat for all-core turbo. An uncapped CPU
doesn't settle — it **sawtooths**: burst high → slam into the thermal throttle
floor (~300 MHz) → cool → burst again. Measured on a Surface Laptop Go 1
(stress-ng, 8 threads, matrixprod):

| Config | Sustained clock | Throughput | Peak temp |
|---|---|---|---|
| uncapped | 1075 MHz (66% of samples at the throttle floor) | 2863 ops/s | 100 °C |
| capped 2.8 GHz | 1013 MHz | 2575 ops/s | 100 °C |
| capped 2.4 GHz | 1223 MHz | 3089 ops/s | 89 °C |
| **capped 2.0 GHz (calibrated)** | **2000 MHz — dead flat** | **4424 ops/s** | **80 °C** |
| capped 1.0 GHz (the blunt old-school fix) | 1100 MHz | 2403 ops/s | 63 °C |

The counter-intuitive result: **a calibrated cap sustains ~55% more throughput
than no cap at all**, because it prevents the 100 °C overshoot that throws the
CPU against the throttle floor for over half the run. The blunt 1 GHz lock
(leaving the CPU at base clock) threw away 46% of the available sustained
performance for cooling headroom it didn't need.

thermaltune automates finding that flat point on any machine instead of
guessing.

## How it works

1. **Baseline** — one sustained all-core run at full speed. If that already
   runs flat within the temperature target, nothing is changed (the tool
   refuses to "fix" machines that don't need fixing).
2. **Descending sweep** — candidates from 85% down to 47% (`--profile quick`
   uses fewer steps, ~15 min). Each step: cooldown → settle → sustained load →
   analysis of floor-dip fraction, sustained-clock/cap ratio, peak temperature.
3. **Refinement** — binary midpoint tests between the last unstable and first
   stable candidate.
4. **Push-up confirmation** — the found candidate is re-tested *and* the cap is
   repeatedly pushed one step up while it stays flat, then confirmed once more
   on the now heat-soaked chassis. Noisy step functions get caught here.
5. **Persist** — result written to `/etc/thermaltune.conf`, applied
   immediately, and re-applied at boot and after suspend/resume via a systemd
   unit + sleep hook.

`thermaltune check` re-runs a 4-minute load and compares against the
calibrated baseline (seasonal drift, dust, different power state → it tells
you to recalibrate).

## Design choices worth knowing

- **Two-phase privilege**: popups and consent run as *your user*; only the
  measurement core runs under sudo. Nothing GUI is ever launched as root.
- **Popups for any DE**: kdialog (KDE) → zenity (GNOME/others) → plain terminal.
  Before starting you get: total time estimate (typical and worst case), and a
  list of detected heavy apps to close yourself (the tool never kills your
  apps without an explicit flag-level decision — warn-only by design).
- **Dependencies auto-installed with your consent** via pacman.
- **Suspend-proof**: calibration runs under `systemd-inhibit`, so an idle
  timer cannot suspend mid-measurement and corrupt the result.
- **Only lowers ceilings**: it never raises voltage, power limits or clocks.
  Ctrl+C restores the previous cap.

## Safety rails

- Default temperature target is derived from your hardware's critical temp
  (`crit − 15`, clamped to ≤ 85 °C).
- Pre-checks abort before wasting your time: idle temperature abnormally high
  (failing cooling — clean the vents first), battery under 50% without AC,
  virtual machines.
- `thermaltune doctor` warns about frequency daemons that fight the cap
  (TLP, auto-cpufreq, thermald, cpupower.service). It does not disable them —
  that's your call; see conflicts note in the docs.

## Honest scope

| Component | Status |
|---|---|
| `intel_pstate` (active) — `max_perf_pct` | **hardware-validated** (Ice Lake i5-1035G1) |
| `amd_pstate` — per-CPU `scaling_max_freq` | implemented per spec, **untested** |
| `acpi-cpufreq` — per-CPU `scaling_max_freq` | implemented per spec, **untested** |
| RAPL power reporting | optional; silently skipped if unavailable |
| Desktop environments | kdialog/zenity/TTY paths; popup rendering under Wayland/X11 tested as user-session processes |

Run `thermaltune --selftest` to validate the measurement analyzer against the
recorded calibration data this tool was developed on.

## Install

```bash
git clone <this repo> && cd thermaltune
./thermaltune doctor      # see what it thinks of your machine
./thermaltune calibrate   # the real thing (asks for sudo internally)
```

Uninstall: `sudo thermaltune uninstall`.

## Roadmap

- thermald adaptive mode (free bursts, clamps when hot) — v2, after it's
  validated on the dev machine
- Per-power-state calibration (AC vs battery)
- Alternative calibration workloads (`--workload`)
- P/E-core aware caps for hybrid Intel CPUs

## License

MIT — see [LICENSE](LICENSE).
