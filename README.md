# homelab-network-guides

Field notes for building a segmented home lab network: an OPNsense firewall, a MikroTik core
switch, a Cisco access switch, a consumer Wi-Fi router in access-point mode, a Home Assistant
IoT segment, and a self-hosted monitoring/SIEM stack.

Each guide is written as a runbook: what to configure, in what order, how to verify it, and
what usually breaks.

| # | Guide | Covers |
|---|---|---|
| 01 | [Network design](docs/01-network-design.md) | Topology, VLANs, addressing, firewall policy between segments |
| 02 | [OPNsense bring-up and troubleshooting](docs/02-opnsense-bringup-troubleshooting.md) | First connection, "General failure", USB NICs, Hyper-V vSwitch conflicts |
| 03 | [Cisco 2960 + MikroTik trunk](docs/03-cisco-2960-mikrotik-trunk.md) | Console recovery, full IOS config, RouterOS bridge VLAN filtering, verification |
| 04 | [Consumer Wi-Fi router as AP + switch](docs/04-wifi-router-ap-mode.md) | Access Point mode behind OPNsense, Wi-Fi settings, single-DHCP rule |
| 05 | [Home Assistant IoT segment](docs/05-home-assistant-iot.md) | IoT VLAN, Zigbee channel planning, coordinator placement, renter-safe device plan |
| 06 | [Home SIEM / SOC / NMS](docs/06-home-siem-soc.md) | Wazuh + Zabbix + Grafana + ntfy, dual-path alerting, phased rollout |
| 07 | [Small rack build](docs/07-small-rack-build.md) | Low-power rack layout, out-of-band KVM, UPS/NUT, concurrency sizing |

## Address plan used in the examples

All addresses use the RFC 5737 documentation ranges. Substitute your own private ranges.

| VLAN | Name | Example subnet | Gateway |
|---|---|---|---|
| 10 | TRUSTED | 192.0.2.0/24 | 192.0.2.1 (OPNsense) |
| 20 | IOT | 203.0.113.0/24 | 203.0.113.1 |
| 50 | LAB (isolated) | 198.51.100.0/24 | 198.51.100.1 |
| 99 | MGMT | `<MGMT_SUBNET>` | OPNsense |
| 999 | NATIVE-UNUSED | none (blackhole native VLAN on trunks) | - |

Fixed infrastructure addresses on TRUSTED: `.1` firewall, `.2` core switch, `.3` Wi-Fi AP,
`.4` access switch. Device names are shown as `<HOSTNAME>`.

## License

MIT - Steven Drew
