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

## WAF detection (WAFW00F)
Fingerprints the WAF/CDN (Cloudflare, Sucuri, Imperva, AWS WAF, ModSecurity, Akamai…) from HTTP responses — passive, normal requests.
```bash
wafw00f -l                                   # list detectable WAFs
wafw00f <DOMAIN>                             # single target  (or https://<DOMAIN>)
wafw00f -a <DOMAIN>                          # test for ALL WAFs (don't stop at first)
wafw00f -v <DOMAIN>                          # verbose
wafw00f -i domains.txt -o batch_results.txt  # batch from file → output
wafw00f http://<DOMAIN>:8080                 # specific port
# cross-validate
whatweb <DOMAIN> | grep -i waf ; nmap --script http-waf-detect <DOMAIN>
```
Output: "is behind Cloudflare WAF" (detected) / "does not seem to be behind a WAF" / "seems to be behind a WAF — could not identify". Custom/proprietary WAFs may not be detected. Identify the WAF → research its bypasses before active testing.

## Subdomain enumeration (Sublist3r + others)
Passive OSINT from search engines + public DBs — no direct target interaction.
```bash
sublist3r -d <DOMAIN>
sublist3r -d <DOMAIN> -e google,bing,yahoo,baidu,ask,netcraft,dnsdumpster,virustotal,threatcrowd
sublist3r -d <DOMAIN> -v -o subdomains.txt   # verbose + save
sublist3r -d <DOMAIN> -b -t 10               # bruteforce + threads
sublist3r -d <DOMAIN> -p 80,443,8080         # verify on ports
# combine multiple sources + crt.sh, dedupe:
sublist3r -d <DOMAIN> -o sublist3r.txt
amass enum -passive -d <DOMAIN> -o amass.txt
assetfinder --subs-only <DOMAIN> > assetfinder.txt
curl -s "https://crt.sh/?q=%25.<DOMAIN>&output=json" | jq -r '.[].name_value' | sort -u > cert.txt
cat *.txt | sort -u > all_subdomains.txt
# validate which resolve:
while read s; do host "$s" &>/dev/null && echo "$s"; done < all_subdomains.txt
```
Reduce threads (`-t 5`) + add delays if search engines CAPTCHA/block. Interesting patterns: `admin.* api.* mail.* dev.* staging.* vpn.* portal.*`.

## Email harvesting (theHarvester)
```bash
theHarvester -d <DOMAIN> -l 500 -b all               # emails + subdomains, all sources, 500 results
theHarvester -d <DOMAIN> -b google,linkedin,bing
theHarvester -d <DOMAIN> -l 1000 -b all -v -f out.txt   # verbose + save (html/xml)
theHarvester -d <DOMAIN> -b linkedin -l 300          # employee discovery
theHarvester -d <DOMAIN> -b crtsh                    # cert transparency
theHarvester -d <DOMAIN> -b dnsdumpster -l 200
# sources: google bing linkedin twitter yahoo dnsdumpster crtsh virustotal threatcrowd anubis
# API keys (shodan/securitytrails/github) → /etc/theHarvester/api-keys.yaml
# extract just emails:
theHarvester -d <DOMAIN> -b all -l 500 | grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}"
```

## Other DNS/subdomain tools
```bash
dnsenum <HOST> ; fierce --domain <HOST>
```

## Google dorking
```text
# targeting
site:<DOMAIN>                         site:*.<DOMAIN>               # domain / subdomains
inurl:admin site:<DOMAIN>             inurl:login site:<DOMAIN>
# file types
filetype:pdf site:<DOMAIN>            filetype:xlsx|doc|sql|bak|env|config site:<DOMAIN>
# title / content
intitle:"index of" site:<DOMAIN>      intitle:"login"|"admin" site:<DOMAIN>
intext:"password"|"confidential" site:<DOMAIN>
# config / creds
inurl:"wp-config.php" filetype:php    intitle:"index of" "wp-config.php.bak"
inurl:"auth_user_file.txt"            filetype:htpasswd           inurl:"password.txt"
filetype:sql "INSERT INTO" site:<DOMAIN>
# combine / exclude
site:<DOMAIN> (inurl:admin | inurl:login) (filetype:php | filetype:html)
site:<DOMAIN> intitle:"index of" -inurl:wp-content -inurl:public
cache:<DOMAIN>                        # Google cache (date-range operators deprecated)
```
- **GHDB** — curated dork database: exploit-db.com/google-hacking-database (juicy info, file containers, vuln files, server disclosures).
- **Wayback Machine** — historical snapshots: `https://web.archive.org/web/*/https://<DOMAIN>` (or a timestamp `.../web/20160101000000/https://<DOMAIN>`).
- Space out queries / rotate IP to avoid CAPTCHAs. Never download unauthorized data.

## Leaked credentials / breach databases
Check whether harvested emails appear in known breaches (passive, org password-hygiene assessment).
```bash
# Have I Been Pwned (haveibeenpwned.com) — free web lookup, or API:
curl -s "https://haveibeenpwned.com/api/v3/breachedaccount/<email>" -H "hibp-api-key: <KEY>"
# password check via k-anonymity (send only first 5 SHA-1 chars):
curl -s "https://api.pwnedpasswords.com/range/<first5-sha1-chars>"
```
- Other sources: **DeHashed**, **BreachDirectory**, **Intelx.io** (more detail, often paid).
- Feeds into password-spray prep — only with explicit written authorization and in scope.
- Defensive takeaway for the report: force resets on breached accounts, enforce MFA, awareness training.

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
