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

* **GATT Service UUID:** `a876f000-7f10-4d70-b606-7df77c3eee0c`
* **GATT Write Characteristic:** `0000f004-0000-1000-8000-00805f9b34fb`
* **GATT Notify Characteristic:** `a876f003-7f10-4d70-b606-7df77c3eee0c`
* **Authorization Sequence:** An initial 20-byte zero payload (`[0x00] * 20`) must be written to authenticate the GATT connection before sending state alterations.

#### 20-Byte Payload Map

| Byte Index | Field Name | Standard Value | Description |
| :--- | :--- | :--- | :--- |
| **`[00 - 01]`** | **Header** | `00 2E` | Operation Command (`0x2E` = Direct Manual Override) |
| **`[02]`** | **Scene ID** | `00` | Target scene / channel index (`0x00` = Scene 1) |
| **`[03]`** | **Timer Control Flags** | `03` | Bitwise flags: <br> • Bit 0: Start time enable (`1`) <br> • Bit 1: Stop time enable (`1`) |
| **`[04]`** | **Day Mask** | `7F` | `0x7F` (`01111111` binary) = Enabled for all 7 days |
| **`[05 - 06]`** | **Start Time** | `11 00` | Timer start time (`17:00` / 5:00 PM) |
| **`[07 - 08]`** | **Stop Time** | `14 1E` | Timer stop time (`20:30` / 8:30 PM) |
| **`[09]`** | **Reserved** | `00` | Unknown / Padding |
| **`[10 - 15]`** | **Padding Block** | `FF 00 00 00 00 00` | Static protocol boundary padding |
| **`[16 - 17]`** | **Fixed Scale** | `64 00` | Scale maximum (`100` in decimal) |
| **`[18]`** | **Brightness Value** | `00` - `FF` | Target dimming level (`0` = OFF, `255` = 100% Brightness) |
| **`[19]`** | **Tail Byte** | `00` | End-of-frame packet delimiter |

#### Command Payload Examples

* **Manual OFF Command:**
  ```text
  00 2e ff 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  ```
* **Manual ON (80% Brightness / Byte 18 = `0xCC`):**
  ```text
  00 2e 00 03 7f 11 00 14 1e 00 ff 00 00 00 00 00 64 00 cc 00
  ```
