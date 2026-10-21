# Active Directory Hardening (Phase 7)

Domain: `cyber.local` · DC: `DC` (Windows Server 2019) · Client: `CLIENT-01` (Windows 10, Wazuh agent name "WIN10")

All changes were made **after** the SIEM was in place (Phase 6), so every change — and every mistake — is visible in Wazuh. Snapshots were taken before each risky step.

## GPO inventory

| GPO | Linked to | Purpose |
|-----|-----------|---------|
| `DC - Advanced Audit Policy` | Domain Controllers OU | Turn on the auditing the DC was not doing |
| Default Domain Policy (edited) | Domain root | Password + lockout policy |
| `Hardening - All Computers` | Domain root | LLMNR off, SMB signing, Kerberos AES |
| `Hardening - Domain Controllers` (**Enforced**) | Domain Controllers OU | LDAP server signing required |
| `LAPS - Workstations` | Workstations OU | Local admin password randomization |

A separate GPO per concern (never editing the built-in policies except where a setting must live there, like password policy). Easy to see what changed, easy to disable.

---

## A. Audit Policy

**Why first:** Windows only logs what the audit policy tells it to. Without it, the DC does not record the events, the agent forwards nothing, and no rule can fire. A detection is only as good as the log underneath it.

Advanced Audit Policy (not Basic — the two override each other unpredictably). The setting **"Audit: Force audit policy subcategory settings to override audit policy category settings"** is Enabled so Advanced wins.

**Important:** a GPO with Advanced Audit **replaces** the whole audit policy — any subcategory not listed becomes `No Auditing`. The GPO must include everything that should be on.

| Category | Subcategory | Setting | Key events |
|----------|-------------|---------|-----------|
| Account Logon | Credential Validation | Success, Failure | 4776 (NTLM) |
| | Kerberos Authentication Service | Success, Failure | 4768 (TGT), 4771 (pre-auth fail) |
| | Kerberos Service Ticket Operations | Success, Failure | 4769 (service ticket) |
| Account Management | User Account Management | Success, Failure | 4720, 4722, 4725, 4740 |
| | Security Group Management | Success | 4728, 4729, 4732 |
| | Computer Account Management | Success | 4741 |
| DS Access | Directory Service Access | Success, Failure | 4662 (DCSync) |
| | Directory Service Changes | Success | 5136 |
| Logon/Logoff | Logon | Success, Failure | 4624, 4625 |
| | Special Logon | Success | 4672 |
| | Account Lockout | Success | 4625 (at the workstation) |
| Policy Change | Audit Policy Change | Success | 4719 (T1562.002) |
| System | Security System Extension | Success | 4697 (service install, T1543) |

**Not configured on purpose:**
- Process Creation (4688) — Sysmon Event 1 already covers it, with richer data.
- Network Policy Server — no NPS in the lab.

### Detection insight: the dangerous attacks "succeed"

The most dangerous AD attacks are **successful** events, not failures:

```
Kerberoasting   →  4769 Success   (Ticket Encryption Type 0x17 = RC4)
AS-REP Roasting →  4768 Success   (Pre-Authentication Type 0)
DCSync          →  4662 Success   (replication GUID)
```

An audit policy with only Failure enabled catches crude brute force and misses all of these. Detection is based on **anomalous fields inside a successful event**, not on failure.

---

## B. Baseline (attack surface)

Mapped before changing anything (PowerShell, ActiveDirectory module):

| Finding | Count | Risk |
|---------|-------|------|
| Domain Admins | 14 (1 built-in + a nested group) | Should be 2-4 |
| Dormant accounts (90+ days, enabled) | ~91 | Unmonitored doors (T1078) |
| AS-REP roastable (`DoesNotRequirePreAuth`) | 1 (`silvie.cordi`) | T1558.004 |
| Accounts with SPN | 1 (`krbtgt` only) | Minimal Kerberoasting surface |
| `PasswordNeverExpires` | Administrator, Guest | Admin password never rotates |

Password policy baseline: min length **4**, lockout **Never** — password spraying almost guaranteed to succeed.

---

## C. Password + Lockout Policy

**Key rule:** domain password policy must be set at the **domain root** (Default Domain Policy). A GPO linked to an OU does **not** change domain users' password policy. Exceptions use Fine-Grained Password Policy (PSO).

| Setting | Before | After |
|---------|--------|-------|
| Minimum password length | 4 | 14 |
| Password complexity | - | Enabled |
| Maximum password age | 42 | 60 |
| Password history | 24 | 24 |
| Account lockout threshold | Never | 15 |
| Lockout duration | - | 15 min |
| Reset lockout counter after | - | 15 min |

### Lockout threshold is a trade-off

A low threshold is not strictly safer. An attacker can send wrong passwords on purpose to **lock out every user** (denial of service). A short auto-unlock `duration` (15 min) limits that damage because accounts unlock themselves.

```
threshold too low (3)  → easy DoS by lockout + false lockouts
threshold = Never      → brute force unlimited
threshold 15 + 15 min  → the balance
```

### Custom rule: detect domain policy changes

Changing the domain policy generates **4739**, but Wazuh's built-in handling leaves it at level 0 (no alert) — the same gap as the R1 ACL event 4716. Custom rule elevates it:

```xml
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

`if_sid 60103` chains from the built-in "Windows audit success event" rule (confirmed by inspecting an existing 4725 alert, which carried the `windows_security` group). The event's `domainPolicyChanged` field states exactly which policy changed (Password / Lockout / Kerberos).

---

## D. Admin accounts (least privilege)

### Domain Admins: 14 → 1

The 14 members were not all direct. `Get-ADGroupMember -Recursive` showed 14, but the direct `member` list held only `Administrator` and a group, `PenTest`. Thirteen users were Domain Admins **through a nested group** — a classic way to hide privilege.

Fix: remove the nested group in one operation, instead of 13 user edits.

```powershell
Remove-ADGroupMember -Identity "Domain Admins" -Members "PenTest" -Confirm:$false
```

- `PenTest` still exists, now with no elevated rights, so it can stay for controlled use later. The risk was the nesting, not the group.
- Alarm the reverse: event **4728** (member added to Domain Admins, T1098) is now meaningful.

### AS-REP roastable account

`silvie.cordi` (dormant + `DoesNotRequirePreAuth`, member of `IT Admins`). Defense in depth — clear the flag **and** disable:

```powershell
Set-ADAccountControl silvie.cordi -DoesNotRequirePreAuth $false
Disable-ADAccount silvie.cordi
```

(All 6 members of `IT Admins` turned out to be dormant — a group of dead accounts with a privileged-sounding name.)

### Dormant accounts: ~91 disabled in one pipeline

```powershell
$keep = @("Administrator","krbtgt","Guest")
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly |
  Where-Object { $_.Enabled -and ($_.SamAccountName -notin $keep) } |
  Disable-ADAccount
```

- `$keep` protects critical accounts — mandatory in a bulk action, one mistake hits dozens.
- No service accounts among them (SPN check), so nothing broke.
- 91 × event **4725** in under a second. That spike also looks like T1531 (mass account lockout, e.g. ransomware) — a future correlation rule should alert on X+ 4725 in a short window.

---

## E. Protocol hardening

| Setting | Attack blocked | MITRE | Where |
|---------|----------------|-------|-------|
| LLMNR off | Responder / NetNTLM theft | T1557.001 | All Computers |
| SMB signing required (server + client) | NTLM relay | T1557 | All Computers |
| SMBv1 removed | EternalBlue | T1210 | PowerShell per host |
| LDAP server signing required | LDAP relay to the DC | T1557 | Domain Controllers (**Enforced**) |

SMBv1 removal (feature, not just disable):

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart
```

NBT-NS was left in place (partial hardening) — per-interface, low value with one client. Noted as a gap.

### LDAP signing needed Enforced (GPO precedence)

The setting was correct in the GPO (`GptTmpl.inf` → `LDAPServerIntegrity=4,2`), the Security CSE was registered, SMB signing (same CSE) applied, gpupdate ran, even a reboot — and the registry stayed `1` (None).

Root cause: **Default Domain Controllers Policy explicitly sets LDAP signing to None** (`LDAPServerIntegrity=4,1`) and has link order 1 (highest precedence), so it won over the Hardening GPO at order 3. Marking the Hardening link **Enforced** makes it win regardless of order, without editing the built-in policy.

```
link order 1 = highest precedence (processed last, wins)
Enforced link overrides precedence and cannot be blocked → the way to guarantee a baseline
```

Verify effective values via the **policy registry keys**, not the cmdlets (the cmdlets read the local, non-policy value):

```powershell
(Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient' -Name EnableMulticast).EnableMulticast                                      # 0
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\LanManServer\Parameters' -Name RequireSecuritySignature).RequireSecuritySignature          # 1
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters' -Name LDAPServerIntegrity).LDAPServerIntegrity                             # 2
```

After requiring signing, confirmed the domain still works: `nltest /sc_query:cyber.local` = `Status = 0 NERR_Success`, and a domain user logged into CLIENT-01.

---

## F. Kerberos: AES only

No custom service accounts (only `krbtgt` has an SPN), so the Kerberoasting surface is already minimal. The remaining hardening is to block the weak encryption that **makes** Kerberoasting practical.

```
RC4 ticket → cracks fast offline (hashcat)
AES ticket → orders of magnitude slower
```

GPO (`Hardening - All Computers`): "Network security: Configure encryption types allowed for Kerberos" = AES128 + AES256 + Future, with DES and RC4 **unchecked**.

```powershell
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\Parameters" -Name SupportedEncryptionTypes).SupportedEncryptionTypes
# 24  = AES128 (8) + AES256 (16), no RC4/DES
```

Verified on CLIENT-01: `klist` shows every ticket (TGT, CIFS, LDAP) as `AES-256-CTS-HMAC-SHA1-96`.

**Protected Users** (forces AES, blocks NTLM/RC4 for members) is a strong control for **dedicated** admin accounts. Microsoft advises **not** adding the built-in Administrator (RID 500) — it is the break-glass account. In a real tiering model we would add dedicated admin accounts. Documented as a recommendation.

---

## G. LAPS (local admin password randomization)

Prevents lateral movement via a shared local admin password (Pass-the-Hash, T1550.002 / T1078). Each machine gets a unique, random local admin password, stored in AD and rotated automatically.

Windows LAPS (native) was not available (Server 2019 not patched to April 2023+), so **Legacy LAPS** (`LAPS.x64.msi` 6.2.0.0, SHA256 `0001DD...F6D1`) was used. The MSI was moved into the VMs by copy-paste through VMware — no network, so the lab stayed isolated.

| Part | Where | Install |
|------|-------|---------|
| Management tools (PowerShell, ADMX) | DC01 | `msiexec /i LAPS.x64.msi ADDLOCAL=Management.PS,Management.ADMX,Management.UI /quiet` |
| CSE (the worker) | CLIENT-01 | `msiexec /i LAPS.x64.msi /quiet` (needs elevation) |

```powershell
# On DC01
Update-AdmPwdADSchema                                                      # adds ms-Mcs-AdmPwd(+ExpirationTime)
Set-AdmPwdComputerSelfPermission -OrgUnit "OU=Workstations,DC=cyber,DC=local"   # computer can write its own password
```

GPO `LAPS - Workstations` (Administrative Templates → LAPS):
- Enable local admin password management = Enabled
- Password Settings: complexity all, length 14, age 30 days
- Do not allow expiration longer than required = Enabled

Retrieve (DC01):

```powershell
Get-AdmPwdPassword -ComputerName CLIENT-01 | Format-List ComputerName, Password, ExpirationTimestamp
# Password = <random 14-char>, ExpirationTimestamp = +30 days
```

- The DC is **not** LAPS-managed (no CSE on it).
- Schema extension is **irreversible** — snapshot `pre-laps` first.
- Production: a dedicated "LAPS Readers" group gets read rights instead of relying on Domain Admins / full control.
