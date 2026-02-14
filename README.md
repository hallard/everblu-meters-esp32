# everblu-meters-esp32

ESP32 firmware to read water usage data from **Itron EverBlu Cyble Enhanced** meters using the RADIAN protocol over 433 MHz, with MQTT integration for home automation.

## Features

- Daily scheduled meter reading with deep sleep between cycles
- MQTT publishing (JSON + optional raw topics) with retained messages
- Automatic 433 MHz frequency scanning and calibration
- Configurable retry logic on read failure
- OTA firmware updates
- RGB LED status indication (NeoPixel)
- ESP32 and meter battery monitoring
- Multiple ESP32 board support (ESP32, ESP32-C3, ESP32-S2)

## Table of Contents

- [Hardware](#hardware)
- [Wiring](#wiring)
- [Configuration](#configuration)
- [Building & Flashing](#building--flashing)
- [MQTT Topics](#mqtt-topics)
- [LED Status Codes](#led-status-codes)
- [Troubleshooting](#troubleshooting)
- [Save Power](#save-power)
- [Origin & License](#origin--license)
- [Links](#links)

## Hardware

### Required

| Component | Description |
|-----------|-------------|
| ESP32 board | Any supported variant (see [Supported Boards](#supported-boards)) |
| CC1101 module | 433 MHz RF transceiver ([E07-400T10S][1]) or 868/915 MHz variant ([E07-900T10S][10]) |
| USB cable | For initial programming |
| WiFi network | 2.4 GHz |

### Optional

| Component | Description |
|-----------|-------------|
| LC709203F | I2C lithium battery fuel gauge |
| WS2812B LED | NeoPixel RGB status indicator |
| LiPo battery | For standalone battery-powered operation |

Ready-made hardware designs are available: [ESP32C3-CC1101][9] (ESP32-C3 + CC1101 integrated board) and [cc1101-e07-pi][7]. See the [ESP32C3-CC1101][9] and [cc1101-e07-pi][7] repos for DIY wiring instructions.

### Supported Boards

| Environment | Board | Notes |
|-------------|-------|-------|
| `esp32c3-cc1101` | [ESP32C3-CC1101][9] | **Default.** LC709203F + NeoPixel |
| `esp32dev` | ESP32 DevKit | NeoPixel support |
| `esp32pico` | Pico32 | Base config |
| `mhetlive` | MHET ESP32 Mini Kit | Base config |
| `lolin32-lite` | Lolin32 Lite | Base config |
| `garagestick` | Adafruit Metro ESP32-S2 | USB CDC |

## Wiring

### [ESP32C3-CC1101][9] (default `esp32c3-cc1101` environment)

| CC1101 Pin | ESP32-C3 GPIO | Function |
|------------|---------------|----------|
| GDO0 | GPIO 1 | Data ready interrupt |
| SCK | GPIO 6 | SPI clock |
| MOSI | GPIO 7 | SPI data out |
| MISO | GPIO 2 | SPI data in |
| CSn | GPIO 8 | SPI chip select |

| Peripheral | ESP32-C3 GPIO | Function |
|------------|---------------|----------|
| LC709203F SCL | GPIO 4 | I2C clock |
| LC709203F SDA | GPIO 5 | I2C data |
| NeoPixel | GPIO 3 | LED data |
| Button | GPIO 9 | User button |

Pin assignments for other boards are defined in their respective `[env:...]` sections of `platformio.ini`.

## Configuration

All configuration is done via build flags in `platformio.ini`.

### Meter Identity

Get the serial number and year from the label on your meter:

![Cyble Meter Label](pictures/meter_label.png)

```ini
-D METER_YEAR=22
-D METER_SERIAL=828979
```

> **Note:** Remove the leading `0` from the serial number. If your label reads `0828979`, enter `828979`.

### Wake Schedule & Retries

```ini
-D WAKE_HOUR=08         ; Hour to wake and read (local time, 0-23)
-D WAKE_MIN=15          ; Minute to wake and read (0-59)
-D RETRIES=6            ; Max retry attempts on failure
-D RETRIES_DELAY=15     ; Minutes between retries
```

With the example above the device wakes daily at **08:15**. On failure it retries every 15 minutes up to 6 times (last attempt at **09:45**), then sleeps until 08:15 the next day.

### WiFi

```ini
-D WIFI_SSID=\"YOUR-SSID\"
-D WIFI_PASS=\"YOUR-PASSWORD\"
-D TIME_ZONE=TZ_Europe_Paris    ; See src/timezone.h for all options
```

### MQTT

```ini
-D MQTT_SERVER=\"192.168.1.8\"
-D MQTT_PORT=1883
-D TOPIC_BASE=\"everblu/\"
-D MQTT_USER=\"\"               ; Leave empty for no auth
-D MQTT_PASS=\"\"               ; Leave empty for no auth
; -D PUBLISH_RAW                ; Uncomment to publish individual metric topics
```

### Optional Flags

| Flag | Description |
|------|-------------|
| `-D FORCE_SCAN` | Force a full frequency scan on next boot |
| `-D PUBLISH_RAW` | Publish individual raw metric topics in addition to JSON |
| `-D DEBUG_OUT=1` | Enable CC1101 SPI debug output on serial |

## Building & Flashing

This project uses [PlatformIO](https://platformio.org/).

```bash
# Build (default esp32c3-cc1101 environment)
pio run

# Build a specific environment
pio run -e esp32dev

# Upload
pio run --target upload

# Serial monitor
pio device monitor
```

OTA updates are available once the device is connected to WiFi (via ArduinoOTA).

## MQTT Topics

All topics are published under a base path built from your configuration:

```
<TOPIC_BASE>/cyble-<YEAR>-0<SERIAL>-esp<MAC>/
```

Example: `everblu/cyble-22-0828979-espbf84/`

The `esp<MAC>` suffix is auto-generated from the ESP32 MAC address, allowing multiple devices on the same broker.

### `json` -- Successful Read (retained)

Published to `<base>/json` on each successful meter read.

```json
{
    "ts": 1689656420,
    "date": "Tue Jul 18 07:00:20 2023",
    "esp_battery": {
        "percent": 97,
        "vbat": 4155
    },
    "liters": 467932,
    "battery": 173,
    "read": 228,
    "rssi": -87,
    "lqi": -120,
    "hours": "06:18"
}
```

| Field | Description |
|-------|-------------|
| `ts` | Unix timestamp |
| `date` | Human-readable date/time |
| `esp_battery` | ESP32 battery state (if hardware present) |
| `liters` | Total water consumption in liters |
| `battery` | Meter battery remaining (months) |
| `read` | Meter internal read counter |
| `rssi` | Received signal strength (dBm) |
| `lqi` | Link quality indicator |
| `hours` | Meter business hours window (e.g. `"06:18"` = 06:00-18:59) |

### `error` -- Read Failure

Published to `<base>/error` when a read attempt fails.

```json
{
    "ts": 1689626877,
    "date": "Mon Jul 17 22:47:57 2023",
    "esp_battery": {
        "percent": 90,
        "vbat": 4094
    },
    "type": "No Data",
    "retries": 4
}
```

The device follows the configured `RETRIES` / `RETRIES_DELAY` schedule before going back to sleep.

### `scanning` -- Frequency Scan Progress (retained)

Published to `<base>/scanning` in real time during a frequency scan.

```json
{
    "ts": 1689416825,
    "date": "Sat Jul 15 12:27:05 2023",
    "frequency": "433.8226",
    "register": "0x10AE7C",
    "rssi": -87,
    "lqi": -120,
    "result": 115
}
```

| `result` value | Meaning |
|----------------|---------|
| `> 0` | Success -- value is the meter read counter |
| `-1` | Read returned an error |
| `-2` | No data received |

### `scan` -- Frequency Scan Complete

Published to `<base>/scan` when scanning finishes.

**Frequency found:**

```json
{
    "ts": 1689419164,
    "date": "Sat Jul 15 13:06:04 2023",
    "frequency": "433.8000",
    "min": "433.7900",
    "max": "433.8100"
}
```

`frequency` is the center of the working range. If `frequency` is `"0.0000"`, no valid frequency was found.

### `sleep_until` -- Next Wake Time (retained)

Published to `<base>/sleep_until` before entering deep sleep.

```json
{
    "seconds": 29526,
    "ts": 1689656406,
    "date": "Tue Jul 18 07:00:06 2023"
}
```

### Raw Topics (optional)

When `PUBLISH_RAW` is enabled, individual values are also published:

| Topic suffix | Value |
|-------------|-------|
| `liters` | Total water consumption |
| `battery` | Meter battery (months remaining) |
| `read` | Meter read counter |
| `ts` | Unix timestamp |
| `date` | Human-readable date |
| `hours` | Business hours window |
| `rssi` | Signal strength |
| `lqi` | Link quality |
| `esp_bat_pc` | ESP32 battery percent (if available) |
| `esp_bat_mv` | ESP32 battery voltage in mV (if available) |

## LED Status Codes

| Color | Meaning |
|-------|---------|
| Yellow | Boot / startup |
| Cyan | Connecting to WiFi |
| Pink | WiFi connected / NTP sync |
| Blue | Testing frequency (during scan) |
| Green | Successful read |
| Orange | Read error during scan |
| Red | Fatal error |

## Troubleshooting

### Frequency Calibration

If the device cannot communicate with the meter, the CC1101 module may need frequency calibration. Enable a full scan by adding `FORCE_SCAN` to your build flags:

```ini
-D FORCE_SCAN
```

The device will scan around 433.82 MHz, find the working range, and persist the calibrated frequency. Subsequent boots use the saved frequency automatically. Remove the flag after a successful scan.

### Business Hours

Meters only respond during their configured business hours (typically 06:00-18:59 local time). If you cannot get a reading, try between **10:00 and 16:00** on a weekday. Some meters do not respond on weekends.

The `hours` field in the JSON response indicates the meter's active window (e.g. `"06:18"` means 06:00 to 18:59).

### Serial Number Leading Zero

The configuration expects the serial **without** the leading zero. If your meter label reads `0828979`, set `METER_SERIAL=828979`. The MQTT topic will automatically include the leading zero.

## Save Power

The meter has an internal battery rated for ~10 years at one read per day.

> **Do not query your meter more than once a day.** Excessive queries will drain the meter battery prematurely, potentially requiring a costly meter replacement by your water utility.

## Origin & License

Original reverse-engineering work by [lamaisonsimon][4], published on GitHub by [@neutrinus][5], then ported to ESP8266/ESP32 by [psykokwak][6]. This fork adds ESP32-C3 support, deep sleep, battery monitoring, RGB LED, and MQTT enhancements.

The license is unspecified. Citing one of the original authors:

> I didn't put a license on this code maybe I should, I didn't know much about it in terms of licensing. This code was made by "looking" at the radian protocol which is said to be open source.

## Links

- [Blog & other projects][2]
- [CC1101 E07 433 MHz module][1]
- [CC1101 E07 868/915 MHz module][10]
- [ESP32C3-CC1101 hardware][9]
- [cc1101-e07-pi hardware][7]
- [everblu-meters (original)][5]
- [everblu-meters-esp8266 (psykokwak fork)][6]
- [everblu-meters-pi][8]

[1]: https://www.cdebyte.com/products/E07-400T10S
[2]: https://hallard.me
[3]: https://oshpark.com/shared_projects/BVwV2j3b
[4]: http://www.lamaisonsimon.fr/wiki/doku.php?id=maison2:compteur_d_eau:compteur_d_eau
[5]: https://github.com/neutrinus/everblu-meters
[6]: https://github.com/psykokwak-com/everblu-meters-esp8266
[7]: https://github.com/hallard/cc1101-e07-pi
[8]: https://github.com/hallard/everblu-meters-pi
[9]: https://github.com/hallard/ESP32C3-CC1101
[10]: https://www.cdebyte.com/products/E07-900T10S
