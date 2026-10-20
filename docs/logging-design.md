# Logging Design (Phase 6)

## Goal

Every device that makes a security decision, and every host that could be attacked, sends its logs to one place (Wazuh), with **one time source**, so events from different sources can be put on a single timeline.

## Why Logging came before AD hardening

The original plan put Active Directory hardening first. I moved Logging earlier: with the SIEM in place first, every hardening change (and every mistake) from now on is visible in the logs. Hardening without logging means you cannot prove what changed.

## 1. Time: NTP

Before NTP, the same event showed three different times: `13:18` on Ubuntu-Web, `15:18` on R1 and `18:xx` on pfSense (see [phase5-dmz.md](../verification/phase5-dmz.md)). A timeline built from mixed clocks puts events in the wrong order.

```
R1  (ntp master 3, clock timezone UTC)
 ├── DC01        w32tm manual peer 192.168.100.254 (reliable time source of the domain)
 │    └── WIN10  domain hierarchy (NT5DS), gets time from DC01
 ├── pfSense     NTP server 10.10.10.254, Etc/UTC
 ├── Ubuntu-Web  timesyncd drop-in -> 192.168.99.254, UTC
 └── WAZUH-01    timesyncd drop-in -> 192.168.30.254, UTC
```

| Decision | Reason |
|----------|--------|
| R1 is the master | The lab is isolated, there is no internet NTP. R1 is reachable by every VLAN (it is every gateway) |
| Each host uses **its own gateway** address | No extra firewall/ACL holes. The DMZ only needs `udp/123` to `192.168.99.254` |
| DC01 syncs from R1, clients from DC01 | This is how AD works: the PDC emulator is the domain's time source, and Kerberos fails if clocks differ by more than 5 minutes |
| Infrastructure in UTC, Windows displays Israel time | The **clock** is synced everywhere. The **time zone** is only how it is displayed. Wazuh stores timestamps in UTC, and the dashboard shows them in the browser's zone |

Note: after `ntp master`, R1 shows `unsynchronized` (state `FREQ`) for roughly 15 minutes before it serves time.

## 2. Collection

```
                          ┌──────────────────────────────┐
DC01   ─ agent TCP/1514 ─▶│                              │
WIN10  ─ agent TCP/1514 ─▶│  WAZUH-01  192.168.30.10     │──▶ indexer ──▶ dashboard
Ubuntu-Web ─ TCP/1514 ───▶│  manager: decode → rules →   │
R1     ─ syslog UDP/514 ─▶│  alerts                      │
pfSense─ syslog UDP/514 ─▶│                              │
                          └──────────────────────────────┘
```

| Source | Method | Why this method |
|--------|--------|-----------------|
| Windows (DC01, WIN10) | Wazuh agent + Sysmon | Event logs are local to the host. Sysmon adds process creation, network connections and DNS queries, which Windows does not log by default |
| Ubuntu-Web | Wazuh agent | auth/syslog + DVWA web logs |
| R1, pfSense | Syslog | Network devices cannot run an agent |

### Sysmon

- Config: SwiftOnSecurity `sysmonconfig-export.xml` (a well-known community baseline that filters noise).
- Collected via the `windows` group's `agent.conf` (`Microsoft-Windows-Sysmon/Operational`, `eventchannel`), not configured host by host.
- Useful event IDs for later phases:

| Event ID | Meaning | Example use |
|----------|---------|-------------|
| 1  | Process creation (with command line, hashes, parent) | `powershell -enc ...` started by Word |
| 3  | Network connection | A process on WIN10 connecting to an unusual port |
| 11 | File created | Payload dropped in `%TEMP%` |
| 13 | Registry value set | Persistence via Run keys |
| 22 | DNS query | Lookup of a suspicious domain |

## 3. Detection

| Source | Built-in coverage | Gap found | Fix |
|--------|-------------------|-----------|-----|
| R1 ACL denies | Decoded, but rule 4716 = level 0 | No alert for DMZ → internal attempts | Rules 100100 / 100101 |
| R1 ICMP denies | Not decoded (catch-all decoder) | Ping sweeps never correlated | Decoder `cisco-ios-acl-icmp` |
| DVWA | `web-accesslog` + rules 31100+ | Logs were not files | Bind mount + `localfile` |
| pfSense | `pf` decoder, rule 87701 | - | - |

Details and XML: [configs/wazuh.md](../configs/wazuh.md).

## 4. Sensor limits (what the SIEM cannot see)

A detection rule is only as good as the sensor that feeds it.

| Limit | Effect | Status |
|-------|--------|--------|
| R1 logs the **first packet** of each flow immediately, then a **summary every 5 minutes** | Repeated activity on known flows appears up to 5 minutes late, as many lines with the same millisecond | Documented. In an incident timeline, write a time **range** for summary lines |
| A **fast** sweep: some new flows are not logged immediately, only in the 5-minute summary | Correlation (100101) fires late, or not at all if fewer than 5 lines arrive together | Proven by experiment ([phase6-logging.md](../verification/phase6-logging.md#5-sensor-limit-fast-vs-slow-sweep)). Exact IOS mechanism not confirmed |
| Frequency rules count **log lines**, not packets | A line saying `2 packets` counts once | Accepted |
| Syslog is UDP | Messages can be lost silently | Sequence numbers in R1 messages (`30:`, `31:` ...) let you detect gaps |
| An alert is not a compromise | `31164 SQL injection attempt` with HTTP `302` = redirected, never reached the vulnerable page | Triage rule: always check the response code |

Future improvement: a dedicated network sensor (IDS such as Suricata) for scan detection instead of router ACL logs.
