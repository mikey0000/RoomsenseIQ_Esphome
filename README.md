# RoomsenseIQ on ESPHome

ESPHome configuration that runs a [RoomsenseIQ](https://www.roomsenselabs.com/)
ESP32-S3 environmental monitor as a pure ESPHome device, reimplementing the
stock firmware's sensor calibration, math, and on-device behaviour (RGB status
LED, calibration flows). The stock ESP-IDF firmware is treated as the spec of
truth; every non-obvious value is annotated with the stock source line it
mirrors.

## What it measures

- Temperature & humidity - SCD4x
- CO2 - SCD4x (with one-tap forced recalibration)
- VOC & NOx indices - SGP4x
- Particulate matter (PM1.0 / PM2.5 / PM10) - MPM10 ([custom component](https://github.com/talmuth/esphome_components/tree/master/components/mpm10))
- Carbon monoxide - TGS5141 (ADC, see Notes)
- Ambient light - photoresistor
- Presence / occupancy - LD2410 mmWave radar
- Motion - IRA-S210ST01 PIR
- Dew point + mold risk - derived from SCD4x (Magnus formula, stock parity)

## On-device behaviour (ported from stock firmware)

- **Occupancy fusion** - PIR + radar state machine with a BedSense mode
  (radar-only) and NVS-persisted latched state.
- **Direction detection** - towards/away movement from radar distance deltas
  with stock hysteresis, suppressed in the radar's unreliable close range.
- **LD2410 auto-calibration** - baseline ("empty room") and blindspot
  ("worst spot occupied") buttons that sample per-gate energies and write the
  radar thresholds, mirroring the stock calibration tasks.
- **Air-quality LED alerts** - a 5 s loop drives the status light off stock
  pollutant thresholds (CO2 / VOC / PM / mold), layered over the occupancy
  state, with a distinct calibration indication.
- **CO2 forced calibration** - one-tap button that runs the stock 3-minute
  fresh-air dwell, performs the FRC at 421 ppm, and records the calibration
  date as a diagnostic sensor.

### Status LED states

| State | Appearance |
|-------|-----------|
| Occupied | steady orange |
| Vacant | steady blue |
| Elevated air pollution | yellow strobe |
| Extreme air pollution | purple strobe |
| CO2 calibrating | cyan strobe |
| Wi-Fi disconnected | orange/blue alternating |

All effect values are stock PWM duties (gamma correction disabled to keep them faithful).

## Hardware

ESP32-S3 (`esp32-s3-devkitc-1`, esp-idf framework). I²C bus on `sda: GPIO12`,
`scl: GPIO13`. Sensors: SCD4x, SGP4x, MPM10, TGS5141, LD2410, IRA-S210ST01,
RGB status LED, photoresistor.

## Configuration structure

These YAML files are **package fragments**, not full device configs - each is
self-contained and meant to be included remotely as an ESPHome `package`.

- **`roomsense-base.yaml`** - base device: bluetooth_proxy, web_server, I²C,
  and the presence/motion/light/LED stack. Pulls in
  `components/{ld2410, ld2410-calibration, ira-s210st01, direction-detection, occupancy, ambient-light, rgb-led}.yaml`.
- **`climatesense.yaml`** - ClimateSense expansion board: environmental
  sensors. Pulls in `components/{mpm10, scd4x, sgp4x, tgs5141, air-quality-alerts, status-led}.yaml`.
- **`components/*.yaml`** - one fragment per sensor/feature.
- **`components/debug.yaml`** - debug settings.

## Why not stock firmware?

The devices make great Bluetooth proxies, and ESPHome makes that trivial.
Beyond that, the stock firmware has some rough edges:

- Devices show up in Home Assistant as one device with a flat set of sensors ([roomsense/firmware#13](https://github.com/roomsense/firmware/issues/13)).
- The setup access point keeps broadcasting with its default password even after Wi-Fi connects ([roomsense/firmware#1](https://github.com/roomsense/firmware/issues/1)).
- No firmware updates since Sep 2024.

## Installation

The device flashes over USB like any other ESP32. Because these files are
package fragments, include them from your own device config:

```yaml
esphome:
  name: roomsense-iq-office
  friendly_name: RoomsenseIQ Office
  name_add_mac_suffix: true

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework:
    type: esp-idf

packages:
  # your own shared config
  wifi: !include common/wifi.yaml

  RoomsenseIQ:
    url: https://github.com/talmuth/RoomsenseIQ_Esphome
    # base device only:
    files: [roomsense-base.yaml]
    # or base + ClimateSense expansion board:
    # files: [roomsense-base.yaml, climatesense.yaml]
    # or specific fragments only:
    # files: [components/mpm10.yaml, components/debug.yaml]
```

## Notes

The ClimateSense CO sensor (TGS5141) is on GPIO20 (ADC2).

On **esphome 2025.8.0 and newer** it builds and runs with Wi-Fi enabled out of the
box, no patch needed: the ESP-IDF v5 ADC rewrite
([esphome#9021](https://github.com/esphome/esphome/pull/9021)) dropped the old
"ESP32S3 doesn't support ADC on this pin when Wi-Fi is configured" restriction (the
`final_validate_config` check no longer exists in the ADC component).

On **esphome older than 2025.8.0** that check rejects the build:

```log
sensor.adc: [source .../components/tgs5141.yaml:2]
    ESP32S3 doesn't support ADC on this pin when Wi-Fi is configured.
```

Either omit `components/tgs5141.yaml`, or patch `final_validate_config` in your
esphome install's `esphome/components/adc/sensor.py` to whitelist GPIO20:

```diff
diff --git a/esphome/components/adc/sensor.py b/esphome/components/adc/sensor.py
index 3309bd04..f39ff3a8 100644
--- a/esphome/components/adc/sensor.py
+++ b/esphome/components/adc/sensor.py
@@ -64,6 +64,7 @@ def final_validate_config(config):
                         CONF_WIFI in fv.full_config.get()
                         and config[CONF_PIN][CONF_NUMBER]
                         in ESP32_VARIANT_ADC2_PIN_TO_CHANNEL[variant]
+            and not (variant == "ESP32S3" and config[CONF_PIN][CONF_NUMBER] == 20)
                 ):
                         raise cv.Invalid(
                                 f"{variant} doesn't support ADC on this pin when Wi-Fi is configured"
```

## Support

If this is useful, you can [buy me a coffee](https://buymeacoffee.com/talmuth).
