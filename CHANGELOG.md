# Changelog

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
