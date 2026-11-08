# WLED ESP32-C5 – Wi-Fi / mDNS Crash Analysis

## Round 1: Understanding the Issue

*Repository:* [WLED](https://github.com/wled/WLED) is open-source firmware that controls addressable LED strips (WS2812B, SK6812 and many others) from an ESP32 board over Wi-Fi. It has about 19k stars, 4.5k forks and 336 contributors, and it is mostly C++ (79%). It is licensed under EUPL-1.2.

The web page, the phone app, Home Assistant, MQTT, Alexa and the JSON API all reach the lights through Wi-Fi, so the Wi-Fi code is central to the project. The core firmware is in ⁠ wled00/ ⁠, the web interface is in ⁠ wled00/data/ ⁠, optional add-ons are in ⁠ usermods/ ⁠, and build settings are in ⁠ platformio.ini ⁠.

*Issue:* [#5945](https://github.com/wled/WLED/issues/5945). The ESP32-C5 is the first ESP32 chip with 5 GHz Wi-Fi, and WLED still marks its builds as experimental. When the Wi-Fi band is switched, the board can crash inside mDNS, the service that gives it the ⁠ wled-xxxxxx.local ⁠ name. A reboot fixes it. The issue is labelled as a bug, is on the 17.0 milestone, and had no pull request or assignee when we checked on 7 November.

*Cause:* when the band changes, ⁠ initConnection() ⁠ in ⁠ wled00/wled.cpp ⁠ destroys the Wi-Fi interface with ⁠ WiFi.mode(WIFI_MODE_NULL) ⁠ while mDNS is still running.


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

### 2. Crash Sequence

1. The user triggers a Wi-Fi connection or band change.
2. `initConnection()` starts the connection restart process.
3. `WiFi.disconnect(true)` and `WiFi.mode(WIFI_MODE_NULL)` tear down the network interface.
4. mDNS is still running, because nothing calls `MDNS.end()` on this path.
5. mDNS leaves a multicast group join pending for the destroyed interface.
6. The join runs on the lwIP thread after the interface is gone and dereferences it.

Steps 1-4 are visible in the source. Steps 5-6 were confirmed from the emulator panic log and backtrace described below, not only from reading the code: the fault is in the lwIP thread inside `join_group`, reached from mDNS, and not in `loop()`.

### 3. Proposed Fix

Manage the mDNS lifecycle alongside Wi-Fi reconnection:

1. Stop mDNS by calling `MDNS.end()` before Wi-Fi teardown, if mDNS is running.
2. Disconnect and rebuild the Wi-Fi connection.
3. Reconnect to the network.
4. Wait until the device has a valid IP address.
5. Restart mDNS after the connection is back, with its service records intact.

Note on step 5: a bare `MDNS.begin()` is not sufficient here. WLED already restarts mDNS inside `initInterfaces()`, which is called once the station has an IP, and that path re-registers the services and TXT records (`http`, `wled`, `mac`). Restarting with `MDNS.begin()` alone would bring the responder back without those records, so the restart should go through the existing path instead of a separate call.

This fix has not been validated on hardware yet; see the limitations at the end.

---

## Testing Through Espressif's Online ESP-EMU

Espressif provides a browser-based ESP emulator:

- **Online emulator:** https://espressif.github.io/esp-emulator/
- **Espressif documentation:** https://github.com/espressif/esp-emulator

The emulator runs the real firmware, but it does not model radio bands. Association always happens on the emulated access point, so a genuine 2.4 GHz-to-5 GHz association cannot be tested there. What the emulator *can* exercise is the code path a band change drives: `initConnection()` running `WiFi.mode(WIFI_MODE_NULL)` while mDNS is still up.

### Test Setup

- ESP32-C5 build with `WLED_DEBUG` enabled, merged into a 4 MB flash image and loaded into the online emulator, which reaches the local network through a relay on `localhost`.
- The emulated access point credentials are seeded in `cfg.json` and `wsec.json`, so the device connects without any UI interaction.
- The emulator has no usable settings UI from the page (the websocket relay is blocked on HTTPS), so a temporary harness at the top of `WLED::loop()` drove the change: once the station was connected **and** mDNS had started (`interfacesInited` is set right after `MDNS.begin()`), it waited 3 s, then set `wifiBandMode = WIFI_BAND_MODE_5G_ONLY` and `forceReconnect` - the same thing `set.cpp` does when the band dropdown changes. It then set the band back to `WIFI_BAND_MODE_AUTO` so the emulated radio could reconnect and restart mDNS, and repeated for 10 cycles.
- The harness is temporary and is not part of the proposed fix.

### Result: the crash reproduces

The device panicked on **cycle 2 of 10**, while mDNS was running and the band switch was pending:

```text
[TEST] band cycle 2/10 -> 5GHz only (mDNS active before teardown)
Initial connect or forced reconnect (@ 35s).
WiFi: Default SSID (myssid) used.
initConnection() called @ 35s.
WiFi disconnect.
WiFi-E: Disconnected! @ 35s
WiFi-E: STA Stopped
WiFi-E: Event 100
getWLEDHostname: 'wled-000001'
WiFi mode_null: driver teardown / re-init.
[TEST] post-teardown: mode NULL applied
WiFi-E: driver ready.
WiFi-E: STA Started
Access point disabled (init).
Guru Meditation Error: Core  0 panic'ed (Load access fault). Exception was unhandled.

MEPC    : 0x42096d4e  RA      : 0x420a34ea  SP      : 0x4082a1e0
MCAUSE  : 0x00000005  MTVAL   : 0x000000f3
```

`MCAUSE 5` is a load access fault and `MTVAL 0x000000f3` is the address being read, which is consistent with a stale netif pointer being dereferenced. The fault comes after `mode NULL applied`, so the interface was already gone when the reader ran. Symbolised against `firmware.elf`:

| Address | Symbol |
|---|---|
| `MEPC 0x42096d4e` | `esp_netif_is_netif_up` |
| `RA 0x420a34ea` | `join_group` |
| `0x420a1830` | `mdns_priv_get_esp_netif` |
| `0x420a36ac` | `pcb_if_init_lwip` |
| `0x4208069a` | `tcpip_thread` |

The fault is on the lwIP (`tcpip_thread`) side, not in `loop()`. `join_group` is reached through mDNS code, and the netif reference it uses is already invalid. This matches the sequence above. It is also why the state matters more than which request triggered the reconnect: the crash needs an `initConnection()` that tears the interface down while mDNS is still running, which is the state a band change produces (`set.cpp` sets `forceReconnect` while the device is connected and mDNS is up).

### Video Evidence

- [Issue reproduction](Issue.mp4) - crash during the second cycle of a 10-cycle test.
- [Successful test](Solved.mp4) - no errors across three rounds of the 10-cycle test.

### What This Test Does Not Show

- **No 5 GHz association.** The emulated radio is single-band, so the station associated with the 2.4 GHz access point throughout. How the device behaves on a real 5 GHz network is not covered.
- **Band request result not yet recorded.** Whether the emulated driver accepts `WIFI_BAND_MODE_5G_ONLY` (rather than returning an error) is being instrumented separately; the crash reproduced on the teardown path regardless.
- **One reproduction.** The crash rate on hardware is unknown, so this shows the mechanism, not how often it triggers.

### What Requires Real Hardware

To validate the reported crash and any fix to it:

1. Build WLED with the proposed mDNS lifecycle change.
2. Flash it to an ESP32-C5 development board.
3. Connect it to a supported 2.4 GHz access point.
4. Trigger the relevant Wi-Fi reconnection or band-selection path.
5. Capture serial logs and any panic/backtrace.
6. Verify that mDNS stops before interface teardown.
7. Verify that mDNS starts again only after the new interface has a valid IP.
8. Repeat the reconnection test and confirm that the device remains stable.

**Important limitation:** the emulator reproduced the teardown race, but it cannot verify a real 2.4 GHz-to-5 GHz switch, because 5 GHz Wi-Fi is not modelled. Use it for the mechanism and for repeatability, and validate the actual crash fix on suitable hardware.

---

## Overall Objective

Investigate and fix the ESP32-C5 Wi-Fi reconnection crash by ensuring that mDNS is stopped before Wi-Fi interface teardown and restarted after a successful reconnection and IP assignment.

---

## Presentation

[Open the MACHZERO presentation](1.pdf)

---

## Software Requirements

- **Node.js 20 or newer** for the WLED web interface build and tests.
- **npm** for installing the Node.js dependencies.
- **Python 3.13** with the packages listed in [`requirements.txt`](requirements.txt) for PlatformIO.
- **PlatformIO 6.1.19** for compiling the firmware.
- A modern web browser for the [Espressif ESP-EMU](https://espressif.github.io/esp-emulator/).
- An ESP32-C5 development board and a supported 2.4 GHz/5 GHz Wi-Fi setup for hardware validation.

Install the project dependencies with:

```bash
npm ci
pip install -r requirements.txt
```
