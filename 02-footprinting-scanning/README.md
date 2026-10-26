# Footprinting & Scanning

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Actively map the target — live hosts, open ports, services, versions, OS — plus active DNS interrogation (zone transfers reveal far more than passive DNS). Placeholders: `<TARGET_IP>`, `<DOMAIN>`, `<NS>`, `<NETWORK>` (e.g. `192.168.2.0/24`).

## Active DNS & zone transfers
DNS record types: **A** (IPv4), **AAAA** (IPv6), **NS** (name servers), **MX** (mail), **CNAME** (alias), **TXT**, **SOA** (authority), **PTR** (reverse). A misconfigured **AXFR** dumps the whole zone → internal IPs, hidden subdomains, dev/staging hosts, SRV records.
```bash
# find the name servers first, then try AXFR against each
host -t NS <DOMAIN>
dig AXFR @<NS> <DOMAIN>                       # e.g. dig AXFR @nsztm1.digi.ninja zonetransfer.me
dnsenum <DOMAIN>                              # automated enum + AXFR attempt
dnsenum --dnsserver <NS> <DOMAIN>
fierce --domain <DOMAIN>                      # subdomain brute via DNS
fierce --domain <DOMAIN> --wordlist /path/wordlist.txt
# individual record lookups
host -t A|AAAA|MX|TXT|SOA|NS <DOMAIN>
```
Transfer **REFUSED** = properly secured; a successful transfer is itself a finding (exposes internal topology). Fix: restrict AXFR to authorized secondaries, DNSSEC, monitor for transfer attempts.

## Find my own network range
```bash
ip a ; ip addr show ; ifconfig ; hostname -I     # e.g. inet 192.168.2.190/24 → scan 192.168.2.0/24
```

## Host discovery
```bash
sudo nmap -sn <NETWORK>                       # ping sweep (host discovery only)
sudo nmap -sn -O <NETWORK>                    # + OS hints
sudo nmap -sn -v <NETWORK>                    # verbose
sudo nmap -sn -PE -PS21,22,23,25,80,443,3389 <NETWORK>   # combine ICMP + TCP SYN probes
sudo netdiscover -i eth0 -r <NETWORK>         # ARP-based (local segment, shows MAC vendor)
fping -I eth1 -g <NETWORK> -a 2>/dev/null     # fping sweep (live only, suppress unreachable)
traceroute <DOMAIN>                           # path  (tracert on Windows)
# stealth
sudo nmap -sn --max-rate 10 <NETWORK>         # slow = quieter
sudo nmap -sn --randomize-hosts <NETWORK>
# output + build target list for next phase
sudo nmap -sn -oA host_discovery <NETWORK>
sudo nmap -sn <NETWORK> | grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' > targets.txt
```
Read the output: `Host is up` = responded, `MAC Address` = NIC vendor, plus latency per host.

## Port scanning — scan types
```bash
nmap <TARGET_IP>                              # default: TCP SYN, top 1000
nmap -Pn <TARGET_IP>                          # skip host discovery (Windows often blocks ICMP)
nmap -F <TARGET_IP>                           # fast (top 100)
nmap -p 80,443,22 <TARGET_IP>                 # specific ports
nmap -p 1-1000 <TARGET_IP>                    # range
nmap -p- <TARGET_IP>                          # all 65535
nmap -sU <TARGET_IP>                          # UDP
nmap -sU -p 53,161,162 <TARGET_IP>            # UDP specific
```

## Detection & NSE
```bash
nmap -sV <TARGET_IP>                          # service/version
nmap -O <TARGET_IP>                           # OS
nmap -A <TARGET_IP>                           # aggressive (OS+ver+scripts+traceroute)
nmap -sC <TARGET_IP>                          # default scripts
# NSE categories / individual scripts
nmap --script safe <TARGET_IP>
nmap --script vuln <TARGET_IP>                # known-CVE checks
nmap --script "safe and discovery" <TARGET_IP>
nmap --script http-enum|smb-security-mode|ssh-auth-methods <TARGET_IP>
```

## Timing & performance
```bash
nmap -T0 … -T5 <TARGET_IP>                    # paranoid→insane (T3 default, T4 labs)
nmap --max-rtt-timeout 100ms <TARGET_IP>
nmap --min-rate 50 --max-rate 100 <TARGET_IP>     # slow network
nmap --min-rate 500 --max-rate 1000 <TARGET_IP>   # fast network
```

## Firewall / IDS evasion (scope + approval only)
```bash
nmap -T2 --max-parallelism 1 --scan-delay 5s <TARGET_IP>
nmap -f <TARGET_IP>                           # fragment packets
nmap -sS -f --data-length 16 <TARGET_IP>
nmap -D RND:10 <TARGET_IP>                    # 10 random decoys
```

## Output
```bash
nmap -oN out.txt <TARGET_IP>                  # normal  (-oX xml / -oG grepable)
nmap -oA scan <TARGET_IP>                     # all three formats
nmap -Pn -sV -T4 -A -p- -iL hosts.txt --open -oN ports.txt   # full sweep from host list
xsltproc scan.xml -o scan.html                # XML → HTML report
grep "open" scan.nmap | cut -d'/' -f1         # extract open ports
ndiff baseline.xml current.xml                # diff scans over time
```

## Real-world recipes
```bash
nmap -Pn -sS -sV -sC -O <TARGET_IP>                              # Windows (ICMP blocked → -Pn)
nmap -Pn -p 135,139,445,3389,5985,5986 -sV <TARGET_IP>          # common Windows services
nmap -Pn -p 80,443,8000,8080,8443 -sV -sC --script http* <TARGET_IP>   # web
nmap -Pn -p 443 --script ssl* <TARGET_IP>                        # TLS checks
```

## Port states & common ports
- **open** = service accepting · **closed** = reachable, nothing listening · **filtered** = firewall blocking (may be open) · **unfiltered** = reachable, state unknown.
- Essential ports: 21 FTP · 22 SSH · 23 Telnet · 25 SMTP · 53 DNS · 80 HTTP · 110 POP3 · 135 MSRPC · 139/445 SMB · 143 IMAP · 443 HTTPS · 993 IMAPS · 995 POP3S · 1433 MSSQL · 3306 MySQL · 3389 RDP · 5432 PostgreSQL · 5900 VNC.

> Flow: active DNS (NS → AXFR per server) → find my range → host discovery (`-sn`/netdiscover) → full TCP `-p-` → `-sV -sC -O` on open ports → targeted NSE → save `-oA` and build the host/port list for Enumeration. `filtered` usually = firewall; use `-Pn` when ICMP is blocked.
