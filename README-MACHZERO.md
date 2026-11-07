## ROUND 1

**THE ISSUE**
So the issue is that whenever you switch the Wi-Fi from 2.4GHz to 5GHz, it crashes. Its using an ESP32-C5. The crash occurs because of the helper mDNS. mDNS helps by creating a name for the IP address, so we don’t have to use the IP address everywhere. But when you switch Wi-Fi, it disconnects and reconnects to the 5GHz network. When that happens, mDNS continues to run, but the memory is wiped, so whatever data was in the object gets erased. Now mDNS is not pointing anywhere as the object got deleted , so when you reconnect it gives an error. This is what happens right now, and it causes the crash.

**WHAT SHOULD HAPPEN**
What should happen is that when you switch from 2.4GHz to 5GHz, mDNS shouldn't be left pointing at the old Wi-Fi object after it gets erased. Our solution, which is what we thought of: when you change the Wi-Fi band from 2.4GHz to 5GHz, we stop mDNS before the Wi-Fi is rebuilt. Once the board is connected to the new Wi-Fi and has an IP, we start mDNS again so it attaches to the new Wi-Fi object, and you won't have to reboot the board to recover.

**CODEBASE PART**
This code is in wled.cpp, in the function initConnection(). Since it is related to Wi-Fi and connection, it will fall in the initConnection() function.

## ROUND 2

THE ARCHITECTURE OF THE REPO

The project is built around three main parts:

**1.Core Firmware (wled00/)**
This is where all the board logic lives.

Entry & Setup (wled_main.cpp, wled.cpp): Manages system boot, main loops, and Wi-Fi connection flows.

Web Server & APIs (wled_server.cpp, json.cpp): Serves the control interface and handles incoming commands.

Lights & Effects (FX.cpp, bus_manager.cpp): Calculates pixel animation colors and sends signal data to the physical GPIO pins.

**2.Web Interface (wled00/data/)**
Contains the frontend files (HTML, JS, CSS) used to control WLED from a browser. During the build step, Node.js compiles these files into C++ header code so they can be stored directly inside the board's flash memory.

**3.Modules & Build Settings (usermods/ & platformio.ini)**

usermods/: A directory for adding optional extras like sensors or displays without altering core code.

platformio.ini: Configures targets and compilation settings for supported microcontrollers like ESP32, ESP32-C5, or ESP8266.

## Understand the codebase and identify the code relevant to your selected issue.

**1. The Vulnerable Code Segment in wled00/wled.cpp**

Inside wled.cpp, the initConnection() function is called whenever WLED needs to start a connection or perform a forced reconnection (e.g., when switching Wi-Fi bands from 2.4 GHz to 5 GHz on an ESP32-C5).

Here is the exact section where the crash is triggered:

void WLED::initConnection()
{
#if defined(ESP8266) || defined(ESP32)
if (WLED_WIFI_CONFIGURED)
{
DEBUG_PRINTLN(F("initConnection() called"));

    //  THE BUG: WiFi driver teardown occurs HERE while mDNS thread is still active!
    WiFi.disconnect(true);

    // ... Wi-Fi band & mode configuration ...
    WiFi.mode(WIFI_STA);

    // ... setup AP/STA parameters ...

}
#endif
}

**2. Why the Crash Happens in wled.cpp**

When the user changes settings to force 5 GHz, initConnection() is triggered.

WiFi.disconnect(true) is executed immediately. This commands the underlying ESP-IDF stack to destroy the active esp_netif interface object.

However, mDNS is still running asynchronously in the background because MDNS.end() was never called before WiFi.disconnect(true).

As esp_netif is destroyed, the mDNS LwIP thread (pcb_if_init_lwip) tries to check interface status via esp_netif_is_netif_up(), leading to a Load Access Fault (Panic Core 0 at address 0x000000e8).

**3. The Proposed Fix in wled00/wled.cpp**

To prevent the crash, gracefully shut down the mDNS service before tearing down the network interface:

void WLED::initConnection()
{
#if defined(ESP8266) || defined(ESP32)
if (WLED_WIFI_CONFIGURED)
{
DEBUG_PRINTLN(F("initConnection() called"));

    //  FIX: Safely stop mDNS before tearing down the network interface
    #ifndef WLED_DISABLE_MDNS
    if (espmDNSstarted) {
      MDNS.end();
      espmDNSstarted = false;
      DEBUG_PRINTLN(F("mDNS stopped prior to network re-init"));
    }
    #endif

    // Now safely tear down the Wi-Fi connection
    WiFi.disconnect(true);

    // Re-initialization logic continues normally...

}
#endif
}

Once the board completes the 5 GHz handshake, initInterfaces() in wled.cpp will automatically restart mDNS via MDNS.begin() when WiFi.status() == WL_CONNECTED.
