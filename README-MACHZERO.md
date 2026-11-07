THE ISSUE
So the issue is that whenever you switch the Wi-Fi from 2.4GHz to 5GHz, it crashes. Its using an ESP32-C5. The crash occurs because of the helper mDNS. mDNS helps by creating a name for the IP address, so we don’t have to use the IP address everywhere. But when you switch Wi-Fi, it disconnects and reconnects to the 5GHz network. When that happens, mDNS continues to run, but the memory is wiped, so whatever data was in the object gets erased. Now mDNS is not pointing anywhere as the object got deleted , so when you reconnect it gives an error. This is what happens right now, and it causes the crash.

WHAT SHOULD HAPPEN
What should happen is that when you switch from 2.4GHz to 5GHz, mDNS shouldn't be left pointing at the old Wi-Fi object after it gets erased. Our solution, which is what we thought of: when you change the Wi-Fi band from 2.4GHz to 5GHz, we stop mDNS before the Wi-Fi is rebuilt. Once the board is connected to the new Wi-Fi and has an IP, we start mDNS again so it attaches to the new Wi-Fi object, and you won't have to reboot the board to recover.

CODEBASE PART
This code is in wled.cpp, in the function initConnection(). Since it is related to Wi-Fi and connection, it will fall in the initConnection() function.
