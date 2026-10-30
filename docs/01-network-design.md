# 01 - Network design

## Topology

```text
ISP modem/ONT (bridge mode)
  └─ OPNsense WAN (2.5G)
       └─ OPNsense LAN 192.0.2.1  -- trunk: VLAN 10/20/50/99
            └─ MikroTik RB5009 core 192.0.2.2  (bridge, VLAN filtering)
                 ├─ living-room drop  → Wi-Fi router in AP mode 192.0.2.3 (AP + 4-port switch)
                 ├─ office drop (trunk) → Cisco Catalyst 2960-CG 192.0.2.4 (access switch)
                 │     ├─ Gi0/2-7  TRUSTED desks
                 │     └─ Gi0/8    LAB VM host (VLAN 50)
                 └─ PoE port      → Zigbee coordinator (IoT VLAN)
```

Roles are kept separate on purpose:

- **OPNsense** is the only router, DHCP server, NAT and firewall. Every inter-VLAN flow passes
  through its rules.
- **RB5009** is a switch. No DHCP, no NAT. It tags VLANs toward OPNsense and the access switch.
- **Access switch and Wi-Fi AP** are layer 2 only, with a single management address each.

## VLANs

| VLAN | Purpose | Notes |
|---|---|---|
| 10 TRUSTED | Laptops, desktops, phones, TV/consoles that cast | Default for anything you log into |
| 20 IOT | Plugs, sensors, bulbs, Zigbee coordinator | 2.4 GHz SSID only, client isolation |
| 50 LAB | Attack/defense lab, malware, throwaway VMs | No route to anything else |
| 99 MGMT | Switch/AP management, IP-KVMs, UPS/PDU | Reachable from an admin workstation only |
| 999 NATIVE-UNUSED | Native VLAN on trunks | No addresses; untagged frames die here |

## Inter-VLAN policy (OPNsense)

| From → To | TRUSTED | IOT | LAB | MGMT | Internet |
|---|---|---|---|---|---|
| TRUSTED | allow | allow (HA, casting) | deny | admin host only | allow |
| IOT | deny (established only) | isolate | deny | deny | allow (or per-device) |
| LAB | deny | deny | allow | deny | optional, logged |
| MGMT | deny | deny | deny | allow | updates only |

Practical rules:

- Block IoT from **initiating** sessions to TRUSTED. Allow Home Assistant (on TRUSTED or its own
  host) to reach IoT.
- Enable the mDNS repeater between TRUSTED and IoT only if casting/discovery needs it.
- LAB never bridges to the home LAN. Monitor it with agents or one-way log shipping only.
- DNS: point every VLAN at OPNsense (Unbound). Block outbound 53/853 from IoT to force it.

## Build order

1. OPNsense WAN/LAN up, single flat LAN, DHCP working ([guide 02](02-opnsense-bringup-troubleshooting.md)).
2. RB5009 as a plain switch, management address on the LAN.
3. Wi-Fi router into AP mode ([guide 04](04-wifi-router-ap-mode.md)).
4. Create VLAN interfaces on OPNsense, then tag them on the RB5009 **before** turning on bridge
   VLAN filtering.
5. Access switch trunk ([guide 03](03-cisco-2960-mikrotik-trunk.md)).
6. IoT segment and Home Assistant ([guide 05](05-home-assistant-iot.md)).
7. Monitoring ([guide 06](06-home-siem-soc.md)).

Change one thing at a time and verify before moving on. Most outages in this build come from
turning on VLAN filtering before the uplink to the firewall is tagged.
