# 05 - Home Assistant IoT segment

A local-first smart home on its own VLAN, designed for a rental: battery sensors, plug-in
controls and bulbs only. No in-wall switches, no thermostat or deadbolt replacement without
written permission from the landlord.

## 1. Network first

- IoT VLAN 20, `203.0.113.0/24`, gateway OPNsense.
- Dedicated SSID, **2.4 GHz only**, WPA2/WPA3, client isolation on.
- Firewall: IoT cannot initiate to TRUSTED; Home Assistant can reach IoT. mDNS repeater only if
  needed. Enable IPv6 on the VLAN if you add Matter-over-Wi-Fi devices.
- **Radio planning:** Zigbee and Thread share 2.4 GHz with Wi-Fi. Put the Wi-Fi 2.4 GHz radio on
  channel 1 or 11 and Zigbee on channel 15, 20 or 25 so they do not overlap.

Test: a phone on the IoT SSID cannot reach a laptop on TRUSTED, and Home Assistant can reach the phone.

## 2. Home Assistant host

- HAOS on a small always-on x86 box (N100/N150 mini PC) or a VM with the radio passed through.
- Wired Ethernet, SSD (not SD card). Home Assistant Green is the low-maintenance option (no radio built in).
- Cloud hubs (SmartThings, voice assistants) become HA clients, never the controller.
- Nightly backups to the lab NAS. Remote access via Nabu Casa or your own VPN, not port forwards.

## 3. Radios

One mesh, placed well, beats three protocols.

| Role | Option | Why |
|---|---|---|
| Zigbee coordinator | SMLIGHT SLZB-06 family (PoE) | Network-attached, so it can sit mid-home instead of in a metal rack. Powered from a PoE switch port |
| USB fallback | Home Assistant Connect ZBT-2, Sonoff ZBDongle-P | Always on a USB extension cable, away from USB 3.0 noise |
| Z-Wave (later) | Home Assistant Connect ZWA-2, Zooz 800 | Only for locks or water valves. 908 MHz, ignores Wi-Fi |
| Thread (later) | Second radio | One radio cannot run Zigbee and Thread at once |

Choose Zigbee2MQTT (more control, OTA, quirks) or ZHA (fewer moving parts) and stay with it;
changing coordinators means re-pairing.

**Mesh rule:** battery sensors do not route. Every room needs at least one mains-powered Zigbee
device (plug or bulb).

## 4. Device plan by area

| Area | Devices | Automations |
|---|---|---|
| Exterior and bedroom doors | Adhesive Zigbee contact sensors | Lights after sunset, presence, away mode |
| Hall, bedrooms | Zigbee PIR (simpler and more private than Wi-Fi mmWave), one bulb or plug per room | Lighting, mesh routing |
| Bathrooms | Leak sensors by supply lines and toilet | Highest value sensor in a rental |
| Kitchen | Leak sensor under sink, plug on small appliance, scene button | "Kitchen off" |
| Living room | Plugs on lamps / bias light | Media stays on TRUSTED for casting |
| Climate | Temp/humidity sensors; energy-monitoring plug on window/portable AC units | Cut units left running; one week of baseline before schedules |
| Entry | Retrofit lock only (existing deadbolt stays), camera on your own door only | After the lease question is settled |
| Smoke/CO | Leave building alarms alone; optional listener device | Notify only |

Skip: in-wall switches, hardwired shutoffs, anything needing a neutral, and Wi-Fi bulbs as the
main fleet (dozens of Wi-Fi clients will clog a consumer AP).

## 5. Order of operations

1. VLAN, IoT SSID, HAOS on Ethernet; verify isolation.
2. Coordinator on PoE, mid-home, Zigbee channel chosen against Wi-Fi. Pair three plugs and confirm routing.
3. Leak sensors and door contacts; notifications.
4. Room lighting. Presence only once the mesh is stable.
5. Energy plugs on the big loads.
6. Lock / Z-Wave last.
7. Dashboards per area, plus an away mode that cuts plugs and notifies on door or leak.

Budget guide: coordinator $40-80, plugs $15-25 each, contact/leak sensors $10-20 each.
