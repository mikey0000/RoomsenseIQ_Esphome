# 🏠 RoomsenseIQ ESPHome Configuration

This repository contains ESPHome configuration files for a [Roomsense IQ](https://www.roomsenselabs.com/) environmental monitoring device based on ESP32-S3.

## 🌟 Overview

RoomsenseIQ is an advanced indoor environmental quality monitor that tracks multiple parameters:

- 🌡️ Temperature and humidity (SCD4x sensor)
- 🟩 CO2 levels (SCD4x sensor)
- 🧪 VOC and NOx indices (SGP4x sensor)
- 🌫️ Particulate matter (MPM10 sensor)
- 🚨 Carbon monoxide (TGS5141 sensor)
- 💡 Ambient light levels
- 🧍 Room occupancy/presence detection (LD2410 mmWave radar)
- 👀 Motion detection (PIR sensor)

## 🛠️ Hardware

- 🖥️ ESP32-S3 development board
- 🟩 SCD4x CO2 sensor
- 🧪 SGP4x gas sensor
- 🌫️ MPM10 particulate matter sensor
- 🚨 TGS5141 CO sensor
- 🧍 LD2410 mmWave radar sensor
- 👀 IRA-S210ST01 PIR motion sensor
- 🌈 RGB LED for status indication
- 💡 Photoresistor for ambient light sensing

## ❓ Why not stock firmware?

I've got multiple devices with intent installing ESPHome on them, so for me it was just "Why Not?" 😄. I wanted to use them as bluetooth proxies, and ESPHome is an easy way to do so.
However, after receiving the devices I discovered a couple of issues with the stock firmware:

- 🏷️ Devices are automatically detected by HomeAssistant via MQTT integration, but are effectively added as a set of sensors under one single device ([roomsense/firmware/issues/13](https://github.com/roomsense/firmware/issues/13));
- 📡 Even after the device has been successfully connected to Wi-Fi, it continues to broadcast its setup access point with the default password, which appears to be a conscious choice by the authors ([roomsense/firmware/issues/1](https://github.com/roomsense/firmware/issues/1));
- 💤 Authors seem to have abandoned the device and haven't released any new firmware updates since Sep 3, 2024.

## 🗂️ Configuration Structure

- `roomsense-base.yaml`: Base configuration with core components, includes:
      - `components/ambient-light.yaml`: 💡 Ambient light sensor configuration
      - `components/rgb-led.yaml`: 🌈 RGB status LED
      - `components/ld2410.yaml`: 🧍 mmWave presence detection
      - `components/ira-s210st01.yaml`: 👀 PIR motion sensor

- `climatesense.yaml`: Configuration focused on environmental sensors present on [ClimateSense](https://www.roomsenselabs.com/climatesense) expansion board
      - `components/mpm10.yaml`: 🌫️ Particulate matter sensor, via custom component ([mpm10 custom component](https://github.com/talmuth/esphome_components/tree/master/components/mpm10))
      - `components/scd4x.yaml`: 🟩 CO2, temperature and humidity sensor
      - `components/sgp4x.yaml`: 🧪 VOC and NOx sensor
      - `components/status-led.yaml`: 🌈 Status LED configuration
      - `components/tgs5141.yaml`: 🚨 CO sensor configuration, see Notes below

- `components/debug.yaml`: 🐞 Debugging settings

## ⚙️ Configuration and Installation

Since the device itself is based on ESP32, installation is not different from any other case when flashed over USB. Here is how my configuration for my devices looks:

```yaml
esphome:
    name: roomsense-iq-office
    friendly_name: RoomsenseIQ Office
    name_add_mac_suffix: true
    comment: Esphome Version of RoomsenseIQ firmware

esp32:
    board: esp32-s3-devkitc-1
    variant: esp32s3
    framework:
        type: esp-idf

packages:
    # I'm using shared configuration across multiple esp-home devices 
    wifi: !include common/wifi.yaml
    common: !include common/common.config.yaml
    ble_presence_sensors: !include common/ble_presence_sensors.yaml
    
    # Here is how configuration from this repository is included
    RoomsenseIQ: 
        url: https://github.com/talmuth/RoomsenseIQ_Esphome
        # if you have base device only
        files: [roomsense-base.yaml]
        
        # or, if you also have ClimateSense attached
        files: [roomsense-base.yaml, climatesense.yaml]
        
        # or, you want to include specific sensors only
        files: [components/mpm10.yaml, components/debug.yaml]
```

## ⚠️ Notes

🚨 **CO sensor from ClimateSense (TGS5141) is connected to PIN20, and current ESPHome version doesn't allow ADC on this pin, so you will get an error while attempting to build/validate the configuration.**

```log
Failed config

sensor.adc: [source /config/.esphome/packages/b925f83b/components/tgs5141.yaml:2]
    
    ESP32S3 doesn't support ADC on this pin when Wi-Fi is configured.
```

To avoid this you can either not include the complete setup for ClimateSense, or modify the `final_validate_config` method in `/esphome/esphome/components/adc/sensor.py` file from your esphome installation:

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

### 📝 Todo

- [ ] Calibrate all ADC sensors according to the original firmware
- [ ] Implement on-device automation to use LED to show different statuses

## 🎉 Support This Project

If you find this project helpful, consider supporting me:  
[☕ Buy Me a Coffee](https://buymeacoffee.com/talmuth)
