# WSTG-CONF-05: Admin Interfészek Felsorolása

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **admin interfészek** gyakran **nincs megfelelően védve** és **könnyen felfedezhetők**. Ezek az interfészek **privileged funkciókat** (user provisioning, config changes, data manipulation) biztosítanak, és ha **unauthorized user** hozzáfér, az **teljes kompromittálást** eredményezhet.

⚠️ **NE KEVERD ÖSSZE:** Ez nem általános admin testing, hanem specifikusan **admin interface discovery** - megtalálni rejtett admin page-eket, default admin path-okat, és tesztelni az access control-t.

---

## Mi a cél?

**Rejtett admin interfészek és funkciók azonosítása:**
- Default admin paths (`/admin`, `/administrator`)
- Framework-specific admin pages (WordPress `/wp-admin/`)
- Hidden functionality (parameter tampering)
- Alternative ports (Tomcat :8080)

---

## Admin Interface Funkciók

### Tipikus privileged operations:
- **User provisioning** (create/delete users)
- **Site design** és layout changes
- **Data manipulation** (database access)
- **Configuration changes** (settings, modules)
- **File management** (upload, delete)
- **System monitoring** (logs, stats)

---

## Tesztelési Technikák

### 1. **Directory & File Enumeration**
### 2. **Source Code Review**
### 3. **Server Documentation**
### 4. **Publicly Available Information**
### 5. **Alternative Ports**
### 6. **Parameter Tampering**

---

## 1. Directory & File Enumeration

### Concept:
**Common admin paths** brute-force-olása wordlist-tel.

### Tool #1: **gobuster**

#### Parancsok:
```bash
# Common admin paths
gobuster dir -u https://pelda.hu -w admin-paths.txt

# Specific extensions
gobuster dir -u https://pelda.hu -w admin-paths.txt -x php,asp,aspx,jsp

# Lowercase + uppercase
gobuster dir -u https://pelda.hu -w admin-paths.txt --lowercase --uppercase
```

**Admin wordlist (admin-paths.txt):**
```
/admin
/administrator
/admin.php
/admin/
/admin/login
/admin/index.php
/admincp
/admin_area
/admin-console
/administration
/administrador
/adm
/backend
/control
/controlpanel
/cpanel
/cms
/manage
/manager
/moderator
/webadmin
/siteadmin
/Login
/login
/signin
/secret
/root
```

---

### Tool #2: **ffuf**

#### Parancsok:
```bash
# Admin path fuzzing
ffuf -u https://pelda.hu/FUZZ -w admin-paths.txt -mc 200,301,302,403

# Admin file fuzzing
ffuf -u https://pelda.hu/FUZZ -w admin-files.txt -mc 200,403

# Hide 404
ffuf -u https://pelda.hu/FUZZ -w wordlist.txt -fc 404
```

---

### Tool #3: **ZAP - Forced Browse**

#### Használat:
```
1. ZAP → Tools → Forced Browse
2. Target: https://pelda.hu
3. Select wordlist (admin directories)
4. Start
5. Nézd a talált path-eket
```

---

### Google Dorking - Admin Discovery:

```
# Generic admin pages
site:pelda.hu inurl:admin
site:pelda.hu inurl:administrator
site:pelda.hu inurl:login
site:pelda.hu inurl:manage

# Specific paths
site:pelda.hu "admin panel"
site:pelda.hu "administration"
site:pelda.hu intitle:"admin login"
```

---

## 2. Source Code Review

### Concept:
**HTML/JS source** kommentekben vagy kódban linkek admin page-ekhez.

### Tool: **curl + grep**

#### Parancsok:
```bash
# Admin links in HTML
curl -s https://pelda.hu | grep -i "admin\|administrator\|manage"

# Hidden links (display:none)
curl -s https://pelda.hu | grep -i "display.*none" -A 5

# JavaScript admin references
curl -s https://pelda.hu | grep -oE '/[a-zA-Z0-9_-]+/(admin|manage|control)' | sort -u
```

---

### Példák:

#### HTML Comment:
```html
<!-- Admin panel: /secret-admin-panel-xyz123 -->
<!-- TODO: Restrict admin access to 10.0.0.0/8 -->
```

---

#### Commented-out Link:
```html
<!-- <a href="/administrator/">Admin Login</a> -->
```

→ Try: `https://pelda.hu/administrator/`

---

#### JavaScript Reference:
```javascript
if (user.role === "admin") {
    window.location = "/admin/dashboard.php";
}
```

→ Try: `https://pelda.hu/admin/dashboard.php`

---

## 3. Server Documentation

### Concept:
**Default configurations** és **documentation** gyakran tartalmaz admin interface információt.

### Default Admin Paths Framework-önként:

#### WordPress:
```
/wp-admin/
/wp-login.php
/wp-admin/admin-ajax.php
/wp-admin/index.php
```

**Tesztelés:**
```bash
curl https://pelda.hu/wp-admin/
curl https://pelda.hu/wp-login.php
```

**Default credentials:**
- Username: `admin`
- Password: Check during setup (no default)

---

#### Joomla:
```
/administrator/
/administrator/index.php
/administrator/index.php?option=com_login
/administrator/index.php?option=com_users
```

**Tesztelés:**
```bash
curl https://pelda.hu/administrator/
```

**Default credentials:**
- Set during installation

---

#### Drupal:
```
/user/login
/admin
/user
/node/add
```

**Tesztelés:**
```bash
curl https://pelda.hu/user/login
```

---

#### phpMyAdmin:
```
/phpmyadmin/
/phpMyAdmin/
/pma/
/dbadmin/
/mysql/
```

**Tesztelés:**
```bash
curl https://pelda.hu/phpmyadmin/
```

**Default credentials:**
- Username: `root`
- Password: (empty) vagy `root`

---

#### Apache Tomcat:
```
/manager/html
/manager/text
/host-manager/html
```

**Tesztelés:**
```bash
curl https://pelda.hu:8080/manager/html
```

**Default credentials:**
- Username: `admin` vagy `tomcat`
- Password: `admin` vagy `tomcat`

---

#### Webmin:
```
:10000/
/session_login.cgi
```

**Tesztelés:**
```bash
curl https://pelda.hu:10000/
```

**Default credentials:**
- Username: `admin`
- Password: `admin` vagy set during install

---

#### Plesk:
```
:8443/
/login_up.php3
```

---

#### cPanel:
```
:2082/
:2083/ (SSL)
/login
```

---

### PHP Admin Tools:
```
/phpinfo.php
/info.php
/test.php
/dbadmin/
/mysqladmin/
/login.php
/admin.php
```

---

## 4. Publicly Available Information

### Concept:
**Public documentation** és **security advisories** gyakran említenek default admin paths.

### Resources:

#### Default Passwords:
- **CIRT Default Password List:** https://cirt.net/passwords
- Search by vendor/product
- Typical defaults

---

#### FuzzDB:
- **Admin Login Paths:** https://github.com/fuzzdb-project/fuzzdb
- `discovery/predictable-filepaths/login-file-locations/Logins.txt`

**Wordlist example:**
```
admin.php
administrator.php
moderator.php
webadmin.php
adminarea.php
bb-admin.php
adminLogin.php
admin_area.php
panel-administracion.php
```

---

## 5. Alternative Server Ports

### Concept:
Admin interfészek gyakran **különböző portokon** futnak.

### Common Admin Ports:

| Port | Service | Admin Interface |
|------|---------|-----------------|
| 8080 | HTTP Alt | Tomcat Manager |
| 8443 | HTTPS Alt | Alternative admin |
| 10000 | Webmin | Linux admin tool |
| 2082 | cPanel | Hosting control |
| 2083 | cPanel SSL | Hosting control (SSL) |
| 8443 | Plesk | Hosting control |
| 9090 | Cockpit | Server admin |
| 3000 | Various | Dev/Admin tools |

---

### Tool: **Nmap - Port Scan**

#### Parancsok:
```bash
# Common admin ports
nmap -p 8080,8443,10000,2082,2083,3000,9090 pelda.hu

# Full port scan
nmap -p- pelda.hu

# Service version detection
nmap -sV -p 8080,8443,10000 pelda.hu
```

**Példa output:**
```
PORT      STATE SERVICE VERSION
8080/tcp  open  http    Apache Tomcat 9.0.45
10000/tcp open  http    Webmin httpd
```

---

### Manual Testing:

```bash
# Check common admin ports
curl http://pelda.hu:8080/
curl https://pelda.hu:8443/
curl https://pelda.hu:10000/

# Tomcat Manager
curl http://pelda.hu:8080/manager/html

# Webmin
curl https://pelda.hu:10000/
```

---

## 6. Parameter Tampering

### Concept:
**Hidden parameters** vagy **cookies** admin funkciót engedélyezhetnek.

### A) **Hidden Form Fields:**

```html
<form action="/dashboard" method="POST">
    <input type="text" name="username">
    <input type="password" name="password">
    <input type="hidden" name="admin" value="no">
    <button>Login</button>
</form>
```

**Attack:**
```bash
# Change admin=no to admin=yes
curl -X POST https://pelda.hu/dashboard \
  -d "username=test&password=test&admin=yes"
```

**Ha sebezhető:**
→ Admin panel access!

---

### B) **Cookie Manipulation:**

**Original Cookie:**
```
Cookie: session=abc123; useradmin=0
```

**Attack:**
```bash
# Change useradmin=0 to useradmin=1
curl https://pelda.hu/dashboard \
  -H "Cookie: session=abc123; useradmin=1"
```

---

### C) **URL Parameter:**

**Original URL:**
```
https://pelda.hu/panel?user=normal
```

**Attack:**
```
https://pelda.hu/panel?user=admin
https://pelda.hu/panel?role=admin
https://pelda.hu/panel?isadmin=1
https://pelda.hu/panel?admin=true
```

---

### Tool: **Burp Suite Intruder**

#### Használat:
```
1. Intercept login/dashboard request
2. Send to Intruder
3. Add position: admin=§no§
4. Payload: yes, 1, true, admin
5. Start attack
6. Analyze responses
```

---

### Common Debug Parameters:

**FuzzDB - Debug Parameters:**
```
debug=true
debug=1
test=1
admin=1
isadmin=true
role=admin
user=admin
access=admin
dev=1
```

**Tesztelés:**
```bash
curl "https://pelda.hu/dashboard?debug=1"
curl "https://pelda.hu/panel?admin=true"
curl "https://pelda.hu/index.php?role=admin"
```

---

## Framework-Specific Admin Paths

### WordPress (részletes):
```bash
# Main admin
curl https://pelda.hu/wp-admin/

# Login
curl https://pelda.hu/wp-login.php

# AJAX endpoint
curl https://pelda.hu/wp-admin/admin-ajax.php

# User management
curl https://pelda.hu/wp-admin/users.php

# Plugin management
curl https://pelda.hu/wp-admin/plugins.php

# Theme editor (CRITICAL!)
curl https://pelda.hu/wp-admin/theme-editor.php
```

---

### Joomla (részletes):
```bash
# Admin panel
curl https://pelda.hu/administrator/

# Component access
curl https://pelda.hu/administrator/index.php?option=com_users
curl https://pelda.hu/administrator/index.php?option=com_config
curl https://pelda.hu/administrator/index.php?option=com_installer
```

---

### Tomcat (részletes):
```bash
# Manager app (WAR deployment)
curl http://pelda.hu:8080/manager/html

# Host manager
curl http://pelda.hu:8080/host-manager/html

# Text interface
curl http://pelda.hu:8080/manager/text/list

# Status
curl http://pelda.hu:8080/manager/status
```

---

## Brute-force Attacks

### Tool: **Hydra**

#### Parancsok:
```bash
# HTTP POST form brute-force
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  pelda.hu http-post-form "/admin/login.php:username=^USER^&password=^PASS^:Invalid"

# WordPress admin
hydra -l admin -P passwords.txt \
  pelda.hu http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:ERROR"

# Basic Auth
hydra -l admin -P passwords.txt \
  http-get://pelda.hu/admin/
```

**FIGYELEM:**
- Lassú
- Account lockout risk
- Detectable (IDS/IPS)
- Csak engedéllyel!

---

## Bypass Techniques

### 1. **IP Restriction Bypass:**

**If admin only from 10.0.0.0/8:**
```bash
# X-Forwarded-For header
curl https://pelda.hu/admin/ \
  -H "X-Forwarded-For: 10.0.0.5"

# X-Real-IP
curl https://pelda.hu/admin/ \
  -H "X-Real-IP: 10.0.0.5"

# Client-IP
curl https://pelda.hu/admin/ \
  -H "Client-IP: 10.0.0.5"
```

---

### 2. **HTTP Method Bypass:**

```bash
# GET blocked, try POST
curl -X POST https://pelda.hu/admin/

# Try other methods
curl -X PUT https://pelda.hu/admin/
curl -X DELETE https://pelda.hu/admin/
curl -X OPTIONS https://pelda.hu/admin/
```

---

### 3. **Path Traversal:**

```bash
# URL encoding
https://pelda.hu/admin/../admin/
https://pelda.hu/%2e%2e/admin/

# Double encoding
https://pelda.hu/%252e%252e/admin/
```

---

## Gray-Box Testing

### Ha van server access:

#### Check Config Files:

**Apache:**
```bash
# Virtual host config
cat /etc/apache2/sites-enabled/000-default.conf | grep -i "admin\|directory"

# htaccess
find /var/www -name ".htaccess" -exec grep -H "admin" {} \;
```

---

**nginx:**
```bash
# nginx config
cat /etc/nginx/sites-enabled/default | grep -i "location.*admin"
```

---

#### Check Application Config:

**WordPress:**
```bash
# wp-config.php
cat wp-config.php | grep -i "admin\|user"
```

---

**PHP:**
```bash
# Config files
find /var/www -name "config.php" -o -name "config.inc.php"
grep -r "admin" /var/www/html/includes/
```

---

## Gyakorlati Workflow

### 1. Directory Enumeration:
```bash
gobuster dir -u https://pelda.hu -w admin-paths.txt -x php
```

---

### 2. Framework Detection:
```bash
# WordPress?
curl https://pelda.hu/wp-admin/

# Joomla?
curl https://pelda.hu/administrator/

# phpMyAdmin?
curl https://pelda.hu/phpmyadmin/
```

---

### 3. Port Scan:
```bash
nmap -p 8080,8443,10000,2082,2083 pelda.hu
```

---

### 4. Source Code Review:
```bash
curl -s https://pelda.hu | grep -i "admin" | grep -v "^$"
```

---

### 5. Parameter Tampering:
```bash
# Test hidden parameters
curl "https://pelda.hu/dashboard?admin=1"
```

---

### 6. Default Credentials:
```
Try: admin/admin, admin/password, root/root
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Admin path discovery | `gobuster dir -u URL -w admin-paths.txt` |
| WordPress admin | `curl https://pelda.hu/wp-admin/` |
| phpMyAdmin | `curl https://pelda.hu/phpmyadmin/` |
| Tomcat Manager | `curl http://pelda.hu:8080/manager/html` |
| Port scan | `nmap -p 8080,10000,2082 pelda.hu` |
| Google dorking | `site:pelda.hu inurl:admin` |
| Parameter tampering | `curl "URL?admin=1"` |
| Source review | `curl -s URL \| grep -i admin` |

---

## Fontos Toolok

### Discovery:
- **gobuster** - Directory brute-force
- **ffuf** - Fast fuzzer
- **ZAP Forced Browse** - OWASP tool
- **dirb** - Web content scanner

### Brute-force:
- **Hydra** - Multi-protocol brute-forcer
- **Medusa** - Alternative brute-forcer
- **Patator** - Python-based

### Port Scanning:
- **Nmap** - Port scanner
- **masscan** - Fast scanner

### Manual:
- **curl** - HTTP requests
- **Burp Suite** - Proxy + Intruder

---

## Védelem (Remediation)

### 1. **Change Default Paths:**
```
DON'T use: /admin, /administrator
DO use: /myCustomAdminPath123
```

---

### 2. **IP Whitelisting:**
```apache
# Apache
<Directory /var/www/html/admin>
    Order Deny,Allow
    Deny from all
    Allow from 203.0.113.5
</Directory>

# nginx
location /admin {
    allow 203.0.113.5;
    deny all;
}
```

---

### 3. **Strong Authentication:**
```
- Multi-Factor Authentication (MFA)
- Strong passwords
- Account lockout after failed attempts
- CAPTCHA
```

---

### 4. **Separate Admin Domain:**
```
Public site: https://pelda.hu
Admin site: https://admin-internal.pelda.hu (VPN only)
```

---

### 5. **Web Application Firewall (WAF):**
```
- Block /admin, /administrator paths from public
- Rate limiting
- Geo-blocking
```

---

## Fontos Megjegyzések

✅ **Default admin paths** (`/admin`, `/wp-admin/`) nagyon gyakori  
✅ **Alternative ports** (8080, 10000) gyakran admin interfészek  
✅ **Parameter tampering** (`admin=1`) működhet weak auth-nál  
✅ **Default credentials** (`admin/admin`) gyakran nem változtatott  
❌ **Account lockout** risk brute-force-nál!  
⚠️ **Framework detection** → known admin paths  
⚠️ **Source code review** → hidden admin links  

---

**Összefoglalva:** Ez a fejezet az **admin interfészek felderítéséről** szól. **Directory brute-forcing** (gobuster, ffuf) megtalálja a **common admin paths** (`/admin`, `/administrator`). **Framework-specific** admin page-ek (WordPress `/wp-admin/`, Joomla `/administrator/`, Tomcat `:8080/manager/`) **well-known** és **könnyen azonosíthatók**. **Parameter tampering** (`admin=1`, cookie manipulation) **authorization bypass**-ot eredményezhet. **Default credentials** (`admin/admin`) gyakran **nem változtatottak**. **IP whitelisting**, **MFA**, és **custom admin paths** jelentik a védelmet!
