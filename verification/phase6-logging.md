# Phase 6: Logging Verification

All timestamps are UTC unless noted. The dashboard shows browser time (Israel, UTC+3).

## 1. All agents connected

```
$ sudo /var/ossec/bin/agent_control -l
   ID: 000, Name: wazuh-01 (server), IP: 127.0.0.1, Active/Local
   ID: 001, Name: DC01, IP: any, Active
   ID: 002, Name: WIN10, IP: any, Active
   ID: 003, Name: ubuntu-web, IP: any, Active
```

## 2. Decoding: `wazuh-logtest`

**ICMP deny (custom decoder `cisco-ios-acl-icmp`)**

```
Oct 20 10:00:00.000 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.100 (8/0), 1 packet

**Phase 2: Completed decoding.
        name: 'cisco-ios'
        parent: 'cisco-ios'
        acl: 'DMZ-IN'
        action: 'denied'
        cisco.facility: 'SEC'
        cisco.mnemonic: 'IPACCESSLOGDP'
        cisco.severity: '6'
        dstip: '192.168.100.100'
        icmp_code: '0'
        icmp_type: '8'
        protocol: 'icmp'
        srcip: '192.168.99.10'

**Phase 3: Completed filtering (rules).
        id: '100100'
        level: '8'
        description: 'R1 ACL DMZ-IN: DMZ host 192.168.99.10 -> 192.168.100.100 (icmp) blocked'
        groups: '['cisco_ios', 'lab_custom', 'firewall_block', 'dmz_violation']'
**Alert to be generated.
```

Before the decoder, the same line produced `'R1 ACL DMZ-IN: DMZ host  tried to reach : - blocked'` (empty fields).

**TCP deny (built-in `cisco-ios-acl`, not broken by the change)**

```
Oct 20 10:00:00.000 UTC: %SEC-6-IPACCESSLOGP: list DMZ-IN denied tcp 192.168.99.10(33370) -> 192.168.100.100(53), 1 packet

**Phase 2:  action: 'denied'  dstip: '192.168.100.100'  dstport: '53'  protocol: 'tcp'  srcip: '192.168.99.10'  srcport: '33370'
**Phase 3:  id: '100100'  level: '8'
```

**Other Cisco messages (catch-all `cisco-ios-default` still works)**

```
Oct 20 10:00:00.000 UTC: %SYS-5-CONFIG_I: Configured from console by console

**Phase 3:  id: '4721'  level: '3'  description: 'Cisco IOS: Router configuration changed'
```

## 3. End to end: ping sweep from the DMZ

**Ubuntu-Web** (new destinations, 2 seconds apart):

```bash
for ip in 192.168.100.50 192.168.100.51 192.168.10.50 192.168.10.51 192.168.30.50; do ping -c1 -W1 $ip; sleep 2; done
```

**R1**

```
Oct 20 11:44:54.300 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.50 (8/0), 1 packet
Oct 20 11:44:56.309 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.51 (8/0), 1 packet
Oct 20 11:44:58.322 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.10.50 (8/0), 1 packet
Oct 20 11:45:00.331 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.10.51 (8/0), 1 packet
Oct 20 11:45:02.341 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.30.50 (8/0), 1 packet
```

**Wazuh**

```
"timestamp":"2026-10-20T11:44:55.331+0000"  "id":"100100"  "dstip":"192.168.100.50"
"timestamp":"2026-10-20T11:44:57.376+0000"  "id":"100100"  "dstip":"192.168.100.51"
"timestamp":"2026-10-20T11:44:59.380+0000"  "id":"100100"  "dstip":"192.168.10.50"
"timestamp":"2026-10-20T11:45:01.366+0000"  "id":"100100"  "dstip":"192.168.10.51"
"timestamp":"2026-10-20T11:45:03.368+0000"  "id":"100101"  "dstip":"192.168.30.50"
```

- Router to SIEM latency: about 1 second.
- The 5th event fired the correlation rule **100101** (level 10, MITRE **T1046**) instead of a 5th 100100.
- `previous_output` in the alert holds the 4 earlier events, which is the evidence an analyst opens first:

```
33: Oct 20 11:45:00.331 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.10.51 (8/0), 1 packet
32: Oct 20 11:44:58.322 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.10.50 (8/0), 1 packet
31: Oct 20 11:44:56.309 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.51 (8/0), 1 packet
30: Oct 20 11:44:54.300 UTC: %SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.50 (8/0), 1 packet
```

The `30:`-`33:` prefixes are R1's syslog sequence numbers. Syslog is UDP, so a gap in this sequence would prove lost messages.

## 4. Second sweep: delayed, not duplicated

The same sweep was run again a few minutes later. Those 5 flows were still in R1's log cache, so R1 did not log them immediately. It reported them in the next 5-minute summary:

```
"timestamp":"2026-10-20T11:50:23.214+0000"  "id":"100100"  "dstip":"192.168.100.51"
"timestamp":"2026-10-20T11:50:23.214+0000"  "id":"100100"  "dstip":"192.168.100.50"
"timestamp":"2026-10-20T11:50:23.214+0000"  "id":"100100"  "dstip":"192.168.30.50"
"timestamp":"2026-10-20T11:50:23.214+0000"  "id":"100100"  "dstip":"192.168.10.50"
"timestamp":"2026-10-20T11:50:23.214+0000"  "id":"100101"  "dstip":"192.168.10.51"
```

- I first called this a duplicate ("echo"). It was not: `1 packet` in a summary line counts **new** packets since the last report. Asking "what did anyone do at that time?" settled it: the sweep had been repeated.
- **Timeline rule:** many lines with the **same millisecond** are an R1 summary. The packets happened somewhere in the previous 5 minutes. An incident report must say "between 11:45 and 11:50", not "at 11:50:23".
- `ignore` on 100101 was considered and rejected: it would have hidden this real second scan. In security, documented noise is better than a blind spot.

## 5. Sensor limit: fast vs slow sweep

**Fast sweep (no delay between pings)**, first run:

```
R1 immediate:      100.100, 10.100, 30.10                    (3 of 5)
R1 5-min summary:  100.1 (2 packets), 10.1 (2 packets), ...  at 11:36:22.156, same millisecond
Wazuh:             3 x 100100 in 60 s  ->  100101 did NOT fire
```

**Slow sweep (2 seconds apart):** all 5 logged immediately, 100101 fired (section 3).

**Nothing was lost in the data plane.** The ACL counter equals the sum of packets in the log lines:

```
R1#show access-lists DMZ-IN
    60 deny ip 192.168.99.0 0.0.0.255 192.168.0.0 0.0.255.255 log (10 matches)

log lines: 1 + 1 + 1 + 1 + 2 + 1 + 2 + 1 = 10
```

```
R1#show running-config all | include logging rate-limit|access-list logging
logging rate-limit console 10 except errors
```

The only default rate limit applies to the **console**, so it does not explain the delay of syslog messages. The behaviour (fast new flows are delayed to the summary) is proven by the experiment. The exact IOS mechanism is not confirmed.

**Conclusion:** R1 ACL logging is a weak scan sensor. A fast scan is detected up to 5 minutes late. See [logging-design.md](../docs/logging-design.md#4-sensor-limits-what-the-siem-cannot-see).

## 6. DVWA web logs

**Container log files are real files now**

```
$ sudo docker exec dvwa-dvwa-1 ls -l /var/log/apache2
-rw-r--r-- 1 root root   0 Oct 20 14:05 access.log
-rw-r--r-- 1 root root 434 Oct 20 14:05 error.log
-rw-r--r-- 1 root root   0 Oct 20 14:05 other_vhosts_access.log

$ sudo tail -3 /var/log/dvwa/access.log
192.168.10.100 - - [20/Oct/2026:14:05:58 +0000] "GET /login.php HTTP/1.1" 200 980 ...
192.168.10.100 - - [20/Oct/2026:14:06:03 +0000] "POST /login.php HTTP/1.1" 302 507 "http://192.168.99.10/login.php" ...
192.168.10.100 - - [20/Oct/2026:14:06:03 +0000] "GET /index.php HTTP/1.1" 200 2850 "http://192.168.99.10/login.php" ...
```

- The real client IP (WIN10) is logged.
- `POST /login.php` → `302` → `GET /index.php` = a **successful** login. A failed DVWA login is also a `302`, but back to `login.php`. Brute-force detection must look at where the redirect goes, not at a `401` code.

**Agent picked up the files** (after the `linux-dmz` group config was pushed)

```
2026/10/20 14:56:36 wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/dvwa/access.log'.
2026/10/20 14:56:36 wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/dvwa/error.log'.
```

**Test request from WIN10**

```
http://192.168.99.10/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit
```

**Alert:** rule **31164** `SQL injection attempt.` (level 6), agent `ubuntu-web`. HTTP status in the event: **302**.

**Triage:** `302` means the request was redirected (the session had expired), so the payload never reached the vulnerable page. An alert means someone *tried*. The response code is the first evidence of whether it worked:

| Response | Meaning |
|----------|---------|
| 302 / 403 / 404 | Did not reach the target. Attempt only |
| 200 | Processed. Check response size and other logs: was data returned? |
| 500 | The input broke something on the server. Suspicious |

## 7. Time is consistent

The same event on R1 (`11:44:54.300 UTC`) and in Wazuh (`11:44:55.331 UTC`), one second apart. Compare with Phase 5, where the same event showed 13:18, 15:18 and 18:xx on three devices.
