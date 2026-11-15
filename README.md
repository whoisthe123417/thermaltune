# thermaltune-universalfixer

**Calibrate the highest stable CPU frequency cap on thermally-limited Linux laptops — universal across desktops, drivers, and shells.**

The installed command is `thermaltune`; the project is thermaltune-universalfixer.

> **Made by AI.** thermaltune was designed, written, and debugged by an AI coding agent
> (OpenCode, GLM model) working with one human tester on one real machine. Every claim
> in this README was verified on real hardware — the full measurement story, including
> the bugs found along the way: [docs/validation.md](docs/validation.md).

```
thermaltune calibrate     # ~40 min, finds + applies + persists the sweet spot
thermaltune calibrate --mode adaptive   # no cap until it gets hot
thermaltune calibrate --defer           # schedule the run for the next AC boot
thermaltune mode adaptive # switch modes later (uses the saved calibration)
thermaltune check         # ~5 min, verifies the saved calibration still holds
thermaltune status        # what's currently applied
thermaltune doctor        # full system audit (incl. adaptive readiness)
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

## What the automated runs achieved (2026-11-15, AC)

| Automated run | Outcome |
|---|---|
| cap calibration (thorough, target 85 °C) | converged to **59% ≈ 2.2 GHz dead flat**, peak 82 °C — 4856 vs 4266 ops/s uncapped in the same session (**+14% sustained**), one refinement step above the manual 55% sweet spot |
| the sweep's individual verdicts | 85/73 unstable (sawtooth) · 63 flat-but-86 °C → correct temp-only fail · 55 stable · push-up to 66 correctly rejected |
| adaptive calibration (thermald engine) | thermald engaged for real (rapl_controller); verify ladder 94 → 85 → **77 °C held** at 1881 MHz; operating target auto-derived at 75 °C |
| adaptive verification check | ran as a true single-controller test after the engine-exclusivity fix (journal-verified: no second controller active) |

Step tables, fixture data and every bug the hardware found along the way:
[docs/validation.md](docs/validation.md).

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

## Adaptive mode (thermald or ttc)

A fixed cap keeps the CPU at the sweet spot all the time — even when you're
just typing. Adaptive mode does the smarter thing: **full speed in bursts,
automatic clamping only when the CPU actually gets hot.** Same peak
temperature protection, more responsiveness.

**What that gets you:**

- **Full speed when the machine is cool** — browsing, typing, opening apps:
  no cap at all, the CPU runs like it's uncapped, because it is
- **Short bursts are faster than under any fixed cap** — launching an app or
  a 30-second compile finishes before heat even builds up, so the clamp
  never touches it
- **No permanent performance tax** — a fixed cap never lets the CPU sprint;
  adaptive only slows it down when it's earned it (read: overheated)
- **Same heat protection as cap mode** — on the dev machine it held 77 °C
  against a 75 °C target under full load
- **Switching is free** — `thermaltune mode cap` / `thermaltune mode
  adaptive` flips instantly from the saved calibration, no recalibration

The one tradeoff: under a heavy all-core load for hours, a fixed cap runs
slightly cooler and flatter. Pick adaptive if you use the machine for
everything, cap mode if you mostly cook it.

```
thermaltune calibrate --mode adaptive
thermaltune mode adaptive   # or cap — later, from the saved calibration
```

Two interchangeable engines:

- **thermald** — the standard Linux thermal daemon. thermaltune writes
  `/etc/thermald/thermal-conf.xml` (a passive trip at the calibrated target
  on the `x86_pkg_temp` zone, driving `rapl_controller` when RAPL is
  readable, otherwise the Processor T-state device), enables the daemon,
  then **empirically verifies** it actually clamps: full-speed load runs
  whose peak must land within target + 3 °C, with the target stepping down
  and retrying (max 3 attempts) otherwise. If it can't engage on your
  hardware, everything is rolled back and thermaltune points you at
  `--engine ttc`.
- **ttc** — thermaltune's own controller (systemd service
  `thermaltune-adaptive.service`): polls package temperature every 2 s,
  lowers the proven cap 5% at temp ≥ target, raises it 2% at temp ≤
  target − 6 °C (max one change per 12 s, floor 35%). It re-reads its target
  from `/etc/thermaltune.conf` every cycle, so `thermaltune mode` switches
  take effect immediately.

The verification loop is the important part. On the dev machine there is no
`rapl_controller` sysfs device, and a controller that fails to engage would
otherwise silently do nothing while looking installed. thermaltune never
ships an unverified controller: if the engine doesn't demonstrably clamp, you
get a full rollback, not a false sense of safety.

## Design choices worth knowing

- **Two-phase privilege**: popups and consent run as *your user*; only the
  measurement core runs under sudo. Nothing GUI is ever launched as root.
- **Popups for any DE**: kdialog (KDE) → zenity (GNOME/others) → plain terminal.
  Before starting you get: total time estimate (typical and worst case), and a
  list of detected heavy apps to close yourself (the tool never kills your
  apps without an explicit flag-level decision — warn-only by design).
- **Dependencies auto-installed with your consent** via pacman.
- **Suspend-proof**: calibration holds a `systemd-inhibit` lock (sleep and
  idle are blocked), and a `/run/thermaltune-calibrating` guard makes the
  resume hook skip re-applying the cap while a run is live — so even a
  forced suspend cannot corrupt a measurement.
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
| adaptive mode — `ttc` engine | implemented, hardware acceptance run pending |
| adaptive mode — `thermald` engine | implemented (man-page-verified config), acceptance run pending |
| `amd_pstate` — per-CPU `scaling_max_freq` | implemented per spec, **untested** |
| `acpi-cpufreq` — per-CPU `scaling_max_freq` | implemented per spec, **untested** |
| RAPL power reporting | optional; silently skipped if unavailable |
| Desktop environments | kdialog/zenity/TTY paths; popup rendering under Wayland/X11 tested as user-session processes |

Run `thermaltune --selftest` to validate the measurement analyzer against the
recorded calibration data this tool was developed on.

## Install & correct usage

**1 — Get it and check your machine** (safe, read-only, no sudo yet):

```bash
git clone https://github.com/whoisthe123417/thermaltune-universalfixer
cd thermaltune-universalfixer
./thermaltune doctor
```

…or without git:

```bash
curl -LO https://github.com/whoisthe123417/thermaltune-universalfixer/releases/download/v0.2.8/thermaltune
sha256sum -c thermaltune.sha256 && chmod +x thermaltune
./thermaltune doctor
```

**2 — Calibrate** (~40 min, hands-off). Close heavy apps, plug in AC, then paste:

```bash
./thermaltune calibrate
```

One consent popup, one sudo password, then it runs itself. While it runs:
lid open, machine idle — fans roaring at 100 °C moments is the *measurement
working*, not a problem. Suspend is blocked for the run; Ctrl+C aborts safely
and restores the previous cap. Can't supervise now?
`./thermaltune calibrate --defer` runs it unattended at the next boot with
AC power and removes itself.

**3 — Verify and live with it:**

```bash
./thermaltune status     # what's applied and saved
./thermaltune check      # 5-min re-verification (seasons, dust, updates)
./thermaltune off        # unclamp until next boot
./thermaltune apply      # re-apply the saved state
```

**Optional, but recommended — adaptive mode** (no cap until it gets hot):

```bash
./thermaltune calibrate --mode adaptive   # engine popup: thermald or ttc
./thermaltune mode adaptive               # quick switch later (saved values)
```

Engines are mutually exclusive — switching stops the other automatically.

Run everything as your user, never `sudo ./thermaltune …` in front — the tool
elevates itself and keeps popups available. Uninstall: `sudo thermaltune uninstall`.

## Roadmap

- Per-power-state calibration (AC vs battery)
- Alternative calibration workloads (`--workload`)
- P/E-core aware caps for hybrid Intel CPUs

## Credits

- **GLM (via OpenCode)** — designed, wrote, and debugged every line of this tool,
  against a real machine, over two days
- **@whoisthe123417** — the hardware, the testing, the calibration runs, and the
  decision to publish

Every commit carries a `Co-Authored-By: GLM` trailer — the AI authorship is in
the history itself, not just this label.

## License

MIT — see [LICENSE](LICENSE).

