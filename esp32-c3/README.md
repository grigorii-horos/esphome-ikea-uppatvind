# ESP32-C3 variant: UPPÅTVIND as a real fan entity

[`uppatvind.yaml`](uppatvind.yaml) is a drop-in ESPHome config that exposes the
purifier as a single `fan` entity with three speeds, keeps it in sync when
someone uses the physical button, and powers off with one long press instead of
cycling the fan up to maximum first.

Written for an **ESP32-C3 SuperMini** on `esp-idf`. The config in the root
README targets an ESP8266 and reads LED1 with `pulse_width`; this one uses
`duty_cycle`, adds control logic on top, and is a separate implementation
rather than a patch.

It addresses three open upstream issues:

| Issue | What this does about it |
|---|---|
| [#2](https://github.com/jonathonlui/esphome-ikea-uppatvind/issues/2) — speed 3 flips between 0 and 3 | Thresholds sit halfway between the measured plateaus, plus a two-sample debounce for the LED fade |
| [#6](https://github.com/jonathonlui/esphome-ikea-uppatvind/issues/6) — fan entity feedback from the HA UI is "tricky" | Target/actual split, optimistic state, retry and a watchdog — see [Control logic](#control-logic) |
| [#7](https://github.com/jonathonlui/esphome-ikea-uppatvind/issues/7) — long press powers off | Used automatically whenever the target is off |

## Wiring

**Every test pad on the UPPÅTVIND board is 5 V, and the ESP32-C3 is not 5 V
tolerant** — absolute maximum on a GPIO is VDD + 0.3 ≈ 3.6 V. Both signals go
through a 4-channel BSS138 level shifter. This applies to TP4 as well as TP7:
even when we are not pressing, the purifier's own pull-up sits on the pin.

| UPPÅTVIND | BSS138 | ESP32-C3 |
|---|---|---|
| TP2 (GND) | GND | GND |
| TP3 (5 V) | HV | `5V` pin |
| — | LV | `3V3` pin |
| TP4 (button) | HV1 ↔ LV1 | GPIO3, open drain |
| TP7 (LED1 PWM) | HV2 ↔ LV2 | GPIO4 |

Take LV from the board's `3V3` pin, not from 5 V — that is what sets the
low-side voltage. The shifter already has 10 kΩ pull-ups on both sides, which
is exactly what the open-drain button output needs.

You also need a `secrets.yaml` next to the config with `wifi_ssid`,
`wifi_password`, `api_encryption_key` and `ota_password`.

## Calibration

Duty cycle on LED1 per speed, measured on real hardware:

| Speed | Duty |
|---|---|
| off | 0.0 % |
| 1 | 5.0 % |
| 2 | 25.0 % |
| 3 | 80.0 % |

Thresholds in the config are `2.5 / 15 / 50`, i.e. **halfway between the
plateaus, never on them**. This matters more than it looks: an earlier draft
used `5 / 45 / 80`, which put two of the three boundaries exactly on a working
point. Idle at 5.0 % jittered across its own boundary and the entity flipped
on and off every few seconds, and speed 2 at 25 % fell inside the range for
speed 1 and was never reachable.

To check your own unit, flash the config, then use the `Press purifier button`
entity and watch `LED1 duty` in Home Assistant. Note the value that each speed
settles on and put the thresholds between them.

LED1 fades in over about 2 s, so during a change the duty passes through
intermediate values — `0.7 / 12.5 / 13.6 / 17.8 / 28.6 / 77.9 %` in one
measured run, each appearing for exactly one sample. Those are the fade, not
speeds. The config only accepts a value seen twice in a row.

## Control logic

The purifier's MCU owns the fan; we only emulate button presses and read the
result back. That makes the two states genuinely separate:

- **`desired_speed`** — what Home Assistant asked for
- **`current_speed`** — what LED1 says the hardware is doing

`go_to_speed` closes the gap, and everything else in the config exists to keep
those two honest:

- **Presses are counted one at a time.** The intermediate state is
  deterministic (`(speed + 1) % 4`), so the target is re-checked after every
  press without waiting for the sensor. Retargeting mid-sequence stops at the
  new speed instead of finishing the old lap — going 3 → 2 and then changing
  your mind to 1 costs two presses, not six.
- **Off uses a ~2 s hold.** Turning off from speed 1 by cycling would run the
  fan up to maximum on the way.
- **Verify and retry.** After a sequence the script waits for the sensor to
  confirm, up to three attempts. A press the purifier missed is corrected
  rather than silently accepted.
- **HA sees the target while a command is in flight.** The sensor stops
  publishing until the two agree. Without this it publishes the old speed into
  the 1–2 s gap between a command and the script starting, and the UI visibly
  jumps back before settling.
- **The sensor's own sync must not look like a command.** `fan.make_call()`
  from the sensor fires the same `on_speed_set` / `on_turn_*` triggers, so a
  `syncing` flag stops it from overwriting a target that just arrived. Without
  it, a command sent while the previous one was still settling was accepted,
  published, and then silently discarded — the classic "the button in HA
  didn't work the first time".
- **A 2 s watchdog.** `script.execute` is dropped while the script runs, so a
  command arriving just before the loop exits would be lost. The watchdog
  restarts the script whenever target and actual disagree.
- **Failure is visible.** After three failed attempts the UI rolls back to the
  actual speed rather than lying about a state the hardware never reached.

Cost of all this: a command that arrives during a press sequence waits for it
to finish, up to about 5 s. That is inherent — the purifier cycles speeds on a
single button and each step has to be confirmed on LED1.

## Notes

- `logger.log` calls in the scripts are kept on purpose. Without them the logs
  cannot distinguish our press from someone pressing the physical button.
- `esphome config` does not compile lambdas — errors inside them only surface
  during a build.
- The give-up-and-roll-back path has not been exercised on hardware; the rest
  has.
- Whether a long press has any other side effect (a filter counter, say) is
  unverified — LED2 / TP6 was not wired up here.
