# Phase 3: Network Verification

## SW1: trunk

```
SW1#show interfaces trunk
Port        Mode             Encapsulation  Status        Native vlan
Et0/0       on               802.1q         trunking      999

Port        Vlans allowed on trunk
Et0/0       10,30,99-100
```

## SW1: port status

```
SW1#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Et0/0     "CONNECT TO R1"    connected    trunk      a-full   auto RJ45
Et0/1     CONNECT TO WIN 10  connected    10         a-full   auto RJ45
Et0/2     CONNECT TO DC01    connected    100        a-full   auto RJ45
Et0/3     CONNECT TO WAZUH   connected    30         a-full   auto RJ45
Et1/0     CONNECT TO UBUNTU  connected    99         a-full   auto RJ45
Et1/1                        disabled     666          auto   auto RJ45
Et1/2                        disabled     666          auto   auto RJ45
Et1/3                        disabled     666          auto   auto RJ45
```

## R1: interfaces

```
R1#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.10.10.254    YES manual up                    up
Ethernet0/1                unassigned      YES manual up                    up
Ethernet0/1.10             192.168.10.254  YES manual up                    up
Ethernet0/1.30             192.168.30.254  YES manual up                    up
Ethernet0/1.99             192.168.99.254  YES manual up                    up
Ethernet0/1.100            192.168.100.254 YES manual up                    up
Ethernet0/1.999            unassigned      YES unset  up                    up
Ethernet0/2                unassigned      YES unset  administratively down down
Ethernet0/3                unassigned      YES unset  administratively down down
```

## R1: routing table

```
Gateway of last resort is 10.10.10.253 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.10.10.253
C        10.10.10.252/30 is directly connected, Ethernet0/0
C        192.168.10.0/24 is directly connected, Ethernet0/1.10
C        192.168.30.0/24 is directly connected, Ethernet0/1.30
C        192.168.99.0/24 is directly connected, Ethernet0/1.99
C        192.168.100.0/24 is directly connected, Ethernet0/1.100
```

## WIN10 (VLAN 10): DHCP relay + DNS + inter-VLAN routing

```
C:\> ipconfig
   Connection-specific DNS Suffix  . : cyber.local
   IPv4 Address. . . . . . . . . . . : 192.168.10.100
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.10.254

C:\> nslookup dc.cyber.local
Server:  DC.cyber.local
Address:  192.168.100.100
Name:    dc.cyber.local
Address:  192.168.100.100

C:\> ping 192.168.100.100
Reply from 192.168.100.100: bytes=32 time=1ms TTL=127
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

**What this proves**

- The DHCP relay works: the client in VLAN 10 got a lease from DC01 in VLAN 100.
- The scope options work: gateway, DNS server and DNS suffix were all delivered.
- Inter-VLAN routing works: TTL=127 means exactly one router hop (Windows starts at TTL 128).
