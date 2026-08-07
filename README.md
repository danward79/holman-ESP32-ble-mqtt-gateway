# ESPHome Bridge for Holman Bluetooth Garden Light Controllers

[![ESPHome Version](https://img.shields.io/badge/ESPHome-v2026.7%2B-blue.svg)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Compatible-41BDF5.svg)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

An asynchronous ESPHome firmware bridge that brings **Holman Vibrance Warm White BLE Garden Light Controllers** into Home Assistant via MQTT with single-tile dashboard support, hardware confirmation, and auto-discovery.

---

## 🌟 Features

* **Home Assistant Auto-Discovery:** Automatically creates a native light entity with integrated power toggle and dimming slider.
* **Non-Optimistic State Updates:** The Home Assistant UI only reflects changes *after* the physical Bluetooth hardware confirms command execution over the air.
* **Command Debouncing:** Rapidly dragging dimming sliders queues only the latest target state, dropping intermediate frames to keep the BLE link stable.
* **Retained State Memory:** Retains confirmed state in MQTT across Home Assistant reboots without draining the controller's battery with continuous polling.
* **Hardware Safe:** Uses manual state override opcodes (`0x2E`) exclusively, ensuring saved controller timer schedules or system clock settings are never modified.

---

## 🛠️ Required Hardware

1. **Holman Garden Light Controller** (Vibrance Warm White Series / Bluetooth BLE).
2. **ESP32 Board** (e.g., Seeed Studio XIAO ESP32C3, ESP32 DevKit, or similar ESP32 board positioned within BLE range of the Holman controller).
3. **MQTT Broker** (e.g., Mosquitto addon running in Home Assistant).

---

## 🚀 Setup & Installation

### Step 1: Find Your Holman Controller's MAC Address
You need the MAC address of your Holman BLE controller. You can find this using:
* An app like **nRF Connect** (iOS/Android) while standing near the controller.
* ESPHome's `esp32_ble_tracker` component.

*Format example:* `F3:61:AC:E8:BB:78`

### Step 2: Configure `secrets.yaml`
1. Copy `secrets.yaml.example` to `secrets.yaml`:
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
2. Edit `secrets.yaml` with your Wi-Fi, MQTT, and MAC address details:
   ```yaml
   wifi_ssid: "MyHomeWiFi"
   wifi_password: "SuperSecretPassword"
   mqtt_broker: "192.168.1.50"
   mqtt_user: "mqtt-user"
   mqtt_password: "mqtt-password"
   holman_ble_mac: "F3:61:AC:E8:BB:78"
   ```

### Step 3: Flash the ESP32
Flash the `holman-garden-controller.yaml` file to your ESP32 board using the ESPHome CLI or ESPHome Dashboard:

```bash
esphome run holman-garden-controller.yaml
```

---

## 🖥️ Home Assistant Integration

Once the ESP32 connects to Wi-Fi and MQTT:
1. It automatically broadcasts an MQTT Auto-Discovery payload to `homeassistant/light/garden_lights/config`.
2. A new entity named **`light.garden_lights`** will appear under **Settings** → **Devices & Services** → **MQTT**.
3. Add a **Tile Card** or **Mushroom Light Card** to your dashboard and select `light.garden_lights`. Ensure the **Brightness Slider** feature is enabled.

---

## 🔬 Protocol Technical Notes

* **GATT Write Characteristic:** `0000f004-0000-1000-8000-00805f9b34fb`
* **GATT Notify Characteristic:** `a876f003-7f10-4d70-b606-7df77c3eee0c`
* **Authorization Sequence:** An initial 20-byte zero payload (`[0x00] * 20`) is required to authenticate the session before sending state parameters.
* **State Payload Structure (20 Bytes):**
  * **Manual OFF:** `00 2e ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00`
  * **Manual Dim:** `00 2e 00 03 7f 11 00 14 1e 00 ff 00 00 00 00 00 64 00 [BRIGHTNESS_BYTE] 00` (where `BRIGHTNESS_BYTE` scales linearly from `0x00` to `0xFF`).
