# Ubuntu-Web Configuration (Phase 5, updated in Phase 6)

| Item       | Value                                           |
|------------|-------------------------------------------------|
| Hostname   | `ubuntu-web`                                    |
| Platform   | EVE-NG QEMU node, Ubuntu 22.04 cloud image      |
| Disk       | 20 GiB (grown from the 2.2 GiB cloud image)     |
| Network    | `ens3` = 192.168.99.10/24, VLAN 99 (DMZ)        |
| Gateway    | 192.168.99.254 (R1 `e0/1.99`)                   |
| DNS        | none (on purpose, see below)                    |
| Services   | Docker Engine + Docker Compose, DVWA + MariaDB  |
| Time       | UTC, synced from R1 (`192.168.99.254`)          |
| Monitoring | Wazuh agent 4.14.9, ID 003, group `linux-dmz`   |

## Network: `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: false
      addresses:
        - 192.168.99.10/24
      routes:
        - to: default
          via: 192.168.99.254
```

- **No DHCP:** VLAN 99 has no DHCP scope. Servers get fixed IPs.
- **No DNS server:** a DMZ server should not query the internal Domain Controller.

## cloud-init overrides

The cloud image is managed by cloud-init, which can rewrite the network config and hostname at boot. Both are disabled:

```
/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg   ->  network: {config: disabled}
/etc/cloud/cloud.cfg.d/99-preserve-hostname.cfg         ->  preserve_hostname: true
```

## Docker

- Installed from Docker's **official apt repository** (not `docker.io` from Ubuntu), so `docker compose` v2 is available.
- The `ubuntu` user is **not** in the `docker` group. Membership in that group is equivalent to root (MITRE ATT&CK T1611), so all Docker commands use `sudo`.

## DVWA (`~/DVWA/compose.yml`)

Source: [github.com/digininja/DVWA](https://github.com/digininja/DVWA) (official Docker Compose setup).

| Setting       | Default                | Changed to             | Why |
|---------------|------------------------|------------------------|-----|
| `ports`       | `127.0.0.1:4280:80`    | `192.168.99.10:80:80`  | Reachable from the lab, but bound **only** to the DMZ interface, never `0.0.0.0` |
| `pull_policy` | `always`               | `missing`              | The server has no internet. `always` would fail on every start |
| `volumes` (dvwa) | none                | `/var/log/dvwa:/var/log/apache2` | Phase 6: Apache logs become real files on the host, so the Wazuh agent can read them (see below) |

- MariaDB (`3306/tcp`) is **not** published to the network. Only the DVWA container can reach it.
- Both containers use `restart: unless-stopped`, so they come back after a reboot.
- Default DVWA security level is `impossible`. It will be lowered to `low` in the attack phase.

## Time (Phase 6): `/etc/systemd/timesyncd.conf.d/lab-ntp.conf`

```ini
[Time]
NTP=192.168.99.254
```

- A drop-in file instead of editing `timesyncd.conf`: package upgrades never overwrite it.
- Time zone: UTC (`timedatectl`). R1's ACL `DMZ-IN` allows only `udp/123` to `192.168.99.254` for this.

## Wazuh agent (Phase 6)

- Agent 4.14.9 installed from the `.deb` in the manager's `agent-kit` (SHA256 verified), enrolled to `192.168.30.10`, group `linux-dmz`.
- R1's ACL `DMZ-IN` allows only TCP 1514 (events) and 1515 (enrollment) from `192.168.99.10` to `192.168.30.10`. The DMZ still cannot reach anything else inside.
- What the agent collects is defined centrally on the manager (`shared/linux-dmz/agent.conf`, see [wazuh.md](wazuh.md)).

## DVWA logs (Phase 6)

The DVWA image is based on `php:apache`, where Apache's log files are **symlinks** to the container's stdout/stderr:

```
$ sudo docker exec dvwa-dvwa-1 ls -l /var/log/apache2
access.log -> /dev/stdout
error.log -> /dev/stderr
other_vhosts_access.log -> /dev/stdout
```

That is why `docker logs` showed the requests in Phase 5, but there was no file for an agent to read.

| Option | Pros | Cons |
|--------|------|------|
| Read `/var/lib/docker/containers/<id>/<id>-json.log` | No change to DVWA | Path changes every time the container is recreated. Each line is wrapped in Docker JSON |
| Docker `syslog` logging driver | Central | Different format, needs another port through the ACL, `docker logs` stops working |
| **Bind mount `/var/log/apache2` to the host** (chosen) | Plain Apache "combined" format. Wazuh decodes it out of the box (`web-accesslog`, rules 31100+: SQLi, XSS, scanners) | `docker logs` no longer shows access logs. Needs log rotation on the host |

The bind mount hides the symlinks from the image, so Apache creates regular files:

```yaml
services:
  dvwa:
    ...
    restart: unless-stopped
    volumes:
      - /var/log/dvwa:/var/log/apache2
```

```
$ ls -l /var/log/dvwa/
-rw-r--r-- 1 root root   0 Oct 20 14:05 access.log
-rw-r--r-- 1 root root 434 Oct 20 14:05 error.log
-rw-r--r-- 1 root root   0 Oct 20 14:05 other_vhosts_access.log
```

### Log rotation: `/etc/logrotate.d/dvwa`

```
/var/log/dvwa/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

- `copytruncate`: Apache runs inside the container and keeps the file open, and there is no simple way to signal it to reopen a new file. logrotate copies the file, then empties it. A few lines written at that exact moment can be lost. Acceptable in the lab.
- Without rotation, an aggressive scan from Kali could fill the 20 GB disk.
