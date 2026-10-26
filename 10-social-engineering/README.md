# Social Engineering

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Run phishing campaigns to harvest credentials / deliver payloads — GoPhish setup. Scope + explicit written approval only.

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
