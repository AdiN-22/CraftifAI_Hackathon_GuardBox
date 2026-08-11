# GuardBox — Tamper-Aware Smart Storage Security Node

Built for the CraftifAI FirmGen Hackathon using FirmGen v0.3.1 on an ESP32.

---

## Problem Statement

Valuable items left in boxes, drawers, delivery parcels, or storage cabinets are usually only discovered as missing or tampered with *after* the fact — there's no active monitoring on the container itself. GuardBox turns any box or enclosure into an armed security node: it watches for physical tampering (being picked up, tilted, or moved) and lid intrusion (the lid being opened) in real time, and gives immediate local feedback the moment either happens — while a system is armed.

## Target Users

- Individuals storing valuables (cash box, jewelry box, small safe alternative)
- Package/delivery security — detect tampering before a parcel reaches its recipient
- Lab or equipment cabinets where unauthorized access needs to be caught immediately
- Dorm/shared-space storage where a lightweight, low-cost tamper alert is useful

## How It Works

GuardBox has two independent sensing pipelines, both gated by an armed/disarmed state:

1. **Tamper detection (MPU-6050):** captures a motion/orientation baseline the moment the system is armed, then continuously compares live readings against that baseline. A pickup, tilt, or drop beyond a configurable threshold triggers an alert.
2. **Intrusion detection (HC-SR04):** captures a baseline lid-gap distance at arming time, then watches for a significant change (lid opening) beyond a configurable threshold.

Either sensor can independently trigger an alert — you don't need both to fire together.

**Status indication** is entirely LED-based (no buzzer):
- **Green** — disarmed / safe
- **Blue** — armed, monitoring
- **Red** — alert triggered (stays on until disarmed)
- **Yellow (blinking)** — active, unacknowledged alert; stops blinking once acknowledged, while red remains on

**Controls** are two buttons on a 4-key momentary pushbutton module:
- **SW0** — toggle armed/disarmed
- **SW1** — acknowledge an active alert (silences the yellow blink without disarming, so monitoring stays active while you investigate)

## Hardware / Bill of Materials

| Component | Qty | Notes |
|---|---|---|
| ESP32 development board (DOIT DevKit V1) | 1 | Main controller |
| MPU-6050 accelerometer/gyroscope (GY-521 breakout) | 1 | Tamper/motion detection, I2C |
| HC-SR04 ultrasonic distance sensor | 1 | Lid-gap / intrusion detection |
| Individual LEDs — Green, Blue, Red, Yellow | 4 | Status indication (with appropriate current-limiting resistors, ~220Ω) |
| 4-key momentary pushbutton module (4KEYS-R1) | 1 | Only SW0 and SW1 used; SW2/SW3 unused |
| Resistors: 1kΩ, 2kΩ | 1 each | Voltage divider for HC-SR04 ECHO line |
| Breadboard + jumper wires | — | Prototyping |

> Note: the project originally used an active buzzer for audible alert feedback. It was replaced with a yellow LED (fast-blink pattern) as the acknowledge-able alert indicator — see [Design Decisions](#design-decisions).

## Wiring

| Signal | Pin | Notes |
|---|---|---|
| MPU-6050 VCC | 3.3V | Do **not** use 5V — GY-521's onboard pull-ups reference VCC, so 5V puts SDA/SCL out of spec for the ESP32 |
| MPU-6050 GND | GND | |
| MPU-6050 SDA | GPIO21 | Direct connection, no divider needed |
| MPU-6050 SCL | GPIO22 | Direct connection, no divider needed |
| HC-SR04 VCC | 5V | Sensor requires 5V to operate reliably |
| HC-SR04 GND | GND | Common ground with ESP32 is required |
| HC-SR04 TRIG | GPIO5 | Direct connection — ESP32's 3.3V HIGH is sufficient to trigger it |
| HC-SR04 ECHO | GPIO18 | **Via voltage divider** (1kΩ in series, 2kΩ to GND) — ECHO outputs 5V, which is out of spec for the ESP32 pin without stepping down |
| LED — Green | GPIO25 | Disarmed/safe indicator |
| LED — Blue | GPIO26 | Armed indicator |
| LED — Red | GPIO27 | Alert indicator (solid) |
| LED — Yellow | GPIO32 | Alert indicator (fast blink; previously the buzzer pin) |
| 4KEYS COM | GND | Common return for both buttons |
| 4KEYS SW0 | GPIO4 | Arm/disarm toggle, internal pull-up, active-low |
| 4KEYS SW1 | GPIO13 | Acknowledge, internal pull-up, active-low |

## Build Instructions

1. Wire the hardware per the table above. **Double-check the HC-SR04 ECHO divider and common ground before powering on** — these were the two most common sources of instability during development (a missing ground reference or missing divider produced ECHO timeout errors).
2. Install FirmGen v0.3.1 and confirm the ESP32 toolchain (ESP-IDF) is detected via Toolchain Status.
3. Create a new FirmGen project targeting the ESP32 board.
4. The firmware was built incrementally through FirmGen using natural-language prompts, in this order:
   - LED + output skeleton (3 status LEDs, initial buzzer — later replaced)
   - 4-key switch module integration (arm/disarm on SW0)
   - MPU-6050 integration (tamper sensing)
   - HC-SR04 integration (intrusion sensing)
   - Combined alert state machine (both sensors gated by armed state, SW1 acknowledge)
   - Wi-Fi + MQTT telemetry and event reporting
   - Buzzer → yellow LED refinement
5. Full prompt history is available in the exported FirmGen chat included in this repository.
6. Flash and monitor via FirmGen's Deploy action; verify boot sequence and state transitions in the serial monitor before testing physically.

## Testing the Device

1. Confirm it boots to **green** (disarmed).
2. Press **SW0** → LED goes **blue** (armed); this captures a fresh sensor baseline.
3. Nudge or lift the enclosure → **red** turns on, **yellow** starts blinking (IMU-triggered alert).
4. Re-arm, then instead approach/cover the HC-SR04 without touching the enclosure → same alert response, ultrasonic-triggered — confirms the two sensors work independently.
5. Press **SW1** during an active alert → yellow stops blinking, red remains on (acknowledged but still armed).
6. Press **SW0** → returns to green, clears the alert, system fully disarmed.
7. Leave the armed system undisturbed for a couple of minutes to confirm no false triggers from ambient vibration/noise.

## Telemetry

The device publishes over MQTT when Wi-Fi is available:
- `<topic-prefix>/telemetry` — periodic status (armed state, last distance reading, connectivity) every 5 seconds
- `<topic-prefix>/events` — published on every armed/disarmed transition and every alert trigger, including which sensor caused it

Local LED/alert behavior continues uninterrupted if Wi-Fi/MQTT is unavailable or drops mid-session.

## Design Decisions

- **Baseline-relative thresholds instead of fixed values:** motion and distance baselines are captured at the moment of arming rather than hardcoded, so the system adapts to wherever the box is actually placed rather than assuming a fixed resting position.
- **Two independent sensor pipelines:** the IMU and ultrasonic sensor are handled in fully separate modules and can each trigger an alert on their own — this was a deliberate choice to avoid a single point of failure if one sensor is obstructed or fails.
- **Acknowledge vs. disarm separation (SW1 vs. SW0):** silencing an active alert doesn't quietly disable monitoring. This mirrors how real alarm systems separate "I heard it" from "stand down," so a user can't accidentally lose protection just to stop the alert indicator.
- **Buzzer replaced with a yellow blinking LED:** for a purely visual alert path — the fast-blink pattern preserves the "urgent, needs attention" signal that the buzzer originally provided, while keeping the design fully silent.

## Limitations

- Thresholds for both sensors are tuned from limited bench testing and may need adjustment for different mounting surfaces or enclosure materials.
- The HC-SR04 requires a few centimeters of clear space in front of it to read reliably; very tight enclosures may need repositioning.
- No physical enclosure/mounting was fabricated for this submission — components are prototyped on a breadboard.
- MQTT telemetry requires the device to be within Wi-Fi range; no offline/store-and-forward buffering is implemented if connectivity is lost for an extended period.
- No battery/power-loss handling — the system assumes continuous power; a power interruption while armed will reset the armed state on reboot.

## Repository Contents

- `/src` — firmware source, organized into separate modules (LED/output, switch input, IMU driver, ultrasonic driver, alert state machine, Wi-Fi/MQTT reporting, configuration)
- `firmgen_chat_export.*` — exported FirmGen chat showing the full prompt/iteration history
- Task List screenshots — included showing the generated plan and iteration progress through each build step
- This README

## Demo Video

(https://drive.google.com/file/d/14juo6N9LKt5onGFpymNOg-u7iKAOcpPS/view?usp=drive_link)
