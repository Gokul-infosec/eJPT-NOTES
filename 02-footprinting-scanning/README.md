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
A `/16` = 65,536 hosts; the job is finding the few that are live. **Never rely on one probe type** — ICMP misses Windows (firewall blocks it by default), so combine methods.
```bash
sudo nmap -sn <NETWORK>                       # ping sweep (sends ICMP echo + TCP SYN:443 + TCP ACK:80 + ICMP timestamp; ARP on local Ethernet)
sudo nmap -sn --send-ip <NETWORK>             # force IP probes instead of ARP on local net
sudo nmap -sn -O <NETWORK>                    # + OS hints
sudo nmap -sn -v <NETWORK>                    # verbose (see WHY: "received syn-ack" / "received reset" / "host seems down")
sudo netdiscover -i eth0 -r <NETWORK>         # ARP-based (local segment, shows MAC vendor)
# native ping / fping
ping -c 5 <TARGET_IP>                         # Linux  (-n 5 on Windows)
fping -a -g <NETWORK> 2>/dev/null             # live hosts only  (-S <ip> to spoof source)
fping -a -g <NETWORK> > active_hosts.txt 2>/dev/null
traceroute <DOMAIN>                           # path  (tracert on Windows)
```

### Probe types (combine for coverage)
```bash
nmap -sn -PS22,80,443,3389 <NETWORK>          # TCP SYN ping (best) — SYN-ACK or RST = alive
nmap -sn -PA80 <NETWORK>                      # TCP ACK ping — RST = alive (often FW-blocked)
nmap -sn -PU53,137,161 <NETWORK>              # UDP ping (DNS/NetBIOS/SNMP) — for TCP/ICMP-filtered hosts
nmap -sn -PE <NETWORK>                        # ICMP echo only
# recommended combined sweep:
sudo nmap -sn -v -T4 -PS21,22,25,80,443,445,3389,8080 -PU53,137,161 <NETWORK>
```

### Target specification
```bash
nmap <TARGET_IP>                              # single
nmap 192.168.1.100 192.168.2.50 10.1.1.1      # multiple (space-separated)
nmap 192.168.1.1-100                          # range
nmap 192.168.1.0/24                           # CIDR  (/24=256, /16=65536)
nmap 192.168.1.1 192.168.1.50-100 10.0.0.0/24 # combination
nmap -iL targets.txt                          # from file (one IP/range/CIDR per line, # comments)
nmap --exclude 192.168.1.1 <NETWORK>
```

### Stealth / output
```bash
sudo nmap -sn --max-rate 10 <NETWORK>         # slow = quieter
sudo nmap -sn -n --randomize-hosts <NETWORK>  # no DNS, random order
sudo nmap -sn -oA host_discovery <NETWORK>
sudo nmap -sn <NETWORK> | grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' > targets.txt   # build target list
```
`Host is up` = responded · `MAC Address` = NIC vendor. If `ping` gets nothing but `nmap -Pn` says "Host is up", it's a Windows box blocking ICMP.

## Port scanning — scan types
```bash
nmap <TARGET_IP>                              # default: TCP SYN if root, else connect; top 1000
nmap -Pn <TARGET_IP>                          # skip host discovery (Windows often blocks ICMP)
nmap -F <TARGET_IP>                           # fast (top 100)
nmap -p 80,443,22 <TARGET_IP>                 # specific ports
nmap -p 1-100,1000-2000 <TARGET_IP>           # multiple ranges
nmap -p- <TARGET_IP>                          # all 65535
nmap --top-ports 1000 <TARGET_IP>
```
- **SYN scan `-sS`** (default w/ root, "half-open"/stealth): sends SYN → SYN-ACK = open, RST = closed, no reply = filtered → Nmap replies RST (never completes handshake). Faster, quieter, fewer logs.
- **TCP Connect `-sT`** (default without root): completes the full handshake → logged on the target. No privileges needed, reliable.
- **UDP `-sU`**: for DNS/SNMP/DHCP etc. Slow.
```bash
nmap -sS -p 80,445,3389 <TARGET_IP>
nmap -sT -T4 -p- <TARGET_IP>
nmap -sU -p 53,137,138,139 <TARGET_IP>
nmap -sS -sU -p- -T4 -A -v <TARGET_IP>        # TCP + UDP combined, aggressive
```

## Service & OS detection
```bash
nmap -sV <TARGET_IP>                          # service/version
nmap -sV --version-intensity 0-9 <TARGET_IP>  # probe depth (5 default, 9 max, 3 light/fast)
nmap -O <TARGET_IP>                            # OS fingerprint
nmap -O --osscan-guess <TARGET_IP>             # guess when no exact match (e.g. "Linux 3.2-4.9 (92%)")
nmap -O --max-os-tries 1 <TARGET_IP>           # faster (5 = thorough)
nmap -sS -sV -O -T4 -p- <TARGET_IP>            # full recon on one host
```
Workflow: host discovery → port scan → `-sV` → `-O`. Services on **non-standard ports** (e.g. MongoDB on 6421) are why you scan `-p-`, not just top-1000.

## NSE (Nmap Scripting Engine)
Lua scripts in `/usr/share/nmap/scripts/` (`.nse`). Categories: **default**, **safe**, **discovery**, **auth**, **brute**, **vuln**, **exploit** (brute/exploit are intrusive — scope + approval).
```bash
nmap -sC <TARGET_IP>                          # default (safe) scripts
nmap -A <TARGET_IP>                           # aggressive = -sV + -sC + -O + --traceroute
# find scripts
ls /usr/share/nmap/scripts/ | grep http
nmap --script-help=ftp-anon                   # what a script does
# run scripts
nmap --script safe <TARGET_IP>
nmap --script vuln <TARGET_IP>                # known-CVE checks
nmap --script "safe and discovery" <TARGET_IP>
nmap --script=mongodb-info,memcached-info -p 6421,4128 <TARGET_IP>   # specific (comma list)
nmap --script=ftp* -p 21 <TARGET_IP>          # wildcard category
nmap --script=http-enum,http-title -p 80,443,8080 <TARGET_IP>
# if a script won't auto-run, pin the port; debug with --script-trace
nmap -sV --script=<name> -p <port> --script-trace <TARGET_IP>
```
NSE often leaks the OS/kernel/hostname and misconfigs (e.g. `Authentication: not required` on memcached/mongodb) — feed that straight into vulnerability research.

## Timing & performance
| Template | Name | Spacing | ~1000 ports | Use |
|---|---|---|---|---|
| `-T0` | Paranoid | 5 min | ~83 h | extreme IDS evasion |
| `-T1` | Sneaky | 15 s | ~4 h | high stealth |
| `-T2` | Polite | 0.4 s | ~7 min | low network impact |
| `-T3` | Normal | default | ~1 min | general |
| `-T4` | Aggressive | fast parallel | ~30 s | reliable/internal nets |
| `-T5` | Insane | max | ~15 s | fast nets, some loss |
```bash
nmap --host-timeout 30s <NETWORK>             # skip hosts that hang (big nets)
nmap --scan-delay 5s <TARGET_IP>              # space probes (stealth)
nmap --max-retries 1 <TARGET_IP>              # fewer retries = faster (5 for flaky nets)
nmap -n <TARGET_IP>                           # skip DNS resolution
nmap --max-rtt-timeout 100ms <TARGET_IP>
nmap --min-rate 50 --max-rate 100 <TARGET_IP>              # slow network
nmap --min-rate 500 --max-rate 1000 <TARGET_IP>           # fast network
nmap -T4 --min-hostgroup 64 --min-parallelism 10 <NETWORK>  # parallel hosts
# recommended defaults:
nmap -T4 -sS -sV --host-timeout 30s <TARGET_IP>           # internal
nmap -T2 -sS --scan-delay 2s --max-retries 2 <TARGET_IP>  # external
```

## Firewall detection & IDS evasion (scope + approval only)
```bash
# ACK scan — detect stateful firewall: unfiltered=no FW, filtered=stateful FW, closed=reachable no service
nmap -sA <TARGET_IP>
nmap -sA -p 80,445,3389 <TARGET_IP>
nmap -sA -T4 --top-ports 50 <TARGET_IP>
# Windows firewall tell: 135/139/445 come back filtered under a SYN scan
nmap -sS --top-ports 100 <TARGET_IP>
```
```bash
# evasion knobs
nmap -sS -f <TARGET_IP>                       # fragment packets
nmap -sS --mtu 8 <TARGET_IP>                  # custom fragment size (multiple of 8)
nmap -sS --data-length 200 <TARGET_IP>        # pad with random bytes
nmap -sS -g 53 <TARGET_IP>                    # spoof source port (look like DNS; -g 80 = HTTP)
nmap -sS -D 10.0.0.1,ME,10.0.0.2 <TARGET_IP>  # decoys (ME = your real IP's slot); or -D RND:10
nmap -sS --ttl 128 <TARGET_IP>                # custom TTL
nmap -sS --ip-options "S 192.168.1.1" <TARGET_IP>
# layered stealth example:
nmap -sS -sV -f -D 10.10.23.1,10.10.23.2 -g 53 --data-length 128 --ttl 64 --scan-delay 500ms -p- <TARGET_IP>
```

## Output formats & Metasploit integration
```bash
nmap -oN out.txt <TARGET_IP>                  # normal (human/report)
nmap -oX scan.xml <TARGET_IP>                 # XML (tool/MSF import)
nmap -oG scan.gnmap <TARGET_IP>               # grepable (scripting)
nmap -oA scan <TARGET_IP>                     # all three at once
nmap -sS -sV --reason -v <TARGET_IP> -oN report.txt   # -vv more, --reason shows why a state
nmap -Pn -sV -T4 -A -p- -iL hosts.txt --open -oN ports.txt   # full sweep from host list
# parse
grep "open" scan.gnmap ; awk '/open/ {print $2}' scan.gnmap
xsltproc scan.xml -o scan.html ; xmllint scan.xml ; ndiff baseline.xml current.xml
```
```text
# import into Metasploit
service postgresql start ; msfconsole
workspace -a <engagement>
db_import /path/scan.xml ; hosts ; services
db_nmap -sS -sV -O <TARGET_IP>                # scan from inside MSF (auto-saved to DB)
db_export -f xml backup.xml
hosts -c address,os_name ; services -c port,proto,name,info
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
