# Troubleshooting Log

Real problems I hit while building the lab, how I diagnosed them, and what I learned.

## Phase 5: Ubuntu-Web (DMZ)

### 1. Ping to DC01 and Kali failed, but ping to R1 worked

| | |
|---|---|
| **Symptom** | With the temporary internet NIC (`ens4`) connected, pings to `192.168.100.100` and `10.20.20.254` failed. Ping to the gateway `192.168.99.254` worked. |
| **Diagnosis** | `ip route get 192.168.100.100` showed `via 192.168.240.2 dev ens4`. Two default routes existed: `ens4` metric 100 and `ens3` metric 200. Lower metric wins. |
| **Cause** | Only the directly connected subnet (`192.168.99.0/24`) used `ens3`. Everything else left through the internet NIC, where nobody knows the lab addresses. |
| **Lesson** | On a multi-homed host, `ip route get <IP>` is the first check. Linux picks the most specific route, then the lowest metric. |

### 2. `apt` failed: "No space left on device"

| | |
|---|---|
| **Symptom** | `E: Write error - write (28: No space left on device)` |
| **Diagnosis** | `df -h /` showed 2.0G, 100% used. `lsblk` showed the whole disk was only 2.2G. `vgs` was empty, so no LVM. Layout `vda1` + `vda14` + `vda15` = Ubuntu **cloud image**. |
| **Fix** | Node powered off. On the EVE-NG host, resized the node's **overlay** disk (never the base image) with `qemu-img resize`. Then inside the guest: `growpart` + `resize2fs`. |
| **Lesson** | Cloud images ship with a tiny disk on purpose and expect to be grown at deployment. |

### 3. Resized the wrong disk, then resized too much

| | |
|---|---|
| **Symptom 1** | `qemu-img resize` failed: `Could not open backing file .../vios-adventerprisek9...`. |
| **Cause 1** | The overlay belonged to a Cisco vIOS node from an old lab, not to Ubuntu-Web. Similar size, wrong file. |
| **Fix 1** | Identified the right overlay by its backing file, not by size: `file /opt/unetlab/tmp/0/*/*/virtioa.qcow2 \| grep -i ubuntu` |
| **Symptom 2** | The disk became **146 GiB** instead of 20 GiB. |
| **Cause 2** | `+18G` is relative. The command ran 8 times (2.2 + 8 x 18 = 146.2). It succeeds silently, so it looked like nothing happened. |
| **Fix 2** | The guest had not grown its partition yet, so shrinking was safe: `qemu-img resize --shrink <file> 20G`. |
| **Lesson** | Prefer **idempotent** commands (absolute size `20G`, not `+18G`). Before shrinking any disk, confirm the partition inside does not use the space being removed. |

### 4. `growpart` failed on a full disk

| | |
|---|---|
| **Symptom** | `mkdir: cannot create directory '/tmp/growpart.856': No space left on device` |
| **Cause** | `growpart` needs a temp directory, and `/tmp` lives on the full root partition. Chicken and egg. |
| **Fix** | `sudo TMPDIR=/run growpart /dev/vda 1`. `/run` is a tmpfs (RAM), not affected by the full disk. Then `resize2fs /dev/vda1`: root went from 2.0G to 20G online. |

### 5. Network config at risk of being overwritten

| | |
|---|---|
| **Finding** | The netplan file was `50-cloud-init.yaml`, with a warning that changes do not persist across an instance reboot. |
| **Fix** | Disabled cloud-init network management and hostname reset (`/etc/cloud/cloud.cfg.d/99-*.cfg`), then rebooted to prove the IP and hostname stayed. |

### 6. Old netplan still active after "editing" it

| | |
|---|---|
| **Symptom** | `ip route` still showed `metric 200`, and `ens4` was still UP. |
| **Cause** | The new file had not been saved/applied. The `metric 200` value existed only in the old version: a fingerprint of which config was live. |
| **Fix** | Re-edited, `sudo netplan try`, `sudo netplan apply`. Route now shows no metric. |

### 7. `docker compose` said "no configuration file provided"

| | |
|---|---|
| **Cause** | Ran from `~` instead of `~/DVWA`. Compose looks for `compose.yml` in the current directory. |

### 8. Forgot the pfSense admin password

| | |
|---|---|
| **Fix** | Console menu option **3** (Reset webConfigurator password), log in with the default, change it immediately, store it in a password manager. Option **4** would have been a factory reset. |

## Phase 6: Logging (NTP + Wazuh)

### 9. R1 stuck on "Clock is unsynchronized"

| | |
|---|---|
| **Symptom** | After `ntp master 3`, `show ntp status` kept showing `unsynchronized`, and DC01's `w32tm /resync` returned "no time data was available". |
| **Cause** | The NTP master first runs in state `FREQ` (measuring its own clock frequency). It only serves time after it stabilizes. |
| **Fix** | Wait about 15 minutes, then resync the clients. |
| **Lesson** | Not every "broken" state is broken. Read what the state means before changing config. |

### 10. "Every device shows a different hour"

| | |
|---|---|
| **Symptom** | After NTP, the dates matched but DC01 showed a different hour than R1. |
| **Cause** | DC01's **time zone** was Pacific. The clock itself was synced. |
| **Lesson** | Clock and time zone are two different things. Infrastructure runs in UTC. Windows displays Israel time but stores UTC internally. Check the offset before calling it drift. |

### 11. Garbled paste into the QEMU console

| | |
|---|---|
| **Symptom** | A `sed` command to edit `timesyncd.conf` failed, and pasted lines arrived with missing or duplicated characters. |
| **Cause** | The EVE-NG/QEMU console does not handle fast pasting. |
| **Fix** | A small drop-in file (`/etc/systemd/timesyncd.conf.d/lab-ntp.conf`) typed line by line, then SSH for longer work. |

### 12. WAZUH-01 could reach its gateway but not DC01

| | |
|---|---|
| **Symptom** | `ping 192.168.30.254` worked, `ping 192.168.100.100` failed. |
| **Diagnosis** | The default route went out through the temporary NAT NIC. Same root cause as problem 1. |
| **Fix** | A static route `192.168.0.0/16 via 192.168.30.254` in netplan (more specific than the default), then the NAT NIC was disconnected. |

### 13. Wazuh agent on Windows: "The specified service name is invalid"

| | |
|---|---|
| **Cause** | `msiexec /q` returns immediately (asynchronous). `Start-Service` ran before the installer finished, and the first attempt was in a non-elevated shell. |
| **Fix** | Elevated PowerShell, and wait for the installer, with a verbose log for evidence: `Start-Process msiexec.exe -ArgumentList '/i wazuh-agent-4.14.9-1.msi /q WAZUH_MANAGER=192.168.30.10 WAZUH_AGENT_GROUP=windows /l*v C:\agent-kit\install.log' -Wait` |
| **Also** | PowerShell commands typed into `cmd.exe` fail. Type `powershell` first. |

### 14. The SIEM was down for 3 hours 21 minutes

| | |
|---|---|
| **Symptom** | `wazuh-logtest error when connecting with wazuh-analysisd`. `wazuh-control status`: every daemon `not running`. |
| **Evidence** | `ossec.log`: `08:03:48 ERROR: (2102): Duplicated decoder with prematch: 'cisco-ios-acl'` then `CRITICAL: (1202): Configuration error at 'etc/decoders/0065-cisco-ios_decoders.xml'` |
| **Cause** | The manager was restarted after the default decoder file had been **copied** to `etc/decoders`, but **before** the original was excluded in `ossec.conf`. Both files loaded, `analysisd` refused to start, and the whole manager stopped with it. |
| **Fix** | With the exclude in place, `wazuh-analysisd -t` passed, then restart. Back up at `11:25:09`. |
| **Impact** | 08:03 → 11:25 with no detection, and nobody noticed. |
| **Lessons** | 1) Order matters: exclude first, copy second, test third, restart last. 2) Check `ossec.log` after every restart. 3) Have a rollback plan before touching the SIEM. 4) Monitor the monitor: a silent log source is a blind spot attackers use on purpose (MITRE ATT&CK **T1562**, Impair Defenses). |

### 15. Dashboard showed "No results" for alerts that existed

| | |
|---|---|
| **Symptom** | `alerts.json` had rule 100100 alerts, the dashboard showed nothing. |
| **Cause** | The dashboard was **pinned** to agent DC01 (filter `agent.id: 001`). Syslog events belong to agent **000** (the manager). |
| **Fix** | Unpin the agent. |
| **Lesson** | When a search returns nothing, check filters and the time range before assuming there is no data. |

### 16. A custom decoder in `local_decoder.xml` was never used

| | |
|---|---|
| **Symptom** | `wazuh-logtest` Phase 2 showed only `cisco.facility/severity/mnemonic` for ICMP lines. |
| **Wrong guess** | I first assumed the decoder name had to match the sibling decoders. Renaming did not help. |
| **Root cause** | Reading the original decoder file showed a catch-all child (`cisco-ios-default`, no `prematch`) at the end. The first matching child wins, and default decoders load before local ones. |
| **Fix** | Copy the default file, add the decoder before the catch-all, exclude the original. See [configs/wazuh.md](../configs/wazuh.md#custom-decoder-icmp-acl-logs). |
| **Lesson** | Read the source before guessing. `logtest` shows the parent name in `name:`, which made the first guess look right. |

### 17. `grep` in archives.log matched itself

| | |
|---|---|
| **Cause** | With `logall` on, the `sudo grep ...` command itself was logged (auth log), and then found by the same grep. |
| **Fix** | Search for a pattern that only the real log line contains: `grep -F -e '->192.168.30.10 '`. |

### 18. Multi-line paste + `sudo` = commands lost

| | |
|---|---|
| **Symptom** | Two commands merged into one line (`restart wazuhsudo systemctl ...`), and `ossec.log` had no new lines, so the restart never ran. |
| **Cause** | `sudo` stopped to ask for a password and consumed the next pasted lines as input. |
| **Fix** | One command per paste. Verify with the log, not with the absence of an error. |

### 19. `tcpdump: command not found`

| | |
|---|---|
| **Cause** | WAZUH-01 was already isolated (NAT disconnected) when the tool was needed. |
| **Lesson** | Install a basic toolkit (`tcpdump net-tools dnsutils traceroute`) while the temporary link is still up. Used `archives.log` (`logall`) as a workaround. |

### 20. `compose.yml`: duplicate keys after editing

| | |
|---|---|
| **Cause** | The example snippet included context lines (`ports`, `restart`, `db:`). They were pasted too, so `dvwa` had two `ports:` and an empty `db:` appeared. |
| **Risk** | In YAML a duplicate key is an error or silently overrides the first one. An empty `db:` could have removed the database service. |
| **Fix** | Remove the duplicates, then validate without starting anything: `docker compose config --quiet && echo OK`. |
