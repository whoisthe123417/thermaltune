# Changelog

## 0.2.3 — 2026-11-15

- fix: the deferred-calibration unit used `Type=simple`, under which
  systemd starts `ExecStartPost` as soon as the process spawns — so the
  cleanup raced the calibration and deleted the answers file before the
  first read ("answers file missing", seen on the first real boot-time
  run). Now `Type=oneshot` + `TimeoutStartSec=2h`: cleanup runs strictly
  after the calibration finishes, and a failed run stays scheduled for
  the next AC boot as documented.
- selftest fixtures relocated into the repo (`tests/data`, committed):
  the /tmp fixture directory did not survive a reboot. The six
  root-recorded CSVs from the first real run are now permanent
  regression data; the six manual-harness CSVs were lost with /tmp
  (their numbers remain in docs/validation.md).

## 0.2.2 — 2026-11-15

- `calibrate --defer`: schedule the calibration to run **unattended at the
  next boot with AC power** (one-shot systemd unit, no popups, self-removes
  after one run). For machines whose owner can't spare the 40 minutes.
- calibrate now parses its own flags, so `thermaltune calibrate --defer`
  and `thermaltune calibrate --profile quick` work as typed.
- honest time estimates: typical duration now assumes the sweep stops early
  (measured real run: ~35 min; old estimate claimed ~63).

## 0.2.1 — 2026-11-14

- fix: kHz/MHz unit mismatch between the cap reference passed to the
  analyzer and the sampler's frequency units made every real calibration
  step look unstable (dips 100%, ratio 0) — the sweep degenerated to the
  lowest tested cap. Found by the first real end-to-end run (battery,
  Surface Laptop Go 1); the sweep data itself was correct and, with the
  fix, correctly identifies 55% as the stable point.
- selftest extended with the root-recorded RAPL data from that run
  (5-column CSVs): now **12/12 passing** against both the manual harness
  data and real thermaltune output.

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
