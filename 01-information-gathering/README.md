# Information Gathering

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Collect everything about the target before touching it hard — networking basics on my own box, then passive OSINT (website footprinting, WHOIS, Netcraft, DNS) and light active recon (subdomains, WAF, host discovery). Passive = no direct/aggressive interaction with target infra → low detection risk, legal only on authorized targets. Placeholders: `<HOST>`, `<DOMAIN>`, `<TARGET_IP>`, `<IP>`.

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
openssl s_client -connect <HOST>:<PORT>      # TLS service  (-debug / -state / -quiet)
```

## Website footprinting
```bash
# DNS resolution — IPs + mail servers (multiple IPs often = behind Cloudflare/proxy)
host <DOMAIN>
#   target.com has address 192.0.2.1
#   target.com mail is handled by 10 mail.target.com

# robots.txt — dirs the site doesn't want indexed (leaks /wp-admin/, /private/, etc.)
curl https://<DOMAIN>/robots.txt
# sitemap.xml — page/post structure, author info, hidden categories
curl https://<DOMAIN>/sitemap.xml

# Technology profiling (server, CMS, versions, frameworks)
whatweb <DOMAIN>
#   HTTPServer[cloudflare], PHP[7.4.29], WordPress[5.8], jQuery, X-Powered-By[PHP/7.4.29]
# GUI: BuiltWith / Wappalyzer browser extensions

# Mirror the whole site for offline analysis
sudo apt-get install webhttrack ; webhttrack      # web UI at http://localhost:8080
```
Interpretation: `/wp-admin/` in robots.txt + WordPress plugins = WordPress; multiple IPs / Cloudflare headers = proxy/CDN hiding origin; author names in sitemap + email patterns in mirrored content = info disclosure for social engineering.

## WHOIS enumeration
```bash
whois <DOMAIN>          # registrar, creation/update/expiry dates, name servers, DNSSEC, contacts (if not redacted)
whois <IP>              # NetRange / CIDR / NetName / Organization / location (maps hosting / ASN)
```
- **Read for:** registrar + registration dates (domain age/renewal patterns), name servers (DNS infra), contact info when not privacy-redacted, DNSSEC status.
- **Privacy-protected** domains show `REDACTED FOR PRIVACY` / "Withheld for Privacy ehf" (Iceland) — owner hidden.
- **Historical WHOIS** (archived records): SecurityTrails, Whoxy, DomainTools.
- Use cases: social-engineering targets, infra mapping, typosquat detection, correlating multiple company domains (`company.com/.net/.org`).

## Netcraft (passive aggregation — netcraft.com → Internet Data Mining / site report)
Single web report aggregating: background (site title, first-seen date, rank, language), network (name servers, registrar, IP, TLD, DNSSEC), **SSL/TLS cert** (issuer, validity, Poodle/Heartbleed checks), hosting history (provider changes over time), web trackers/analytics, and full tech stack (proxy/WAF, server-side PHP/.NET, client-side JS, CMS, DB).
```bash
# complementary CLI checks for the same intel
whois <DOMAIN> ; host <DOMAIN> ; whatweb <DOMAIN>
sslscan <DOMAIN>                                   # detailed TLS/cipher analysis
```
CDN/WAF (Cloudflare/Sucuri) can obscure the real origin — note cert expiry (near-term renewal), outdated software versions, and third-party tracker dependencies as attack surface.

## DNS reconnaissance
Record types worth enumerating: **A** (IPv4), **AAAA** (IPv6), **MX** (mail), **NS** (name servers), **TXT** (verification/SPF), **CNAME** (aliases).
```bash
dnsrecon -d <DOMAIN>                 # general enumeration (A/AAAA/MX/NS/TXT)
dnsrecon -d <DOMAIN> -a              # include AXFR zone-transfer attempt
dnsrecon -d <DOMAIN> -t brt -D subdomains.txt    # brute-force subdomains from a wordlist
dnsrecon -d <DOMAIN> -t mx           # mail servers only
dig <DOMAIN> ; dig axfr @<DNS-server> <DOMAIN>   # zone transfer
# Certificate transparency → subdomains/hosts
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq .
# GUI: DNSDumpster (dnsdumpster.com) — graphical infra map, MX/TXT/NS, subdomains, PNG/XLSX export
sudo nano /etc/hosts                 # map target FQDN → IP for lab access
```
Indicators: `*.ns.cloudflare.com` NS = Cloudflare; `MX *.google.com` + `include:_spf.google.com` TXT = G Suite; interesting subdomains → `admin.*` `api.*` `dev.*` `staging.*` `mail.*`.

## Subdomains, WAF, emails
```bash
sublist3r -d <DOMAIN>                        # subdomain enum
theHarvester -d <DOMAIN> ; theHarvester -d <DOMAIN> -b all   # emails / hosts / names
wafw00f -l ; wafw00f <HOST> -a               # WAF detection
dnsenum <HOST> ; fierce --domain <HOST>
```

## Google dorks
```text
site:            site:*.sitename.com         intitle:        intitle:index of
inurl:           filetype:                   cache:
inurl:auth_user_file.txt   inurl:passwd.txt   inurl:wp-config.bak
```

## Host discovery (nmap basics)
```bash
sudo nmap -sn <TARGET_IP/NETWORK>            # ping sweep
netdiscover -i eth1 -r <TARGET_IP/NETWORK>   # ARP scan
nmap -sn <TARGET_IP>/<SUB> > hosts.txt       # save live hosts
nmap -sn -T4 <TARGET_IP>/<SUB> -oG - | awk '/Up$/{print $2}'   # just the IPs
```

> Automate the passive pass by wrapping `host` / `curl robots+sitemap` / `whatweb` / `whois` / `sslscan` / `dnsrecon` into one `recon.sh -d <domain> -o <dir>` script and loop it over a domain list.
>
> Flow: networking sanity-check → website footprint (host/robots/sitemap/whatweb/HTTrack) → WHOIS (domain + IP) → Netcraft + sslscan → DNS (dnsrecon + crt.sh + DNSDumpster) → subdomains/WAF/emails → host discovery → hand off to Footprinting & Scanning. All passive except light host discovery.
