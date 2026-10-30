# 07 - Small rack build

A home rack sized for VMs, NAS, media, VPN, local language models and labs, all running at once, without
data-center power bills. Target: ~100 W idle, ~150-220 W with everything busy, on one 120 V circuit.

## Bill of materials (by role)

| Role | Example | Notes |
|---|---|---|
| Frame | 25U 4-post open frame, adjustable depth, casters | 4-post, not 2-post (UPS weight). Open sides for airflow |
| Firewall | 1U OPNsense box, 2.5G WAN + 2.5G/10G LAN | Dedicated hardware so internet/VPN survive a hypervisor outage |
| Core switch | MikroTik CRS310-8G+2S+IN (8x 2.5G, 2x 10G SFP+) | |
| Hypervisor | Minisforum MS-01, 96 GB RAM, 2x NVMe (boot + VM pool) | Proxmox. Dual 10G SFP+ to the core |
| NAS | Synology DS923+/DS1522+, 4x large HDD | Media, photo originals, Proxmox backup target |
| Lab (optional) | N100 mini PC | Throwaway k8s/malware/firewall-breaking box on the LAB VLAN |
| UPS | 1500 VA line-interactive 2U rack UPS | NUT for coordinated shutdown |
| PDU | Switched PDU | Per-outlet control |
| Out-of-band | 2x JetKVM (firewall, hypervisor), USB-serial cable | On the MGMT VLAN, not the LAN |

## Sizing for "everything on"

| State | Approx. watts |
|---|---|
| Idle, disks spun down | ~100 |
| All VMs up, NAS awake | 140-160 |
| Plus media transcode and a 7-13B local model | 180-220 |
| Plus a ~70 W GPU | 250-280 |
| Used 2U dual-Xeon + 8 HDDs + 300 W GPU | 450-700 (this is when you split rooms) |

The real limits are RAM and disk I/O, not amps:

- 64 GB is tight for Nextcloud, Immich, Jellyfin, *arr, Home Assistant, lab VMs and a local model
  all resident. Use 96 GB.
- Keep VM disks on NVMe and media on HDDs. Concurrent backup + library scan + photo ingest on the
  same spindles will stutter.

### Proxmox guest map

| Guest | RAM |
|---|---|
| Jellyfin (Quick Sync) | 4 GB |
| *arr + download client (behind VPN) | 4 GB |
| Nextcloud | 4-6 GB |
| Immich | 6-8 GB |
| Home Assistant OS | 2-4 GB |
| Ollama | 8-16 GB |
| Lab VMs (2-3) | 8-16 GB |
| Proxmox + ZFS | ~8 GB |

## Elevation (bottom to top)

| U | Item |
|---|---|
| 1-2 | UPS |
| 3 | PDU (or 0U vertical on a rear rail) |
| 4 | Blank / power slack |
| 5 | Patch panel |
| 6 | Cable manager |
| 7 | Firewall |
| 8 | Core switch |
| 9 | Blank |
| 10-11 | NAS on vented shelf |
| 12 | Hypervisor on vented shelf + KVM |
| 13 | Optional lab box |
| 14-24 | Spare / airflow |
| Top rail | Photoelectric smoke + heat detector |

## Power and safety

```text
Dedicated 20 A / 120 V circuit
  → surge protection
  → smoke/heat-triggered latching relay on UPS input (installed by an electrician)
  → rack UPS
  → switched PDU
```

NUT shuts hosts down before the relay drops UPS input; manual reset afterwards. Detectors go on
the top rail, where heat collects.

## Out-of-band access

- IP-KVM on the firewall and the hypervisor (the one you will actually use for boot menus and
  hung kernels). Keep a USB-serial cable for the firewall console.
- Skip 1U LCD console drawers.

## When to split into another room

Only for noise (HDD seek on an open frame next to a bedroom) or a gaming-class GPU. Power and
heat at this scale are not reasons to split.
