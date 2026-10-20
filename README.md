# Home SOC Lab

A home lab that simulates a small enterprise network (`cyber.local`). I use it to practice the full cycle: **attack → logs → detection → investigation → incident report**.

It's built in **EVE-NG** (Cisco IOL router and switch, pfSense) and **VMware Workstation** (Active Directory, a Windows client, Kali Linux, and a Wazuh SIEM).

> Status: 🚧 In progress. Network, firewall and DMZ phases complete. Logging is in place: a Wazuh SIEM collects from all hosts and network devices, with custom detections for the DMZ.

---

## Architecture

![Topology](diagrams/topology-v3.png)

| Segment       | VLAN | Subnet            | Gateway          | Hosts                          |
|---------------|------|-------------------|------------------|--------------------------------|
| Fake Internet | -    | 10.20.20.0/24     | -                | Kali (attacker), pfSense WAN   |
| Transit       | -    | 10.10.10.252/30   | -                | pfSense LAN `.253`, R1 `.254`  |
| Users         | 10   | 192.168.10.0/24   | 192.168.10.254   | WIN10 (DHCP `.100-.200`)       |
| Management    | 30   | 192.168.30.0/24   | 192.168.30.254   | Wazuh `.10`                    |
| DMZ           | 99   | 192.168.99.0/24   | 192.168.99.254   | Ubuntu-Web `.10`               |
| Servers       | 100  | 192.168.100.0/24  | 192.168.100.254  | DC01 `.100` (AD / DNS / DHCP)  |

More detail is in [docs/ip-plan.md](docs/ip-plan.md).

## Stack

| Component        | Technology                                          |
|------------------|-----------------------------------------------------|
| Virtualization   | VMware Workstation Pro + EVE-NG Community 6.2       |
| Routing          | Cisco IOL L3 15.7 (Router-on-a-Stick)               |
| Switching        | Cisco IOL L2 (802.1Q trunk, access ports)           |
| Firewall         | pfSense CE 2.6.0                                    |
| Identity         | Windows Server 2019 (AD DS, DNS, DHCP) `cyber.local`|
| Endpoint         | Windows 10 Enterprise (domain-joined)               |
| Attacker         | Kali Linux                                          |
| Web target       | Ubuntu 22.04 cloud image + Docker + DVWA (DMZ)      |
| SIEM             | Wazuh 4.14.9 all-in-one (Ubuntu 24.04)              |
| Endpoint telemetry | Sysmon (SwiftOnSecurity config) on Windows        |
| Time             | NTP, R1 as lab master, UTC                          |

## Project Phases

| # | Phase                         | Status       |
|---|-------------------------------|--------------|
| 1 | Planning                      | ✅ Done       |
| 2 | Virtualization & Images       | ✅ Done       |
| 3 | Network (VLANs, Routing, DHCP)| ✅ Done       |
| 4 | Firewall (pfSense)            | ✅ Done       |
| 5 | DMZ web server (DVWA) + DMZ isolation | ✅ Done       |
| 6 | Logging (NTP, Wazuh, Sysmon, syslog, custom detections) | ✅ Done |
| 7 | Active Directory hardening    | 🔄 Next       |
| 8 | Attack simulation             | ⏳ Planned    |
| 9 | Detection & Investigation     | ⏳ Planned    |
| 10 | Incident Report              | ⏳ Planned    |

## Network Hardening Applied (so far)

- Native VLAN changed from 1 to an unused VLAN (999) to mitigate VLAN hopping (double tagging)
- DTP disabled on the trunk (`switchport nonegotiate`)
- Only the required VLANs are allowed on the trunk
- PortFast + BPDU Guard on all access ports
- Unused ports shut down and moved to a black-hole VLAN (666)
- VTP transparent mode
- DHCP is relayed via `ip helper-address`. The servers VLAN uses static IPs only.

## Firewall Policy (pfSense)

- Default deny: the "allow LAN to any" rule is disabled. Each allowed flow has its own rule.
- The DMZ may not initiate any outbound connection (logged).
- Users: web only (80, 443). DC01: DNS only. Management: HTTPS only.
- Hybrid outbound NAT hides all internal VLANs behind the WAN address.
- Details: [configs/pfSense.md](configs/pfSense.md), [docs/firewall-design.md](docs/firewall-design.md), [verification/phase4-firewall.md](verification/phase4-firewall.md)

## DMZ (Phase 5)

- Ubuntu-Web (`192.168.99.10`) runs DVWA in Docker, bound to the DMZ interface only.
- pfSense port forward publishes only TCP/80 (`WAN -> 192.168.99.10`), with logging on allowed traffic.
- R1 ACL `DMZ-IN` blocks the DMZ from starting connections to internal networks and to the firewall. pfSense blocks it from the internet. Both log.
- Details: [configs/ubuntu-web.md](configs/ubuntu-web.md), [docs/dmz-design.md](docs/dmz-design.md), [verification/phase5-dmz.md](verification/phase5-dmz.md)

## Logging and Detection (Phase 6)

- **One time source:** R1 is the NTP master. Every device syncs to its own gateway, and the infrastructure runs in UTC. One event now has one timestamp on every device.
- **Collection:** Wazuh agents on DC01, WIN10 and Ubuntu-Web (configured centrally per group). Syslog from R1 and pfSense. Sysmon on Windows.
- **DVWA web logs:** Apache logs were symlinks to the container's stdout. A bind mount turned them into files the agent reads. Built-in web rules now detect attacks (e.g. rule 31164, SQL injection attempt).
- **Custom detections:** the built-in rule for R1 ACL denies is level 0 (no alert). Rules **100100** (DMZ host blocked) and **100101** (5+ attempts in 60 s, MITRE T1046) close that gap. A custom decoder adds ICMP denies, so ping sweeps are correlated too.
- **Sensor limits found by testing:** R1 reports repeated and fast flows up to 5 minutes late. Documented, with triage rules for timelines.
- **Incident during the build:** a decoder change stopped the manager for 3 h 21 min without anyone noticing. Root cause, fix and a safe change procedure are documented.
- Details: [docs/logging-design.md](docs/logging-design.md), [configs/wazuh.md](configs/wazuh.md), [verification/phase6-logging.md](verification/phase6-logging.md), [docs/troubleshooting-log.md](docs/troubleshooting-log.md)

## Known Risks

- Kali has a second adapter (VMware NAT) that bypasses pfSense. It is kept **disconnected** and enabled only for updates.
- Any internal VLAN can open SSH to the DMZ and to Wazuh (planned: allow only from VLAN 30). SSH uses passwords (planned: keys).
- pfSense is managed from the Users VLAN (planned: admin GUI reachable only from VLAN 30).
- pfSense CE 2.6.0 (2022) is outdated; newer CE releases exist. Acceptable in an isolated lab, but it would be patched in production.
- Wazuh has no internet, so its vulnerability feed is frozen. The server has 6 GB RAM, below the recommended size for production.
- The Cisco decoder file is a modified copy of the default and does not receive Wazuh updates. It must be compared after every upgrade.
- R1 ACL logging is a weak scan sensor (delayed reporting). A dedicated IDS is a future improvement.
- No alert yet when a log source or the manager itself goes silent.

## Repository Structure

```
home-soc-lab/
├── README.md
├── configs/          # Device configurations (R1, pfSense, Ubuntu-Web, Wazuh)
├── diagrams/         # Topology diagrams
├── docs/             # Design decisions, troubleshooting log
├── screenshots/      # GUI evidence per phase (no credentials)
└── verification/     # Test outputs proving each phase works
```

## Disclaimer

Everything here runs in an isolated lab that I own. No real systems are targeted. Cisco and Microsoft images are **not** included in this repository.
