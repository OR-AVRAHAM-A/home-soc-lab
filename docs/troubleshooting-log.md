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
