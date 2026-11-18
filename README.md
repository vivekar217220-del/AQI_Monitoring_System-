# ESP32 Air Quality Monitor

An ESP32 prototype that displays environmental readings and uses simulated PM2.5 to demonstrate an AQI scale, RGB status lights, and a buzzer alert.

**Demos** · [Hardware walkthrough](<WhatsApp Video 2026-11-02 at 7.13.06 PM.mp4>) · [HTTP data sharing](<HTTP_DATA_SHARING (1).mp4>) · [MQTT data sharing](<MQTT_DATA_SHARING (1).mp4>)

| Input or output | Role |
| --- | --- |
| DHT11 | Measures temperature and humidity. |
| MQ-7 | Reports analog sensor voltage after warm-up. |
| Simulated PM2.5 | Drives the illustrative AQI, alarm threshold, and displayed fan-speed percentage. |
| SSD1306 OLED and serial monitor | Display readings and device status. |
| RGB LED and buzzer | Indicate the AQI range and alert at the configured threshold of 100. |

## Try the sketch

1. Follow the [connection table](Connections), including the MQ-7 analog voltage divider.
2. Open the [source](Code) as an Arduino sketch for **ESP32 Dev Module**. Install the DHT sensor, Adafruit SSD1306, and Adafruit GFX libraries.
3. Configure the Wi-Fi placeholders for a local 2.4 GHz network. Upload and open the Serial Monitor at **115200 baud**.

The checked-in sketch connects to Wi-Fi and reports locally. The HTTP and MQTT recordings show separate data-sharing experiments.
