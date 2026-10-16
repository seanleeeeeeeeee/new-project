# Roadmap — Aimable Ball Launcher

Baseline hardware (from `parts.md`): 1000 KV brushless motor + 30 A ESC (flywheel), 28BYJ-48 stepper + ULN2003 (aiming), ESP32 (control), 12 V / 15 A battery.

Each milestone ends with a concrete **exit check** so you know when it's done.

---

## M0 — Requirements & decisions
- [ ] Pick the ball: type, diameter, mass (ping-pong / foam / tennis / Nerf Rival, etc.). Drives every dimension downstream.
- [ ] Target performance: range, launch speed, rate of fire, aim range (pan °, tilt °), aim accuracy.
- [ ] Launch architecture: single flywheel + curved hood, or dual counter-rotating flywheels (needs a 2nd motor/ESC).
- [ ] Aim axes: pan only, or pan + tilt (tilt needs a 2nd stepper).
- [ ] Feed: gravity hopper + servo gate, solenoid pusher, or manual.
- [ ] Control interface: physical buttons/joystick, web UI over ESP32 Wi-Fi, BLE app, or serial.
- [ ] Size/weight/budget targets.

**Exit check:** one-page spec with numbers filled in.

## M1 — Physics & sizing (design)
- [ ] Estimate exit speed: `v ≈ k · ω · r` (k ≈ 0.4–0.6 single wheel w/ hood, higher for dual wheels). 1000 KV × 12 V ≈ 12,000 rpm no-load, ~9–10k under load.
- [ ] Choose flywheel diameter and ball compression/gap (typically 5–15 % of ball diameter).
- [ ] Projectile range vs. launch angle → decide tilt range needed.
- [ ] Torque budget for pan/tilt: turret inertia + flywheel gyroscopic load. Check 28BYJ-48 (~300 g·cm, ~15 rpm max) is enough; if not, plan a NEMA 17 + TMC2209/A4988.
- [ ] Power budget: ESC peak current + steppers + ESP32 → confirm 15 A pack and pick fuse rating.

**Exit check:** spreadsheet/notes with flywheel size, gearing, and motor choices justified.

## M2 — Electronics bring-up (breadboard)
- [ ] Power tree: battery → fuse → main switch → ESC (12 V); buck converter → 5 V (steppers, servo) → ESP32 (3.3 V onboard reg).
- [ ] Common ground between ESC, drivers, and ESP32.
- [ ] ESC: calibrate throttle range, spin motor with PWM (50 Hz servo-style or OneShot) from ESP32.
- [ ] Stepper: drive via ULN2003 (or new driver), verify direction and step count/rev (28BYJ-48 ≈ 2048 steps/rev in full-step).
- [ ] Add limit/home switches for each aim axis (open-loop steppers need homing).
- [ ] Feeder actuator (servo/solenoid with flyback diode + MOSFET).
- [ ] Arming switch / E-stop wired in hardware, not just software.

**Exit check:** every actuator moves independently from test sketches; nothing gets hot.

## M3 — Firmware architecture (code)
- [ ] Project setup: PlatformIO or Arduino IDE, repo layout (`firmware/`), pin map header.
- [ ] Modules:
  - `flywheel` — arm/disarm, spin-up ramp, target speed, spin-down
  - `aim` — homing, absolute positioning in degrees, soft limits, accel ramps (AccelStepper or FastAccelStepper)
  - `feeder` — single-shot / burst, interlocked with flywheel-at-speed
  - `safety` — arming state machine, watchdog, timeout auto-disarm, low-battery cutoff
  - `ui` — input handling (buttons / web / BLE)
- [ ] State machine: `DISARMED → HOMING → IDLE → SPINNING_UP → READY → FIRING → …`
- [ ] Non-blocking main loop (or FreeRTOS tasks: one for steppers, one for comms).
- [ ] Battery voltage monitoring via ADC + divider.

**Exit check:** full fire sequence runs on the bench without the mechanism attached.

## M4 — CAD v1 (mechanical)
- [ ] Pick tool (Onshape / Fusion / FreeCAD) and keep files in `cad/`.
- [ ] Flywheel + hood/barrel: ball path, compression gap adjustable (slots or shims).
- [ ] Flywheel hub: mount to motor shaft/prop adapter; balance it.
- [ ] Turret: pan bearing (lazy-susan or 608 bearings), tilt pivot near the center of mass.
- [ ] Drive train: gears/belts from stepper to each axis (GT2 belt works well, adds reduction).
- [ ] Hopper + feed gate.
- [ ] Electronics bay: ESC airflow, battery retention, switch/E-stop access, cable routing through the pan axis (slack loop or slip ring).
- [ ] Guards over spinning flywheel.

**Exit check:** full assembly with no interferences, all fasteners/hardware listed in a BOM.

## M5 — Prototype build & test
- [ ] Print/cut parts; assemble flywheel module alone first.
- [ ] Spin test at increasing throttle behind a shield — check vibration, balance, hub retention.
- [ ] Hand-feed balls: measure exit speed (phone slow-mo or chronograph) vs. throttle.
- [ ] Add turret; verify homing and pointing repeatability.
- [ ] Integrate feeder; test jam rate over 50+ shots.

**Exit check:** launches reliably at target speed, aims within spec.

## M6 — Calibration & aiming
- [ ] Throttle → exit-speed curve; account for speed drop between rapid shots.
- [ ] Measure range vs. (tilt angle, speed); build a lookup table or fit a ballistic model.
- [ ] "Aim at distance X" mode: firmware picks angle + speed.
- [ ] Optional: backlash compensation, speed feedback (hall/IR tach on flywheel) for closed-loop RPM.

**Exit check:** hits a target at 3 known distances with acceptable spread.

## M7 — Iterate (CAD v2 / electronics v2)
- [ ] Fix pain points from testing (stiffness, wobble, jams, wiring mess).
- [ ] Move from breadboard to perfboard or a custom PCB (KiCad, `pcb/`).
- [ ] Clean enclosure, strain relief, labeled connectors.

## M8 — Stretch goals
- [ ] Web/phone UI with live aim control.
- [ ] Camera + target tracking (ESP32-CAM or a Pi doing vision, sending aim commands).
- [ ] Spin control (dual flywheels at different speeds → topspin/backspin).
- [ ] Ball-count sensor, shot logging.
- [ ] Battery-level display, sounds/LEDs.

---

## Cross-cutting
- **Safety:** flywheel guard, hardware arm switch, inline fuse, never put fingers near the feed path when armed, eye protection during testing.
- **Docs:** keep `parts.md` as the BOM with prices/links; add wiring diagram (`docs/wiring.png`), pin map, test log.
- **Repo layout:** `firmware/`, `cad/`, `pcb/`, `docs/`.

## Open questions / gaps in the current parts list
- **Ball type is undefined** — blocks flywheel and hood design.
- **Only one stepper** — enough for pan *or* tilt; need two for both.
- **28BYJ-48 is slow and weak** — fine for a light pan-only turret with gearing; likely underpowered for tilting a flywheel assembly. NEMA 17 + TMC2209 is the usual upgrade.
- **28BYJ-48 is 5 V** — needs a buck converter from the 12 V pack (12 V variants exist).
- **No feeder actuator** listed (servo or solenoid).
- **No buck converter, fuse, switch, or limit switches** listed yet.
- **Battery chemistry** (LiPo 3S vs. lead-acid vs. NiMH) — affects charger, low-voltage cutoff, weight.
