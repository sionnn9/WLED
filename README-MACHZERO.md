# WLED ESP32-C5 – Wi-Fi / mDNS Crash Analysis

## Round 1: Understanding the Issue

### Problem

When the ESP32-C5 switches from a 2.4 GHz Wi-Fi network to a 5 GHz network, the device crashes.

The suspected cause is related to **mDNS (Multicast DNS)**. mDNS provides a hostname for the device, allowing users to access WLED without entering its IP address.

During a Wi-Fi network change, the existing network interface is disconnected and rebuilt. However, mDNS may continue running and retain references to the old network interface. When that interface is destroyed, these references can become invalid, potentially causing a crash.

### Expected Behaviour

mDNS should stop before the existing Wi-Fi interface is destroyed and restart only after the device reconnects and obtains a valid IP address.

This should prevent stale network references and allow the device to switch networks without requiring a reboot.

### Relevant Codebase

- **File:** `wled00/wled.cpp`
- **Function:** `initConnection()`

Since this function handles Wi-Fi initialization and reconnection, it is the appropriate place to manage the mDNS lifecycle during network restarts.

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

**Important:** Networking crashes, such as the reported ESP32-C5 mDNS issue, may occur in the lwIP thread rather than the main application loop.

---

## Round 3: Root Cause Analysis

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

### 2. Why the Crash Happens

The suspected sequence is:

1. The user triggers a Wi-Fi band switch.
2. `initConnection()` starts the connection restart process.
3. `WiFi.disconnect(true)` disconnects and tears down the existing network interface.
4. mDNS remains active because `MDNS.end()` was not called.
5. mDNS may attempt to access the old network interface.
6. Invalid interface references can cause a crash in the networking stack.

The underlying issue is the lack of proper mDNS cleanup before Wi-Fi teardown.

### 3. Proposed Fix

The proposed solution is to manage the mDNS lifecycle alongside Wi-Fi reconnection.

1. Detect the Wi-Fi connection restart.
2. Call `MDNS.end()` before disconnecting the existing interface.
3. Disconnect and rebuild the Wi-Fi connection.
4. Reconnect to the new network.
5. Wait until the device obtains a valid IP address.
6. Restart mDNS using `MDNS.begin()`.

This ensures mDNS is initialized against the new network interface rather than retaining references to the old one.

---

## Round 4: Testing Strategy

The fix should be validated on an ESP32-C5 environment.

### Testing Steps

1. Build the patched WLED firmware for ESP32-C5.
2. Connect the device to a 2.4 GHz Wi-Fi network.
3. Verify that mDNS starts successfully.
4. Trigger the Wi-Fi band/network switch through the web interface.
5. Monitor serial logs during disconnection and reconnection.
6. Verify that mDNS stops before Wi-Fi teardown.
7. Confirm that mDNS restarts after the new IP address is assigned.
8. Check that the device remains operational without crashing or requiring a reboot.

### Expected Execution Flow

```text
       2.4 GHz Wi-Fi
              |
              v
       Trigger Wi-Fi switch
              |
              v
          MDNS.end()
              |
              v
       Wi-Fi disconnect
              |
              v
     Rebuild network interface
              |
              v
       Connect to new network
              |
              v
        Obtain valid IP
              |
              v
          MDNS.begin()
              |
              v
       Normal WLED operation
```

### Expected Result

- mDNS stops before the old Wi-Fi interface is destroyed.
- The new Wi-Fi connection is initialized successfully.
- mDNS restarts using the new network interface.
- No stale network references remain.
- The ESP32-C5 continues operating without a crash or manual reboot.

---

## Overall Objective

Fix the ESP32-C5 Wi-Fi switching crash by ensuring that mDNS is properly stopped before Wi-Fi teardown and restarted only after a successful reconnection.

> **Note:** The root cause and fix described above are the current working hypothesis and proposed approach. They should be confirmed through code inspection, serial logs and testing on the target hardware.
