# AD Hardening Design & Detection Mapping (Phase 7)

## Order of work

Logging (Phase 6) was done before AD hardening on purpose. With the SIEM already collecting, every hardening change here is visible in Wazuh, and so is every mistake. Hardening without logging means you cannot prove what changed, or notice when an attacker changes it back.

Each sub-step followed: see the current state → change one thing → **verify in the actual result, not in the management tool** → confirm the event reached Wazuh. "GPO applied" is not the same as "setting took effect" — proven twice in this phase (the audit CSE, and LDAP signing precedence).

## Attack surface before and after

| Dimension | Before | After |
|-----------|--------|-------|
| Domain Admins | 14 (nested group) | 1 (built-in Administrator) |
| Enabled dormant accounts | ~91 | 0 |
| AS-REP roastable | 1 | 0 |
| Min password length | 4 | 14 |
| Account lockout | Never | 15 / 15 min |
| LLMNR / SMBv1 | on | off / removed |
| SMB & LDAP signing | not required | required |
| Kerberos encryption | RC4 allowed | AES only |
| Local admin password | shared/static | LAPS, unique, rotated |
| Audit coverage | partial, Success-only in key areas | full, Success+Failure where it matters |

## Detection mapping

What the hardened audit policy now lets Wazuh see. The pattern to learn: **the serious attacks are successful events; the signal is an anomalous field, not a failure.**

| Attack | MITRE | Event | Subcategory | Signal |
|--------|-------|-------|-------------|--------|
| Password spraying / brute force | T1110 | 4625, 4771, 4776 | Logon, Kerberos Auth, Credential Validation | many failures, one source, many users |
| Valid account logon | T1078 | 4624 | Logon | Logon Type, source IP |
| Account lockout | T1110 | 4740 | User Account Management | burst of lockouts |
| Kerberoasting | T1558.003 | 4769 **Success** | Kerberos Service Ticket Ops | Ticket Encryption Type `0x17` (RC4) |
| AS-REP Roasting | T1558.004 | 4768 **Success** | Kerberos Auth Service | Pre-Auth Type `0` |
| User created | T1136.002 | 4720 | User Account Management | off-hours, by non-admin |
| Added to Domain Admins | T1098 | **4728** | Security Group Management | targetSid ends `-512` |
| DCSync | T1003.006 | 4662 **Success** | Directory Service Access | replication-get-changes GUID |
| Service installed | T1543.003 | 4697 | Security System Extension | unexpected service |
| AD object changed | - | 5136 | Directory Service Changes | GPO / sensitive object edits |
| **Audit policy changed** | T1562.002 | 4719 | Audit Policy Change | someone turning auditing off |
| **Domain policy changed** | T1484.001 | 4739 → rule 100200 | (syslog of DC) | password/lockout/Kerberos weakened |

### Triage notes learned this phase

- **4719 / group changes by `DC$`**: when a GPO applies a change, the subject is the computer account (SYSTEM), not the person who edited the GPO. A **human** subject on `auditpol`/policy change is far more suspicious. To find who edited a GPO, look at 5136 on the GPO object.
- **Well-known RIDs**: `-512` Domain Admins, `-519` Enterprise Admins, `-500` built-in Administrator. Identify the group by RID even without the name.
- **Mass 4725**: 91 disables in one second is maintenance here, but the same shape is T1531 (ransomware locking users). Context — change window, source, known admin — decides.

## Custom detections added (cumulative)

| Rule | Level | Fires on | MITRE |
|------|-------|----------|-------|
| 100100 | 8 | R1 ACL DMZ deny (one host) | - |
| 100101 | 10 | 5+ DMZ denies in 60s (scan) | T1046 |
| 100200 | 8 | Domain policy changed (4739) | T1484.001 |

Planned: X+ 4725 in a short window (mass account disable, T1531); Kerberoasting (4769 with RC4); AS-REP (4768 pre-auth 0).

## What was deferred (honest gaps)

| Item | Why | Plan |
|------|-----|------|
| NBT-NS disable | per-interface, low value with one client | registry/DHCP option later |
| Windows LAPS (native) | Server 2019 not patched to April 2023+ | used Legacy LAPS instead |
| Protected Users for admins | break-glass Administrator must not be a member | add dedicated admin accounts in a tiering model |
| Dedicated LAPS Readers group | lab relies on Domain Admins/full control to read | create the group in production |
| DDCP weakening of LAPS signing left in place | best practice: don't edit the built-in policy | overridden with an Enforced hardening GPO |
