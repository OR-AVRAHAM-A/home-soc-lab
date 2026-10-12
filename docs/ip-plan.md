# IP Plan

## Design decisions

- **VLAN ID = third octet.** Example: VLAN 10 → 192.168.10.0/24. The servers VLAN was renumbered from 20 to 100 so it matches the existing DC address (192.168.100.100). This avoids re-addressing a Domain Controller.
- **Gateway = .254** in every VLAN, on R1 subinterfaces (Router-on-a-Stick). All inter-VLAN traffic passes through one point, where ACLs and logging can be enforced.
- **Transit /30** between pfSense and R1. This is a point-to-point link with only 2 hosts.
- **Fake Internet** uses 10.20.20.0/24, chosen so it does not overlap the home LAN or VMware NAT.
- **Static IPs** for servers and infrastructure. **DHCP** only for user endpoints (VLAN 10).

## Addressing

```
Segment           Subnet              Gateway          Device           IP
----------------  ------------------  ---------------  ---------------  ------------------------
Fake Internet     10.20.20.0/24       -                pfSense WAN      10.20.20.1
                                      10.20.20.1       Kali             10.20.20.254
Transit           10.10.10.252/30     -                pfSense LAN      10.10.10.253
                                      -                R1 e0/0          10.10.10.254
VLAN 10 Users     192.168.10.0/24     192.168.10.254   R1 e0/1.10       192.168.10.254
                                                       WIN10            DHCP .100 - .200
VLAN 30 Mgmt      192.168.30.0/24     192.168.30.254   R1 e0/1.30       192.168.30.254
                                                       Wazuh            192.168.30.10  (static)
VLAN 99 DMZ       192.168.99.0/24     192.168.99.254   R1 e0/1.99       192.168.99.254
                                                       Ubuntu-Web       192.168.99.10  (static)
VLAN 100 Servers  192.168.100.0/24    192.168.100.254  R1 e0/1.100      192.168.100.254
                                                       DC01             192.168.100.100 (static)
VLAN 666          black hole (unused ports)
VLAN 999          native VLAN (unused)
```

## DHCP (DC01)

| Scope         | Range                          | 003 Router      | 006 DNS          | 015 Domain   |
|---------------|--------------------------------|-----------------|------------------|--------------|
| VLAN10-USERS  | 192.168.10.100 - 192.168.10.200| 192.168.10.254  | 192.168.100.100  | cyber.local  |

R1 relays DHCP for VLAN 10 with `ip helper-address 192.168.100.100` on `e0/1.10`.

## VMware ↔ EVE-NG mapping

```
VMnet   EVE adapter   EVE Linux   EVE Cloud   VMware VM
------  ------------  ----------  ----------  -----------------
NAT     Adapter 1     eth0        Cloud0      EVE management
VMnet2  Adapter 2     eth1        Cloud1      Kali
VMnet3  Adapter 3     eth2        Cloud2      WIN10
VMnet4  Adapter 4     eth3        Cloud3      DC01
VMnet5  Adapter 5     eth4        Cloud4      Wazuh
```

VMnet2-5 are host-only. The host adapter and VMware DHCP are disabled, so the lab stays isolated.
