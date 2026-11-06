# Phase 8 — External Attack Verification (Part A)

Attacker: Kali on the fake internet (`10.20.20.254`), no credentials. Target: DVWA, published by pfSense port-forward (`WAN:80 → 192.168.99.10:80`), which preserves the attacker's real source IP. All timestamps UTC.

## 1. Recon — port scan

```bash
nmap -Pn -p- 10.20.20.1
```

Only **TCP/80** is open; every other port is `filtered`. pfSense default-deny works: one published service, nothing else.

### Gap found: firewall blocks were not alerting

pfSense was forwarding `filterlog` to Wazuh, but nothing showed up for blocked traffic. Cause: pfSense sends `filterlog` **without a hostname field**, so Wazuh's pre-decoder treats `filterlog[pid]:` as the hostname, and the built-in pfSense decoder (which keys on the program name) never matches.

Fix — a custom `pfsense-filterlog` decoder plus two rules:

```
100300  level 3   pfSense block (one connection)
100301  level 10  15+ blocks from one source in 30s  → port scan (T1046, T1595), ignore=60
```

`wazuh-logtest` confirmed the decoder extracts `srcip`, `dstip`, `dstport`, `protocol`, `interface`. A repeat scan then fired 100301. (`ignore=60` was added after the first run flooded ~85 alerts per scan.)

## 2. SQL injection

DVWA at **security = low**. Classic authentication bypass and UNION extraction dumped the user table (MD5 hashes), cracked offline:

```
admin   / password     gordonb / abc123
1337    / charley      pablo   / letmein
smithy  / password
```

**Detection:** built-in rule **31164** *SQL injection attempt* fired (agent `ubuntu-web`).

Triage note learned: the HTTP response code is the first evidence of success. `302 / 403 / 404` = the request did not reach the target (attempt only). `200` = processed — check response size. `500` = the input broke something (suspicious).

## 3. XSS

Reflected and stored XSS executed; a session cookie was captured via an injected `<script>`.

**Detection: gap.** No alert fired. XSS is a client-side attack — the malicious script runs in the victim's browser, so from Apache's point of view the request is an ordinary `GET`/`POST`. There is no server-side signal to key on. Documented as a gap; a WAF would be needed.

## 4. Web login brute force

```bash
# repeated POSTs to login.php with a password list
```

`admin:password` succeeded. A successful DVWA login is `POST /login.php → 302 → GET /index.php`; a failed one is `302` back to `login.php`. Brute-force detection must look at **where the redirect goes**, not at a `401` (DVWA never returns one).

### Gap found: web brute force was not alerting

Built-in rule **31108** *Ignored URLs (simple queries)* consumes `login.php` web-access events at level 0, so a flood of login attempts produced no alert.

Fix — chain off 31108 instead of fighting it:

```xml
<rule id="100400" level="3">
  <if_sid>31108</if_sid>
  <match>login.php</match>
  <description>Web request to login page from $(srcip)</description>
</rule>

<rule id="100401" level="10" frequency="6" timeframe="30" ignore="60">
  <if_matched_sid>100400</if_matched_sid>
  <same_source_ip />
  <description>Web login brute force: 6+ requests to login.php from $(srcip) in 30s</description>
  <mitre><id>T1110</id></mitre>
</rule>
```

Lesson learned the hard way: **100400 had to be level ≥ 1.** At level 0 it was not counted by `if_matched_sid`, so 100401 never fired. Once raised to level 3, the brute force fired **100401** (level 10, T1110) from source `10.20.20.254`.

Two false starts were also recorded, both static-vs-dynamic field issues: `<field name="url">` errored (`url` is static), and `<url>login.php</url>` matched the field name but not the value `/login.php`. `<match>login.php</match>` (substring on the full log) was the correct form.

## 5. Command injection → RCE

DVWA command injection at **security = low**:

```
127.0.0.1; id   →   uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Remote code execution confirmed, as the low-privilege `www-data` account. (At security = **impossible** the same payload returns "invalid IP" — input validation rejects the chained command. A useful contrast of the same bug hardened.)

**Detection: visibility gap.** The DVWA command-injection form is a **POST**. Apache's `access.log` records the request URL, not the POST body, so the payload (`127.0.0.1; id`) never reaches the SIEM. This is deeper than a missing rule — the data the rule would need is not collected. A WAF, `mod_security`, or a network IDS that inspects request bodies would be required.

## 6. Lateral movement — the pivot (the climax of Part A)

From the compromised web container, probe the internal networks:

```
127.0.0.1; ping -c 3 192.168.100.100; ping -c 3 192.168.10.100; ping -c 3 192.168.30.10
```

Result — every internal target:

```
From 192.168.99.254 icmp_seq=1 Packet filtered
100% packet loss
```

`192.168.99.254` is R1's DMZ interface. The **`DMZ-IN` ACL blocked the pivot** — the web server fell, but network segmentation kept the blast radius inside the DMZ. This is defense-in-depth working exactly as designed.

**Detection:** rule **100100** fired **3 times** (one per destination subnet), group `dmz_violation`, level 8.

Two triage points worth keeping:

- **Attribution.** The ACL log `srcip` is `192.168.99.10` — the DMZ **host**, not the container. Container ICMP egresses through the host's network namespace, so the firewall and the SIEM attribute the activity to the web server. The red flag an analyst reads: *why is a DMZ web server suddenly pinging the DC?*
- **Why 100101 did not fire.** The correlation rule needs 5+ denies from one source in 60 s; only 3 subnets were probed. Correct behaviour — a careful attacker who touches only a few targets stays under the correlation threshold, which is exactly why the single-event rule 100100 (level 8) must still be triaged, not dismissed as noise.
