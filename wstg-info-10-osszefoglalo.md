# WSTG-INFO-10: Alkalmazás Architektúra Feltérképezése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **meg kell érteni az alkalmazás teljes architektúráját** - milyen komponensek vannak használatban (webszerver, adatbázis, load balancer, CDN, WAF, stb.), hogyan kapcsolódnak össze, és **mi van scope-on belül** és mi kívül a teszt szempontjából.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM konkrét komponens fingerprinting! Ez a **teljes rendszer architektúrájának megértése** - egy big picture, hogy **hogyan épül fel az infrastruktúra**.

---

## Mi a cél?

**Az alkalmazás architektúrájának és használt technológiáknak a megértése**:
- Application komponensek (webserver, database, PaaS, serverless)
- Network komponensek (reverse proxy, load balancer, CDN)
- Security komponensek (firewall, IDS/IPS, WAF)
- Third-party services (APIs, payment gateways, social media)

### Miért fontos?

- **Scope meghatározás** - Mi van tesztelve és mi nem?
- **Attack surface** - Hol lehet támadni?
- **Bypass lehetőségek** - CDN/WAF bypass?
- **Proper reporting** - Kinek szól a recommendation? (PaaS-nál nem a kliensnek)
- **Tesztelési stratégia** - Más technikák kellenek microservice vs monolith app-hoz

---

## 3 Fő Komponens Kategória

### 1. **Application Components** (Alkalmazás Komponensek)
### 2. **Network Components** (Hálózati Komponensek)
### 3. **Security Components** (Biztonsági Komponensek)

---

## 1. Application Components

### 1.1 Web Server

**Concept:** Hagyományos webszerver (Apache, nginx, IIS).

**Hogyan azonosítsd:**
- Lásd: [WSTG-INFO-02 Webserver Fingerprinting](02-Fingerprint_Web_Server.md)

**Parancs:**
```bash
curl -I https://pelda.hu | grep -i server
```

**Kimenet:**
```
Server: nginx/1.18.0
```

---

### 1.2 Platform-as-a-Service (PaaS)

**Concept:** A szolgáltató kezeli az infrastruktúrát, te csak az alkalmazást deploy-olod (pl. Heroku, Azure App Service, Google App Engine).

**Jellemzők:**
- ✅ Nincs access az underlying infrastructure-hoz
- ✅ Infrastructure testing **OUT OF SCOPE**
- ✅ Csak az alkalmazás kódjáért felelős a kliens

**Hogyan azonosítsd:**

#### Azure App Service:
```bash
# Domain pattern
curl -I https://myapp.azurewebsites.net

# X-Powered-By header
curl -I https://pelda.hu | grep -i x-powered-by
```

**Kimenet:**
```
X-Powered-By: ASP.NET
# És a domain: *.azurewebsites.net
```

---

#### Heroku:
```bash
curl -I https://myapp.herokuapp.com
```

**Kimenet:**
```
Via: 1.1 vegur
Server: Cowboy
```

**Azonosítás:** `Via: vegur` = Heroku proxy

---

#### Google App Engine:
```bash
curl -I https://myapp.appspot.com
```

**Kimenet:**
```
Server: Google Frontend
X-Cloud-Trace-Context: ...
```

---

**Parancs összefoglaló:**
```bash
# Check domain patterns
echo "pelda.hu" | grep -E "azurewebsites|herokuapp|appspot|elasticbeanstalk"

# Check headers
curl -I https://pelda.hu | grep -iE "via|x-cloud|x-powered-by"
```

---

### 1.3 Serverless

**Concept:** Funkció-alapú architektúra (AWS Lambda, Azure Functions, Google Cloud Functions).

**Jellemzők:**
- ✅ Nincs hagyományos webserver
- ✅ Individual functions futnak
- ✅ Microservice-friendly
- ✅ Infrastructure testing **OUT OF SCOPE**

**Hogyan azonosítsd:**

#### AWS Lambda:
```bash
curl -I https://api.pelda.hu/function
```

**Kimenet:**
```http
X-Amz-Invocation-Type: RequestResponse
X-Amz-Log-Type: None
X-Amz-Request-Id: 12345-67890
X-Amzn-Trace-Id: Root=1-...
```

**Azonosítás:** `X-Amz-*` header-ek = AWS Lambda

---

#### Azure Functions:
```bash
curl -I https://myfunction.azurewebsites.net/api/function
```

**Kimenet:**
```
Server: Kestrel
```

**Figyelem:** `Server: Kestrel` önmagában **NEM elég**! Lehet bármilyen .NET Core app is.

---

**Parancs összefoglaló:**
```bash
# AWS Lambda detection
curl -I https://api.pelda.hu | grep -i "x-amz"

# Azure Functions (kevésbé megbízható)
curl -I https://pelda.hu | grep -i "kestrel"
```

---

### 1.4 Microservices

**Concept:** Az alkalmazás **több független service-ből** áll (különböző nyelveken, containerekben - gyakran Kubernetes).

**Jellemzők:**
- ✅ Több nyelv használata
- ✅ Docker/Kubernetes
- ✅ API Gateway előtt
- ✅ Service mesh (Istio, Linkerd)

**Hogyan azonosítsd:**

#### Különböző nyelvek error message-ekben:
```bash
# Trigger errors különböző endpoint-okon
curl https://api.pelda.hu/users/error
curl https://api.pelda.hu/orders/error
curl https://api.pelda.hu/payments/error
```

**Ha látod:**
- `/users` → Python traceback
- `/orders` → Java stack trace
- `/payments` → Node.js error

→ **Microservices!**

---

#### Kubernetes header-ek:
```bash
curl -I https://api.pelda.hu
```

**Kimenet:**
```
X-Forwarded-For: 10.244.0.5
X-Request-ID: abc-123
```

**Figyelem:** Internal IP (10.x.x.x) = lehet Kubernetes pod

---

#### Service Mesh header-ek (Istio):
```bash
curl -I https://api.pelda.hu
```

**Kimenet:**
```
X-Envoy-Upstream-Service-Time: 15
X-B3-TraceId: 1234567890abcdef
X-B3-SpanId: abcdef123456
```

**Azonosítás:** `X-Envoy-*`, `X-B3-*` = Istio/Envoy proxy

---

### 1.5 Static Storage (S3, Azure Blob)

**Concept:** Statikus fájlok (képek, CSS, JS) külön storage-on (S3, Azure Blob).

**Hogyan azonosítsd:**

#### Amazon S3:
```bash
# HTML source code-ban keress S3 URL-eket
curl -s https://pelda.hu | grep -oE 'https?://[^"]*s3[^"]*'

# Direct S3 URL patterns
# Pattern 1: bucket.s3.amazonaws.com
# Pattern 2: s3.region.amazonaws.com/bucket
```

**Példák:**
```
https://mybucket.s3.amazonaws.com/images/logo.png
https://s3.eu-west-1.amazonaws.com/mybucket/style.css
```

---

#### Azure Blob Storage:
```bash
# Keress Azure Blob URL-eket
curl -s https://pelda.hu | grep -oE 'https?://[^"]*blob.core.windows.net[^"]*'
```

**Példa:**
```
https://mystorageaccount.blob.core.windows.net/images/banner.jpg
```

---

**Parancs összefoglaló:**
```bash
# S3 detection
curl -s https://pelda.hu | grep -oE '(s3\.amazonaws\.com|s3-[a-z0-9-]+\.amazonaws\.com)'

# Azure Blob detection
curl -s https://pelda.hu | grep -oE 'blob\.core\.windows\.net'

# Google Cloud Storage
curl -s https://pelda.hu | grep -oE 'storage\.googleapis\.com'
```

---

### 1.6 Database

**Concept:** Adatbázis backend (MySQL, PostgreSQL, MSSQL, MongoDB, stb.).

**Hogyan azonosítsd:**

#### Port Scanning:
```bash
# Nmap port scan
nmap -sV -p 1433,3306,5432,27017,6379 pelda.hu

# Specifikus adatbázis portok:
# 1433 - Microsoft SQL Server
# 3306 - MySQL
# 5432 - PostgreSQL
# 27017 - MongoDB
# 6379 - Redis
# 1521 - Oracle
```

---

#### Error Messages Trigger:
```bash
# SQL injection karakterek error trigger-hez
curl "https://pelda.hu/product.php?id=1'"

# NoSQL injection
curl "https://pelda.hu/api/user?id[$ne]=null"
```

**MySQL Error példa:**
```
You have an error in your SQL syntax; check the manual that 
corresponds to your MySQL server version...
```

**PostgreSQL Error:**
```
ERROR: syntax error at or near "'" 
LINE 1: SELECT * FROM users WHERE id='1''
```

**MongoDB Error:**
```
MongoError: invalid operator: $ne
```

---

#### Educated Guess (Platform alapján):

| Platform | Tipikus Adatbázis |
|----------|-------------------|
| Windows + IIS + ASP.NET | **Microsoft SQL Server** |
| PHP | **MySQL** vagy **PostgreSQL** |
| Python (Django) | **PostgreSQL** |
| Ruby on Rails | **PostgreSQL** |
| Node.js | **MongoDB** vagy **PostgreSQL** |
| Java | **Oracle** vagy **PostgreSQL** |
| Embedded systems | **SQLite** |
| Oracle APEX | **Oracle Database** |

---

### 1.7 Authentication

**Concept:** Hogyan történik a user authentication?

**Típusok:**

#### 1. HTTP Basic Authentication:
```bash
curl -I https://pelda.hu/admin
```

**Kimenet:**
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Admin Area"
```

**Azonosítás:** Browser popup + `WWW-Authenticate: Basic`

---

#### 2. NTLM Authentication (Active Directory):
```bash
curl -I https://pelda.hu/admin
```

**Kimenet:**
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM
```

**Vagy:**
- Username format: `DOMAIN\username`
- Domain dropdown a login form-on

---

#### 3. LDAP Authentication:
**Azonosítás:**
- Form-based login
- Enterprise environment
- Nincs user registration (csak admin add-ol user-t)

---

#### 4. Single Sign-On (SSO):
```bash
# OAuth/OpenID Connect
curl https://pelda.hu/login

# Redirect SSO provider-hez
# Example: https://accounts.google.com/...
# Example: https://login.microsoftonline.com/...
```

**Azonosítás:**
- Redirect external provider-hez (Google, Microsoft, Okta)
- "Login with Google/Facebook/GitHub" gombok
- SAML request/response (XML)

**SAML detection:**
```bash
curl -s https://pelda.hu/login | grep -i "SAMLRequest\|SAMLResponse"
```

---

### 1.8 Third Party Services

**Concept:** External resources (scripts, APIs, images, stb.).

**Hogyan azonosítsd:**

#### Browser DevTools:
```
1. F12 → Network tab
2. Reload page
3. Nézd meg az összes request-et
4. Filter: Third-party
```

**Vagy curl + grep:**
```bash
# External scripts
curl -s https://pelda.hu | grep -oE 'src="https?://[^"]*"' | grep -v pelda.hu

# External stylesheets
curl -s https://pelda.hu | grep -oE 'href="https?://[^"]*\.css"' | grep -v pelda.hu

# External images
curl -s https://pelda.hu | grep -oE 'src="https?://[^"]*\.(jpg|png|gif)"' | grep -v pelda.hu
```

---

**Gyakori Third-Party Services:**

| Service | Domain Pattern | Purpose |
|---------|----------------|---------|
| Google Analytics | `google-analytics.com`, `googletagmanager.com` | Analytics |
| Facebook Pixel | `facebook.net`, `connect.facebook.net` | Tracking |
| Stripe | `js.stripe.com` | Payment |
| reCAPTCHA | `google.com/recaptcha` | Bot protection |
| Cloudflare | `cdnjs.cloudflare.com` | CDN |
| jQuery CDN | `ajax.googleapis.com` | JavaScript library |

**FONTOS:** Third-party services **általában OUT OF SCOPE!**

---

## 2. Network Components

### 2.1 Reverse Proxy

**Concept:** Frontend proxy a backend server(ek) előtt.

**Funkciók:**
- Load balancing
- SSL termination
- Caching
- WAF
- Multiple apps on one IP

**Hogyan azonosítsd:**

#### Server Header Mismatch:
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
Server: nginx/1.18.0
X-Powered-By: ASP.NET
```

**Azonosítás:** nginx frontend + ASP.NET backend = **Reverse Proxy!**

---

#### Duplicate Headers:
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
Server: nginx
Server: Apache/2.4.41
```

**Azonosítás:** Két `Server` header = Reverse Proxy

---

#### Multiple Apps on Same IP:
```bash
# Same IP, different apps
curl -I https://app1.pelda.hu | grep Server
curl -I https://app2.pelda.hu | grep Server
```

**Ha:**
- `app1` → PHP
- `app2` → ASP.NET
- Same IP

→ **Reverse Proxy!**

---

### 2.2 Load Balancer

**Concept:** Több backend server közötti terheléselosztás.

**Hogyan azonosítsd:**

#### Multiple Requests → Different Responses:
```bash
# 10 request és nézd a különbségeket
for i in {1..10}; do
    curl -I https://pelda.hu | grep -E "Date|Server"
    sleep 1
done
```

**Mit keress:**
- **Inconsistent system times** (másodperc eltérés)
- **Different internal IPs** error message-ekben
- **Different Server header values**

---

#### Load Balancer Cookies:

**F5 BIG-IP:**
```bash
curl -I https://pelda.hu | grep -i cookie
```

**Kimenet:**
```
Set-Cookie: BIGipServerPOOL_NAME=...
```

**Azonosítás:** `BIGipServer*` cookie = F5 BIG-IP load balancer

---

**HAProxy:**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
Server: HAProxy/2.2.0
```

---

#### SSRF-based Detection:
Ha van SSRF vulnerability:
```bash
# Trigger SSRF to localhost
# Backend server IP lehet más minden request-nél
```

---

### 2.3 Content Delivery Network (CDN)

**Concept:** Geographically distributed caching proxy-k (Cloudflare, Akamai, Fastly).

**Hogyan azonosítsd:**

#### WHOIS Lookup:
```bash
# DNS resolve
dig pelda.hu +short

# WHOIS check
whois 104.26.10.123
```

**Kimenet példa:**
```
OrgName: Cloudflare, Inc.
```

**Azonosítás:** IP belongs to Cloudflare → **CDN!**

---

#### Specific Headers:

**Cloudflare:**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
CF-Ray: 1234567890abc-VIE
CF-Cache-Status: HIT
Server: cloudflare
```

---

**Akamai:**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
X-Akamai-Request-ID: 1234567
X-Cache: HIT from akamai.net
```

---

**Fastly:**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
X-Served-By: cache-vie21632-VIE
X-Cache: HIT
Fastly-Stats: ...
```

---

#### CDN Bypass (Backend Server Discovery):

**Módszerek:**

**1. Email Headers:**
```bash
# Regisztráció után nézd az email header-ét
# Lehet, hogy direct from backend:
Received: from backend-server.pelda.hu (192.168.1.100)
```

---

**2. DNS History / Certificate Transparency:**
```bash
# DNS history check
curl "https://crt.sh/?q=%.pelda.hu&output=json"

# Historical DNS
curl "https://securitytrails.com/domain/pelda.hu/dns"
```

---

**3. Subdomain Scan:**
```bash
# Subdomains may not be behind CDN
amass enum -d pelda.hu

# Direct access to origin
curl -I https://origin.pelda.hu
```

---

**4. SSRF Exploit:**
Ha van SSRF:
```
# Internal IP disclosure
```

---

**FONTOS:**
- ✅ CDN IP-k **OUT OF SCOPE** infrastructure testing-hez
- ✅ CDN gyakran WAF-ot is tartalmaz
- ✅ Caching miatt változások nem azonnal látszanak

---

## 3. Security Components

### 3.1 Network Firewall

**Concept:** Packet filtering firewall (iptables, Windows Firewall, hardware firewall).

**Hogyan azonosítsd:**

#### Port Scan:
```bash
# Nmap teljes port scan
nmap -p- pelda.hu

# Check port states
nmap -sS -p1-1000 pelda.hu
```

**Port States:**
- **Open** - Port nyitva és szolgáltatás válaszol
- **Closed** - RST packet visszajön → Nincs firewall
- **Filtered** - Nincs válasz → **Firewall!**

**Ha:**
- Majority = **Closed** → Valószínűleg nincs firewall
- Majority = **Filtered** → **Firewall van!**

---

#### Exposed Services Check:
```bash
nmap -sV -p 25,110,143,3306,1433,5432 pelda.hu
```

**Ha látod:**
- Port 25 (SMTP) open
- Port 3306 (MySQL) open
- Port 1433 (MSSQL) open

→ **Rossz firewall konfig vagy nincs firewall!**

---

### 3.2 Intrusion Detection/Prevention System (IDS/IPS)

**Concept:** Network-level suspicious activity detection (Snort, Suricata).

**IDS:** Csak alert-el
**IPS:** Blokkol is (source IP)

**Hogyan azonosítsd:**

#### Automated Scan + IP Block:
```bash
# Nmap aggressive scan
nmap -A -T4 pelda.hu

# Várj pár másodpercet, próbálj újra
curl https://pelda.hu
```

**Ha:**
- Connection timeout
- Connection refused
- 403 Forbidden minden request-re

→ **IPS blokkolt!**

---

#### Behavior Change:
```bash
# Normál browsing OK
curl https://pelda.hu

# Vulnerability scan
nikto -h https://pelda.hu

# Utána normál browsing blocked
curl https://pelda.hu
```

→ **IPS trigger!**

---

### 3.3 Web Application Firewall (WAF)

**Concept:** HTTP request inspection és malicious request blokkolás (ModSecurity, Cloudflare WAF, AWS WAF).

**Hogyan azonosítsd:**

#### Attack String Injection:
```bash
# SQL injection pattern
curl "https://pelda.hu/product?id=1' UNION SELECT 1--"

# XSS pattern
curl "https://pelda.hu/search?q=<script>alert(1)</script>"

# Path traversal
curl "https://pelda.hu/file?path=../../../etc/passwd"

# Command injection
curl "https://pelda.hu/ping?host=127.0.0.1;cat /etc/passwd"
```

**Ha blokkolva:**
- 403 Forbidden
- Custom block page
- WAF-specific error message

→ **WAF van!**

---

#### WAF-Specific Headers/Cookies:

**ModSecurity:**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
Server: Apache/2.4.41 (Unix) mod_security/2.9.3
```

---

**Cloudflare WAF:**
```bash
curl "https://pelda.hu/?test=<script>"
```

**Block page:**
```html
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
```

---

**AWS WAF:**
```bash
curl "https://pelda.hu/?xss=<script>"
```

**Kimenet:**
```
X-Amzn-Waf-Action: BLOCK
X-Amzn-RequestId: ...
```

---

**Imperva (Incapsula):**
```bash
curl -I https://pelda.hu
```

**Kimenet:**
```
X-CDN: Incapsula
Set-Cookie: incap_ses_...
```

---

#### WAF Fingerprinting Tool: **wafw00f**

```bash
# Telepítés
pip3 install wafw00f

# Használat
wafw00f https://pelda.hu

# Aggressive mode
wafw00f -a https://pelda.hu
```

**Kimenet példa:**
```
[*] Checking https://pelda.hu
[+] The site https://pelda.hu is behind Cloudflare (Cloudflare Inc.) WAF.
```

---

#### WAF Bypass (Origin Server Access):

Ugyanazok a technikák mint CDN bypass-nál:
- Email headers → origin IP
- DNS history
- Subdomain scanning
- Direct IP access

**Ha megtalálod az origin IP-t:**
```bash
# Bypass WAF
curl -H "Host: pelda.hu" http://ORIGIN_IP/
```

---

## Architecture Mapping Workflow

### Teljes Feltérképezési Folyamat:

```bash
#!/bin/bash

TARGET="pelda.hu"
OUTPUT="architecture_map.txt"

echo "=== Architecture Mapping: $TARGET ===" > $OUTPUT

# 1. DNS & IP Resolution
echo -e "\n[+] DNS Resolution:" >> $OUTPUT
dig $TARGET +short >> $OUTPUT
whois $(dig $TARGET +short | head -1) | grep -i "OrgName\|Organization" >> $OUTPUT

# 2. Web Server Detection
echo -e "\n[+] Web Server:" >> $OUTPUT
curl -I https://$TARGET | grep -iE "server|x-powered-by|x-generator" >> $OUTPUT

# 3. PaaS Detection
echo -e "\n[+] PaaS Detection:" >> $OUTPUT
if echo $TARGET | grep -qE "azurewebsites|herokuapp|appspot"; then
    echo "  [!] PaaS detected in domain!" >> $OUTPUT
fi

# 4. Serverless Detection
echo -e "\n[+] Serverless Detection:" >> $OUTPUT
curl -I https://$TARGET | grep -i "x-amz\|kestrel" >> $OUTPUT

# 5. Static Storage
echo -e "\n[+] Static Storage:" >> $OUTPUT
curl -s https://$TARGET | grep -oE 's3\.amazonaws\.com|blob\.core\.windows\.net' | head -5 >> $OUTPUT

# 6. Load Balancer Cookies
echo -e "\n[+] Load Balancer:" >> $OUTPUT
curl -I https://$TARGET | grep -i "BIGipServer\|haproxy" >> $OUTPUT

# 7. CDN Detection
echo -e "\n[+] CDN Detection:" >> $OUTPUT
curl -I https://$TARGET | grep -iE "cf-ray|x-akamai|fastly|x-cache" >> $OUTPUT

# 8. WAF Detection
echo -e "\n[+] WAF Detection:" >> $OUTPUT
curl -s "https://$TARGET/?test=<script>" | grep -i "cloudflare\|incapsula\|akamai\|blocked" >> $OUTPUT

# 9. Third-Party Services
echo -e "\n[+] Third-Party Services:" >> $OUTPUT
curl -s https://$TARGET | grep -oE 'https?://[^"]*' | grep -vE "$TARGET" | sort -u | head -10 >> $OUTPUT

echo -e "\n[*] Architecture mapping complete!"
cat $OUTPUT
```

---

## Gyakorlati Cheat Sheet

| Component | Detection Command |
|-----------|------------------|
| Web Server | `curl -I https://pelda.hu \| grep Server` |
| PaaS | Check domain (azurewebsites.net, herokuapp.com) |
| Serverless | `curl -I https://pelda.hu \| grep X-Amz` |
| S3 Storage | `curl -s https://pelda.hu \| grep s3.amazonaws` |
| Load Balancer | `curl -I https://pelda.hu \| grep BIGipServer` |
| CDN | `whois $(dig pelda.hu +short)` |
| Firewall | `nmap -p- pelda.hu` (check filtered ports) |
| WAF | `curl "https://pelda.hu/?x=<script>"` |
| Reverse Proxy | Check Server header mismatch |

---

## Architecture Diagram Example

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │   Cloudflare   │  (CDN + WAF)
              │   104.26.x.x   │
              └────────┬───────┘
                       │
                       ▼
              ┌────────────────┐
              │  AWS WAF       │  (Optional extra WAF)
              └────────┬───────┘
                       │
                       ▼
         ┌─────────────────────────┐
         │   Load Balancer (F5)    │
         │   BIGipServer cookie    │
         └─────────┬───────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ Web 1  │ │ Web 2  │ │ Web 3  │  (nginx reverse proxy)
   │nginx   │ │nginx   │ │nginx   │
   └───┬────┘ └───┬────┘ └───┬────┘
       │          │          │
       ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ App 1  │ │ App 2  │ │ App 3  │  (PHP-FPM)
   │PHP     │ │PHP     │ │PHP     │
   └───┬────┘ └───┬────┘ └───┬────┘
       │          │          │
       └──────────┼──────────┘
                  ▼
          ┌───────────────┐
          │    MySQL      │  (Database)
          │   Port 3306   │
          └───────────────┘
                  │
                  ▼
          ┌───────────────┐
          │   Redis       │  (Cache)
          │   Port 6379   │
          └───────────────┘

External:
  - S3 (static files): mybucket.s3.amazonaws.com
  - Stripe API (payment): js.stripe.com
  - Google Analytics: google-analytics.com
```

---

## Scope Determination

### IN SCOPE:
✅ Web Application code
✅ Application-level vulnerabilities
✅ Business logic
✅ Authentication/Authorization
✅ Input validation

### OUT OF SCOPE (általában):
❌ CDN infrastructure (Cloudflare IP-k)
❌ Third-party services (Stripe, Google Analytics)
❌ PaaS infrastructure (Azure, Heroku)
❌ Serverless platform (AWS Lambda infrastruktúra)
❌ Shared hosting provider

### GRAY AREA (client-tel egyeztetni):
⚠️ Load balancer config
⚠️ WAF rules
⚠️ Firewall rules
⚠️ SSL/TLS configuration
⚠️ Origin server (CDN bypass)

---

## Fontos Megjegyzések

✅ **Architecture mapping = első lépés** minden pentest előtt  
✅ **Scope tisztázása kritikus** - mi van benne, mi nincs  
✅ **CDN/WAF bypass** gyakran lehetséges (origin server discovery)  
✅ **Third-party OUT OF SCOPE** - de azonosítsd és dokumentáld!  
❌ **Ne teszteld a PaaS infrastructure-t** - nem a kliensé  
⚠️ **Multiple requests** kellenek load balancer detektálásához!  

---

**Összefoglalva:** Ez a fejezet az **alkalmazás teljes architektúrájának feltérképezéséről** szól. Azonosítanod kell **minden komponenst**: webserver, PaaS/serverless, database, load balancer, CDN, WAF, third-party services. Ez **meghatározza a scope-ot** (mi van tesztelve), a **támadási felületet**, és a **bypass lehetőségeket** (pl. CDN/WAF bypass origin server-en keresztül). **Black-box testing-ben** header-ek, cookie-k, error message-ek, és port scan alapján építed fel a big picture-t!
