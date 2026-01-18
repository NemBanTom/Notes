# WSTG-CONF-02: Alkalmazás Platform Konfiguráció Tesztelése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **webszerver és application server konfigurációját** kell átvizsgálni **default fájlok, debug code, és logging mechanizmusok** szempontjából. **Rossz konfiguráció** komoly biztonsági kockázatokat okozhat, még akkor is, ha az alkalmazás kódja biztonságos.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM az alkalmazás kódjának tesztelése! Ez a **platform konfiguráció** átvizsgálása - default sample files, comments, error handling, logging.

---

## Mi a cél?

**Biztosítani, hogy:**
- Default és ismert fájlok el legyenek távolítva
- Debug kód és extension-ök NE legyenek production-ban
- Logging mechanizmusok megfelelően működjenek

### Miért fontos?

- **Default sample apps** gyakran sebezhetőek (CVE-k)
- **HTML kommentek** érzékeny információt szivárogtathatnak
- **Debug mode** teljes stack trace-eket ad vissza
- **Rossz logging** nem detektálja a támadásokat

---

## 3 Fő Tesztelési Terület

### 1. **Sample és Known Files**
### 2. **Comment Review**
### 3. **Logging Mechanisms**

---

## 1. Sample és Known Files (Default Fájlok)

### Concept:
**Default telepítés** során sok webszerver **sample application-öket és test page-eket** telepít, amelyek **ismert sebezhetőségeket** tartalmaznak.

### Ismert Sebezhetőségek Példák:

#### CVE-1999-0449 (IIS Exair Sample):
```
Default IIS install → Exair sample site
Denial of Service vulnerability
```

#### CAN-2002-1744 (IIS CodeBrws.asp):
```
Microsoft IIS 5.0 → CodeBrws.asp
Directory traversal vulnerability
Attackers can read arbitrary files
```

#### CAN-2002-1630 (Oracle 9iAS sendmail.jsp):
```
Oracle 9iAS → sendmail.jsp
Email sending vulnerability
Attackers can send spam
```

#### CAN-2003-1172 (Apache Cocoon view-source):
```
Apache Cocoon → view-source sample
Directory traversal
Source code disclosure
```

---

### Tool #1: **Nikto (CGI Scanner)**

#### Parancsok:
```bash
# Alapvető scan (default files check)
nikto -h https://pelda.hu

# Specific tuning (sample files)
nikto -h https://pelda.hu -Tuning 2

# All tests
nikto -h https://pelda.hu -Tuning 123456789
```

**Mit keres:**
- Default admin pages
- Sample applications
- Test scripts
- Example CGI scripts
- Documentation directories

---

### Tool #2: **gobuster - Known Files**

#### Parancsok:
```bash
# Default files wordlist
gobuster dir -u https://pelda.hu -w default-files.txt

# Common CGI/sample files
gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirb/common.txt -x asp,jsp,php,cgi
```

**Default files példák:**
```
/examples/
/docs/
/manual/
/test/
/sample/
/demo/
/admin/
/_vti_bin/
/cgi-bin/
/scripts/
```

---

### Apache Default Files:

```bash
# Apache manual
curl https://pelda.hu/manual/

# Apache server-status
curl https://pelda.hu/server-status

# Apache server-info
curl https://pelda.hu/server-info

# Test pages
curl https://pelda.hu/test/
curl https://pelda.hu/examples/
```

**Ha elérhetőek:**
- `/manual/` → Documentation (information disclosure)
- `/server-status` → ⚠️ **CRITICAL!** Real-time server stats
- `/server-info` → ⚠️ **CRITICAL!** Server configuration info

---

### IIS Default Files:

```bash
# IIS help files
curl https://pelda.hu/iishelp/

# Sample apps
curl https://pelda.hu/iissamples/

# Admin scripts
curl https://pelda.hu/scripts/

# Virtual directory
curl https://pelda.hu/_vti_bin/
```

---

### Tomcat Default Files:

```bash
# Manager app
curl https://pelda.hu:8080/manager/html

# Host manager
curl https://pelda.hu:8080/host-manager/

# Examples
curl https://pelda.hu:8080/examples/

# Documentation
curl https://pelda.hu:8080/docs/
```

**Ha elérhetőek:**
- `/manager/html` → ⚠️ **CRITICAL!** Deploy malicious WAR files
- `/examples/` → Sample servlets (known vulnerabilities)

---

### Oracle WebLogic Default Files:

```bash
# Console
curl https://pelda.hu:7001/console

# uddiexplorer
curl https://pelda.hu:7001/uddiexplorer/
```

---

## 2. Comment Review (Komment Elemzés)

### Concept:
**HTML kommentek** gyakran tartalmaznak **érzékeny információkat**, developer jegyzetek, vagy **kikommentezett kódot**.

### Tool #1: **Browser View Source**

#### Használat:
```
1. Jobb klikk → View Page Source (Ctrl+U)
2. Ctrl+F keresés: <!--
3. Olvasd végig az összes kommentet
```

---

### Tool #2: **curl + grep**

#### Parancsok:
```bash
# Kommentek kinyerése
curl -s https://pelda.hu | grep -oP '<!--.*?-->'

# Többsoros kommentek
curl -s https://pelda.hu | sed -n '/<!--/,/-->/p'

# Rekurzív site crawl + comment extraction
wget -r -l 2 https://pelda.hu
grep -r "<!--" pelda.hu/
```

---

### Veszélyes Kommentek Példái:

#### 1. TODO jegyzetek:
```html
<!-- TODO: Remove hardcoded admin password before production -->
<!-- FIXME: SQL injection vulnerability in search.php -->
<!-- Note: Temporarily disabled authentication for testing -->
```

---

#### 2. Database credentials:
```html
<!-- Database connection: mysql://admin:P@ssw0rd@10.10.10.5/production -->
<!-- MongoDB: mongodb://dbuser:secret123@localhost:27017/myapp -->
```

---

#### 3. API keys:
```html
<!-- Google Maps API Key: AIzaSyDUEBnKgwiqMNpDplT6ozE4Z0XxuAbqDi4 -->
<!-- Stripe API: sk_live_1234567890abcdef -->
```

---

#### 4. Internal IPs és infrastructure:
```html
<!-- Load balancer: 192.168.1.100 -->
<!-- Backend API: http://10.0.0.50:8080/api -->
<!-- Old server (deprecated): server-old.internal.com -->
```

---

#### 5. Kikommentezett kód:
```html
<!--
<form action="/admin/delete_all_users.php" method="POST">
    <input type="hidden" name="confirm" value="yes">
    <button>Delete All</button>
</form>
-->
```

**Probléma:** Functionality hidden, de attacker láthatja a HTML-ben!

---

#### 6. Debug információk:
```html
<!-- User ID: 12345, Role: admin, Session: abc123 -->
<!-- Query: SELECT * FROM users WHERE id=5 -->
<!-- Response time: 2.3s, DB queries: 47 -->
```

---

### Tool #3: **Burp Suite - Comment Extraction**

#### Használat:
```
1. Burp → Proxy → HTTP History
2. Filter → Show only HTML
3. Response → Search: <!--
4. Vagy használj Burp Extender plugin-okat
```

---

## 3. Configuration Review (Konfiguráció Átvizsgálás)

### Gray-Box Testing (Access to Server)

#### Apache Configuration Review:

**Config file lokáció:**
```bash
# Main config
/etc/apache2/apache2.config
/etc/httpd/conf/httpd.conf

# Virtual hosts
/etc/apache2/sites-enabled/
/etc/httpd/conf.d/

# Check config syntax
apachectl -t
apache2ctl -t
```

---

**Fontos beállítások:**

#### 1. Disable Directory Listing:
```apache
<Directory /var/www/html>
    Options -Indexes
</Directory>
```

**Teszt:**
```bash
curl https://pelda.hu/uploads/
```

**Ha Directory Listing enabled:**
```html
Index of /uploads/
- file1.txt
- secret_document.pdf
- backup.zip
```

---

#### 2. Disable Server Signature:
```apache
ServerSignature Off
ServerTokens Prod
```

**Teszt:**
```bash
curl -I https://pelda.hu/nonexistent
```

**Előtte:**
```
Server: Apache/2.4.41 (Ubuntu)
```

**Utána:**
```
Server: Apache
```

---

#### 3. Custom Error Pages:
```apache
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
```

**Miért fontos:**
- Default error pages → version disclosure
- Stack traces → code paths disclosure

---

#### 4. Disable Unnecessary Modules:
```bash
# List loaded modules
apachectl -M

# Disable module
a2dismod status
a2dismod info
systemctl restart apache2
```

**Veszélyes modulok (ha nincs rá szükség):**
- `mod_status` → /server-status
- `mod_info` → /server-info
- `mod_userdir` → User home directories

---

#### IIS Configuration Review:

**IIS Manager GUI:**
```
1. Start → IIS Manager
2. Site → Configuration Editor
3. Check: Error Pages, Directory Browsing, Default Documents
```

**PowerShell:**
```powershell
# Get IIS configuration
Get-WebConfiguration

# Disable directory browsing
Set-WebConfigurationProperty -Filter /system.webServer/directoryBrowse -Name enabled -Value false

# Custom error pages
Set-WebConfigurationProperty -Filter /system.webServer/httpErrors -Name errorMode -Value Custom
```

---

#### nginx Configuration Review:

**Config file:**
```bash
/etc/nginx/nginx.conf
/etc/nginx/sites-enabled/default

# Check syntax
nginx -t
```

**Fontos beállítások:**

```nginx
# Disable server tokens
server_tokens off;

# Custom error pages
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# Disable autoindex
autoindex off;
```

---

## 4. Logging Mechanisms (Naplózási Mechanizmusok)

### Concept:
**Proper logging** kritikus a **security monitoring** és **incident response** szempontjából.

### 7 Kritikus Kérdés:

#### 1. **Sensitive Information in Logs?**

**Probléma példa:**
```
Apache access log:
GET /login?username=admin&password=Secret123! HTTP/1.1
```

**Érzékeny adatok a log-ban:**
- Passwords (GET parameters)
- Session tokens
- Credit card numbers
- API keys
- Personal data (GDPR!)

---

**Tesztelés:**
```bash
# Check access logs
tail -f /var/log/apache2/access.log

# Login request GET-tel (BAD!)
curl "https://pelda.hu/login?username=admin&password=test123"

# Nézd a logot
grep "password" /var/log/apache2/access.log
```

**Ha látod a password-öt a log-ban → KRITIKUS!**

---

#### 2. **Log Location - Dedicated Server?**

**Probléma:**
Ha logok ugyanazon a szerveren vannak:
- Attacker törli a logokat
- Disk space attack (DoS)

**Best practice:**
```
Central logging server:
- Syslog server
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Splunk
- Graylog
```

**Tesztelés:**
```bash
# Check log location
ls -la /var/log/apache2/
ls -la /var/log/nginx/

# Check if local or remote
netstat -an | grep 514  # Syslog port
```

---

#### 3. **Log Storage - DoS Condition?**

**Probléma:**
Attacker tömi a logokat → disk full → service failure

**Tesztelés:**
```bash
# Check disk usage
df -h /var/log

# Generate massive logs (TEST ENVIRONMENT ONLY!)
for i in {1..10000}; do
    curl "https://pelda.hu/nonexistent?param=$(head -c 1000 /dev/urandom | base64)"
done

# Check disk again
df -h /var/log
```

**Ha nő gyorsan → log rotation nincs vagy rossz**

---

#### 4. **Log Rotation?**

**Config check:**

**Linux logrotate:**
```bash
# Check logrotate config
cat /etc/logrotate.d/apache2

# Typical config:
/var/log/apache2/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
}
```

**Tesztelés:**
```bash
# Check log age
ls -lh /var/log/apache2/access.log*

# Should see:
# access.log
# access.log.1
# access.log.2.gz
# ...
```

---

#### 5. **Log Review - Attack Detection?**

**Mit keress:**

**40x Errors (CGI Scanner):**
```bash
# Count 404 errors by IP
awk '($9 ~ /404/)' /var/log/apache2/access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Output:
# 1523 192.168.1.50  ← Suspicious! Automated scan?
# 12 10.0.0.5
# 5 203.0.113.10
```

---

**50x Errors (SQL Injection Attempts):**
```bash
# 500 errors
grep " 500 " /var/log/apache2/access.log

# Példa log:
GET /product.php?id=1' UNION SELECT 1,2,3-- HTTP/1.1" 500
GET /search?q=<script>alert(1)</script> HTTP/1.1" 500
```

**Ha sok 50x ugyanabból az IP-ből → Attack!**

---

#### 6. **Log Backup?**

**Tesztelés:**
```bash
# Check backup schedule
crontab -l | grep log

# Check if logs are backed up
ls -la /backup/logs/

# Check remote backup (scp, rsync)
ps aux | grep rsync
```

---

#### 7. **Log Validation?**

**Probléma:**
Ha nincs input validation a logging-nál → **Log Injection**

**Log Injection példa:**
```bash
# Attacker request
curl "https://pelda.hu/login?username=admin%0A[2024-01-01] SUCCESS: admin logged in"

# Log output:
[2024-01-18] FAILED: admin
[2024-01-01] SUCCESS: admin logged in  ← FAKE entry!
```

**Tesztelés:**
```bash
# Newline injection
curl "https://pelda.hu/test?param=test%0AFake%20Log%20Entry"

# Check log
tail -f /var/log/apache2/access.log
```

---

### Apache Log Configuration:

**httpd.conf vagy apache2.conf:**
```apache
# Access log format (don't log passwords!)
LogFormat "%h %l %u %t \"%r\" %>s %b" common
CustomLog /var/log/apache2/access.log common

# Error log
ErrorLog /var/log/apache2/error.log
LogLevel warn

# Exclude sensitive parameters
SetEnvIf Request_URI "password=" dontlog
CustomLog /var/log/apache2/access.log common env=!dontlog
```

---

### nginx Log Configuration:

**nginx.conf:**
```nginx
http {
    # Access log
    access_log /var/log/nginx/access.log;
    
    # Error log
    error_log /var/log/nginx/error.log warn;
    
    # Custom log format (no sensitive data)
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request_method $uri" $status $body_bytes_sent';
    
    access_log /var/log/nginx/access.log main;
}
```

---

### IIS Logging:

**IIS Manager:**
```
1. Site → Logging
2. Format: W3C
3. Fields: Select (ne include sensitive fields)
4. Directory: Separate partition
```

**PowerShell:**
```powershell
# Check logging settings
Get-WebConfiguration -Filter '/system.applicationHost/sites/siteDefaults/logFile'

# Enable logging
Set-WebConfigurationProperty -Filter '/system.applicationHost/sites/siteDefaults/logFile' -Name enabled -Value $true
```

---

## Sensitive Information Types in Logs

### CRITICAL:
- **Passwords** (plaintext vagy encoded)
- **Session tokens**
- **API keys**
- **Credit card numbers** (PCI-DSS violation!)
- **Database connection strings**
- **Encryption keys**

### HIGH:
- **Authentication tokens**
- **Personal data** (GDPR - name, email, address)
- **Internal IP addresses**
- **File paths** (source code location)
- **Stack traces** (code logic disclosure)

### MEDIUM:
- **Usernames**
- **Email addresses**
- **System component names**
- **Debug information**

---

## System Configuration Hardening

### CIS Benchmarks:

**Download:**
- https://www.cisecurity.org/cis-benchmarks/

**Available for:**
- Apache HTTP Server
- Microsoft IIS
- nginx
- Tomcat

**Használat:**
```
1. Download CIS Benchmark PDF
2. Checklist minden item-re
3. Implement recommendations
4. Re-test
```

---

### CIS-CAT Lite (Automated Assessment):

**Tool:**
- https://www.cisecurity.org/cybersecurity-tools/cis-cat-lite/

**Használat:**
```bash
# Run assessment
./CIS-CAT.sh -b benchmarks/CIS_Apache_HTTP_Server_2.4_Benchmark_v2.0.0.xml

# Generate report
# HTML report: CIS-CAT-Report.html
```

---

### NIST Checklists:

**URL:** https://nvd.nist.gov/ncp/repository

**Available:**
- Web servers
- Operating systems
- Databases
- Applications

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs/Tool |
|---------|--------------|
| Default files scan | `nikto -h https://pelda.hu` |
| Directory listing check | `curl https://pelda.hu/uploads/` |
| Comment extraction | `curl -s https://pelda.hu \| grep "<!--"` |
| Server-status check | `curl https://pelda.hu/server-status` |
| Apache modules list | `apachectl -M` |
| nginx config test | `nginx -t` |
| Log location check | `ls -la /var/log/apache2/` |
| Log rotation check | `cat /etc/logrotate.d/apache2` |
| 404 errors by IP | `awk '($9 ~ /404/)' access.log \| awk '{print $1}' \| uniq -c \| sort -rn` |
| Disk usage | `df -h /var/log` |

---

## Fontos Toolok

### Scanners:
- **Nikto** - Web server scanner
- **gobuster** - Directory/file brute-force
- **wfuzz** - Web fuzzer

### Config Assessment:
- **CIS-CAT Lite** - CIS Benchmark automated assessment
- **Lynis** - Security auditing tool (Linux)
- **Microsoft Attack Surface Analyzer** - Windows

### Log Analysis:
- **grep/awk/sed** - Command-line log parsing
- **GoAccess** - Real-time web log analyzer
- **Splunk** - Commercial log management
- **ELK Stack** - Open source log management

---

## Védelem (Remediation)

### 1. **Remove Default Files**
```bash
# Apache
rm -rf /var/www/html/manual
rm -rf /var/www/html/examples

# IIS
Remove-Item C:\inetpub\wwwroot\iisstart.htm
```

---

### 2. **Disable Directory Listing**
```apache
# Apache
Options -Indexes

# nginx
autoindex off;
```

---

### 3. **Custom Error Pages**
```apache
# Apache
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
```

---

### 4. **Remove HTML Comments (Production)**
```bash
# Build process-ben:
# Minify HTML → kommentek törlődnek
```

---

### 5. **Proper Logging**
```apache
# Don't log passwords
SetEnvIf Request_URI "password=" dontlog
CustomLog /var/log/apache2/access.log common env=!dontlog

# Use POST for sensitive data (not GET)
```

---

### 6. **Log Rotation**
```bash
# /etc/logrotate.d/apache2
/var/log/apache2/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    create 640 root adm
}
```

---

### 7. **Centralized Logging**
```bash
# rsyslog config
*.* @@log-server.internal.com:514

# Or use modern solutions: ELK, Splunk, Graylog
```

---

## Fontos Megjegyzések

✅ **Default files = ismert sebezhetőségek** - töröld őket!  
✅ **HTML kommentek** gyakran érzékeny infót tartalmaznak  
✅ **Directory listing** = information disclosure  
✅ **Passwords in logs** = CRITICAL (PCI-DSS, GDPR violation!)  
✅ **Log rotation** kötelező (disk space management)  
❌ **Server-status/server-info** NE legyen elérhető public-an!  
⚠️ **GET parameters with passwords** → access log-ban látszik!  
⚠️ **Stack traces** production-ban = code disclosure  

---

**Összefoglalva:** Ez a fejezet az **alkalmazás platform konfigurációjának teszteléséről** szól. **Default sample files** (Tomcat examples, IIS samples) ismert sebezhetőségeket tartalmaznak - **töröld őket**! **HTML kommentek** érzékeny információkat szivárogtathatnak (TODO, credentials, internal IPs). **Logging** megfelelő konfigurálása kritikus - ne logolj **sensitive data-t** (passwords, tokens), használj **log rotation-t**, és **central logging**-ot. **Custom error pages** megakadályozzák a **version disclosure** és **stack trace**-eket!
