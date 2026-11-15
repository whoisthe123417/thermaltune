# Changelog

## 0.2.7 — 2026-11-15

- ttc engine: **proportional response** instead of a fixed-rate ratchet.
  The first adaptive check exposed the weakness: starting a full burst
  from idle, the fixed -5%/12 s steps needed ~96 s to walk the cap down
  and the CPU overshot to 95 C (check failed; the second, warm-started
  check passed at 87 C). Now the cut scales with overshoot: -15% every
  4 s at target+8 C, -10% every 6 s at target+3 C, -5% every 12 s near
  the target; the +2%/12 s recovery is unchanged.
- install-persistence now restarts an active adaptive service, so a
  controller update takes effect with the one command (skipped while a
  calibration is running).

## 0.2.6 — 2026-11-15

- fix: `thermaltune mode adaptive` died with "unknown mode: mode" — the
  dispatcher passed the subcommand through unshifted, so cmd_mode saw the
  literal word "mode" as its argument (found on the first real use,
  right after the AC acceptance run). Also validates the argument before
  the sudo re-exec, so a typo no longer costs a password prompt.

## 0.2.5 — 2026-11-15

- flag validation before anything else: `--profile garbage` dies instead
  of silently running the thorough sweep, `--target-temp` must be an
  integer 40-95 C (a garbage value previously became 0 inside the
  analyzer and judged every step unstable), `--load-seconds` must be
  30-600, and `--mode`/`--engine` are checked before any popup or
  elevation. Found by static review during the first AC acceptance run.
- flags with a missing value (`thermaltune calibrate --profile`) now die
  with a clear message instead of an unbound-variable crash.
- the TTY engine picker could not be answered: its prompt offered [1/2]
  but only the words ttc/thermald were accepted — typing 1 or 2 died
  with "no engine selected". Both now work.
- `check` printed "limit 8%" for floor dips while the analyzer gates at
  5% — text aligned to 5%.
- kdialog engine radiolist: dropped `--separate-output` (a checklist
  flag; radiolist already outputs the plain tag).
- TTY consent/deferred prompts printed their message twice (once
  directly, once via ui_ask) — collapsed to a single call.
- doctor: thermald readiness now reports `rapl_controller` correctly
  when run as root, and notes it when run as an unprivileged user.
- the battery warning no longer prints the -1 sentinel on machines
  without a battery.

## 0.2.4 — 2026-11-15

- suspend protection is now real (found by the first suspended-calibration
  incident): the privileged phase runs under `systemd-inhibit
  --what=sleep:idle`, and the resume hook skips re-applying the cap while
  `/run/thermaltune-calibrating` exists. Previously the README claimed
  inhibit protection the code didn't have — a mid-run suspend let the
  resume hook stomp the cap the calibration had just set (the analyzer's
  ratio check caught the impostor step and flagged it unstable).
- the deferred unit's ExecStart is inhibit-wrapped too (a 40-minute
  boot-time run can outlast an idle timer).

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
