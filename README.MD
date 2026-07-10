# ESP8266 Rogue AP / Evil-Twin Detector

A small, passive WiFi monitoring tool for the ESP8266 that watches your home
network's SSID and alerts you if something suspicious shows up — like a
second access point broadcasting the same network name from a different
BSSID (a classic evil-twin / rogue-AP setup).

It only listens. It does not send deauth frames, spin up a fake access
point to intercept traffic, or capture any credentials. Think of it as a
tripwire, not a weapon.

## What it does

- Scans nearby WiFi networks every 15 seconds
- Learns the legitimate BSSID(s) of your home SSID over several baseline
  scans, along with the **RSSI (signal strength) range** and channel each
  one normally shows
- Flags an alert if:
  - Your SSID appears from a **completely unknown BSSID** — likely evil
    twin with no attempt to spoof the MAC
  - A **known BSSID** (i.e. a MAC address matching your real router)
    suddenly shows a signal strength well outside its learned range, or
    switches channel unexpectedly — this catches an attacker who **spoofs
    your router's MAC address** to slip past a plain whitelist check,
    since a cloned device broadcasting from a different physical location
    almost never matches your router's RSSI footprint exactly
  - Alerts require the suspicious behavior to repeat across **2
    consecutive scan cycles** before firing, cutting down on false
    positives from normal signal fluctuation while still catching a
    sustained impersonation attempt
- Serves a live dashboard over its own local access point so you can check
  status from your phone without needing your main router online, including
  a live view of "pending" (not-yet-confirmed) suspicious sightings
- Blinks the onboard LED as a physical alert indicator
- Logs the last 20 alerts with timestamps

### Why RSSI fingerprinting matters

A naive detector just checks "is this BSSID in my known-good list?" — but
an attacker can trivially spoof their WiFi adapter's MAC address to match
your real router's BSSID, defeating that check instantly. This version
also fingerprints the **physical radio behavior** of your real router
(the RSSI range it's been observed at, its channel) so that even a
perfectly MAC-spoofed clone gets flagged if it's broadcasting from a
different location or channel than your actual hardware.

## What it deliberately does NOT do

- No deauthentication / disassociation packet injection
- No fake AP that impersonates your network to capture traffic
- No credential capture or captive portal
- No `wifi_send_pkt_freedom` or raw 802.11 frame injection anywhere in the code

This keeps it in "monitor and alert" territory — safe to run continuously
on your own network without doing anything that could be mistaken for an
attack against anyone else's.

## Hardware

Any ESP8266 board works: NodeMCU, Wemos D1 Mini, etc.

## Setup

1. Open `rogue_ap_detector.ino` in the Arduino IDE (or PlatformIO)
2. Set `homeSSID` at the top of the file to your network's name
3. Flash to your ESP8266
4. On boot, it creates its own AP:
   - SSID: `rogue-detector`
   - Password: `detector123`
   (change these in the code before deploying)
5. Connect to that AP from your phone and visit `http://192.168.4.1`
   to view the dashboard

## How the baseline works

On first boot, the device scans for your home SSID over 6 scan cycles and
remembers every BSSID (MAC address) it sees broadcasting that name, along
with the RSSI range and channel each one shows — this handles mesh routers
or repeaters that legitimately use multiple BSSIDs for one SSID. After
that, any new BSSID, or a known BSSID behaving abnormally, triggers an
alert once confirmed across 2 consecutive sc