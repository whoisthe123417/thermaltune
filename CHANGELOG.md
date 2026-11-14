# Changelog

## 0.2.0 — 2026-11-14

Adaptive mode.

- New operating mode: full speed in bursts, automatically clamped near the
  temperature target (`calibrate --mode adaptive`, `mode [cap|adaptive]`)
- Two engines: `thermald` (config written, engagement empirically verified,
  automatic rollback with a ttc suggestion if it can't clamp) and `ttc`
  (`thermaltune-adaptive.service` controller on the proven cap knob,
  2 s polling, reads its target from the config every cycle)
- Verification loop: repeated full-speed runs, target steps down until the
  engine demonstrably clamps within target + 3 °C
- Mode-aware `apply`, `off`, `check`, `status`, `doctor` (adaptive readiness
  section) and `uninstall` (thermald config backup/restore)
- thermald config backed up to `thermal-conf.xml.pre-thermaltune` and
  restored on mode switch/uninstall; adaptive systemd unit for ttc
- `--mode` / `--engine` flags, engine-choice popup, mode-aware consent copy

## 0.1.0 — 2026-11-14

Initial release.

- `calibrate` (thorough/quick profiles): baseline → descending sweep →
  refinement → push-up + final confirmation → apply + persist
- `check` drift verification, `status`, `doctor`, `apply|on|off`,
  `install-persistence`, `uninstall`
- Two-phase privilege architecture (user-session popups, root measurement core)
- kdialog / zenity / TTY popup abstraction, warn-only heavy-app detection
- `systemd-inhibit` protection during calibration and checks
- Dependency auto-install with consent (pacman)
- intel_pstate `max_perf_pct` path (hardware-validated on i5-1035G1);
  `amd_pstate` / `acpi-cpufreq` per-CPU `scaling_max_freq` paths (untested)
- `--selftest` analyzer validation against recorded calibration data
- Hardware validation data in `docs/validation.md` (Surface Laptop Go 1)
