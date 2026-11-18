# ESP32 Air Quality Monitoring System

An embedded prototype for reading environmental sensors, displaying measurements on a local screen, and testing the path from a device to a web dashboard. The ESP32 reads temperature and humidity from a DHT11, samples the MQ-7 module's analog output, and uses simulated PM2.5 to calculate a demonstration air quality index (AQI).

**Project status:** The checked-in firmware joins Wi-Fi and reports locally. The repository also contains separate HTTP and MQTT demonstrations. Integrating telemetry into the main sketch and building the web dashboard are next steps.

## Project demonstrations

| Recording | What it covers |
| --- | --- |
| [Hardware demonstration](<WhatsApp Video 2026-11-02 at 7.13.06 PM.mp4>) | ESP32 prototype and local monitoring hardware |
| [HTTP data sharing](<HTTP_DATA_SHARING (1).mp4>) | Experiment with sending readings to an HTTP endpoint |
| [MQTT data sharing](<MQTT_DATA_SHARING (1).mp4>) | Experiment with publishing readings through a broker |

## System architecture

```text
DHT11 (temperature / humidity) ───────┐
MQ-7 (analog voltage through divider) ─┼──> ESP32 ──> OLED + Serial Monitor
Simulated PM2.5 ──────────────────────┘      │
       │                                     └──> RGB status LED + buzzer
       └──> Demo AQI and fan-speed estimate
```

- **Measured:** temperature and relative humidity from the DHT11, plus the analog output of the MQ-7 module. The firmware reports the raw ADC value and estimates the module output voltage. After its configured warm-up interval, it also collects a resistance baseline for the serial diagnostic.
- **Simulated:** a bounded random walk generates PM2.5 samples. The sketch interpolates between its programmed concentration breakpoints to calculate a PM2.5-based demonstration index. Temperature, humidity, and MQ-7 readings are separate outputs.
- **Local feedback:** The OLED presents the latest AQI category, PM2.5, temperature, humidity, MQ-7 warm-up or voltage, and Wi-Fi signal strength. Serial output adds a sequence number, uptime, ADC data, and diagnostics.
- **Fan display:** A weighted blend of low, medium, and high PM2.5 conditions produces a suggested fan percentage. In this sketch the percentage is displayed only.

## Hardware and wiring

The build uses an ESP32 Dev Module, DHT11, MQ-7 module, 128×64 SSD1306 I²C OLED, common-cathode RGB LED, buzzer, breadboard, jumper wires, and two 10 kΩ resistors. Add an individual series resistor to each RGB channel if the LED module has none.

| Component | Power and ground | Signal connection |
| --- | --- | --- |
| MQ-7 module | 5 V VIN and common GND | AO to the divider shown below, midpoint to GPIO34. Leave DO unconnected. |
| DHT11 | 3.3 V and common GND | DATA to GPIO4 |
| SSD1306 OLED | 3.3 V and common GND | SDA to GPIO21, SCL to GPIO22 |
| RGB LED | Common cathode to GND | Red to GPIO26, green to GPIO27, blue to GPIO14 |
| Buzzer | Common GND | Control on GPIO25 |

```text
MQ-7 AO ── 10 kΩ ──┬── 10 kΩ ── GND
                   │
                GPIO34
```

The equal-value resistors halve the analog voltage at GPIO34. The sketch applies the inverse divider ratio when it estimates the module's output voltage. See the [full connection notes](Connections).

## Build and run the local monitor

1. Wire the modules with power disconnected and check that all grounds are shared. Verify the MQ-7 divider before connecting its midpoint to GPIO34.
2. Install the ESP32 board package in the Arduino IDE and select **ESP32 Dev Module**. Install **DHT sensor library**, **Adafruit SSD1306**, and **Adafruit GFX Library** using Library Manager.
3. Create a new Arduino sketch and copy in the [firmware](Code). The repository stores the sketch in an extensionless file called `Code`.
4. Set the `WIFI_SSID` and `WIFI_PASS` placeholders locally for a 2.4 GHz network. Select the ESP32 serial port, upload the sketch, and open Serial Monitor at **115200 baud**.
5. At startup, check the RGB self-test and beep. Once Wi-Fi connects, look for a report with the device IP and subsequent readings roughly every 10 seconds. The MQ-7 voltage field on the OLED appears after the configured 120-second warm-up interval.

### Local settings and alerts

| Setting | Purpose | Default |
| --- | --- | --- |
| `READ_INTERVAL_MS` | Target interval between sensor reports | 10,000 ms |
| `ALERT_AQI` | Turns on the red LED and buzzer at or above this index | 100 |
| `CO_WARMUP_MS` | Firmware warm-up interval before collecting baseline samples | 120,000 ms |
| `OLED_ADDR` | I²C display address | `0x3C` |
| `RGB_COMMON_ANODE` | Selects the LED output polarity | `false` |
| `BUZZER_PULSED` | Selects a pulsing alarm | `true` |

| Demonstration AQI | RGB LED | Buzzer |
| --- | --- | --- |
| 50 or below | Green | Off |
| 51 to 99 | Yellow | Off |
| 100 or above | Red | Pulses by default |

The alert threshold and category label are separate. At an index of exactly 100, the category still displays “Satisfactory” while the red LED and buzzer activate.

## HTTP and MQTT integration design

The two transports can send the same reading format after each sampling cycle. The following is an example schema for integrating the demos with a receiver.

```json
{
  "device_id": "aqi-esp32-01",
  "sequence": 42,
  "temperature_c": 28.4,
  "humidity_pct": 56,
  "mq7_voltage_v": 0.84,
  "pm25_ug_m3": 61.0,
  "pm25_source": "simulated",
  "demo_aqi": 101,
  "alert": true
}
```

Use JSON `null` for unavailable values, such as a failed DHT11 read, and add a server-side UTC timestamp on receipt. The device sequence counter restarts on boot. Use a per-boot identifier with the device ID and sequence number if the backend needs to deduplicate retried messages.

### HTTP request flow

```text
ESP32 ── POST /api/readings ──> HTTP receiver ──> storage
Dashboard ── GET /api/readings ──> HTTP receiver ──> history
```

1. The ESP32 serializes the reading as JSON and sends a POST with `Content-Type: application/json`.
2. The receiver validates the payload, records the receive time, and returns a `2xx` response after accepting it.
3. The ESP32 can retry temporary network or server failures with a delay. For failed validation, correct the payload before resending.
4. The dashboard requests the latest reading and history through read endpoints. Use HTTPS for a remote deployment.

### MQTT publish and subscribe flow

```text
ESP32 ── publish ──> broker ── subscription ──> ingestion service ──> dashboard/storage
```

1. The ESP32 connects to a broker and publishes the reading on a per-device topic such as `aqi/aqi-esp32-01/telemetry`.
2. A subscriber receives that topic, validates and timestamps the message, and writes the reading to the same storage used by the dashboard.
3. For a remote deployment, use a TLS connection and credentials scoped to the allowed topics. If the client uses QoS 1, handle potential duplicate deliveries in the subscriber.

When bringing either transport into the main sketch, keep reconnect and upload work separate from local sampling and alerts. The current sketch waits for Wi-Fi at startup and blocks while reconnecting.

## Bring-up and troubleshooting

| Check | What to look for |
| --- | --- |
| Wi-Fi | Serial output distinguishes an SSID it cannot find from a failed association. Verify the 2.4 GHz network name and local credentials. |
| OLED | Confirm SDA and SCL are on the documented pins. Check the module address if the screen is blank. The sketch uses `0x3C` and suggests trying `0x3D`. |
| DHT11 | If the serial report shows “NO READING,” check DATA on GPIO4, 3.3 V, and common ground. |
| MQ-7 | Verify 5 V supply, common ground, divider orientation, and the GPIO34 midpoint if the raw ADC value is near zero. |
| RGB and buzzer | Check the startup self-test first. For a repeatable bench test, use a temporary fixed simulated input and exercise values below and at the alert threshold. |
| Telemetry integration | Compare the serial sequence number with the HTTP receiver log or MQTT subscriber. Check that the stored sample preserves the simulation flag and timestamp. |

## Current limits and next steps

- The PM2.5 input is synthetic. The displayed index is a demonstration and the MQ-7 output is reported as voltage, without a calibrated CO concentration.
- The fan percentage is a firmware calculation. The repository does not contain a fan actuator circuit.
- A future hardware step is to add a particulate sensor and test the concentration-to-index boundary values. A separate software step is to integrate the data-sharing path and show the latest reading and history on a web dashboard.

## Repository contents

| Item | Contents |
| --- | --- |
| [Code](Code) | ESP32 sketch for sampling, Wi-Fi, local output, and alerts |
| [Connections](Connections) | Wiring table and MQ-7 voltage divider |
| [Demonstrations](#project-demonstrations) | Hardware, HTTP, and MQTT recordings |

**Author:** Vivek Chaudhary
