# WLED ESP32-C5 Wi‑Fi / mDNS Crash Analysis

## Round 1

### The issue

When the device switches from a 2.4 GHz Wi‑Fi network to a 5 GHz network, it crashes on an ESP32-C5.

The root cause is the mDNS helper. mDNS creates a name for the device's IP address so WLED does not need to reference raw IPs everywhere. But when the Wi‑Fi network changes, the device disconnects and reconnects to the new band. During that transition, mDNS keeps running even though the old network object has already been torn down. The in-memory state is wiped, so mDNS is left pointing at stale or invalid memory. Once the board reconnects, the stale pointer causes the crash.

### What should happen

When switching from 2.4 GHz to 5 GHz, mDNS should not remain attached to the old Wi‑Fi object after it has been destroyed. The fix is to stop mDNS before rebuilding the Wi‑Fi connection, then start it again only after the board reconnects and has a valid IP address. That prevents the stale pointer and avoids needing a reboot.

### Codebase part

This logic lives in `wled.cpp`, specifically inside `initConnection()`. Since it is directly tied to Wi‑Fi connection setup and teardown, it belongs in the connection initialization flow.

---

## Round 2

### Repo architecture

The project is organized into three main areas:

1. Core Firmware (`wled00/`)
   - `wled_main.cpp`, `wled.cpp` handle startup, loop logic, and Wi‑Fi connection flow.
   - `wled_server.cpp` and `json.cpp` serve the web UI and API endpoints.
   - `FX.cpp` and `bus_manager.cpp` manage light effects and output timing.

2. Web Interface (`wled00/data/`)
   - Contains the HTML, CSS, and JS used by the browser UI.
   - During the build step, Node.js compiles these assets into embedded C++ headers that are stored in flash.

3. Modules and Build Settings (`usermods/` and `platformio.ini`)
   - `usermods/` holds optional add-ons such as sensors or displays.
   - `platformio.ini` defines supported builds for chips like ESP32, ESP32-C5, and ESP8266.

---

## Relevant code and root cause

### 1) Vulnerable code segment in `wled00/wled.cpp`

Inside `initConnection()`, the connection restart path calls the Wi‑Fi teardown immediately before the stack is fully reset.

```
DEBUG_PRINTLN(F("WiFi disconnect."));
WiFi.disconnect(true); // close old connections but the mdns keeps running
delay(5); // wait for hardware to be ready
#ifdef ESP8266
WiFi.setPhyMode(force802_3g ? WIFI_PHY_MODE_11G : WIFI_PHY_MODE_11N);
#endif
```

### 2) Why the crash happens

When the user forces a band switch from 2.4 GHz to 5 GHz, `initConnection()` runs.

`WiFi.disconnect(true)` destroys the active network interface object in the ESP-IDF stack. However, mDNS is still active in the background because `MDNS.end()` was not called before the disconnect. Once the underlying `esp_netif` object is torn down, the mDNS/LwIP thread tries to query interface state and crashes with a fault. This is consistent with the reported panic where the system faults while checking interface state.

### 3) Proposed fix

The intended repair is:

1. Trigger the band switch and call `initConnection()`.
2. Stop mDNS before tearing down the old Wi‑Fi object.
3. Disconnect and rebuild the network interface.
4. Reconnect to the new Wi‑Fi network and obtain a new IP.
5. Start mDNS again so it binds to the fresh Wi‑Fi object.
6. Continue running without a crash or reboot.

---

## Testing strategy

The fix should be validated with the Espressif simulator for the ESP32-C5:

- Load the patched firmware into the simulator.
- Connect to a 2.4 GHz network.
- Trigger the switch to 5 GHz from the web UI.
- Monitor serial logs during the transition.
- Confirm that mDNS stops before teardown and restarts cleanly after reconnect.

```text
2.4 GHz -> switch band -> 5 GHz
    |            |
    |            +--> WiFi reconnects
    |                     |
    +--> MDNS stopped before teardown
                             |
                             +--> MDNS restarted after IP acquired
```

This should confirm that the old Wi‑Fi object is no longer being referenced after destruction, eliminating the crash path.
