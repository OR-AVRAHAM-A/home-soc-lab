# Wazuh Configuration (Phase 6)

| Item        | Value                                                        |
|-------------|--------------------------------------------------------------|
| Hostname    | `wazuh-01`                                                   |
| Platform    | VMware VM, Ubuntu 24.04 LTS (26.04 is not supported by Wazuh 4.14) |
| Resources   | 4 vCPU, 6 GB RAM, 60 GB disk                                 |
| Network     | `ens34` = 192.168.30.10/24, VLAN 30 (Management)             |
| Wazuh       | 4.14.9, all-in-one (manager + indexer + dashboard)           |
| Dashboard   | `https://192.168.30.10`                                      |
| Time        | UTC, synced from R1 (`192.168.30.254`)                       |

## Network: `/etc/netplan/*.yaml`

```yaml
network:
  version: 2
  ethernets:
    ens34:
      dhcp4: false
      addresses: [192.168.30.10/24]
      routes:
        - to: default
          via: 192.168.30.254
        - to: 192.168.0.0/16
          via: 192.168.30.254
```

- The VM had a second NIC (VMware NAT, `ens33`) for the installation only. It is now **disconnected**.
- While it was connected, the default route went out through NAT and the server could not reach DC01. The specific `192.168.0.0/16` route keeps all lab traffic on R1 no matter which default route wins (most specific route first).

## Installation notes

- Installed with the official all-in-one assisted installation, while the temporary NAT link was up.
- After the installation, the Wazuh apt repository was **commented out**. A routine `apt upgrade` must not upgrade the manager by accident: manager, indexer, dashboard and agents must stay on matching versions.
- `~/agent-kit` holds the agent packages (`.deb`, `.msi`), Sysmon and its config, plus `SHA256SUMS`. Files are verified before they are copied to the endpoints (SHA256 on Linux, SHA256 + Authenticode signature on Windows).
- The dashboard "Deploy new agent" page generates a `wget` + `dpkg` command. It must **not** be run on the manager itself. The manager already has a local agent (ID 000).

## Log sources

| ID  | Source      | Group      | Transport          | What is collected |
|-----|-------------|------------|--------------------|-------------------|
| 000 | wazuh-01    | -          | local              | Manager's own logs + **syslog received from R1 and pfSense** |
| 001 | DC01        | `windows`  | agent, TCP 1514    | Windows Security/System/Application + Sysmon |
| 002 | WIN10       | `windows`  | agent, TCP 1514    | Windows Security/System/Application + Sysmon |
| 003 | ubuntu-web  | `linux-dmz`| agent, TCP 1514    | auth/syslog + DVWA Apache access/error logs |
| -   | R1          | -          | syslog, UDP 514    | ACL denies (`%SEC-6-IPACCESSLOG*`), config changes, interface events |
| -   | pfSense     | -          | syslog, UDP 514    | Firewall (filterlog) + system logs |

Syslog events are decoded by the manager, so they appear under **agent 000 (wazuh-01)** with `location` = the sender's IP. A dashboard view pinned to another agent hides them.

## `ossec.conf`: syslog listener

A second `<remote>` block, next to the default agent listener:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.10.253</allowed-ips>     <!-- pfSense (LAN interface) -->
  <allowed-ips>192.168.30.254</allowed-ips>   <!-- R1 (logging source-interface e0/1.30) -->
  <local_ip>192.168.30.10</local_ip>
</remote>
```

- Only the two network devices may send syslog. Anything else is dropped.
- `logall` is `no`. It was enabled temporarily (with `archives.log`) to confirm that raw events arrived before any rule matched, then turned off.

## Centralized agent configuration (groups)

Endpoint settings live on the manager, in `/var/ossec/etc/shared/<group>/agent.conf`. A new agent placed in a group receives the configuration automatically. Validate with `/var/ossec/bin/verify-agent-conf` after every change.

### `shared/windows/agent.conf`

```xml
<agent_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```

### `shared/linux-dmz/agent.conf`

```xml
<agent_config>
  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/dvwa/access.log</location>
  </localfile>
  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/dvwa/error.log</location>
  </localfile>
</agent_config>
```

See [ubuntu-web.md](ubuntu-web.md) for how the DVWA container writes these files.

## Custom rules: `/var/ossec/etc/rules/local_rules.xml`

**Gap:** Wazuh decodes R1's ACL log lines, but the built-in rule 4716 (Cisco IOS informational) has **level 0**: no alert. A compromised DMZ server probing the Domain Controller would have been invisible.

```xml
<group name="cisco_ios,lab_custom,">

  <rule id="100100" level="8">
    <if_sid>4716</if_sid>
    <match>list DMZ-IN denied</match>
    <description>R1 ACL DMZ-IN: DMZ host $(srcip) -> $(dstip) ($(protocol)) blocked</description>
    <group>firewall_block,dmz_violation,</group>
  </rule>

  <rule id="100101" level="10" frequency="5" timeframe="60">
    <if_matched_sid>100100</if_matched_sid>
    <same_source_ip />
    <description>R1 ACL DMZ-IN: 5+ blocked attempts from DMZ host $(srcip) in 60s - possible scan or lateral movement</description>
    <mitre>
      <id>T1046</id>
    </mitre>
    <group>firewall_block,dmz_violation,</group>
  </rule>

</group>

<!-- Phase 7: AD domain policy change (4739) matches only the level-0 generic
     Windows rule 60103, so it never alerts. Elevate it. -->
<group name="windows,lab_custom,">

  <rule id="100200" level="8">
    <if_sid>60103</if_sid>
    <field name="win.system.eventID">^4739$</field>
    <description>AD: Domain policy changed on $(win.system.computer) by $(win.eventdata.subjectUserName) - review password/lockout/Kerberos settings</description>
    <mitre>
      <id>T1484.001</id>
    </mitre>
    <group>policy_changed,ad_hardening,</group>
  </rule>

</group>
```

| Rule   | Level | Meaning |
|--------|-------|---------|
| 100100 | 8     | One blocked connection attempt from the DMZ to an internal network |
| 100101 | 10    | Correlation: 5+ blocked attempts from the same DMZ host within 60 seconds. When it fires, it replaces the 100100 alert for that event |
| 100200 | 8     | AD domain policy changed (event 4739): password/lockout/Kerberos settings modified (T1484.001) |

Custom rule IDs use the 100000-120000 range reserved for local rules.

## Custom decoder: ICMP ACL logs

**Gap 2:** TCP/UDP denies (`%SEC-6-IPACCESSLOGP`) are parsed by the built-in child decoder `cisco-ios-acl`. ICMP denies (`%SEC-6-IPACCESSLOGDP`) are not, so `srcip` stayed empty. Rule 100101 uses `<same_source_ip />`, so a **ping sweep** from the DMZ could never be correlated.

**Why `local_decoder.xml` did not work:** the built-in file ends with a catch-all child decoder, `cisco-ios-default`, which has no `prematch` and matches every Cisco line. Wazuh uses the first child decoder that matches, and the default ruleset is loaded before `etc/decoders`. A local decoder is never reached.

**Fix (official method for changing a default decoder):**

1. Copy `ruleset/decoders/0065-cisco-ios_decoders.xml` to `etc/decoders/` (owner `wazuh:wazuh`, mode `660`).
2. In the copy, add the new child decoder **before** `cisco-ios-default`.
3. Exclude the original in `ossec.conf`.
4. `wazuh-analysisd -t`, then `wazuh-logtest`, then restart.

```xml
<!-- LAB: ICMP ACL logs (IPACCESSLOGDP) are not parsed by default
  - %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.100 (8/0), 1 packet
  -->
<decoder name="cisco-ios-acl-icmp">
  <parent>cisco-ios</parent>
  <prematch>%SEC-6-IPACCESSLOGDP: list \S+ \S+ icmp </prematch>
  <regex>%(\w+)-(\d)-(\w+): list (\S+) (\w+) (\w+) (\d+.\d+.\d+.\d+) -> (\d+.\d+.\d+.\d+) \((\d+)/(\d+)\)</regex>
  <order>cisco.facility, cisco.severity, cisco.mnemonic, acl, action, protocol, srcip, dstip, icmp_type, icmp_code</order>
</decoder>
```

```xml
<ruleset>
  ...
  <!-- User-defined ruleset -->
  <decoder_dir>etc/decoders</decoder_dir>
  <decoder_exclude>ruleset/decoders/0065-cisco-ios_decoders.xml</decoder_exclude>
  <rule_dir>etc/rules</rule_dir>
</ruleset>
```

- The ICMP decoder also extracts `cisco.facility/severity/mnemonic`, because for these lines it replaces `cisco-ios-default`.
- `icmp_type` 8 = echo request (someone is looking for hosts), 0 = echo reply.
- **Trade-off:** the copied file no longer receives Wazuh updates. After every upgrade, compare it with the new original (`diff`) and merge changes.

## Safe change procedure (learned the hard way)

```
1. Back up the file you change
2. Make the change (for a copied default file: add the exclude FIRST, then copy)
3. sudo /var/ossec/bin/wazuh-analysisd -t        # config test, no output = OK
4. sudo /var/ossec/bin/wazuh-logtest             # test with a real log line
5. sudo systemctl restart wazuh-manager
6. sudo /var/ossec/bin/wazuh-control status      # analysisd + remoted running?
7. sudo tail -n 30 /var/ossec/logs/ossec.log     # no ERROR/CRITICAL
8. Rollback plan ready (what to remove to get back to the last working state)
```

See the incident in [troubleshooting-log.md](../docs/troubleshooting-log.md#14-the-siem-was-down-for-3-hours-21-minutes).
