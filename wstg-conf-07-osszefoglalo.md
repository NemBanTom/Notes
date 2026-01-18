# WSTG-CONF-07: HTTP Strict Transport Security (HSTS) Tesztelése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **HSTS header** arra utasítja a böngészőt, hogy **soha ne használjon HTTP-t**, csak **HTTPS-t** az adott domain-hez. Ha a **HSTS hiányzik**, akkor **MITM attackok**, **session hijacking**, és **downgrade attackok** lehetségesek.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM általános HTTPS/SSL testing, hanem specifikusan **HSTS header** ellenőrzése - van-e, helyesen van-e konfigurálva (`max-age`, `includeSubDomains`, `preload`).

---

## Mi a cél?

**HSTS header jelenlétének és validitásának ellenőrzése:**
- Van-e HSTS header
- `max-age` értéke megfelelő-e
- `includeSubDomains` direktíva van-e
- `preload` direktíva van-e

---

## HSTS Header Felépítése

### Syntax:
```
Strict-Transport-Security: max-age=<seconds>; includeSubDomains; preload
```

---

### Directives:

#### 1. **`max-age=<seconds>`** (KÖTELEZŐ)

**Concept:** Hány másodpercig jegyezze meg a böngésző, hogy csak HTTPS-t használjon.

**Értékek:**
```
max-age=31536000  → 1 év (recommended)
max-age=63072000  → 2 év
max-age=0         → HSTS disable (átmeneti)
```

**Minimum recommended:** `31536000` (1 év)

---

#### 2. **`includeSubDomains`** (OPCIONÁLIS, de ajánlott)

**Concept:** Subdomain-ek is HTTPS-only.

**Példa:**
```
Domain: example.com
HSTS header: Strict-Transport-Security: max-age=31536000; includeSubDomains

Covered:
- example.com → HTTPS
- www.example.com → HTTPS
- api.example.com → HTTPS
- blog.example.com → HTTPS
```

**FIGYELEM:** Ha van subdomain **csak HTTP-n**, akkor nem használhatod `includeSubDomains`-t!

---

#### 3. **`preload`** (OPCIONÁLIS, unofficial)

**Concept:** Domain bekerül a **HSTS preload list**-be, amit a böngészők beépítenek.

**Preload list:**
- Chromium, Firefox, Safari, Edge beépített listája
- Böngésző **soha nem** próbál HTTP connection-t, még első alkalommal sem

**Submission:**
- https://hstspreload.org/
- Requirements: `max-age` >= 31536000, `includeSubDomains`, `preload`

---

### Példa HSTS Header-ek:

#### Minimal (elfogadható):
```
Strict-Transport-Security: max-age=31536000
```

#### Recommended:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

#### Best practice (preload):
```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

---

## Veszélyek HSTS Nélkül

### 1. **SSL Strip Attack (MITM)**

**Scenario:**
```
1. User types: example.com (HTTP)
2. Browser → HTTP request
3. Attacker intercepts
4. Attacker → user: HTTP response
5. User stays on HTTP (no HTTPS upgrade!)
```

**HSTS-vel:**
```
1. User types: example.com
2. Browser checks HSTS cache
3. Browser → HTTPS request (automatic upgrade)
4. Attacker cannot downgrade to HTTP
```

---

### 2. **Cookie Hijacking**

**Without HSTS:**
```
User sends cookie via HTTP:
Cookie: sessionid=abc123

Attacker sniffs:
sessionid=abc123 → Session hijacking!
```

**With HSTS:**
```
Browser refuses HTTP connection
All traffic via HTTPS (encrypted)
Cookie safe
```

---

### 3. **Certificate Error Override**

**Without HSTS:**
```
User sees certificate error
User clicks "Proceed anyway"
MITM attack succeeds
```

**With HSTS:**
```
Browser sees certificate error
Browser blocks connection (no override option!)
MITM attack prevented
```

---

## Tesztelési Módszerek

### Tool #1: **curl**

#### Parancsok:
```bash
# Basic HSTS check
curl -s -D- https://pelda.hu | grep -i strict

# Csak header
curl -I https://pelda.hu | grep -i strict-transport-security

# Verbose
curl -v https://pelda.hu 2>&1 | grep -i strict
```

---

**Output példák:**

**Good (HSTS enabled):**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Warning (short max-age):**
```
Strict-Transport-Security: max-age=86400
```
→ Only 1 day! Should be 31536000 (1 year)

**Bad (missing HSTS):**
```
(no Strict-Transport-Security header)
```

---

### Tool #2: **Browser DevTools**

#### Használat:
```
1. F12 → Network tab
2. Navigate to https://pelda.hu
3. Click first request
4. Headers → Response Headers
5. Look for: Strict-Transport-Security
```

**Screenshot:**
```
Response Headers:
Content-Type: text/html
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Frame-Options: DENY
```

---

### Tool #3: **Online Checkers**

#### SecurityHeaders.com:
```
URL: https://securityheaders.com/?q=pelda.hu&followRedirects=on

Results:
✓ Strict-Transport-Security: A+
  max-age=31536000
  includeSubDomains
  preload
```

---

#### SSL Labs:
```
URL: https://www.ssllabs.com/ssltest/analyze.html?d=pelda.hu

HSTS:
✓ Yes
  max-age=31536000
  includeSubDomains: Yes
  preload: Yes
```

---

### Tool #4: **Nmap NSE Script**

#### Parancs:
```bash
nmap --script http-security-headers pelda.hu -p 443
```

**Output:**
```
PORT    STATE SERVICE
443/tcp open  https
| http-security-headers:
|   Strict-Transport-Security:
|     Header: Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## HSTS Header Evaluation

### Best Practice Values:

| Directive | Recommended Value | Minimum |
|-----------|-------------------|---------|
| `max-age` | `31536000` (1 year) | `10886400` (126 days) |
| `includeSubDomains` | Yes | - |
| `preload` | Yes (if eligible) | - |

---

### Scoring:

**A+ (Perfect):**
```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

**A (Good):**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**B (Acceptable):**
```
Strict-Transport-Security: max-age=31536000
```

**C (Weak):**
```
Strict-Transport-Security: max-age=86400
```
→ max-age too short

**F (Missing):**
```
(no HSTS header)
```

---

## Common Misconfigurations

### 1. **Missing HSTS Header:**

```bash
curl -I https://pelda.hu | grep -i strict
# (no output)
```

**Impact:** No HSTS protection!

---

### 2. **Too Short max-age:**

```
Strict-Transport-Security: max-age=300
```

**Impact:** Only 5 minutes! After expiry, vulnerable again.

---

### 3. **HSTS on HTTP:**

```bash
curl -I http://pelda.hu | grep -i strict
Strict-Transport-Security: max-age=31536000
```

**Impact:** HSTS header **CSAK HTTPS-en** érvényes! HTTP-n elküldve **ignored by browser**.

---

### 4. **includeSubDomains but HTTP subdomain exists:**

```
Domain: example.com
HSTS: max-age=31536000; includeSubDomains

But: http://old.example.com (no HTTPS support)
```

**Impact:** Users cannot access `old.example.com` (browser blocks)!

---

### 5. **Preload without requirements:**

```
Strict-Transport-Security: max-age=86400; preload
```

**Impact:** Not eligible for preload (max-age < 31536000, no includeSubDomains).

---

## HSTS Preload List

### Concept:
**Browsers beépített listája** domain-ekről, amelyek **mindig HTTPS**.

### Benefits:
- **First visit** is védett (no HTTP ever)
- No trust-on-first-use (TOFU) problem

### Requirements:
1. Valid certificate
2. Redirect HTTP → HTTPS (same host)
3. All subdomains serve HTTPS
4. `max-age` >= 31536000 (1 year)
5. `includeSubDomains` directive
6. `preload` directive

---

### Submission:

**URL:** https://hstspreload.org/

**Check eligibility:**
```
https://hstspreload.org/?domain=pelda.hu
```

**Submit:**
```
1. Ensure requirements met
2. Submit domain
3. Wait for approval (weeks/months)
4. Domain added to Chromium source
5. Propagates to other browsers
```

**⚠️ FIGYELEM:**
- **Removal nehéz** (months)
- Gondold át előre!

---

## Testing Workflow

### 1. **Basic HSTS Check:**
```bash
curl -I https://pelda.hu | grep -i strict-transport-security
```

**Expected:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

### 2. **Verify max-age:**
```bash
curl -I https://pelda.hu | grep strict-transport-security
```

**Check:**
- >= 31536000? ✓
- < 31536000? ✗

---

### 3. **Check includeSubDomains:**
```bash
curl -I https://pelda.hu | grep includeSubDomains
```

**Present?** ✓  
**Missing?** ⚠️

---

### 4. **Test Subdomain:**
```bash
curl -I https://www.pelda.hu | grep strict-transport-security
curl -I https://api.pelda.hu | grep strict-transport-security
```

**Same header?** ✓

---

### 5. **HTTP Check (should redirect):**
```bash
curl -I http://pelda.hu
```

**Expected:**
```
HTTP/1.1 301 Moved Permanently
Location: https://pelda.hu/
```

---

### 6. **Preload Check:**
```bash
curl -I https://pelda.hu | grep preload
```

**Present?** ✓ (eligible for preload list)

---

## Implementation

### Apache:

**httpd.conf vagy .htaccess:**
```apache
# HSTS Header
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
```

**With mod_headers:**
```apache
<IfModule mod_headers.c>
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" env=HTTPS
</IfModule>
```

---

### nginx:

**nginx.conf:**
```nginx
server {
    listen 443 ssl;
    
    # HSTS Header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

---

### IIS:

**web.config:**
```xml
<system.webServer>
  <httpProtocol>
    <customHeaders>
      <add name="Strict-Transport-Security" value="max-age=31536000; includeSubDomains; preload"/>
    </customHeaders>
  </httpProtocol>
</system.webServer>
```

---

### Node.js (Express):

```javascript
const helmet = require('helmet');

app.use(helmet.hsts({
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
}));
```

---

## Browser Behavior

### HSTS Cache:

**Chrome:**
```
chrome://net-internals/#hsts

Query HSTS/PKP domain:
Domain: pelda.hu

Result:
static_sts_domain: pelda.hu
static_upgrade_mode: FORCE_HTTPS
static_sts_include_subdomains: true
```

---

**Firefox:**
```
about:config

Search: network.stricttransportsecurity

Site Security Service State:
pelda.hu:HSTS
```

---

### Clear HSTS Cache (Testing):

**Chrome:**
```
chrome://net-internals/#hsts
→ Delete domain security policies
→ Domain: pelda.hu
→ Delete
```

**Firefox:**
```
about:preferences#privacy
→ Clear Data
→ Site Settings
→ Remove pelda.hu
```

---

## Gyakorlati Cheat Sheet

| Teszt | Parancs |
|-------|---------|
| HSTS check | `curl -I https://pelda.hu \| grep strict` |
| max-age értéke | `curl -I https://pelda.hu \| grep max-age` |
| includeSubDomains | `curl -I https://pelda.hu \| grep includeSubDomains` |
| preload | `curl -I https://pelda.hu \| grep preload` |
| HTTP redirect | `curl -I http://pelda.hu` |
| SecurityHeaders.com | `https://securityheaders.com/?q=pelda.hu` |
| SSL Labs | `https://www.ssllabs.com/ssltest/analyze.html?d=pelda.hu` |
| HSTS preload check | `https://hstspreload.org/?domain=pelda.hu` |

---

## Fontos Toolok

### Manual Testing:
- **curl** - Command line
- **Browser DevTools** - F12 → Network

### Online Scanners:
- **SecurityHeaders.com** - Header checker
- **SSL Labs** - Comprehensive SSL test
- **hstspreload.org** - Preload eligibility

### Automated:
- **Nmap** - http-security-headers script
- **testssl.sh** - SSL/TLS tester

---

## Védelem (Remediation)

### 1. **Enable HSTS:**
```apache
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
```

---

### 2. **Proper max-age:**
```
Minimum: 31536000 (1 year)
Recommended: 63072000 (2 years)
```

---

### 3. **includeSubDomains:**
```
Only if ALL subdomains support HTTPS!
```

---

### 4. **Redirect HTTP → HTTPS:**
```apache
# Apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
```

```nginx
# nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

---

### 5. **Consider Preload:**
```
Requirements:
✓ Valid certificate
✓ All subdomains HTTPS
✓ max-age >= 31536000
✓ includeSubDomains
✓ preload directive
```

---

## Fontos Megjegyzések

✅ **HSTS** megakadályozza **SSL strip attack**-ot  
✅ **max-age** minimum **31536000** (1 év)  
✅ **includeSubDomains** ajánlott (ha minden subdomain HTTPS)  
✅ **preload** extra védelem (browser built-in list)  
❌ **HSTS HTTP-n** = ignored by browser!  
⚠️ **includeSubDomains** + HTTP subdomain = users blocked!  
⚠️ **Preload removal** nehéz (months)  

---

**Összefoglalva:** Ez a fejezet a **HSTS header** teszteléséről szól. **HSTS** arra utasítja a böngészőt, hogy **csak HTTPS-t használjon** egy adott domain-hez, megakadályozva **SSL strip attack**-okat és **certificate error override**-okat. **Három directive** létezik: **`max-age`** (minimum 31536000 = 1 év), **`includeSubDomains`** (subdomain-ek is HTTPS-only), és **`preload`** (browser beépített listája). **curl**-lel vagy **browser DevTools**-zal ellenőrizhető a header jelenléte. **SecurityHeaders.com** és **SSL Labs** online checker-ek comprehensive audit-ot adnak. **HSTS hiánya** = **MITM vulnerability**!
