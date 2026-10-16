# Ubuntu-Web Configuration (Phase 5)

| Item       | Value                                           |
|------------|-------------------------------------------------|
| Hostname   | `ubuntu-web`                                    |
| Platform   | EVE-NG QEMU node, Ubuntu 22.04 cloud image      |
| Disk       | 20 GiB (grown from the 2.2 GiB cloud image)     |
| Network    | `ens3` = 192.168.99.10/24, VLAN 99 (DMZ)        |
| Gateway    | 192.168.99.254 (R1 `e0/1.99`)                   |
| DNS        | none (on purpose, see below)                    |
| Services   | Docker Engine + Docker Compose, DVWA + MariaDB  |

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

- MariaDB (`3306/tcp`) is **not** published to the network. Only the DVWA container can reach it.
- Both containers use `restart: unless-stopped`, so they come back after a reboot.
- Default DVWA security level is `impossible`. It will be lowered to `low` in the attack phase.
