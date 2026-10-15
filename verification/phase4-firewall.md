# Phase 4: Firewall Verification

## Test 1: End-to-end path + NAT (ICMP)

**WIN10 → Kali**

```
C:\> ping 10.20.20.254
Reply from 10.20.20.254: bytes=32 time=2ms TTL=62
```

**Kali: who is pinging me?**

```
$ sudo tcpdump -ni eth0 icmp
IP 10.20.20.1 > 10.20.20.254: ICMP echo request ...
IP 10.20.20.254 > 10.20.20.1: ICMP echo reply ...
```

**What this proves**

- TTL=62: Kali (Linux) starts at 64, minus 2 router hops (R1 + pfSense).
- The source is `10.20.20.1` (pfSense WAN), not `192.168.10.100`. Outbound NAT works, and the internal addressing is hidden.

## Test 2: Allowed traffic (HTTP, rule 3)

**Kali:** `sudo python3 -m http.server 80`
**WIN10 browser:** `http://10.20.20.254`

```
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.20.20.1 - - [14/Oct/2026 11:20:27] "GET / HTTP/1.1" 200 -
10.20.20.1 - - [14/Oct/2026 11:20:27] code 404, message File not found
10.20.20.1 - - [14/Oct/2026 11:20:27] "GET /favicon.ico HTTP/1.1" 404 -
```

**What this proves**

- Port 80 is allowed for VLAN 10 (`WEB_PORTS` alias).
- NAT works for TCP as well (source `10.20.20.1`).
- The 404 for `favicon.ico` is normal: the browser always requests an icon.

## Test 3: Blocked traffic (default deny)

**Kali:** `sudo python3 -m http.server 8080`
**WIN10 browser:** `http://10.20.20.254:8080` → `ERR_CONNECTION_TIMED_OUT`

**pfSense: Status → System Logs → Firewall**

```
Action  Time             If   Rule                                Source                Destination           Proto
BLOCK   Oct 14 14:46:02  LAN  Default deny rule IPv4 (1000000103) 192.168.10.100:58354  10.20.20.254:8080     TCP:S
BLOCK   Oct 14 14:46:10  LAN  Default deny rule IPv4 (1000000103) 192.168.10.100:58354  10.20.20.254:8080     TCP:S
BLOCK   Oct 14 14:47:15  LAN  Default deny rule IPv4 (1000000103) 192.168.10.100:58357  10.20.20.254:8080     TCP:S
```

**What this proves**

- No pass rule matches port 8080, so the implicit default deny drops it.
- `TCP:S`: only the SYN was seen. The TCP handshake never started.
- A timeout (not "refused") means the packet was **blocked** silently, not **rejected**.
- The same source port repeats (58354 at :02 and :10). These are SYN retransmissions of one connection, not separate connections.

**SOC note**

Many blocked SYNs from one source to **one** destination port = one failed connection with retries.
Blocked SYNs from one source to **many** ports or hosts = likely port scan (MITRE ATT&CK **T1046**).

## Test 4: Kali backdoor adapter is disconnected

```
$ ip -br link show eth1
eth1   DOWN   00:0c:29:a5:45:c6 <NO-CARRIER,BROADCAST,MULTICAST,UP>

$ ip route
default via 10.20.20.1 dev eth0 ...
```

**What this proves**

- `NO-CARRIER`: the virtual cable is unplugged in VMware.
- The only route out of Kali is through pfSense.

## Observation: time zones

The Kali log shows `11:20` (UTC), and pfSense shows `14:46` (Israel time, UTC+3).
In an investigation, a timeline built from mixed time zones gives the wrong order of events. This will be fixed (NTP + one time zone) before the SIEM phase.
