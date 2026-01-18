# WSTG-CONF-04: Régi Backup és Unreferenced Fájlok Átvizsgálása

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **régi, backup, és nem hivatkozott fájlok** (`.old`, `.bak`, `~`, `.zip` archives) gyakran **érzékeny információkat tartalmaznak** (source code, credentials, vulnerabilities). Ezeket a fájlokat gyakran **elfelejtik** törölni production-ból, és **könnyen letölthetővé válnak**.

⚠️ **NE KEVERD ÖSSZE:** Ez hasonló a CONF-03-hoz, de **szélesebb scope** - nem csak file extension handling, hanem **teljes forgotten files discovery** (backup archives, snapshots, unreferenced pages, commented-out functionality).

---

## Mi a cél?

**Unreferenced fájlok megtalálása és elemzése**, amelyek:
- Sensitive information-t tartalmaznak
- Régi sebezhetőségeket tartalmaznak
- Source code-ot felfednek
- Admin funkciókat rejtenek

---

## Veszélyek (Threats)

### 1. **Sensitive Information Disclosure**
```
Include files → Database credentials
Config files → API keys, internal paths
Log files → Session IDs, user data
```

### 2. **Hidden Functionality**
```
Admin pages → Not linked, but accessible
Unreferenced APIs → Powerful operations
Debug interfaces → Sensitive operations
```

### 3. **Old Vulnerabilities**
```
viewdoc.old.jsp → Fixed vulnerability in viewdoc.jsp
But old version still exploitable!
```

### 4. **Source Code Disclosure**
```
login.asp.bak → Source code visible
Reveals: Business logic, SQL queries, credentials
```

### 5. **Backup Archives**
```
myservlets.jar.old → All servlet classes
Can be decompiled → Reverse engineered
```

### 6. **File System Snapshots**
```
/.snapshot/monthly.1/view.php → Old vulnerable version
Still accessible via web!
```

---

## Tesztelési Technikák

### 1. **Naming Scheme Inference** (Elnevezési Séma)
### 2. **Published Content Clues** (Publikált Tartalom Nyomok)
### 3. **Blind Guessing** (Vak Találgatás)
### 4. **Server Misconfiguration** (Szerver Hibakonfiguráció)
### 5. **Public Information** (Publikus Információ)

---

## 1. Naming Scheme Inference

### Concept:
Ha találsz egy fájlt, **logikai következtetéssel** keress hasonlókat.

### Példák:

#### Felhasználó kezelés:
```
Találtad: viewuser.asp

Keress:
- edituser.asp
- adduser.asp
- deleteuser.asp
- listusers.asp
- adminuser.asp
```

---

#### Directory Structure:
```
Találtad: /app/user/

Keress:
- /app/admin/
- /app/manager/
- /app/moderator/
- /app/superuser/
```

---

#### Verzió séma:
```
Találtad: api_v2.php

Keress:
- api_v1.php (régi verzió, lehet sebezhető)
- api_v3.php (beta verzió?)
- api_old.php
- api_legacy.php
```

---

### Tool: **Manual Logic + gobuster**

#### Parancsok:
```bash
# Ha találtál viewuser.asp
gobuster dir -u https://pelda.hu -w user-functions.txt

# user-functions.txt:
adduser.asp
edituser.asp
deleteuser.asp
listusers.asp
```

---

## 2. Published Content Clues

### Concept:
**HTML/JS source code** kommentek, kikommentezett kód, JavaScript logic → hidden pages.

### A) **HTML Comments:**

```html
<!-- <A HREF="uploadfile.jsp">Upload a document to the server</A> -->
<!-- Link removed while bugs in uploadfile.jsp are fixed          -->
```

**Mit csinálj:**
```bash
curl https://pelda.hu/uploadfile.jsp
# Lehet, hogy elérhető!
```

---

### B) **JavaScript Conditional Logic:**

```javascript
var adminUser = false;
if (adminUser) {
    menu.add(new menuItem("Maintain users", "/admin/useradmin.jsp"));
}
```

**Mit látsz:**
- `/admin/useradmin.jsp` létezik
- Csak admin user-eknek jelenik meg
- De a fájl közvetlen elérése lehet, hogy NEM védett!

**Teszt:**
```bash
curl https://pelda.hu/admin/useradmin.jsp
# Elérhető authorization nélkül?
```

---

### C) **Hidden Forms:**

```html
<form action="forgotPassword.jsp" method="post">
    <input type="hidden" name="userID" value="123">
    <!-- <input type="submit" value="Forgot Password"> -->
</form>
```

**Mit látsz:**
- `forgotPassword.jsp` létezik
- `userID` parameter
- Lehet IDOR vulnerability

**Teszt:**
```bash
curl -X POST https://pelda.hu/forgotPassword.jsp -d "userID=1"
curl -X POST https://pelda.hu/forgotPassword.jsp -d "userID=2"
# Más user jelszavát lehet resetelni?
```

---

### D) **robots.txt Clues:**

```
User-agent: *
Disallow: /Admin
Disallow: /uploads
Disallow: /backup
Disallow: /~jbloggs
Disallow: /include
```

**Tesztek:**
```bash
curl https://pelda.hu/Admin/
curl https://pelda.hu/uploads/
curl https://pelda.hu/backup/
curl https://pelda.hu/include/
```

---

### Tool: **Manual Source Review + Burp**

#### Burp Suite:
```
1. Proxy → HTTP History
2. Find HTML pages
3. Search in Response: "<!--" (comments)
4. Search: ".jsp", ".asp", ".php" (hidden references)
5. Extract URLs
```

---

## 3. Blind Guessing (Backup File Discovery)

### Concept:
**Wordlist-tel** próbálgatni gyakori backup file neveket.

### Gyakori Backup Patterns:

#### A) **Extension-based:**
```
index.php.bak
index.php.old
index.php.orig
index.php.backup
index.php.save
index.php.tmp
index.php~
index.php.swp
index.php.copy
```

---

#### B) **Prefix-based (Windows):**
```
Copy of index.php
index - Copy.php
index (2).php
```

---

#### C) **Archive files:**
```
backup.zip
backup.tar
backup.tar.gz
site-backup.zip
www.tar.gz
database.sql
```

---

#### D) **Editor backups:**
```
# Emacs
index.php~
#index.php#

# Vim
.index.php.swp
index.php.swo

# Vi
index.php~
```

---

### Tool #1: **gobuster - Backup Discovery**

#### Parancsok:
```bash
# Extension-based
gobuster dir -u https://pelda.hu -w php-files.txt -x bak,old,orig,backup,tmp,save,copy

# Archive files
gobuster dir -u https://pelda.hu -w common.txt -x zip,tar,gz,tgz,rar,7z

# Emacs/Vim backups
gobuster dir -u https://pelda.hu -w files.txt -x "~,swp"
```

---

### Tool #2: **ffuf**

#### Parancsok:
```bash
# Backup pattern fuzzing
ffuf -u https://pelda.hu/index.phpFUZZ -w backup-suffixes.txt -mc 200

# backup-suffixes.txt:
.bak
.old
.orig
~
.swp
.backup
```

---

### Tool #3: **Manual netcat script** (eredeti dokumentumból):

```bash
#!/bin/bash
server=pelda.hu
port=80

while read url; do
    echo -ne "$url\t"
    echo -e "GET /$url HTTP/1.0\nHost: $server\n\n" | nc $server $port | head -1
done | tee outputfile
```

**Használat:**
```bash
# wordlist.txt:
backup.zip
backup.tar.gz
index.php.bak
login.asp.old

cat wordlist.txt | ./check_files.sh
```

---

## 4. Server Vulnerabilities & Misconfiguration

### A) **Directory Listing:**

```bash
# Check minden directory-ra
curl https://pelda.hu/uploads/
curl https://pelda.hu/includes/
curl https://pelda.hu/backup/
```

**Ha enabled:**
```html
Index of /backup/
- database.sql
- source-code.zip
- config.inc.bak
```

→ **CRITICAL!** Minden fájl letölthető!

---

### Tool: **Nmap - HTTP Enum Script**

```bash
nmap --script http-enum pelda.hu
```

**Mit keres:**
- Common directories
- Backup directories
- Admin interfaces

---

### B) **Apache ?M=D Directory Listing Vulnerability:**

```bash
# Régi Apache verziókon
curl "https://pelda.hu/?M=D"
curl "https://pelda.hu/includes/?M=D"
```

**Ha sebezhető:**
→ Directory listing akkor is, ha ki van kapcsolva!

---

### C) **IIS WebDAV Directory Listing:**

```bash
# WebDAV PROPFIND
curl -X PROPFIND https://pelda.hu/

# Vagy
davtest -url https://pelda.hu
```

---

## 5. Publicly Available Information

### A) **Google Cache:**

```bash
# Google Search
cache:pelda.hu/admin

# Wayback Machine
https://web.archive.org/web/*/pelda.hu/admin
```

**Mit talál:**
- Régi, már törölt oldalak
- Linkek régi admin interfészekhez
- Obsolete functionality

---

### B) **Google Dorking:**

```
# Backup files
site:pelda.hu ext:bak
site:pelda.hu ext:old
site:pelda.hu ext:sql

# Archive files
site:pelda.hu ext:zip
site:pelda.hu ext:tar

# Old versions
site:pelda.hu inurl:old
site:pelda.hu inurl:backup
site:pelda.hu inurl:v1
site:pelda.hu inurl:dev
```

---

### C) **Search Engine Cache:**

```
# Google cached version
cache:pelda.hu

# Bing cached version
# Keresd a "Cached" linket a search result-ban
```

**Miért hasznos:**
- Lehet, hogy a fájl már törölve van a szerverről
- De cache-ben még elérhető
- Lehet benne link hidden pages-hez

---

## 6. File System Snapshots

### Concept:
Bizonyos file systemek **automatikus snapshot-okat** készítenek (ZFS, NetApp).

### Snapshot Locations:

```bash
# NetApp WAFL
/.snapshot/
/.snapshot/hourly.0/
/.snapshot/daily.0/
/.snapshot/weekly.0/
/.snapshot/monthly.0/

# ZFS
/.zfs/snapshot/

# Veritas
/.vxfs/
```

---

### Tesztelés:

```bash
# Check snapshot directories
curl https://pelda.hu/.snapshot/
curl https://pelda.hu/.snapshot/hourly.0/view.php

# Old vulnerable version lehet
curl https://pelda.hu/.snapshot/monthly.1/login.php
```

---

### Tool: **gobuster - Snapshot Discovery**

```bash
# Check for snapshot directories
gobuster dir -u https://pelda.hu -w snapshot-paths.txt

# snapshot-paths.txt:
.snapshot
.snapshot/hourly.0
.snapshot/daily.0
.zfs/snapshot
```

---

## Windows 8.3 Filename Bypass

### Concept:
Windows **8.3 short filename** expansion-nel lehet elérni fájlokat.

### Példa:

```
Long filename: program files
Short name: PROGRA~1

backup copy of index.php
→ BACKUP~1.PHP
```

---

### Attack:

```bash
# Long filename
curl https://pelda.hu/backup/source-code-backup.zip
# → 404 Not Found

# 8.3 short name
curl https://pelda.hu/backup/SOURCE~1.ZIP
# → 200 OK! Downloaded!
```

---

## Comprehensive Wordlist Examples

### Backup Extensions:
```
.bak
.backup
.old
.orig
.original
.copy
.tmp
.temp
.swp
.swo
~
.save
.bck
.back
```

### Archive Extensions:
```
.zip
.tar
.tar.gz
.tgz
.rar
.7z
.gz
.bz2
```

### Common Backup Filenames:
```
backup.zip
backup.tar.gz
site-backup.zip
www.tar
database.sql
db.sql
dump.sql
backup.sql
```

---

## Gyakorlati Workflow

### 1. Spider + Source Review:
```bash
# Spider the site
wget -r -l 2 https://pelda.hu

# Search for clues in HTML
grep -r "<!--" pelda.hu/
grep -r "href=" pelda.hu/ | grep -E "(admin|backup|old)"
```

---

### 2. Backup File Discovery:
```bash
# Extension-based
gobuster dir -u https://pelda.hu -w php-files.txt -x bak,old,orig

# Archive discovery
gobuster dir -u https://pelda.hu -w common.txt -x zip,tar,gz
```

---

### 3. robots.txt Check:
```bash
curl https://pelda.hu/robots.txt
# Note Disallow entries
# Test each manually
```

---

### 4. Google Dorking:
```
site:pelda.hu ext:bak
site:pelda.hu ext:sql
site:pelda.hu inurl:backup
```

---

### 5. Snapshot Check:
```bash
curl https://pelda.hu/.snapshot/
curl https://pelda.hu/.zfs/snapshot/
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Backup file check | `curl https://pelda.hu/index.php.bak` |
| Archive check | `curl https://pelda.hu/backup.zip` |
| Directory listing | `curl https://pelda.hu/backup/` |
| Google dorking | `site:pelda.hu ext:bak` |
| robots.txt | `curl https://pelda.hu/robots.txt` |
| Snapshot check | `curl https://pelda.hu/.snapshot/` |
| Extension fuzzing | `gobuster dir -u URL -w files.txt -x bak,old` |
| Source review | `curl https://pelda.hu \| grep "<!--"` |

---

## Fontos Toolok

### Discovery:
- **gobuster** - Directory/file brute-force
- **ffuf** - Fast fuzzer
- **dirb** - Web content scanner
- **Burp Suite** - Proxy + Spider

### Scanners:
- **Nikto** - Web server scanner
- **Nessus** - Vulnerability scanner

### Manual:
- **wget** - Recursive download
- **curl** - HTTP requests
- **netcat** - Raw HTTP

### Public Info:
- **Google** - Dorking + Cache
- **Wayback Machine** - Archive.org

---

## Védelem (Remediation)

### 1. **Never Edit in Production:**
```
DON'T:
- Edit files in-place on production
- Create backups on live server
- Leave temporary files

DO:
- Edit locally
- Deploy via CI/CD
- Automated deployment
```

---

### 2. **File System Policies:**
```apache
# Apache - Block snapshots
<Location ~ "\.snapshot">
    Order deny,allow
    Deny from all
</Location>

# Block backup extensions
<FilesMatch "\.(bak|old|orig|backup|tmp)$">
    Order allow,deny
    Deny from all
</FilesMatch>
```

---

### 3. **Automated Cleanup:**
```bash
# Cron job - Find and remove backup files
0 2 * * * find /var/www/html -name "*.bak" -delete
0 2 * * * find /var/www/html -name "*.old" -delete
0 2 * * * find /var/www/html -name "*~" -delete
```

---

### 4. **Configuration Management:**
```
- Version control (Git)
- Proper deployment process
- No manual file editing
- Automated testing
```

---

### 5. **Disable Directory Listing:**
```apache
# Apache
Options -Indexes

# nginx
autoindex off;
```

---

## Fontos Megjegyzések

✅ **Backup files** (`.bak`, `.old`) = source code disclosure  
✅ **Archive files** (`.zip`, `.tar`) = complete application download  
✅ **Editor backups** (`~`, `.swp`) gyakran elfelejt találtatnak  
✅ **File system snapshots** = old vulnerable versions  
✅ **robots.txt** = roadmap to hidden directories  
❌ **SOHA ne editálj production-ban!**  
⚠️ **Google cache** régi deleted pages-t is tartalmaz  
⚠️ **Windows 8.3** filename bypass lehetséges!

---

**Összefoglalva:** Ez a fejezet a **forgotten és backup fájlok felderítéséről** szól. **Editor backups** (`index.php~`, `.swp`), **manual backups** (`.bak`, `.old`), és **archive files** (`backup.zip`) gyakran **elfelejt találtatnak production-ról** és **érzékeny információkat** tartalmaznak (source code, credentials, old vulnerabilities). **Naming scheme inference** (ha `viewuser.asp` → keress `edituser.asp`), **HTML comment review**, **robots.txt**, **Google dorking**, és **file system snapshots** mind hatékony discovery módszerek. **NE editálj in-place production-ban** és **automatizált cleanup** folyamatot kell bevezetni!
