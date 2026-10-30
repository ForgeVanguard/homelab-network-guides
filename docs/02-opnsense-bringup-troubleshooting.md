# 02 - OPNsense bring-up and troubleshooting

Getting the first workstation talking to a new OPNsense LAN through the core switch. The
rule throughout: **prove the laptop's own NIC works before touching the firewall or switch.**

## Expected state

| Device | Address |
|---|---|
| OPNsense LAN | 192.0.2.1/24 |
| Core switch (RB5009) | 192.0.2.2/24 |
| Test laptop (static) | 192.0.2.20/24, gateway and DNS 192.0.2.1 |
| DHCP pool | 192.0.2.100-199 |

## Test ladder

Run in order. Stop at the first failure; everything after it is meaningless until it passes.

```cmd
ping 127.0.0.1          :: TCP/IP stack on the laptop
ping 192.0.2.20         :: the laptop's own address on this NIC
ping -4 192.0.2.2       :: the switch
ping -4 192.0.2.1       :: the firewall
ping 1.1.1.1            :: the internet
```

| Result | Meaning |
|---|---|
| `127.0.0.1` fails | Laptop TCP/IP stack is broken: `netsh int ip reset`, `netsh winsock reset`, reboot |
| Own address returns **General failure** | Windows has not bound IPv4 on that NIC. The network is not in the test yet. |
| `Request timed out` | Packet left the PC, nothing answered (ICMP filtered or wrong host) |
| `Destination host unreachable` | No ARP reply: wrong port, wrong VLAN, or the firewall LAN is not on that cable |

## "Cable disconnected" but the adapter shows an IP and internet works

A static address is displayed even on a dead adapter. Internet is coming from **another**
adapter (Wi-Fi, a dock, phone tethering, a virtual switch).

1. `ipconfig /all` and check **Media State** on the Ethernet adapter. `Media disconnected` = no link.
2. Look for `(Duplicate)` next to the IPv4 address. If so, pick a different static address.
3. Disable Wi-Fi, WWAN, VPN clients and virtualization adapters for the test. Only one adapter
   should hold an address in 192.0.2.0/24.
4. Get a link light: different cable, different switch port (avoid port 1 if it is the uplink),
   built-in jack instead of a dongle.

## Hyper-V took the NIC

Symptoms: a `vEthernet` adapter with `169.254.x.x`, and a different "Ethernet N" adapter holding
your LAN address while the physical port reports unplugged. An **External** Hyper-V switch is
bound to the physical NIC.

Quick fix, keep Hyper-V:

1. `ncpa.cpl` and disable the `169.254` vEthernet adapter (or Device Manager > Network adapters >
   Disable device).
2. Hyper-V Manager > Virtual Switch Manager: no switch should be External on this NIC; set it
   to Internal or remove it.
3. Put the static address on the adapter that changes to "unplugged" when you pull the cable.

Turn Hyper-V off completely (elevated prompt):

```cmd
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V-All
bcdedit /set hypervisorlaunchtype off
:: reboot, then confirm
bcdedit /enum | findstr hypervisorlaunchtype
systeminfo
```

Also clear Virtual Machine Platform, Windows Hypervisor Platform and Windows Sandbox in
*Turn Windows features on or off*, and turn off Memory integrity if it keeps the hypervisor
loaded. WSL2 stops working until these are re-enabled:

```cmd
bcdedit /set hypervisorlaunchtype auto
dism.exe /Online /Enable-Feature:Microsoft-Hyper-V-All /All
```

## USB-to-RJ45 dongles

General failure on your own address with a USB dongle is almost always the dongle or its
binding, not the firewall.

- Plug directly into the laptop (not a hub or monitor dock), USB 3.x data port.
- Device Manager > the USB Ethernet adapter > Power Management: uncheck *Allow the computer to
  turn off this device*. Do the same on each USB Root Hub.
- Uninstall the device (delete driver if offered), unplug, replug.
- Confirm no Hyper-V External switch is bound to it.
- Common chipsets: Realtek RTL8153, ASIX AX88179. Use the vendor driver if the inbox one misbehaves.

## DHCP works but ping does not

A DHCP lease proves the path laptop → switch → OPNsense works. ICMP can still be filtered.

```cmd
ipconfig
arp -a
tracert -d 192.0.2.1
```

| Works | Broken | Action |
|---|---|---|
| DHCP lease + `https://192.0.2.1` loads | ping only | Fine. Fix ICMP rules later |
| Switch reachable by MAC (WinBox) only | `.2` and `.1` web | Add 192.0.2.2/24 on the bridge; keep the default address until verified |
| Lease received but gateway is not `.1` | Firewall web UI | A second DHCP server is running. Disable DHCP on the switch/AP |

Do not factory-reset either box to fix a laptop problem.
