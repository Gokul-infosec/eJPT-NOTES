# Web Enumeration & Scanning

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Fingerprint a web app and map its content/endpoints — HTTP methods, directories/files, server/tech, and scanners. Placeholders: `<TARGET_IP>`.

## Tools (install)
```bash
sudo apt install -y gobuster dirb nikto burpsuite sqlmap xsser wpscan hydra
```

## Service scan
```bash
nmap -sS -sV -p 80,443,3306 <TARGET_IP>
```

## HTTP methods (curl)
```bash
curl -I <TARGET_IP>                                   # headers
curl -X GET <TARGET_IP>
curl -X OPTIONS <TARGET_IP> -v                        # allowed methods
curl -X POST <TARGET_IP>/login.php -d "name=john&password=password" -v
curl -X PUT <TARGET_IP>
curl <TARGET_IP>/uploads/ --upload-file hello.txt      # PUT upload
curl -X DELETE <TARGET_IP>/uploads/hello.txt -v
```

## Content discovery
```bash
# dirb
dirb http://<TARGET_IP>
# gobuster
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -b 403,404
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -b 403,404 -x .php,.xml,.txt -r
# ffuf
ffuf -w wordlist.txt -u http://<TARGET_IP>/FUZZ
ffuf -w wordlist.txt -u http://<TARGET_IP>/FUZZ -e .aspx,.php,.txt,.html
ffuf -w /usr/share/wordlists/dirb/small.txt -u http://<TARGET_IP>/FUZZ -mc 200,301
ffuf -w wordlist.txt -u http://<TARGET_IP>/FUZZ -t 64 -maxtime 60
```

## Scanners
```bash
nikto -h http://<TARGET_IP> -o niktoscan.txt
nikto -h "http://<TARGET_IP>/index.php?page=x.php" -Tuning 5 -o nikto.html -Format htm
# WordPress
wpscan --url http://<TARGET_IP> --enumerate u
wpscan --url http://<TARGET_IP> -e vp --plugins-detection mixed --api-token <API_TOKEN>
wpscan --url http://<TARGET_IP> -U admin -P /usr/share/wordlists/rockyou.txt
```

> Flow: scan ports → probe HTTP methods (`OPTIONS`/`PUT`/`DELETE` with curl) → dir/file brute (gobuster/ffuf, filter 403/404, add extensions) → nikto + CMS scanner → feed findings into Web Attacks.
