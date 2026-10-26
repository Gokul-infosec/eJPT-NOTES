# Networking Fundamentals

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
The networking theory scanning depends on — OSI layers, IP, the TCP handshake and flags, ports, and TCP vs UDP. Understanding this is *why* scan types behave the way they do.

## OSI model (7 layers)
| # | Layer | Does | Examples |
|---|---|---|---|
| 7 | Application | network services to apps | HTTP, FTP, SSH, DNS |
| 6 | Presentation | format/encrypt/compress | SSL/TLS, JPEG, GIF |
| 5 | Session | manage connections/sessions | NetBIOS, RPC, APIs |
| 4 | Transport | end-to-end, flow control, segmentation | **TCP, UDP** |
| 3 | Network | logical addressing + routing | **IP**, ICMP, IPsec |
| 2 | Data Link | framing + MAC addressing on the local medium | Ethernet, switches |
| 1 | Physical | the physical medium | cables, fiber, hubs |

Each layer wraps the one above (encapsulation): e.g. App data → TLS → TCP → IP → Ethernet. Every packet = **Header** (metadata: src/dst, protocol structure) + **Payload** (the actual data).

## Network layer (L3)
- **IPv4** — 32-bit (`192.168.1.1`); **IPv6** — 128-bit hex (`2001:0db8:85a3::8a2e:0370:7334`).
- Key IP header fields: Version, Header Length, **TTL** (stops infinite loops), **Protocol** (TCP=6, UDP=17, ICMP=1), Source/Dest IP, Flags + Identification (fragmentation/reassembly).
- **ICMP** — error reporting + diagnostics (ping, traceroute). **DHCP** — auto-assigns IPs.
- Comms types: **unicast** (1:1), **broadcast** (1:all in subnet), **multicast** (1:many).
- Special addresses: `0.0.0.0` current network · `127.0.0.1` localhost · `192.168.0.0/16` private.

## Transport layer (L4)
**TCP** — connection-oriented, reliable, ordered, flow-controlled (slower). **UDP** — connectionless, no guarantees, 8-byte header, fast (DNS, DHCP, SNMP, VoIP, streaming, gaming).

| | TCP | UDP |
|---|---|---|
| Connection | 3-way handshake | connectionless |
| Reliability | guaranteed + ordered | none |
| Header | 20+ bytes | 8 bytes |
| Use | web, email, file transfer | streaming, gaming, DNS |

### TCP three-way handshake
```
Client  --- SYN ------>  Server      # client: let's connect (ISN)
Client  <-- SYN-ACK ---  Server      # server: ok + my ISN
Client  --- ACK ------>  Server      # client: confirmed → data flows
```
This is exactly what scanning exploits: SYN-ACK back = **open**, RST back = **closed**, no response = **filtered**.

### TCP flags
`URG` urgent · `ACK` acknowledgment · `PSH` push · `RST` reset/refuse · `SYN` start · `FIN` graceful close. Scan types set different flag combos to probe port state.

### Port ranges
- **Well-known 0–1023** (IANA): 21 FTP, 22 SSH, 25 SMTP, 80 HTTP, 443 HTTPS.
- **Registered 1024–49151**: 1433 MSSQL, 3306 MySQL, 3389 RDP, 5432 PostgreSQL.
- **Dynamic/private 49152–65535**: ephemeral client source ports. Total range 1–65535 (16-bit).

## Inspecting connections
```bash
netstat -atp                                  # Linux: active TCP + process  (netstat -ano on Windows)
ss -tunp                                       # modern replacement
watch -n 1 "netstat -atp | grep ESTABLISHED"   # live connections while generating traffic
```

## Wireshark / tcpdump (see the layers)
```bash
sudo wireshark -i eth1
# display filters:
icmp                                           # ping traffic (type 8 req / type 0 reply)
tcp                                            # TCP (watch SYN → SYN-ACK → ACK)
tcp.port == 80 and tcp.flags.syn == 1          # SYN packets to :80
udp.port == 53                                 # DNS
```

> This layer-by-layer picture is the foundation for Footprinting & Scanning: a SYN scan is just the first packet of the handshake, an ACK scan probes firewall state with the ACK flag, and UDP scanning works because DNS/SNMP/DHCP ride UDP. Know the handshake and the flags and every nmap scan type makes sense.
