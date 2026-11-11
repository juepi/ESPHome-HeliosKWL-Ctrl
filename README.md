# ESPHome Controller for Helios KWL EC 45-160

## Preface

This is a fork of [Fototeddy's ESPHome based controller](https://github.com/fototeddy/ESP-PWM-Control-HeliosKWL/issues), thanks a lot for his work!  
I've modified the code for my needs, mainly:
- Control a single pair of KWL fans
- Adopted operational modes and possible settings
- Adopted logic for counting power-on-hours (using API time from HA)
- Allow controller to fully operate without HomeAssistant
  - either via Webinterface or
  - by using a hardware button

It is not a requirement, but i will be using my [ESP-Mini-Base](https://github.com/juepi/ESP-Mini-Base) with a Wemos S2 Mini. As the Mini-Base already has several N-channel MOSFETs onboard which can be used for triggering the PWM signals for the fans (pull-down to GND) i have removed the schematic files and hardware related infos.

## ESP32 control for HELIOS KWL via ESPHome

Controller for the decentralized room ventilation [HELIOS KWL EC 45-160 Verso](https://www.heliosventilatoren.de/de/produkte/lueftung-mit-waermerueckgewinnung/einzelraum-lueftungsgeraete/kwl-ec-45-160-09361).
It drives one pair of alternating (pendulum) fans:

| Fan | Name | PWM GPIO | Mini-Base Connector |
|-----|------|-------|--------|
| 1   | WZ   | GPIO1 | J1.1 |
| 2   | SZ   | GPIO2 | J1.2 |

Device name in Home Assistant: **HeliosPWM** (entity IDs `*.heliospwm_<name>`, e.g. `sensor.heliospwm_operating_hours`).

## Operating modes

One of the following `Operating Mode` can be selected:

- **Pendulum**: Heat recovery mode. One fan runs on supply air, the other on exhaust air, and the
  directions are swapped periodically. The interval is adjustable via `Pendulum Interval [s]`
  (20-60 s, default 45 s). This is the default mode.
- **Supply**: Both fans run on supply air, e.g. to compensate for other exhausting devices such as bathroom fans or
  cooker hoods.
- **Cooking**: The SZ fan runs on supply air, the WZ fan on exhaust air (to avoid smells being dragged to the bedroom)

> [!IMPORTANT]
> Both cooking mode and supply mode are not recommended to run for a longer time period during winter to avoid condensation issues!

Selecting Supply or Cooking sets the fan speed to the value configured in `Supply Mode Speed [%]` /
`Cooking Mode Speed [%]` (0-100 % in 25 % steps, default 50 %). The speed is not changed when switching back to Pendulum.  
The actual PWM output to the fans is limited to 4% to 96% (spec for installed [ebmpapst fans](https://at.farnell.com/ebm-papst/4412fgpr-203/axiall-fter-127mm-12v-dc-0-16a/dp/2914153)), additionally the range between 41% and 59% PWM duty cycle is treated as off (which means that the minimum operating `fan_speed` is **20%**).

## Web interface

The ESPHome web interface (`http://heliospwm/`, login see Secrets) is grouped for use on a phone:

- **Fan Speed**: preset buttons `Off` / `Low` / `Medium` / `High` / `Ultra` (0 / 25 / 50 / 75 / 100 %, from `speed_1..5`)
  plus the slider `Fan Speed [%]` for any value in 25 % steps
- **Modes**: `Operating Mode` (Pendulum / Supply / Cooking)
- **Status**: operating hours
- **Settings**: upper limit, pendulum interval, mode speeds, operating hours reset, log level

`Log Level` changes the logger level at runtime without reflashing. The default after every boot is
`INFO`; with `DEBUG` the log additionally shows the mode and speed of each fan (`WZ: Supply, 75%`)
whenever they change.

`Upper Limit [%]` caps the effective fan speed (`min(Fan Speed, Upper Limit)`), e.g. for a night
reduction set from Home Assistant. 100 = no limit (default), 0 = fans off. `Fan Speed` itself is
not changed, so the previous speed applies again as soon as the limit is lifted.

The preset buttons are also available in Home Assistant (`button.heliospwm_off`, `_low`, `_medium`, `_high`, `_ultra`). Their names are fixed, so changing a preset value does not change the entity IDs.

## Hardware button

A push button on GPIO16 (Mini-Base SW1) allows local control:

- **Single click**: cycles the fan speed through 0% → 25% → 50% → 75% → 0% … (`Ultra` / 100% is not part of the cycle)
- **Double click**: cycles the operating mode through Pendulum → Cooking → Pendulum …

Both lists can be adapted in the `substitutions` section of `heliospwm.yml`:

```yaml
button_pin: GPIO16
speed_1: "0"                            # Off     fan speed presets
speed_2: "25"                           # Low     used for the preset buttons (web UI / HA)
speed_3: "50"                           # Medium  and cycled by the hardware button
speed_4: "75"                           # High
speed_5: "100"                          # Ultra
button_speeds: "${speed_1}, ${speed_2}, ${speed_3}, ${speed_4}"   # presets cycled by the hardware button
button_modes: '"Pendulum", "Cooking"'   # options of "Operating Mode": "Pendulum", "Supply", "Cooking"
```

Changes made with the button are reported to Home Assistant like changes from the UI.

## Operating hours

The operating hours are counted on the ESP. Time is counted while at least one fan is set to run;
the short pause during a direction change in pendulum mode is not treated as standstill.

- The counter is kept in RAM only and is **never written to flash**.
- Home Assistant keeps the last reported value in `input_number.ophours_heliospwm`
  (updated by the automation `ophours_heliospwm_store`).
- After a boot (update, power loss) the ESP reads this value from Home Assistant as base value and
  continues counting from there. If Home Assistant is not available during boot, the ESP counts
  with `millis()` and adds the base value as soon as the connection is established.
- The ESP only reports a value after it has received the base value, so a reboot cannot overwrite
  the stored value with a lower one.
- The elapsed time is taken from the Home Assistant time (ESPHome `homeassistant` time platform);
  until the first time sync `millis()` is used. Counting continues while Home Assistant is not connected.
- The value is reported once per hour (full hour) and immediately on (re)connect.
- On an ESP reboot (update, power loss), the running time since the last report (max. 1 hour) is lost.
- After maintenance, the counter is reset with the button `Reset Operating Hours` (web UI or HA).
  Home Assistant detects the reset (the value drops) and records the reset time and a notification.

## Secrets

`heliospwm.yml` references `secrets.yaml` (not committed) for the API encryption key and the web
interface login. Copy `secrets.example.yaml` to `secrets.yaml` and fill in your own values before
compiling. OTA updates use the API encryption key.
For WiFi settings, the global secrets `wifi_ssid` and `wifi_password` are used (see `heliospwm.yml`)

## Home Assistant setup (2026.11)

Home Assistant is optional for operating the fans (web UI and hardware button work without it). It is
needed for: keeping the operating hours across ESP reboots, the maintenance notification, time sync,
and remote control. The steps below assume a YAML-managed installation with the default
`configuration.yaml` includes (`automation: !include automations.yaml`).

### 1. Flash the ESP

1. Copy `secrets.example.yaml` to `secrets.yaml` and fill in an API encryption key and the web login.
2. Compile and flash `heliospwm.yml` with ESPHome (dashboard add-on or CLI, `esphome run heliospwm.yml`).
3. After boot, it announces itself as `heliospwm` via mDNS.

### 2. Add the HA configuration files

| File in this repo | Where it goes in Home Assistant |
|---|---|
| `configuration.yaml` | Merge the `input_number:` and `input_datetime:` blocks into your `configuration.yaml` (or include them from a package). |
| `template.yaml` | Save as `template.yaml` next to `configuration.yaml` and add `template: !include template.yaml`. If you already have a `template:` list, append the entries instead. |
| `automations.yaml` | Append the three automations to your `automations.yaml`. |

Then run **Developer tools → YAML → Check configuration** and restart Home Assistant (helpers and
template entities require a restart; automations can be reloaded).

### 3. Set the operating hours start value (before the ESP connects)

The ESP keeps its counter in RAM and reads the base value from `input_number.ophours_heliospwm`
**once, right after it connects to HA**. If you want to carry over an existing counter reading, enter it
into this helper (Settings → Devices & services → Helpers → *OpHours HeliosPWM Stored*) **before** the
device is added in step 4. For a new installation leave it at 0. If you change the helper later, reboot
the ESP so it picks up the new base value.

### 4. Add the device

1. Settings → Devices & services. The ESPHome integration normally discovers `heliospwm` automatically;
   otherwise add the ESPHome integration manually with host `heliospwm.local` (or its IP), port 6053.
2. When asked, enter the **API encryption key** from your `secrets.yaml`.
3. The device appears as **HeliosPWM**. The "Allow the device to perform Home Assistant actions"
   option is *not* required; the device only reads states and time.

Entity IDs are derived from the device name, e.g. `number.heliospwm_fan_speed`,
`select.heliospwm_operating_mode`, `sensor.heliospwm_operating_hours`,
`button.heliospwm_reset_operating_hours`. Verify this one in particular, it is referenced by the
automation `ophours_heliospwm_store`:

- `sensor.heliospwm_operating_hours`

### 5. Verify

- Within a few seconds after the connection the ESP log shows `Base value from HA: … h` and
  `sensor.heliospwm_operating_hours` gets a value. From then on it updates once per hour.
- `input_number.ophours_heliospwm` follows the sensor (automation `ophours_heliospwm_store`).
- `sensor.ophours_heliospwm_months` and `binary_sensor.maint_heliospwm_due` are available.
- Set the maintenance interval with `input_number.maint_heliospwm_threshold_months` (3, 6, 9 or 12 months).

### 6. After maintenance

Press **Reset Operating Hours** (`button.heliospwm_reset_operating_hours`) in Home Assistant or in the
ESP web UI. The ESP sets its counter to 0 and reports it immediately; Home Assistant then

- stores the 0 in `input_number.ophours_heliospwm` (automation `ophours_heliospwm_store`),
- records the time in `input_datetime.maint_heliospwm_last_reset`, dismisses the "maintenance due"
  notification and posts a reset notification (automation `maint_heliospwm_reset_detected`,
  triggered by the stored value dropping; this also works if the reset was done in the web UI
  while Home Assistant was not connected).

No script is involved. The ESP must be online for the button to be available.

### Entity overview

| Entity | Purpose |
|---|---|
| `number.heliospwm_fan_speed` | Fan speed 0–100 % (25 % steps) |
| `button.heliospwm_off` / `_low` / `_medium` / `_high` / `_ultra` | Speed presets (0 / 25 / 50 / 75 / 100 %) |
| `select.heliospwm_operating_mode` | Operating mode: Pendulum / Supply / Cooking |
| `number.heliospwm_upper_limit` | Speed cap, e.g. night reduction (100 = no limit) |
| `number.heliospwm_pendulum_interval_s` | Pendulum interval 20–60 s |
| `number.heliospwm_supply_mode_speed` / `_cooking_mode_speed` | Speed applied when the mode is activated |
| `select.heliospwm_log_level` | Logger level at runtime (`INFO` after boot, `DEBUG` for troubleshooting) |
| `sensor.heliospwm_operating_hours` | Operating hours (ESP counter, hourly) |
| `input_number.ophours_heliospwm` | Stored operating hours (HA side) |
| `sensor.ophours_heliospwm_readable` / `_months` | Derived display values |
| `binary_sensor.maint_heliospwm_due` | Maintenance due |
| `input_number.maint_heliospwm_threshold_months` | Maintenance interval |
| `input_datetime.maint_heliospwm_last_reset` | Time of the last reset (set automatically) |

A typical night reduction is an automation that sets `number.heliospwm_upper_limit` to 25 in the
evening and back to 100 in the morning; `Fan Speed` itself stays untouched.
