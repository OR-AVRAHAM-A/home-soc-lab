# Phase 7: AD Hardening Verification

Evidence per sub-step. Dashboard times are browser time (Israel, UTC+3); stored timestamps are UTC.

## A. Audit policy applied

`Compare-Object` of `auditpol /get /category:*` before vs after — only the planned changes, nothing turned off by accident:

```
Credential Validation              Success  ->  Success and Failure
Kerberos Authentication Service    Success  ->  Success and Failure
Kerberos Service Ticket Operations Success  ->  Success and Failure
User Account Management            Success  ->  Success and Failure
Directory Service Access           Success  ->  Success and Failure
Directory Service Changes          No Auditing -> Success
Security System Extension          No Auditing -> Success
Network Policy Server              Success and Failure -> No Auditing   (intended, no NPS)
```

**4719 in Wazuh** (the audit change itself, T1562.002):

```
data.win.system.eventID        4719
data.win.eventdata.subjectUserName  DC$          (GPO-applied = computer account, not a person)
data.win.eventdata.auditPolicyChanges  Success added, Failure added
```

**4771 in Wazuh** (Kerberos pre-auth failure — this event was NOT recorded before the policy change):

```
data.win.system.eventID        4771
targetUserName                 Administrator
ipAddress                      ::ffff:192.168.10.100   (CLIENT-01)
status                         0x18                    (bad password)
```

## D. Admin accounts

**Domain Admins after** — one member:

```
Get-ADGroupMember "Domain Admins" -Recursive
  Administrator
```

**4729 in Wazuh** (PenTest group removed from Domain Admins):

```
eventID         4729
targetUserName  Domain Admins
targetSid       S-1-5-21-...-512        (well-known RID for Domain Admins)
memberName      CN=PenTest,OU=Groups,DC=cyber,DC=local
subjectUserName Administrator
```

**Dormant disable** — 91 accounts, then zero enabled-inactive:

```
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly | Where Enabled | Measure | Count
  0
```

Wazuh: **91 × 4725** within ~1 second (16:26:06.4 → 16:26:07.0), subject Administrator. That spike is the T1531 shape to recognise.

## C. Password policy + rule 100200

`net accounts /domain` after:

```
Minimum password length:     14
Lockout threshold:           15
Lockout duration (minutes):  15
Lockout observation window:  15
Maximum password age (days): 60
```

Custom rule 100200 fired on a test policy change:

```
rule.id 100200  level 10... (2 alerts)
data.win.system.eventID         4739
data.win.eventdata.domainPolicyChanged   Password Policy
subjectUserName                 Administrator
rule.mitre.id                   T1484.001
```

Note: 4739 is in the DC Security log and reaches Wazuh, but with no custom rule it matched only the level-0 generic rule (60103), so it never appeared as an alert — the same gap class as the R1 ACL event 4716. Rule 100200 elevates it.

## E. Protocol hardening (verified via policy registry keys)

```
EnableMulticast (DNSClient policy)        0     LLMNR off
RequireSecuritySignature (LanManServer)   1     SMB server signing
RequireSecuritySignature (Workstation)    1     SMB client signing
EnableSMB1Protocol                        False SMBv1 removed
LDAPServerIntegrity (NTDS)                2     LDAP signing required
```

LDAP signing needed an **Enforced** link to beat Default Domain Controllers Policy:

```
GpoLinks (Domain Controllers OU):
  Order 1  Default Domain Controllers Policy   Enforced False   (sets LDAPServerIntegrity=1)
  Order 2  DC - Advanced Audit Policy          Enforced False
  Order 3  Hardening - Domain Controllers      Enforced True    (sets LDAPServerIntegrity=2, wins)
```

Domain still healthy after requiring signing:

```
nltest /sc_query:cyber.local
  Trusted DC Connection Status Status = 0 0x0 NERR_Success
```

## F. Kerberos AES

```
SupportedEncryptionTypes = 24      (AES128 + AES256, no RC4/DES)
```

`klist` on CLIENT-01 — every ticket AES-256:

```
Server: krbtgt/CYBER.LOCAL          KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
Server: cifs/DC.cyber.local         KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
Server: LDAP/DC.cyber.local         KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
```

## G. LAPS

Schema extended, self-permission delegated:

```
Update-AdmPwdADSchema
  ms-Mcs-AdmPwdExpirationTime   Success
  ms-Mcs-AdmPwd                 Success
  cn=computer (modify class)    Success

Set-AdmPwdComputerSelfPermission -OrgUnit "OU=Workstations,..."   Delegated
```

CSE installed on CLIENT-01 (elevated, with log):

```
Local Administrator Password Solution 6.2.0.0
Installation success or error status: 0
C:\Program Files\LAPS\CSE\AdmPwd.dll  present
```

Password retrieved from the DC (value redacted):

```
Get-AdmPwdPassword -ComputerName CLIENT-01
  ComputerName        CLIENT-01
  Password            <random 14-char, complex>
  ExpirationTimestamp 11/20/2026   (+30 days, auto-rotates)
```

## Post-reboot / post-change health

After the DC reboot (during the LDAP debugging) and all signing/AES changes, all three Wazuh agents reconnected:

```
agent_control -l
  001 DC01        Active
  002 WIN10       Active
  003 ubuntu-web  Active
```
