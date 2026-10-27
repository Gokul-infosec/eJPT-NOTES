# Social Engineering

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
"Hacking humans" — manipulate people into giving access/info, bypassing exploits entirely. Scope + explicit written approval only.

## The social engineer
Builds **rapport** (fast trust), or leans on **intimidation/fear/urgency** ("needs to be done by tomorrow!", impersonating a boss), and acts like they **own the place** (confidence = looking like they belong). Outgoing + confident, at least on the job.

## Core techniques
- **Pretexting** — stage a believable scenario so the target hands over info they shouldn't.
- **Emotional pulls** — create a situation where the victim feels forced to act.

## Physical techniques (in-person / phone)
- **Tailgating** — follow an employee through a secure door without a badge.
- **The IT guy** — show up asking to "check" their laptop (they assume IT knows best).
- **Occupied hands** — hands full (coffee) so a helpful employee badges/opens for you.
- **Shoulder surfing** — read creds/info over someone's shoulder.
- **Dumpster diving** — sift trash for cleartext creds, post-its, records.
- **Baiting** — USB dead-drops left for employees to plug into corporate PCs.

## Digital techniques
- **Phishing** — mass trick emails. **Spear-phishing** — targeted. **Whaling** — targets C-suite/board.
- **Vishing** — phishing by phone. **Smishing** — by SMS/messaging.
- Mechanisms: **Pharming** (link text says X, goes to Y), **Watering hole** (compromise a site the target trusts), **BEC** (Business Email Compromise — hijack a trusted internal/partner domain).

## Prevention
**Security awareness / user training** is the only real defense — the best firewalls/EDR don't matter if someone walks in or clicks. User awareness > technology here.

## Run phishing campaigns

## GoPhish
```bash
# Linux install (grab latest release link from github.com/gophish/gophish/releases)
cd /opt/
sudo wget https://github.com/gophish/gophish/releases/download/v0.12.1/gophish-v0.12.1-linux-64bit.zip
sudo unzip -d gophish gophish-v0.12.1-linux-64bit.zip
sudo chmod +x gophish/gophish
cd /opt/gophish && sudo ./gophish          # admin UI login printed on first run

# or run the demo in Docker
docker run -ti -p 3333:3333 --rm gophish/demo
```

Campaign pieces (built in the GoPhish UI): **Sending Profile** (SMTP) → **Landing Page** (clone the login) → **Email Template** → **Users & Groups** → **Campaign** (launch + track opens/clicks/submitted creds).

> Social engineering is the top real-world entry vector after misconfig. Only ever run against targets with signed authorization — phishing without written scope is illegal.
