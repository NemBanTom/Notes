# WSTG-CONF-01: Hálózati Infrastruktúra Konfiguráció Tesztelése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **webszerver infrastruktúra konfigurációját** kell átvizsgálni **ismert sebezhetőségekre** és **rossz konfigurációkra**. Egyetlen sebezhető komponens (webszerver, adatbázis, auth server) az **egész alkalmazást veszélyeztetheti**.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM alkalmazás-szintű vulnerability testing! Ez az **infrastruktúra szintű** konfiguráció és patch management tesztelése - webserver, database, admin tools, authentication systems.

---

## Mi a cél?

**Az alkalmazás konfigurációjának átvizsgálása a hálózaton keresztül**:
- Nincs-e ismert sebezhetőség a komponensekben
- Framework-ök és rendszerek biztonságosak-e
- Default beállítások és credentials meg vannak-e változtatva
- Admin interfészek védettek-e

### Miért fontos?

- **Egy sebezhető webszerver** → teljes alkalmazás kompromittálása
- **Elavult szoftver** → ismert exploitok
- **Default credentials** → unauthorized access
- **Rossz konfiguráció** → information disclosure, RCE

---

## Tesztelési Lépések

### 1. **Infrastruktúra elemek meghatározása**
### 2. **Ismert sebezhetőségek ellenőrzése**
### 3. **Admin toolok átvizsgálása**
### 4. **Authentication rendszerek review**
### 5. **Port lista karbantartása** (change control)

---

## 1. Known Server Vulnerabilities (Ismert Szerver Sebezhetőségek)

### Concept:
**Elavult szoftver verziók** ismert CVE-ket tartalmaznak, amelyek exploitálhatók.

### Problémák:

#### A) Remote File Upload Vulnerability:
```
Példa:
- Apache Struts RCE (CVE-2017-5638)
- Webszerver sebezhető verzió
- Attacker feltölt backdoor-t
- Teljes szerver kompromittálva
```

#### B) Information Disclosure:
```
Példa:
- IIS source code disclosure vulnerability
- Attacker letölti az alkalmazás forráskódját
- Database credentials, API keys a kódban
- Teljes adatbázis kompromittálva
```

---

### Tool #1: **Nmap - Version Detection**

#### Parancsok:
```bash
# Service version detection
nmap -sV pelda.hu

# Aggressive version detection
nmap -sV --version-intensity 5 pelda.hu

# Specific ports
nmap -sV -p 80,443,22,3306,5432 pelda.hu

# OS detection is
nmap -O pelda.hu
```

**Kimenet példa:**
```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 5.3 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.2.15 ((CentOS))
3306/tcp open  mysql    MySQL 5.0.95
```

**Mit csinálj ezzel:**
1. Apache 2.2.15 → Keress CVE-ket
2. MySQL 5.0.95 → Elavult verzió, sok CVE
3. OpenSSH 5.3 → Régi verzió

---

### Tool #2: **Nessus (Vulnerability Scanner)**

#### Használat:
```
1. Telepítsd Nessus-t (Professional vagy Essentials)
2. New Scan → Basic Network Scan
3. Target: pelda.hu
4. Launch scan
5. Nézd az eredményeket (Critical, High, Medium, Low)
```

**Mit ad:**
- Részletes CVE lista
- CVSS score
- Exploit elérhetőség
- Remediation javaslatok

**Előnyök:**
- Nagyon átfogó (100,000+ plugin)
- Friss vulnerability database
- False positive szűrés

**Hátrányok:**
- Fizetős (Professional)
- Lassú (teljes scan órákig tart)

---

### Tool #3: **OpenVAS (Open Source Alternative)**

#### Parancsok:
```bash
# Telepítés (Kali-ban)
apt-get update
apt-get install openvas

# Setup
openvas-setup

# Web UI
https://localhost:9392
```

**Használat:**
```
1. Login a web UI-ba
2. Configuration → Targets → New Target
3. Scans → Tasks → New Task
4. Start scan
```

---

### Tool #4: **Nikto (Web Server Scanner)**

#### Parancsok:
```bash
# Alapvető scan
nikto -h https://pelda.hu

# Specific tests
nikto -h https://pelda.hu -Tuning 1,2,3,4

# Save report
nikto -h https://pelda.hu -o nikto_report.html -Format html

# Verbose output
nikto -h https://pelda.hu -Display V
```

**Mit keres:**
- Elavult szerver verziók
- Misconfiguration-ok
- Dangerous files/CGIs
- Default files
- Known vulnerabilities

---

### Vulnerability Database-ek:

#### 1. **CVE Details** (https://www.cvedetails.com)
```
1. Search: "Apache 2.2.15"
2. Nézd a CVE listát
3. CVSS score alapján priorizálj
```

#### 2. **Exploit-DB** (https://www.exploit-db.com)
```
1. Search: "MySQL 5.0.95"
2. Nézd, van-e public exploit
3. Teszteld (responsible disclosure!)
```

#### 3. **NVD - National Vulnerability Database** (https://nvd.nist.gov)
```
1. Search: CVE-2017-5638
2. Részletes technikai leírás
3. Affected versions
4. Remediation
```

---

### False Positives és False Negatives:

#### False Positive:
```
Scanner output:
"Apache 2.2.15 vulnerable to CVE-XXXX"

Reality:
- Vendor backported patch
- Debian/Red Hat security update
- Verzió szám nem változott, de patch alkalmazva van

Solution:
- Manual verification szükséges
- Check patch level: dpkg -l | grep apache2
```

#### False Negative:
```
Scanner output:
"No vulnerabilities found"

Reality:
- Version header obfuscated (Server: Website)
- Sebezhető verzió fut, de scanner nem ismerte fel

Solution:
- Manual testing
- Assume worst case (legrégebbi supported verzió)
```

---

### Manual Verification:

#### Példa: Apache Version Check
```bash
# Remote version check
curl -I https://pelda.hu | grep Server

# Ha van SSH access (white-box):
ssh user@pelda.hu
apache2 -v

# Red Hat/CentOS
rpm -q httpd

# Debian/Ubuntu
dpkg -l | grep apache2
```

---

## 2. Administrative Tools (Admin Eszközök)

### Concept:
**Admin interfészek** gyakran **gyenge hitelesítéssel** vagy **default credentials**-szel rendelkeznek.

### Admin Interface Típusok:

#### A) Web-based Admin Panels:
- **phpMyAdmin** (MySQL)
- **Adminer** (Database)
- **Webmin** (Linux admin)
- **cPanel/WHM** (Hosting)
- **Plesk** (Hosting)

#### B) FTP/File Management:
- **FTP servers** (vsftpd, ProFTPD)
- **WebDAV**
- **SFTP**

#### C) Network File Systems:
- **NFS** (Network File System)
- **CIFS/SMB** (Windows file sharing)

---

### Tool #1: **Directory Brute-force - gobuster**

#### Parancsok:
```bash
# Admin panel discovery
gobuster dir -u https://pelda.hu -w admin-panels.txt

# Common admin paths
gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirb/common.txt -x php,asp,aspx
```

**Admin wordlist példa:**
```
/admin
/administrator
/admin.php
/admin/login
/phpmyadmin
/pma
/webmin
/cpanel
/plesk
/wp-admin
/manager
/console
```

---

### Tool #2: **Default Credentials Testing**

#### Concept:
Sok admin interface **default username/password**-del települ.

**Gyakori default credentials:**

| Product | Default User | Default Pass |
|---------|--------------|--------------|
| phpMyAdmin | root | (empty) vagy root |
| Webmin | admin | admin |
| Tomcat | admin | admin vagy tomcat |
| MySQL | root | (empty) |
| PostgreSQL | postgres | postgres |
| MongoDB | (none) | (none) |
| Jenkins | admin | admin |

**Manual testing:**
```
1. Találd meg az admin interfészt (/phpmyadmin)
2. Próbáld: root / (empty)
3. Próbáld: admin / admin
4. Check default credentials listákat online
```

---

### Tool #3: **Hydra (Brute Force)**

#### Parancsok:
```bash
# HTTP POST login brute-force
hydra -l admin -P /usr/share/wordlists/rockyou.txt pelda.hu http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"

# FTP brute-force
hydra -l admin -P passwords.txt ftp://pelda.hu

# SSH brute-force
hydra -l root -P passwords.txt ssh://pelda.hu

# MySQL brute-force
hydra -l root -P passwords.txt mysql://pelda.hu
```

**FIGYELEM:** Brute-force **zajos** és **detektálható**! Csak engedéllyel!

---

### Common Admin Interfaces:

#### phpMyAdmin:
```bash
# Keresés
curl https://pelda.hu/phpmyadmin
curl https://pelda.hu/pma

# Default creds teszt
Username: root
Password: (empty)
```

**Ha bejutsz:**
- Teljes adatbázis hozzáférés
- SQL query futtatás
- File upload (into outfile)
- Potentially RCE

---

#### Tomcat Manager:
```bash
# URL
https://pelda.hu:8080/manager/html

# Default creds
admin / admin
tomcat / tomcat
admin / tomcat
```

**Ha bejutsz:**
- WAR file upload
- Deploy malicious application
- Remote Code Execution

---

#### Webmin:
```bash
# URL
https://pelda.hu:10000

# Default creds
admin / admin
root / changeme
```

---

### Remote Admin Access:

#### SSH:
```bash
# Version check
ssh -V

# Connect
ssh user@pelda.hu

# Weak algorithms check
nmap --script ssh2-enum-algos pelda.hu
```

---

#### RDP (Remote Desktop):
```bash
# Port scan
nmap -p 3389 pelda.hu

# RDP security check
nmap --script rdp-enum-encryption pelda.hu
```

---

#### VNC:
```bash
# Port scan
nmap -p 5900-5910 pelda.hu

# VNC auth check
nmap --script vnc-info pelda.hu
```

---

## 3. Authentication Systems Review

### Concept:
**Authentication backend** gyengeségei az egész alkalmazást veszélyeztetik.

### Authentication Types:

#### 1. **Local Database Authentication:**
- User credentials az alkalmazás DB-ben
- Password hashing ellenőrzés
- Brute-force protection?

---

#### 2. **LDAP/Active Directory:**
```bash
# LDAP port check
nmap -p 389,636 pelda.hu

# LDAP enumeration
ldapsearch -x -h pelda.hu -b "dc=pelda,dc=hu"

# Anonymous bind check
ldapsearch -x -h pelda.hu -b "" -s base
```

**Mit keress:**
- Anonymous bind engedélyezett?
- User enumeration lehetséges?
- Weak encryption (LDAP vs LDAPS)

---

#### 3. **NTLM Authentication:**
```bash
# Check NTLM header
curl -I https://pelda.hu/admin
```

**Kimenet:**
```
WWW-Authenticate: NTLM
```

**Ismert problémák:**
- NTLM relay attacks
- Pass-the-hash
- Credential harvesting

---

#### 4. **SSO (Single Sign-On):**

**OAuth/OpenID Connect:**
```bash
# Check OAuth endpoints
curl https://pelda.hu/.well-known/openid-configuration
```

**Mit tesztelj:**
- Redirect URI validation
- State parameter check
- Token security

**SAML:**
```bash
# Check SAML metadata
curl https://pelda.hu/saml/metadata
```

**Mit tesztelj:**
- XML signature validation
- XXE vulnerability
- Replay attacks

---

## 4. Port Management & Change Control

### Concept:
**Csak szükséges portok** legyenek nyitva.

### Tool: **Nmap Full Port Scan**

#### Parancsok:
```bash
# Teljes port scan
nmap -p- pelda.hu

# Top 1000 ports (gyorsabb)
nmap pelda.hu

# Service detection
nmap -sV -p- pelda.hu

# Script scan
nmap -sC -sV -p- pelda.hu
```

---

### Expected Ports (Web Application):

| Port | Service | Required? |
|------|---------|-----------|
| 80 | HTTP | ✅ Yes |
| 443 | HTTPS | ✅ Yes |
| 22 | SSH | ⚠️ Should be restricted to admin IPs |
| 21 | FTP | ❌ Usually not needed (use SFTP) |
| 3306 | MySQL | ❌ Should NOT be public |
| 5432 | PostgreSQL | ❌ Should NOT be public |
| 1433 | MSSQL | ❌ Should NOT be public |
| 27017 | MongoDB | ❌ Should NOT be public |
| 6379 | Redis | ❌ Should NOT be public |
| 11211 | Memcached | ❌ Should NOT be public |
| 8080 | Alt HTTP | ⚠️ Dev/Test only |
| 3389 | RDP | ❌ Should NOT be public |

---

### Unnecessary Open Ports = Security Risk!

**Példa scan output:**
```
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
3306/tcp  open  mysql
8080/tcp  open  http-proxy
```

**Problémák:**
- ⚠️ FTP nyitva (port 21) - régi, insecure protokoll
- ⚠️ MySQL nyitva (port 3306) - **CRITICAL!** Direkten elérhető internet-ről
- ⚠️ Port 8080 nyitva - Lehet dev/test server

---

### Tool: **Masscan (Gyors Port Scanner)**

#### Parancsok:
```bash
# Teljes port range gyorsan
masscan -p1-65535 pelda.hu --rate=1000

# Specific ports
masscan -p22,80,443,3306,5432 pelda.hu --rate=10000
```

---

## Vulnerability Patching & Lifecycle

### Concept:
**Vendor support lifecycle** - elavult verziók nem kapnak patch-et.

### Támogatási Modellek:

#### Red Hat Enterprise Linux (RHEL):
- **Full Support:** 10 év
- **Extended Support:** További 3 év (fizetős)
- **Backported patches** - verzió szám NEM változik

---

#### Debian:
- **Stable:** ~3 év security support
- **LTS (Long Term Support):** +2 év
- **Backported patches**

---

#### Ubuntu:
- **LTS releases:** 5 év support
- **Standard releases:** 9 hónap
- **ESM (Extended Security Maintenance):** +5 év (fizetős)

---

#### Windows Server:
- **Mainstream Support:** 5 év
- **Extended Support:** További 5 év
- **End of Life** után NINCS patch!

---

### End-of-Life (EOL) Software = CRITICAL RISK!

**Példák:**
- **Windows Server 2008** - EOL: 2020 January
- **PHP 5.6** - EOL: 2018
- **Python 2.7** - EOL: 2020 January
- **MySQL 5.0** - EOL: 2012

**Ha EOL szoftvert használsz:**
- ✅ Nincs security patch
- ✅ Új CVE-k nem javítódnak
- ✅ Exploitok publikusan elérhetők
- ✅ Compliance problémák (PCI-DSS, stb.)

---

## Gyakorlati Workflow

### 1. Infrastructure Mapping
```bash
# Port scan
nmap -sV -p- pelda.hu -oA nmap_scan

# Service identification
nmap -sC -sV pelda.hu
```

---

### 2. Version Extraction
```bash
# Web server
curl -I https://pelda.hu | grep Server

# SSH
ssh -V

# Database (ha van access)
mysql --version
psql --version
```

---

### 3. CVE Research
```
1. Google: "Apache 2.2.15 CVE"
2. CVE Details: cvedetails.com
3. Exploit-DB: exploit-db.com
4. Vendor advisories
```

---

### 4. Admin Interface Discovery
```bash
# Directory brute-force
gobuster dir -u https://pelda.hu -w admin-paths.txt

# Default credentials testing
# Manual browser testing
```

---

### 5. Unnecessary Services Check
```bash
# Full port scan
nmap -p- pelda.hu

# Identify unnecessary ports
# Report to client
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Port scan | `nmap -sV -p- pelda.hu` |
| Web server version | `curl -I https://pelda.hu \| grep Server` |
| Vulnerability scan | `nessus` vagy `openvas` |
| Web server scan | `nikto -h https://pelda.hu` |
| Admin panel discovery | `gobuster dir -u https://pelda.hu -w admin-paths.txt` |
| Default creds test | Manual browser testing |
| LDAP enumeration | `ldapsearch -x -h pelda.hu` |
| SSH version | `nmap --script ssh2-enum-algos pelda.hu` |
| Database port check | `nmap -p 3306,5432,1433,27017 pelda.hu` |

---

## Fontos Toolok

### Vulnerability Scanners:
- **Nessus** - Commercial, comprehensive
- **OpenVAS** - Open source alternative
- **Nikto** - Web server scanner
- **Nmap NSE scripts** - Specific vuln checks

### Port Scanners:
- **Nmap** - Standard, reliable
- **Masscan** - Very fast
- **Unicornscan** - Alternative

### Brute-force:
- **Hydra** - Multi-protocol brute-forcer
- **Medusa** - Alternative to Hydra
- **Patator** - Python-based

### Admin Panel Discovery:
- **gobuster** - Directory brute-force
- **ffuf** - Fast fuzzer
- **dirsearch** - Python-based

---

## Védelem (Remediation)

### 1. **Keep Software Updated**
```bash
# Debian/Ubuntu
apt-get update && apt-get upgrade

# Red Hat/CentOS
yum update

# Check for security updates only
apt-get upgrade --security
```

---

### 2. **Disable Unnecessary Services**
```bash
# Check running services
systemctl list-units --type=service --state=running

# Stop unnecessary service
systemctl stop ftp
systemctl disable ftp
```

---

### 3. **Firewall Configuration**
```bash
# iptables - csak 80, 443 port
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -j DROP

# ufw (Ubuntu)
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

---

### 4. **Change Default Credentials**
```
1. phpMyAdmin → root password változtatás
2. Tomcat → manager user létrehozása (nem admin)
3. SSH → disable root login
4. Database → remove anonymous users
```

---

### 5. **Restrict Admin Access**
```bash
# SSH only from specific IP
iptables -A INPUT -p tcp --dport 22 -s 203.0.113.5 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP

# Apache .htaccess (admin directory)
Order Deny,Allow
Deny from all
Allow from 203.0.113.5
```

---

## Fontos Megjegyzések

✅ **Configuration management = kritikus** minden infrastruktúrán  
✅ **Egy sebezhető komponens** = egész alkalmazás veszélyben  
✅ **EOL software = CRITICAL RISK** - azonnal frissítsd!  
✅ **Default credentials** = gyakori probléma  
❌ **Database portok NE legyenek publikusak!**  
⚠️ **False positives** gyakori vulnerability scanner-ekben (backported patches)  
⚠️ **Admin interfészek** legyenek IP-restricted!

---

**Összefoglalva:** Ez a fejezet az **infrastruktúra konfiguráció teszteléséről** szól. **Port scan-nel** (nmap) azonosítod a service-eket, **vulnerability scanner-rel** (Nessus, OpenVAS) keresd az ismert CVE-ket, **admin interfészeket** (phpMyAdmin, Webmin) brute-force-old (gobuster), és teszteld **default credentials**-szel. **Elavult szoftver**, **nyitott database portok**, és **default jelszavak** mind **kritikus kockázatok**. A **patch management** és **port management** proper karbantartása alapvető biztonsági követelmény!
