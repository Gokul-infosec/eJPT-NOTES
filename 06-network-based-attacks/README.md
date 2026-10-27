# Network-Based Attacks

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Capture, filter and analyse network traffic, and abuse L2 to intercept it (ARP poisoning). Placeholders: `<FILE>.pcap`, `<IP>`, `<SSID>`, `<TARGET_IP>`, `<HOST_IP>`.

## Concept
Network-based attacks target **protocols** (ARP, DNS, DHCP…) not a specific OS. The main one here is **MITM** — the attacker sits as a proxy between two endpoints, done via **ARP poisoning**. **Promiscuous mode** (Wireshark/tshark) lets you read all packets on the segment.

## TShark — capture flags
```text
tshark -i <IFACE>            # live capture       -r <FILE>.pcap   # read a file
       -Y "<display filter>" # post-capture filter  -f "<capture filter>"  # BPF capture filter
       -w <out> -F pcapng    # write out (format)  -z <stat> -q     # statistics
       -c <N> --color -Tfields -e <field>           # count / colour / extract fields
# live examples (same filters work on -r too):
tshark -i eth1 -Y "http"
tshark -i eth1 -Y "http.request.method==GET && http.host==google.com" -Tfields -e ip.src
tshark -i eth1 -Y "http contains password"
```

## Wireshark display filters
```text
wireshark -i eth1
ip.addr == 10.10.10.9                    # by host
ip.dst  == 10.10.10.15                   # by destination
ip.src  == 10.10.16.33                   # by source
tcp.port == 25                           # by port
ip.addr == 10.10.14.22 and tcp.port == 8080
tcp.flags.syn == 1 and tcp.flags.ack == 0   # SYN only
eth.dst == ff:ff:ff:ff:ff:ff             # broadcast
```

## TShark (CLI capture/analysis)
```bash
tshark -D                                 # list interfaces
tshark -i eth1
tshark -r <FILE>.pcap
tshark -r <FILE>.pcap -c 100              # first 100 packets
tshark -r <FILE>.pcap -z io,phs -q        # protocol hierarchy
tshark -r <FILE>.pcap -Y 'http' | more    # HTTP traffic
tshark -r <FILE>.pcap -Y "http.request.method==GET"
tshark -r <FILE>.pcap -Y "http.request.method==GET" -Tfields -e frame.time -e ip.src -e http.request.full_uri
tshark -r <FILE>.pcap -Y "http contains password"
tshark -r <FILE>.pcap -Y "ip.src==<IP> && http" -Tfields -e http.user_agent    # OS/UA
tshark -r <FILE>.pcap -Y "ip contains amazon.in && ip.src==<IP>" -Tfields -e ip.src -e http.cookie
```

### Wi-Fi analysis (TShark)
```bash
tshark -r <FILE>.pcap -Y "wlan"
tshark -r <FILE>.pcap -Y "wlan.fc.type_subtype==0x000c"                        # deauth packets
tshark -r <FILE>.pcap -Y "wlan.fc.type_subtype==0x000c" -Tfields -e wlan.ra    # + devices
tshark -r <FILE>.pcap -Y "eapol"                                               # WPA handshake
tshark -r <FILE>.pcap -Y "wlan.fc.type_subtype==8" -Tfields -e wlan.ssid -e wlan.bssid
tshark -r <FILE>.pcap -Y "wlan.ssid==<SSID>" -Tfields -e wlan_radio.channel    # channel
tshark -r <FILE>.pcap -Y "wlan.ta==<DEVICE_MAC> && http" -Tfields -e http.user_agent
tshark -r <FILE>.pcap -Y "wlan.ssid==<SSID>" -Tfields -e wlan.bssid               # BSSID of an SSID
tshark -r <FILE>.pcap -Y "wlan.fixed.capabilities.privacy==0"                      # OPEN (unencrypted) wifi
# eapol = the 4-way WPA handshake (crack target / cafe-latte attack)
```

## ARP poisoning (MITM — arpspoof)
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward     # enable IP forwarding first
arpspoof -i eth1 -t <TARGET_IP> -r <HOST_IP>
```

> Flow: capture (`tshark -i`/Wireshark) → filter down to the protocol/host of interest → pull creds/cookies/UA/SSID with `-Tfields`. For active MITM, enable forwarding then `arpspoof` both directions — scope + approval only.
