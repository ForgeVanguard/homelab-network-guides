# 04 - Consumer Wi-Fi router as an access point and switch

Example hardware: TP-Link Archer BE6500 (Wi-Fi 7, one 2.5G WAN, one 2.5G LAN, three 1G LAN).
The same approach applies to most consumer routers with an Access Point mode.

Goal: living-room Wi-Fi plus a small switch for the TV and consoles, with **OPNsense remaining
the only DHCP, NAT and firewall**.

```text
OPNsense LAN 192.0.2.1
  └─ Core switch 192.0.2.2
       └─ living-room drop
            └─ Wi-Fi router in AP mode 192.0.2.3
                 ├─ 2.5G LAN → TV or console with 2.5G
                 ├─ 1G LAN   → consoles, streaming box, soundbar
                 └─ Wi-Fi    → phones, laptops, handhelds
```

## 1. Initial access

1. Power it on, connect a laptop to a LAN port (not WAN yet).
2. Browse to the vendor setup address (for TP-Link `http://tplinkwifi.net`) and set the admin password.

## 2. Switch to Access Point mode

TP-Link: **Advanced > System > Operation Mode > Access Point Mode > Save**. It reboots.

Then:

- Run Quick Setup for SSIDs and passwords.
- Set a static management address (disable "Smart IP"):
  IP `192.0.2.3`, mask `255.255.255.0`, gateway and DNS `192.0.2.1`.
- Confirm DHCP is off on the AP.
- Reserve `192.0.2.3` on OPNsense (Services > DHCPv4 > LAN) as well, plus reservations for TVs
  and consoles if you want stable addresses.

## 3. Cabling

Core switch port → the AP's **WAN** port. In AP mode every port, including WAN, is a switch
port; using WAN as the uplink keeps the 2.5G LAN port free for a device.

Use Cat6 and a 2.5G port on the core. Internet speed is still capped by the firewall-to-core link.

## 4. Wi-Fi settings

| Band | SSID | Notes |
|---|---|---|
| 5 GHz | `<SSID>` | Primary. 80 MHz default; 160 MHz only if neighbours are quiet |
| 2.4 GHz | `<SSID>` (same or separate) | Legacy devices. 20 MHz when crowded. Channel 1 or 11 (see guide 05 for Zigbee) |
| Security | WPA3-Personal, or WPA2/WPA3 mixed | Mixed only if a TV or console is picky |
| Guest | Skip | Not isolated from the LAN in AP mode. Do guest on OPNsense instead |

Leave EasyMesh off unless you add more APs from the same vendor. Wire the TV and consoles.

Consumer APs generally cannot map SSIDs to VLANs properly. Keep this AP on TRUSTED and give IoT
its own AP or VLAN-capable radio when you split segments.

## 5. What moves to OPNsense

DHCP, NAT / port forwarding / UPnP / DMZ, QoS and parental controls, VPN server, isolated guest
network. USB sharing and Wi-Fi schedules still work on the AP.

## 6. Verify

From a client on the AP (Wi-Fi or wired):

- Address is 192.0.2.x with gateway 192.0.2.1 (lease from OPNsense, not the AP).
- `ping 192.0.2.1` and `ping 192.0.2.2` succeed.
- `http://192.0.2.3` opens the AP UI.

If a device has no internet, it is almost always: still in Router mode, DHCP left on, or the
uplink was connected while it was still doing NAT.

## Don't

- Plug the AP's WAN into the core while it is still in Router mode (double NAT; consoles hate it).
- Leave the stock `192.168.0.1` LAN address configured after the change.
- Run two DHCP servers on one segment.
