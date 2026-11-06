# Attack Simulation Design & Detection Mapping (Phase 8)

Phase 8 validates everything built in phases 1–7 by attacking the lab the way a real adversary would, then checking the SIEM for each step. It is run **purple-team** style: every attack is paired with the question *"was it blocked, and was it detected?"*.

The phase has two parts, matching two realistic adversary positions:

- **Part A — external** (`verification/phase8-external.md`): an attacker on the fake internet with no credentials, hitting the only published service (DVWA on TCP/80).
- **Part B — internal / assume-breach** (`verification/phase8-internal-ad.md`): an attacker who already has a foothold on the Users VLAN (a phished workstation) and attacks Active Directory directly.

## Method

For each technique:

1. Run it from Kali.
2. Record what the **hardening** did (blocked / allowed, and why).
3. Record what the **SIEM** saw (which rule, or nothing — a detection gap).
4. If there is a gap, write a custom rule to close it and re-test.

The point is not to "win" as the attacker. A blocked attack and a fired alert are both successes for the defender. Honest gaps are documented and, where it matters, closed.

## Attacker positions

| Part | Kali location | Starting access | Reaches |
|------|---------------|-----------------|---------|
| A | Fake internet, `10.20.20.254` | none | pfSense WAN → port-forward → DVWA only |
| B | Users VLAN 10, `192.168.10.50` | none, then one phished domain user | the DC on all AD ports (users must reach the DC — intentional) |

For Part B, Kali was re-cabled in EVE-NG from the WAN segment to a free SW1 access port in VLAN 10. The key realisation: a workstation VLAN is **trusted** to reach the servers VLAN (no ACL between them, unlike the DMZ). That trust is exactly what makes a single phished workstation dangerous.

## Detection scorecard

| # | Attack | MITRE | Hardening result | Detection |
|---|--------|-------|------------------|-----------|
| **A — External** | | | | |
| 1 | Port scan (nmap `-p-`) | T1046, T1595 | pfSense default-deny: only TCP/80 open | **100300/100301** (new) |
| 2 | SQL injection (DVWA) | T1190 | app-level (DVWA is the target) | 31164 (built-in) |
| 3 | Reflected/stored XSS | T1059.007 | app-level | **gap** — client-side, no server signal |
| 4 | Web login brute force | T1110 | no app lockout (DVWA) | **100400/100401** (new) |
| 5 | Command injection → RCE | T1190, T1059 | app-level | **gap** — payload is in the POST body, not in `access.log` |
| 6 | Lateral movement (DMZ → internal) | T1046 | **R1 ACL `DMZ-IN` blocked** | 100100 ×3 (Phase 6 rule) |
| **B — Internal (AD)** | | | | |
| 7 | SMB/LDAP enumeration | T1087, T1018 | null session + anonymous bind **denied** | 60205 (built-in) |
| 8 | Kerberos user enumeration | T1087.002 | names leak (inherent to Kerberos) | 60205 (built-in) |
| 9 | Password spray | T1110.003 | password policy held, no weak creds | 60122 / 60204 (built-in) |
| 10 | AS-REP roasting | T1558.004 | **0 roastable accounts** (Phase 7) | n/a — no attack surface |
| 11 | Kerberoasting | T1558.003 | AES-only + strong password = uncrackable | **100500** honeypot (new) |
| 12 | DCSync | T1003.006 | **denied** to a normal user; works only for Domain Admin | **100510** (new) |

## What the hardening proved

Every attack from a normal position was **blocked or made useless**:

- The firewall exposed exactly one port. The DMZ could not pivot inward (R1 ACL). Anonymous AD enumeration returned nothing. The password policy survived a spray. There were no AS-REP-roastable accounts and no crackable service tickets. A normal user could not DCSync.

The only successes required either an application bug (DVWA, which is *meant* to be vulnerable) or a credential the attacker had to be *given* (the assume-breach premise).

## Detection gaps found, and how they were closed

This is the real output of the phase. Three gaps were closed with custom rules; two are documented honestly.

| Gap | Why it existed | Fix |
|-----|----------------|-----|
| pfSense firewall blocks never alerted | pfSense forwards `filterlog` **without a hostname**, so the pre-decoder reads `filterlog[pid]:` as the hostname and the built-in pfSense decoder (keyed on program name) never matches | custom `pfsense-filterlog` decoder + rules **100300/100301** |
| Web login brute force never alerted | built-in rule 31108 consumes `login.php` web-access events at level 0 | **100400** chains `if_sid 31108` + `<match>login.php</match>`; **100401** correlates 6+ in 30 s |
| Kerberoasting (successful) invisible | a successful TGS request looks identical to legitimate traffic; AES removes the classic RC4 (`0x17`) signature | **honeypot** service account `svc-sql` + rule **100500**: any `4769` for it is malicious by definition |
| DCSync invisible | `4662` with the replication GUID is decoded but matches no alerting rule by default | rule **100510**: `4662` + GUID `1131f6aa` + requester is **not a machine account** |
| XSS / RCE not detected | XSS runs in the victim's browser (no server event); the command-injection payload travels in the POST body, which Apache's `access.log` does not record | documented — needs a WAF / network IDS that sees request bodies (future work) |

### The key lesson: hardening can erase detection signatures

The two most dangerous AD attacks — Kerberoasting and DCSync — produce only **benign-looking success events**. The failures (enumeration, spraying) trip built-in correlation rules; the successes do not. Worse, the AES-only hardening from Phase 7, which *prevents* Kerberoasting, also deletes the RC4 ticket-encryption flag that classic detections look for.

So in a well-hardened 2026 AD, you do not detect these by signature. You detect them by **behaviour**:

- **Kerberoasting** → a honeypot SPN that no legitimate process should ever request (rule 100500).
- **DCSync** → directory replication requested from a principal that is not a Domain Controller (rule 100510).

## Password policy is not retroactive (two real gaps found and fixed)

Phase 7 set a 14-character minimum, but Windows does not force existing passwords to change until they expire. Two accounts still carried short, pre-policy passwords:

- `danih` — 11 characters.
- **`Administrator`** — `Pa$$word`, 8 characters.

The second is the serious one. Minimising Domain Admins to a single account is good, but that single account then becomes the entire domain's single point of failure — and it had a dictionary password a spray would crack. Both were reset to long random pass-phrases. **Minimising privilege and strengthening the surviving account are the same job.**

## Custom detections — cumulative

| Rule | Level | Fires on | MITRE | Phase |
|------|-------|----------|-------|-------|
| 100100 | 8 | R1 ACL DMZ deny (one host) | — | 6 |
| 100101 | 10 | 5+ DMZ denies in 60 s | T1046 | 6 |
| 100200 | 8 | AD domain policy changed (4739) | T1484.001 | 7 |
| 100300 | 3 | pfSense firewall block | — | 8 |
| 100301 | 10 | 15+ pfSense blocks from one source in 30 s (port scan) | T1046, T1595 | 8 |
| 100400 | 3 | request to `login.php` | — | 8 |
| 100401 | 10 | 6+ `login.php` requests from one source in 30 s (brute force) | T1110 | 8 |
| 100500 | 12 | TGS requested for honeypot account `svc-sql` | T1558.003 | 8 |
| 100510 | 13 | directory replication by a non-machine account (DCSync) | T1003.006 | 8 |

Full rule and decoder XML is in [../configs/wazuh.md](../configs/wazuh.md).
