# WLED ESP32-C5 – Wi-Fi / mDNS Crash Analysis

## Round 1: Understanding the Issue

### Problem

When the ESP32-C5 switches from a 2.4 GHz Wi-Fi network to a 5 GHz network, the device reportedly crashes.

The suspected cause is related to **mDNS (Multicast DNS)**. mDNS provides a hostname for the device, allowing users to access WLED without entering its IP address.

During a Wi-Fi network change, the existing network interface is disconnected and rebuilt. However, mDNS may continue running and retain references to the old network interface. If that interface is destroyed, those references may become invalid and cause a crash.

### Expected Behaviour

mDNS should stop before the existing Wi-Fi interface is destroyed. It should restart only after the device reconnects and obtains a valid IP address.

This should prevent mDNS from using stale network-interface references during reconnection.

### Relevant Codebase

- **File:** `wled00/wled.cpp`
- **Function:** `initConnection()`

This function handles Wi-Fi initialization and reconnection, making it the relevant place to investigate the mDNS lifecycle during a network restart.

---

## Round 2: Understanding the Repository Architecture

### 1. Core Firmware – `wled00/`

| File | Responsibility |
|---|---|
| `wled_main.cpp` | Arduino `setup()` and `loop()` entry points |
| `wled.cpp` | Core WLED class, boot, Wi-Fi initialization and interfaces |
| `wled_server.cpp` | Web server routes |
| `json.cpp` | JSON API |
| `ws.cpp` | WebSocket communication |
| `set.cpp` | Settings POST handling |
| `xml.cpp` | Server-generated settings pages |
| `cfg.cpp` | Configuration persistence |
| `network.cpp` | Wi-Fi events |
| `udp.cpp` | UDP synchronization, E1.31 and DDP |
| `mqtt.cpp` | MQTT communication |
| `ntp.cpp` | Network time synchronization |
| `improv.cpp` | Improv Wi-Fi provisioning |

**Effects and LED output:**

- `FX.cpp`, `FX_fcn.cpp`, `FX_2Dfcn.cpp`, `FXparticleSystem.cpp` – LED effects.
- `palettes.cpp`, `colors.cpp` – Colour management.
- `bus_manager.cpp` – LED bus management.
- `src/WLEDpixelBus/` – Pixel output backends.

### 2. Web Interface – `wled00/data/`

Contains the HTML, CSS and JavaScript sources for WLED's web interface.

- Settings are separated into individual files, such as `settings_wifi.htm`.
- `npm run build` runs `tools/cdata.js`.
- The build process minifies and compresses the interface into generated `.h` files.
- These generated headers are compiled into firmware and should not be edited manually.

### 3. Modules and Build Configuration

- `usermods/` – Optional user extensions.
- `um_manager.cpp` and `usermod.cpp` – Usermod management.
- `platformio.ini` – Build environments, feature flags and board configurations.
- `pio-scripts/` – Python build scripts.
- `tools/` – Build utilities and tests.
- `.github/workflows/wled-ci.yml` – CI firmware compilation and module validation.

### 4. Runtime Architecture

WLED is not limited to a single-threaded execution model.

It uses:

- Arduino `loop()` for the main application.
- FreeRTOS for asynchronous tasks.
- ESP-IDF Wi-Fi and lwIP TCP/IP threads for networking.
- An asynchronous TCP server for web communication.

Networking failures may occur in a networking task or system thread rather than directly in the main application loop. The exact crash context should be confirmed from the panic/backtrace.

---

## Relevant Code and Root Cause

### 1. Vulnerable Code

**File:** `wled00/wled.cpp`  
**Function:** `initConnection()`

The relevant Wi-Fi restart code:

```cpp
DEBUG_PRINTLN(F("WiFi disconnect."));
WiFi.disconnect(true); // Close old connections, but mDNS keeps running
delay(5);               // Wait for hardware to be ready

#ifdef ESP8266
WiFi.setPhyMode(force802_3g ? WIFI_PHY_MODE_11G : WIFI_PHY_MODE_11N);
#endif
```

### 2. Suspected Crash Sequence

1. The user triggers a Wi-Fi connection or band change.
2. `initConnection()` starts the connection restart process.
3. `WiFi.disconnect(true)` disconnects or tears down the existing network interface.
4. mDNS may remain active because `MDNS.end()` was not called.
5. mDNS may attempt to access an interface that is no longer valid.
6. This may cause a fault in the networking stack.

This is the current root-cause hypothesis and should be checked against the ESP32-C5 panic log, backtrace and the exact Wi-Fi/mDNS library implementation.

### 3. Proposed Fix

Manage the mDNS lifecycle alongside Wi-Fi reconnection:

1. Stop mDNS by calling `MDNS.end()` before Wi-Fi teardown, if mDNS is running.
2. Disconnect and rebuild the Wi-Fi connection.
3. Reconnect to the network.
4. Wait until the device has a valid IP address.
5. Start mDNS again using `MDNS.begin()` and handle any initialization failure.

The implementation should follow the mDNS API and connection-event flow used by the ESP32 Arduino core version in the WLED build.

---

## Testing Through Espressif's Online ESP-EMU

Espressif provides a browser-based ESP emulator:

- **Online emulator:** https://espressif.github.io/esp-emulator/
- **Espressif documentation:** https://github.com/espressif/esp-emulator

The emulator lists ESP32-C5 support, but its current documented limitations are important:

- ESP32-C5 2.4 GHz Wi-Fi is modelled.
- 5 GHz Wi-Fi is not modelled.
- The hosted browser version does not provide networking by itself; network testing requires its local relay.
- The emulator expects a compatible merged firmware image. Confirm that the WLED build can be produced in a supported format before attempting to load it.

### What Can Be Checked Online

If a compatible WLED firmware image can be loaded, use the emulator to inspect whether the firmware boots and to review available console output. This may help with basic startup checks, but it cannot reproduce the actual 2.4 GHz-to-5 GHz transition.

### What Requires Real Hardware or a Suitable RF Test Setup

To validate the reported band-switch crash:

1. Build WLED with the proposed mDNS lifecycle change.
2. Flash it to an ESP32-C5 development board.
3. Connect it to a supported 2.4 GHz access point.
4. Trigger the relevant Wi-Fi reconnection or band-selection path.
5. Capture serial logs and any panic/backtrace.
6. Verify that mDNS stops before interface teardown.
7. Verify that mDNS starts again only after the new interface has a valid IP.
8. Repeat the reconnection test and confirm that the device remains stable.

**Important limitation:** The Espressif online ESP-EMU cannot verify a real 2.4 GHz-to-5 GHz switch because 5 GHz Wi-Fi is not currently modelled. Use it only for the checks it supports; validate the actual crash fix on suitable hardware.

---

## Overall Objective

Investigate and fix the ESP32-C5 Wi-Fi reconnection crash by ensuring that mDNS is stopped before Wi-Fi interface teardown and restarted after a successful reconnection and IP assignment.
