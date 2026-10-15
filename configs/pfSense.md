# pfSense Configuration (Phase 4)

> This is a summary of the running configuration. The full `config.xml` backup is **not** stored in this repository, because it contains password hashes.

## Interfaces

| Interface | Network           | Address           | Upstream gateway |
|-----------|-------------------|-------------------|------------------|
| WAN       | Fake Internet     | 10.20.20.1/24     | None             |
| LAN       | Transit to R1     | 10.10.10.253/30   | None             |

- **WAN:** *Block private networks* is **unchecked**, because the fake internet uses an RFC1918 range (10.20.20.0/24).
- **LAN:** no gateway is set on the interface. If one were set, pfSense would treat LAN as an upstream (WAN-like) interface.

## Routing

| Destination       | Gateway                | Description                 |
|-------------------|------------------------|-----------------------------|
| 192.168.0.0/16    | R1_GW (10.10.10.254)   | Return path to all VLANs    |

- **Default gateway IPv4:** `None`. Setting it to R1 would create a routing loop (R1's default route points back to pfSense).

## Aliases

| Name           | Type     | Value              |
|----------------|----------|--------------------|
| DC01           | Host     | 192.168.100.100    |
| INTERNAL_NETS  | Network  | 192.168.0.0/16     |
| VLAN10_USERS   | Network  | 192.168.10.0/24    |
| VLAN30_MGMT    | Network  | 192.168.30.0/24    |
| VLAN99_DMZ     | Network  | 192.168.99.0/24    |
| WEB_PORTS      | Port     | 80, 443            |

## Firewall Rules: LAN (first match wins)

| # | Action | Source        | Protocol | Destination | Port       | Log | Description                          |
|---|--------|---------------|----------|-------------|------------|-----|--------------------------------------|
| - | Pass   | any           | TCP      | LAN address | 443, 80    | -   | Anti-Lockout Rule (system)           |
| 1 | Block  | VLAN99_DMZ    | any      | any         | any        | Yes | DMZ may not initiate outbound        |
| 2 | Pass   | DC01          | TCP/UDP  | any         | 53         | -   | DC01 DNS forwarding                  |
| 3 | Pass   | VLAN10_USERS  | TCP      | any         | WEB_PORTS  | -   | Users web browsing                   |
| 4 | Pass   | VLAN30_MGMT   | TCP      | any         | 443        | -   | Wazuh updates (HTTPS)                |
| 5 | Pass   | INTERNAL_NETS | ICMP     | any         | echoreq    | -   | Ping for troubleshooting (lab only)  |
| - | ~~Pass~~ | ~~LAN net~~ | any      | any         | any        | -   | Default allow LAN (IPv4+IPv6), **disabled** |
| - | Block  | any           | any      | any         | any        | Yes | Implicit default deny (tracker 1000000103) |

## Outbound NAT

- **Mode:** Hybrid Outbound NAT

| Interface | Source         | Translation  | Description                     |
|-----------|----------------|--------------|---------------------------------|
| WAN       | INTERNAL_NETS  | WAN address  | NAT internal VLANs to WAN address |

- **Reason:** automatic outbound NAT only creates rules for interfaces that have an upstream gateway. WAN has none, so the automatic list was empty.
