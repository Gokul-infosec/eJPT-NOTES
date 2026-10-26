# Footprinting & Scanning

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Map the live hosts, their open ports, services, versions and OS — the full nmap workflow plus network-discovery helpers. Placeholders: `<TARGET_IP>`, `<SUB>`.

## Network discovery
```bash
sudo arp-scan -I eth1 <TARGET_IP/NETWORK>
ping <TARGET_IP>
sudo nmap -sn <TARGET_IP/NETWORK>            # ping sweep
traceroute google.com                        # Linux  (tracert on Windows)

# fping sweep (live hosts only, suppress unreachable noise)
fping -I eth1 -g <TARGET_IP/NETWORK> -a
fping -I eth1 -g <TARGET_IP/NETWORK> -a 2>/dev/null
```

## Nmap — host/port scanning
```bash
nmap <TARGET_IP>
nmap -Pn <TARGET_IP>                         # skip host discovery
nmap -p- <TARGET_IP>                         # all 65535 ports
nmap -p 80 <TARGET_IP>                       # single port
nmap -p 80,445,3389,8080 <TARGET_IP>         # custom list
nmap -p1-2000 <TARGET_IP>                    # range
nmap -F <TARGET_IP> -v                        # fast (top 100)
nmap -sU <TARGET_IP>                          # UDP
```

## Nmap — detection & scripts
```bash
nmap -sV <TARGET_IP>                          # service/version
sudo nmap -sV -O <TARGET_IP>                  # + OS detection
nmap -sC <TARGET_IP>                          # default NSE scripts
nmap -Pn -F -sV -O -sC <TARGET_IP>            # common combo
nmap -Pn -F -A <TARGET_IP>                    # aggressive (OS+ver+scripts+traceroute)
nmap -Pn -F -T5 -sV -O -sC <TARGET_IP> -v     # timing T0 slow … T5 insane
```

## Nmap — open-ports sweep from a host list
```bash
nmap -Pn -sV -T4 -A -oN ports.txt -p- -iL hosts.txt --open
```

## Nmap — output
```bash
nmap -Pn -F -oN out.txt <TARGET_IP>           # normal
nmap -Pn -F -oX out.xml <TARGET_IP>           # XML
nmap -Pn -sV -sC -O -oA out <TARGET_IP>       # all three formats
nmap -A -oA out <TARGET_IP>
```

> Flow: discover live hosts (arp-scan/fping/`-sn`) → full TCP `-p-` → `-sV -sC -O` on the open ports → save with `-oA` for later import (e.g. into Metasploit `db_import`). `filtered` usually = firewall. Save everything.
