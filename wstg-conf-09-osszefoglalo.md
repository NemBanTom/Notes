# WSTG-CONF-09: File Permission Tesztelése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **helytelen file permission-ök** (túl megengedő jogosultságok) **érzékeny fájlokhoz** (config, credentials, logs) **unauthorized access**-t tesznek lehetővé. **World-readable config file**-ok gyakran tartalmaznak **API key-ket**, **database credentials**-t, vagy más **sensitive data**-t.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM web application access control testing, hanem **filesystem-level permission** átvizsgálás - Linux/Windows file permission-ök ellenőrzése.

---

## Mi a cél?

**Rogue file permission-ök azonosítása:**
- World-readable sensitive files
- World-writable directories
- Executable files unauthorized user-ek számára
- Configuration files weak permission-nel

---

## Veszélyek (Threats)

### 1. **Sensitive Information Exposure:**

**Példa:**
```bash
# Config file with database credentials
-rw-r--r-- 1 www-data www-data config.php

# World-readable (r--) → VESZÉLY!
# Anyone can read database password
```

---

### 2. **Configuration Modification:**

**Példa:**
```bash
# Writable config
-rw-rw-rw- 1 root root app.conf

# World-writable (rw-) → CRITICAL!
# Attacker can modify configuration
```

---

### 3. **Code Execution:**

**Példa:**
```bash
# Executable by everyone
-rwxrwxrwx 1 root root admin_script.sh

# World-executable (x) → DANGER!
# Unauthorized users can run privileged script
```

---

### 4. **Log File Access:**

**Példa:**
```bash
# Sensitive logs
-rw-r--r-- 1 root root admin.log

# World-readable → INFO DISCLOSURE!
# Contains admin actions, IPs, session IDs
```

---

## Linux File Permissions Alapok

### Permission Structure:

```
-rwxr-xr-x 1 owner group size date filename
│││││││││
│││││││││
│││└┴┴┴┴┴─ Other permissions (r-x)
││└┴┴┴──── Group permissions (r-x)
│└┴┴────── Owner permissions (rwx)
└───────── File type (- = file, d = directory)
```

---

### Permission Codes:

| Char | Octal | Meaning |
|------|-------|---------|
| `r` | 4 | Read |
| `w` | 2 | Write |
| `x` | 1 | Execute |
| `-` | 0 | No permission |

---

### Common Permissions:

| Octal | Symbolic | Description |
|-------|----------|-------------|
| `644` | `-rw-r--r--` | Owner: read/write, Others: read |
| `600` | `-rw-------` | Owner: read/write, Others: none |
| `755` | `-rwxr-xr-x` | Owner: all, Others: read/execute |
| `700` | `-rwx------` | Owner: all, Others: none |
| `777` | `-rwxrwxrwx` | **DANGEROUS!** Everyone: all |
| `666` | `-rw-rw-rw-` | **DANGEROUS!** Everyone: read/write |

---

## Tesztelési Módszerek

### Tool #1: **ls -la**

#### Parancsok:
```bash
# List files with permissions
ls -la /var/www/html/

# Recursive
ls -laR /var/www/html/

# Only files
ls -la /var/www/html/*.php

# Sort by modification time
ls -lat /var/www/html/
```

---

**Output példa:**
```
-rw-r--r-- 1 www-data www-data 1234 Jan 18 config.php
-rw-rw-rw- 1 www-data www-data 5678 Jan 17 db.conf      ← CRITICAL!
-rwxr-xr-x 1 www-data www-data 9012 Jan 16 script.sh
drwxrwxrwx 2 www-data www-data 4096 Jan 15 uploads/     ← DANGEROUS!
```

**Problémák:**
- `db.conf` → `666` (world-writable!)
- `uploads/` → `777` (world-writable directory!)

---

### Tool #2: **find**

#### Parancsok:
```bash
# Find world-readable files
find /var/www -type f -perm -004

# Find world-writable files
find /var/www -type f -perm -002

# Find world-writable directories
find /var/www -type d -perm -002

# Find SUID files
find /var/www -type f -perm -4000

# Find files with 777 permissions
find /var/www -type f -perm 0777

# Find config files world-readable
find /var/www -name "*.conf" -perm -004
find /var/www -name "*.config" -perm -004
find /var/www -name "*.ini" -perm -004
```

---

**Output példa:**
```bash
find /var/www -type f -perm -002

/var/www/html/config.php
/var/www/html/database.conf
/var/www/html/.env
```

→ **World-writable files found!**

---

### Tool #3: **namei -l**

#### Parancs:
```bash
# Recursive permission listing
namei -l /var/www/html/config.php
```

**Output:**
```
f: /var/www/html/config.php
drwxr-xr-x root root /
drwxr-xr-x root root var
drwxr-xr-x root root www
drwxr-xr-x www-data www-data html
-rw-r--r-- www-data www-data config.php
```

**Mit mutat:**
- Teljes path permission-je
- Minden directory-ban milyen permission-ök vannak

---

### Tool #4: **stat**

#### Parancs:
```bash
# Detailed file info
stat /var/www/html/config.php
```

**Output:**
```
  File: config.php
  Size: 1234      	Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d	Inode: 123456      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/www-data)   Gid: ( 1000/www-data)
Access: 2024-01-18 10:30:00.000000000 +0100
Modify: 2024-01-18 10:25:00.000000000 +0100
Change: 2024-01-18 10:25:00.000000000 +0100
```

**Access:** `0644` → Owner: rw-, Group: r--, Other: r--

---

## Critical Files to Check

### 1. **Web Files:**

```bash
# Check web root
ls -la /var/www/html/

# Check for world-writable
find /var/www/html -type f -perm -002
```

**Expected:**
```
-rw-r--r-- index.php
-rw-r--r-- style.css
-rw-r--r-- script.js
```

**BAD:**
```
-rw-rw-rw- index.php    ← World-writable!
-rwxrwxrwx admin.php    ← World-executable!
```

---

### 2. **Configuration Files:**

```bash
# Find config files
find /var/www -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name ".env"

# Check permissions
ls -la /var/www/html/config.php
ls -la /etc/apache2/apache2.conf
ls -la /etc/nginx/nginx.conf
```

**Expected (secure):**
```
-rw------- config.php    (600)
-rw-r----- db.conf       (640)
```

**BAD:**
```
-rw-r--r-- config.php    (644) ← World-readable!
-rw-rw-rw- db.conf       (666) ← World-writable!
```

---

### 3. **Sensitive Files:**

```bash
# API keys, passwords
find /var/www -name ".env" -o -name "*key*" -o -name "*secret*"

# Check
ls -la /var/www/html/.env
ls -la /var/www/html/api_keys.txt
```

**Expected:**
```
-rw------- .env          (600)
-rw------- api_keys.txt  (600)
```

---

### 4. **Log Files:**

```bash
# Application logs
ls -la /var/log/apache2/
ls -la /var/log/nginx/
ls -la /var/www/html/logs/

# Find log files
find /var/www -name "*.log"
```

**Expected:**
```
-rw-r----- access.log    (640)
-rw------- admin.log     (600)
```

**BAD:**
```
-rw-r--r-- admin.log     (644) ← Anyone can read admin logs!
```

---

### 5. **Executables:**

```bash
# Scripts
find /var/www -name "*.sh" -o -name "*.py" -o -name "*.pl"

# Check
ls -la /var/www/scripts/
```

**Expected:**
```
-rwxr-x--- backup.sh     (750)
-rwx------ admin.sh      (700)
```

**BAD:**
```
-rwxrwxrwx admin.sh      (777) ← Anyone can execute!
```

---

### 6. **Database Files:**

```bash
# SQLite databases
find /var/www -name "*.db" -o -name "*.sqlite"

# Check
ls -la /var/www/html/database.db
```

**Expected:**
```
-rw------- database.db   (600)
```

---

### 7. **Upload Directories:**

```bash
# Check upload dirs
ls -lad /var/www/html/uploads/
ls -lad /var/www/html/files/
```

**Expected:**
```
drwxr-x--- uploads/      (750)
```

**BAD:**
```
drwxrwxrwx uploads/      (777) ← Anyone can write!
```

---

### 8. **Temp Files:**

```bash
# Temp directories
ls -lad /tmp/
ls -lad /var/tmp/
ls -lad /var/www/html/tmp/
```

**Expected:**
```
drwxrwxrwt tmp/          (1777 - sticky bit)
```

---

## Automated Scanning

### Script: **find_insecure_permissions.sh**

```bash
#!/bin/bash

echo "=== Scanning for insecure file permissions ==="
echo ""

# World-writable files
echo "[!] World-writable files:"
find /var/www -type f -perm -002 2>/dev/null

# World-writable directories
echo ""
echo "[!] World-writable directories:"
find /var/www -type d -perm -002 2>/dev/null

# Config files world-readable
echo ""
echo "[!] Config files world-readable:"
find /var/www \( -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name ".env" \) -perm -004 2>/dev/null

# SUID files
echo ""
echo "[!] SUID files:"
find /var/www -type f -perm -4000 2>/dev/null

# 777 permissions
echo ""
echo "[!] Files/dirs with 777 permissions:"
find /var/www -perm 0777 2>/dev/null

echo ""
echo "=== Scan complete ==="
```

---

## Windows File Permissions

### Tool #1: **icacls**

#### Parancsok:
```powershell
# Check file permissions
icacls C:\inetpub\wwwroot\config.xml

# Recursive
icacls C:\inetpub\wwwroot /T
```

**Output:**
```
C:\inetpub\wwwroot\config.xml Everyone:(R)
                                NT AUTHORITY\SYSTEM:(F)
                                BUILTIN\Administrators:(F)
```

**Problematic:**
- `Everyone:(R)` → Anyone can read!
- `Everyone:(W)` → Anyone can write!

---

### Tool #2: **AccessChk (Sysinternals)**

#### Parancsok:
```powershell
# Download from Sysinternals
# https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk

# Check directory
accesschk.exe -dqv C:\inetpub\wwwroot

# Check file
accesschk.exe -qv C:\inetpub\wwwroot\web.config
```

---

### Tool #3: **AccessEnum (Sysinternals)**

#### GUI Tool:
```
1. Download AccessEnum
2. Run as Administrator
3. Scan: C:\inetpub\wwwroot
4. Review permissions
```

---

## Permission Remediation

### 1. **Config Files (600 or 640):**

```bash
# Owner only
chmod 600 config.php
chmod 600 .env
chmod 600 database.conf

# Owner + group read
chmod 640 app.conf
chown www-data:www-data app.conf
```

---

### 2. **Web Files (644):**

```bash
# PHP/HTML files
chmod 644 index.php
chmod 644 *.html
chmod 644 *.css
chmod 644 *.js
```

---

### 3. **Executables (750 or 700):**

```bash
# Owner + group execute
chmod 750 backup.sh

# Owner only
chmod 700 admin.sh
```

---

### 4. **Directories (755 or 750):**

```bash
# Standard web directory
chmod 755 /var/www/html/

# Restricted directory
chmod 750 /var/www/html/admin/
```

---

### 5. **Upload Directory (750 with proper ownership):**

```bash
# Create upload dir
mkdir /var/www/html/uploads

# Set permissions
chmod 750 /var/www/html/uploads
chown www-data:www-data /var/www/html/uploads

# Files uploaded should be 640
# (web server can read, but not execute)
```

---

### 6. **Log Files (640 or 600):**

```bash
# Application logs
chmod 640 /var/log/app/application.log
chown www-data:adm /var/log/app/application.log

# Admin logs
chmod 600 /var/log/app/admin.log
```

---

## Bulk Permission Fix

### Recursive Permission Set:

```bash
# Set all files to 644
find /var/www/html -type f -exec chmod 644 {} \;

# Set all directories to 755
find /var/www/html -type d -exec chmod 755 {} \;

# Set all .sh files to 750
find /var/www/html -name "*.sh" -exec chmod 750 {} \;

# Set config files to 600
find /var/www/html \( -name "*.conf" -o -name ".env" \) -exec chmod 600 {} \;
```

---

## Security Best Practices

### 1. **Principle of Least Privilege:**
```
Only grant minimum necessary permissions
```

---

### 2. **Sensitive Files:**
```
- Config: 600 (owner only)
- Credentials: 600
- API keys: 600
- Private keys: 400 (read-only by owner)
```

---

### 3. **Web Files:**
```
- HTML/CSS/JS: 644
- PHP/Scripts: 640 or 644
- Executables: 750 (not in webroot!)
```

---

### 4. **Directories:**
```
- Web root: 755
- Upload: 750 (with proper ownership)
- Temp: 1777 (sticky bit)
```

---

### 5. **Ownership:**
```bash
# Web files owned by web server user
chown -R www-data:www-data /var/www/html/

# Config files owned by root
chown root:www-data /var/www/html/config.php
chmod 640 /var/www/html/config.php
```

---

## Gyakorlati Cheat Sheet

| Parancs | Cél |
|---------|-----|
| `ls -la /var/www/` | List permissions |
| `find /var/www -perm -002` | World-writable files |
| `find /var/www -perm 0777` | 777 permissions |
| `namei -l /path/to/file` | Recursive permissions |
| `stat /path/to/file` | Detailed file info |
| `chmod 600 file` | Owner read/write only |
| `chmod 644 file` | Owner rw, others read |
| `chmod 755 dir` | Standard directory |
| `chown www-data:www-data file` | Change ownership |

---

## Fontos Toolok

### Linux:
- **ls** - List files with permissions
- **find** - Search files by permissions
- **namei** - Recursive permission listing
- **stat** - Detailed file info
- **chmod** - Change permissions
- **chown** - Change ownership

### Windows:
- **icacls** - Built-in permission tool
- **AccessChk** - Sysinternals tool
- **AccessEnum** - Sysinternals GUI
- **cacls** - Legacy tool

---

## Fontos Megjegyzések

✅ **World-readable config** = credentials exposure  
✅ **World-writable files** = modification by anyone  
✅ **777 permissions** = NEVER use!  
✅ **Sensitive files** = 600 (owner only)  
✅ **Upload directories** = 750 with proper ownership  
❌ **Config in webroot** = bad practice!  
⚠️ **SUID files** = privilege escalation risk!  
⚠️ **Log files world-readable** = info disclosure!

---

**Összefoglalva:** Ez a fejezet a **file permission-ök teszteléséről** szól. **Helytelen permission-ök** (world-readable config, world-writable files) **sensitive information exposure**-t és **unauthorized modification**-t tesznek lehetővé. **`ls -la`**, **`find`**, és **`namei`** tool-okkal ellenőrizheted a permission-öket. **Critical files** (config, credentials, logs) **600 vagy 640** permission-nel kell rendelkezzenek (owner only vagy owner+group). **Web files** általában **644** (read/write owner, read others). **Directories** **755** (execute needed for traversal). **777 permission SOHA** ne használd! **Principle of least privilege** - csak a minimálisan szükséges jogokat add meg!
