# Phase 5: DMZ Verification

## 1. Ubuntu-Web network (after removing the temporary internet link)

```
$ ip -br a
lo      UNKNOWN  127.0.0.1/8 ::1/128
ens3    UP       192.168.99.10/24 fe80::250:ff:fe00:500/64
...

$ ip route
default via 192.168.99.254 dev ens3 proto static
192.168.99.0/24 dev ens3 proto kernel scope link src 192.168.99.10

$ ping -c 3 8.8.8.8
3 packets transmitted, 0 received, 100% packet loss
```

Only one way out: through R1. No internet.

## 2. DVWA is running and bound to the DMZ interface only

```
$ sudo docker ps
IMAGE                           STATUS        PORTS                      NAMES
ghcr.io/digininja/dvwa:latest   Up 6 minutes  192.168.99.10:80->80/tcp   dvwa-dvwa-1
mariadb:10                      Up 6 minutes  3306/tcp                   dvwa-db-1

$ sudo ss -tlnp | grep ':80'
LISTEN 0  4096  192.168.99.10:80  0.0.0.0:*  users:(("docker-proxy",...))

$ curl -I http://192.168.99.10
HTTP/1.1 302 Found
Server: Apache/2.4.68 (Debian)
Location: login.php
```

- Containers started on their own after a reboot (`restart: unless-stopped`).
- MariaDB (3306) has no host binding: it is not reachable from the network.
- `302 -> login.php` is the normal response for an unauthenticated user.

## 3. Before the ACL: the gap

```
$ ping -c 3 192.168.100.100          # Ubuntu-Web -> DC01
64 bytes from 192.168.100.100: icmp_seq=1 ttl=127 time=1.14 ms
3 packets transmitted, 3 received, 0% packet loss
```

TTL=127 = Windows (128) minus one hop (R1). The DMZ could reach the Domain Controller.

## 4. After the ACL: all requirements

| # | From       | Test                          | Expected | Result |
|---|------------|-------------------------------|----------|--------|
| 1 | Ubuntu-Web | ping DC01 (192.168.100.100)   | blocked  | `From 192.168.99.254 icmp_seq=1 Packet filtered` |
| 2 | Ubuntu-Web | `curl http://10.10.10.253`    | blocked  | `Failed to connect ... No route to host` |
| 3 | WIN10      | browse http://192.168.99.10   | works    | DVWA main page loads |
| 4 | WIN10      | ping 192.168.99.10            | works    | `Reply from 192.168.99.10 ... TTL=63` |
| 5 | Ubuntu-Web | ping Kali (10.20.20.254)      | blocked by **pfSense** | 100% loss + pfSense log (below) |

**R1 counters after the tests**

```
R1#show access-lists DMZ-IN
Extended IP access list DMZ-IN
    10 permit tcp 192.168.99.0 0.0.0.255 192.168.0.0 0.0.255.255 established (13 matches)
    20 permit icmp 192.168.99.0 0.0.0.255 192.168.0.0 0.0.255.255 echo-reply (7 matches)
    30 deny ip 192.168.99.0 0.0.0.255 192.168.0.0 0.0.255.255 log (3 matches)
    40 deny ip 192.168.99.0 0.0.0.255 10.10.10.252 0.0.0.3 log (56 matches)
    50 permit ip 192.168.99.0 0.0.0.255 any (3 matches)
    60 deny ip any any log
```

**R1 log (selection)**

```
%SEC-6-IPACCESSLOGDP: list DMZ-IN denied icmp 192.168.99.10 -> 192.168.100.100 (8/0), 1 packet
%SEC-6-IPACCESSLOGP:  list DMZ-IN denied tcp 192.168.99.10(34810) -> 10.10.10.253(80), 1 packet
```

**pfSense log (test 5)**

```
Oct 16 18:37:00  LAN  DMZ may not initiate outbound (1791981600)  192.168.99.10 -> 10.20.20.254  ICMP
Oct 16 18:37:01  LAN  DMZ may not initiate outbound (1791981600)  192.168.99.10 -> 10.20.20.254  ICMP
Oct 16 18:37:02  LAN  DMZ may not initiate outbound (1791981600)  192.168.99.10 -> 10.20.20.254  ICMP
```

**What this proves**

- Test 5 is visible in **both** devices: 3 pings = 3 matches on R1 line 50 = 3 lines in the pfSense log. R1 lets internet-bound traffic through, and pfSense blocks it. That is defense in depth.
- Line 60 has 0 matches: no spoofed traffic.
- R1 logs the first packet, then a summary every 5 minutes (`1 packet`, later `10 packets`). Analysts see counts, not every packet.

## 5. Mini investigation: unexpected denies to 10.10.10.253

Right after the ACL was applied, R1 logged denies that were not part of the tests:

```
denied tcp  192.168.99.10(22) -> 10.10.10.253(36763), 1 packet
denied icmp 192.168.99.10     -> 10.10.10.253 (0/0),  1 packet
denied tcp  192.168.99.10(22) -> 10.10.10.253(36763), 10 packets
```

| Step | Finding |
|------|---------|
| Read the log | Source port 22 = Ubuntu-Web was the SSH **server**. These were **replies** to a session opened from 10.10.10.253. ICMP 0/0 = echo-reply to someone pinging it. |
| Hypothesis | An admin session from the pfSense side, cut by line 40 when the ACL went live. |
| Verify on the endpoint | `last -i` on Ubuntu-Web: `ubuntu pts/0 10.10.10.253 Fri Oct 16 13:01 - 13:18 (00:17)` |
| Root cause | The admin (me) used SSH from the pfSense shell to paste commands into the server. |
| Verdict | Benign. The ACL worked as designed: the DMZ may not talk to the transit network. |

**Lesson about time:** the same event (the session being cut) appears as `13:18` on Ubuntu-Web and `15:18` on R1, while pfSense shows local time (`18:xx`). Correlating sources required manual offset math. This will be fixed with NTP before the SIEM phase.
