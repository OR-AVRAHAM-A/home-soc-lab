# DMZ Design (Phase 5)

## Goal

Host an intentionally vulnerable web application (DVWA) that will be the target of later attacks, **without** giving an attacker who compromises it a path into the internal network.

## The gap found during this phase

pfSense rule #1 blocks the DMZ from initiating outbound traffic. But traffic **between VLANs** is routed by R1 and never reaches pfSense:

```
Ubuntu-Web (VLAN 99) -> R1 -> DC01 (VLAN 100)      pfSense never sees this
```

Before the fix, `ping 192.168.100.100` from Ubuntu-Web succeeded (TTL=127). A compromised web server could have reached the Domain Controller directly.

## Fix: extended ACL `DMZ-IN` on R1

Applied **inbound on `e0/1.99`**, so it inspects every packet that enters the router from the DMZ. The source of every packet it sees is therefore a DMZ address.

| Seq | Action | Match                                         | Purpose |
|-----|--------|-----------------------------------------------|---------|
| 10  | permit | tcp DMZ -> 192.168.0.0/16 `established`       | Replies to sessions opened from inside (users browsing DVWA) |
| 20  | permit | icmp DMZ -> 192.168.0.0/16 `echo-reply`       | Ping from inside to the DMZ works for troubleshooting |
| 30  | deny   | ip DMZ -> 192.168.0.0/16 `log`                | DMZ may not start connections to internal networks |
| 40  | deny   | ip DMZ -> 10.10.10.252/30 `log`               | DMZ may not reach the firewall (pfSense GUI on .253 is open to any source via Anti-Lockout) |
| 50  | permit | ip DMZ -> any                                 | Internet-bound traffic continues to pfSense, which blocks and logs it |
| 60  | deny   | ip any any `log`                              | Any other source entering from the DMZ is spoofed |

### Design notes

- **Stateless vs stateful:** Cisco ACLs do not track sessions. `established` matches TCP packets with ACK or RST set, which allows replies but not new connections (pure SYN). It can be bypassed by a crafted packet with ACK set, so it is weaker than pfSense's state table. A production design would use a Zone-Based Firewall on the router, or route the DMZ through the firewall.
- **No "permit ... eq 80" line:** a rule matching source port 80 could be abused with a forged source port (`nmap --source-port 80`). Line 10 already covers the web server's replies.
- **Defense in depth:** the DMZ is blocked twice, by R1 (internal + transit) and by pfSense (internet). Each device logs its own decision.
- **Side effect (intended):** the DMZ cannot reach R1's own addresses (e.g. `192.168.99.254`), since they are inside 192.168.0.0/16. Routing through R1 is not affected.

## Publishing the web server

pfSense **Port Forward** (destination NAT): `WAN address:80 -> 192.168.99.10:80`, with an associated, logged WAN pass rule.

```
Kali 10.20.20.254 -> pfSense WAN 10.20.20.1:80 --(dst NAT)--> 192.168.99.10:80 -> R1 -> Ubuntu-Web -> DVWA
```

- Only the destination is rewritten, so the web server logs the attacker's real IP.
- Only TCP/80 is exposed (verified with nmap: every other port is `filtered`).
- Allowed traffic is logged, not only blocked traffic, so attacks against DVWA are visible later in the SIEM.

## Temporary internet link for installation

The lab has no real internet. To install Docker and DVWA, Ubuntu-Web got a second NIC (`ens4`) connected to EVE-NG's `Management(Cloud0)` network (VMware NAT) for the installation only:

1. Second NIC with DHCP, default route metric 100 (preferred over the lab route, metric 200).
2. Install packages, pull images, `apt upgrade` (the cloud image was from 2022).
3. Remove `ens4` from netplan, delete the cloud network in EVE-NG, reboot.
4. Verify: `ping 8.8.8.8` fails, only one default route via R1.

`last -i` later showed an SSH login from `192.168.240.1` (the host, through VMware NAT) while the link was up. This confirms why the link had to be removed: it was a path into the DMZ that bypassed both R1 and pfSense.

## Known risks

| Risk | Plan |
|------|------|
| Any internal VLAN (including Users) can open SSH to the DMZ. `DMZ-IN` only filters traffic leaving the DMZ. | Hardening phase: outbound ACL on `e0/1.99` allowing SSH only from VLAN 30 (Mgmt). |
| `established` can be bypassed with forged ACK packets. | Accepted for the lab. Documented above. |
| R1 sends ICMP "administratively prohibited" on denied packets, which reveals the ACL to an attacker. | Hardening phase: `no ip unreachables` on `e0/1.99`. |
| Clocks differ between devices (Ubuntu-Web, R1 and pfSense all show different times). | NTP + a single time zone before the SIEM phase. |
| DVWA is deliberately vulnerable and published on the (fake) internet. | That is the point: it is the attack target. It is reachable only from the isolated lab, bound to the DMZ IP, and boxed in by R1 + pfSense. |
| WAN rule allows any source to DVWA. | Expected for a public web server. Rate limiting / a WAF are possible future improvements. |
