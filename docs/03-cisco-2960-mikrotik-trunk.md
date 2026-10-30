# 03 - Cisco Catalyst 2960-CG access switch on a MikroTik trunk

Target: a WS-C2960CG-8TC-L (8x 1G copper Gi0/1-8, 2x dual-purpose uplinks Gi0/9-10, no PoE)
in the office, uplinked on Gi0/1 to a MikroTik RB5009 core.

MikroTik does not speak DTP and the 2960 ships in `dynamic auto` on VLAN 1, so the trunk will
not form by itself. Hard-set it on both ends.

## 1. Console

Rollover or USB-serial cable into the RJ-45 console port (not Gi0/1). 9600 8N1:

```bash
screen /dev/ttyUSB0 9600
```

If it asks for a password you do not have, hold MODE while powering on, then:

```text
flash_init
del flash:config.text
del flash:vlan.dat
boot
```

Answer **no** to the initial configuration dialog.

## 2. Switch configuration

Management address on TRUSTED is 192.0.2.4. Trunk native VLAN is 999 (unused) so stray untagged
frames are dropped instead of landing on the LAN.

```text
enable
configure terminal
hostname <HOSTNAME>
no ip domain-lookup
ip domain-name lab.home.arpa
crypto key generate rsa modulus 2048
ip ssh version 2
username admin privilege 15 secret <CHANGE_ME>
enable secret <CHANGE_ME>
service password-encryption
line vty 0 4
 transport input ssh
 login local
 exit
vtp mode transparent
spanning-tree mode rapid-pvst
spanning-tree vlan 1-4094 priority 61440

vlan 10
 name TRUSTED
vlan 20
 name IOT
vlan 50
 name LAB
vlan 99
 name MGMT
vlan 999
 name NATIVE-UNUSED
exit

interface Vlan10
 ip address 192.0.2.4 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.0.2.1
no ip http server
no ip http secure-server

interface GigabitEthernet0/1
 description UPLINK-CORE
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,50,99
 spanning-tree portfast trunk
 no shutdown
 exit

interface range GigabitEthernet0/2 - 8
 description TRUSTED-ACCESS
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
 exit

end
write memory
```

Notes:

- `portfast trunk` on Gi0/1 is safe only because it is a single switch-to-switch uplink. Never
  plug Gi0/1 into another port on the same switch.
- BPDU Guard err-disables a desk port if something downstream sends BPDUs. That is intended.
- The priority 61440 keeps this switch from ever becoming root; the core should be root.
- Once MGMT (VLAN 99) is routed on OPNsense, move the SVI from `Vlan10` to `Vlan99`.

Move a port to the lab later:

```text
interface GigabitEthernet0/8
 description LAB-VM-HOST
 switchport access vlan 50
```

Leave VLAN 20 unassigned until the IoT VLAN exists on the firewall.

## 3. MikroTik side

On the RB5009 port feeding the 2960 (`etherX`), tag 10/20/50/99 and accept tagged frames only:

```text
/interface bridge port
set [find interface=etherX] pvid=999 frame-types=admit-only-vlan-tagged

/interface bridge vlan
add bridge=bridge vlan-ids=10 tagged=bridge,etherX
add bridge=bridge vlan-ids=20 tagged=bridge,etherX
add bridge=bridge vlan-ids=50 tagged=bridge,etherX
add bridge=bridge vlan-ids=99 tagged=bridge,etherX
```

Do **not** enable `vlan-filtering=yes` on the bridge until VLAN 10 is also tagged toward
OPNsense, or the whole LAN drops. Use Safe Mode (Ctrl+X in the terminal) while making the change.

### Interim: core is still a flat untagged LAN

Make the uplink a plain access port so the room works today:

```text
interface GigabitEthernet0/1
 description UPLINK-CORE-UNTAGGED
 switchport mode access
 switchport access vlan 10
 no shutdown
```

## 4. Verify

```text
show vlan brief
show interfaces trunk
show interface status
show ip interface brief
show spanning-tree vlan 10
```

- Gi0/1: trunking, 802.1Q, native 999, allowed 10,20,50,99.
- A PC on Gi0/2 gets a 192.0.2.x lease from OPNsense and reaches 192.0.2.1.
- `ssh admin@192.0.2.4` works.

| Symptom | Cause |
|---|---|
| Gi0/1 `notconnect` | Cable or core port, not IOS |
| Link up, no DHCP on access ports | Native VLAN or bridge VLAN table mismatch on the core |
| Port `err-disabled` | BPDU Guard tripped: find the downstream switch, then `shutdown` / `no shutdown` |
