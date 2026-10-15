# Firewall Design Decisions (Phase 4)

## Goals

1. **Default deny:** only traffic that is explicitly needed is allowed out.
2. **Segmentation:** the DMZ is treated as untrusted and may not initiate connections.
3. **Visibility:** blocked traffic is logged, so it can later be forwarded to the SIEM (Wazuh).

## Traffic Flow

```
WIN10 (192.168.10.100)
   │  VLAN 10
   ▼
SW1 ──trunk──► R1 (Router-on-a-Stick)
                 │  10.10.10.252/30
                 ▼
              pfSense LAN (.253)  ── rules + state table
              pfSense WAN (10.20.20.1)  ── outbound NAT
                 │
                 ▼
              Kali (10.20.20.254)  "Internet"
```

## Decisions

| Decision | Reason |
|----------|--------|
| *Block private networks* **off** on WAN | The fake internet uses 10.20.20.0/24, which is RFC1918. With the option on, all WAN traffic was dropped (tracker 12001). |
| No gateway on the LAN interface | A gateway on an interface makes pfSense treat it as upstream (reply-to, NAT, and so on). R1 is internal. |
| Static route `192.168.0.0/16` → R1 | pfSense only knows the /30 transit network. Without the route, replies to the VLANs would be sent nowhere. |
| Default gateway IPv4 = `None` | R1's default route points to pfSense. If pfSense's default pointed to R1, unknown traffic would loop between them. |
| Hybrid outbound NAT | Automatic NAT is generated only for interfaces with a gateway, and WAN has none. Hybrid keeps the automatic rules and adds one manual mapping. |
| Default "allow LAN to any" **disabled** | Least privilege. Every allowed flow has its own rule and description. |
| DMZ block is rule #1 | pfSense evaluates rules top-down and the first match wins. If the block were lower, the ICMP rule (`INTERNAL_NETS`) would match DMZ traffic first. |
| `WEB_PORTS` = `80, 443` (not `80-443`) | A range would open 364 ports, including 135 (RPC), 139 (NetBIOS) and 389 (LDAP). The alias lists exact ports only. |
| Aliases instead of raw IPs | Rules are easier to read and audit, and a change in one place updates every rule. |
| Logging on the DMZ block rule | A DMZ host trying to reach out is a strong indicator of compromise (C2 beaconing, MITRE T1071). |

## Known Risks

| Risk | Mitigation |
|------|------------|
| Kali has a second adapter (VMware NAT, `eth1`) that reaches the real internet and the EVE-NG management IP, bypassing pfSense. | `eth1` is **disconnected** in VMware (*Connected* and *Connect at power on* both unchecked). It is enabled only for `apt` updates and disconnected right after. |
| The ICMP rule allows all internal networks to ping out. | Lab-only rule for troubleshooting. To be removed or limited in the hardening phase. |
| Log timestamps differ between systems (Kali UTC, pfSense local time). | To be fixed with NTP and a single time zone before the Wazuh phase. |
| Firewall logs are stored only locally on pfSense. | To be forwarded with remote syslog to Wazuh (Phase 6). |
