# Information Gathering

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Collect everything about the target before touching it hard — networking basics on my own box, then passive OSINT and light active recon (DNS, subdomains, WAF, host discovery). Placeholders: `<HOST>`, `<TARGET_IP>`, `<IP>`.

## Networking refresher (my own box)
```bash
# Routing
ip route            # Linux
route print         # Windows
netstat -r          # macOS / Linux

# IP / interfaces
ip a ; ip -br -c a  # Linux
ipconfig /all       # Windows
ifconfig            # macOS / Linux

# ARP table
ip neighbour        # Linux
arp -a              # Windows
arp                 # macOS / Linux

# Listening ports / sockets
netstat -tunp ; netstat -tulpn ; ss -tnl   # Linux
netstat -ano                                # Windows
netstat -p tcp -p udp ; lsof -n -i4TCP -i4UDP   # macOS / Linux
```

## Connect & probe
```bash
nc -v <HOST> 80
nc -zv <HOST> <PORT>                         # port scan with netcat
openssl s_client -connect <HOST>:<PORT>      # TLS service
openssl s_client -connect <HOST>:<PORT> -debug   # (or -state / -quiet)
```

## Passive recon
```bash
host <HOST>
whois <HOST> ; whois <IP>
whatweb <HOST>
dnsrecon -d <HOST>
wafw00f -l ; wafw00f <HOST> -a               # WAF detection
sublist3r -d <HOST>                          # subdomain enum
theHarvester -d <HOST> ; theHarvester -d <HOST> -b all   # emails/hosts/names
```

## Google dorks
```text
site:            site:*.sitename.com         intitle:        intitle:index of
inurl:           filetype:                   cache:
inurl:auth_user_file.txt   inurl:passwd.txt   inurl:wp-config.bak
```

## DNS
```bash
sudo nano /etc/hosts                         # map target FQDN → IP
dnsenum <HOST>                               # e.g. dnsenum zonetransfer.me
dig <HOST>
dig axfr @<DNS-server> <HOST>                # zone transfer
fierce --domain <HOST>
```

## Host discovery (nmap basics)
```bash
sudo nmap -sn <TARGET_IP/NETWORK>            # ping sweep
netdiscover -i eth1 -r <TARGET_IP/NETWORK>   # ARP scan
nmap -sn <TARGET_IP>/<SUB> > hosts.txt       # save live hosts
nmap -sn -T4 <TARGET_IP>/<SUB> -oG - | awk '/Up$/{print $2}'   # just the IPs
```

> Flow: networking sanity-check on my box → passive OSINT (whois/DNS/subdomains/WAF) → host discovery to build the live-host list → hand off to Footprinting & Scanning. Nothing intrusive here beyond light host discovery.
