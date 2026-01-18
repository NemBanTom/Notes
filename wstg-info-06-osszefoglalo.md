# WSTG-INFO-06: Alkalmazás Belépési Pontok Azonosítása

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **minden HTTP request és response elemzésével azonosítsd az alkalmazás támadási felületét** (attack surface). Meg kell találni **MINDEN paramétert, headert, cookie-t** és injection pontot, mert ezek mindegyike potenciális sebezhetőség lehet.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM a sebezhetőségek tesztelése! Ez a **mapping fázis** - fel kell térképezni az összes input pontot, MIELŐTT tesztelnéd őket (SQL injection, XSS, stb.).

---

## Mi a cél?

**Azonosítani az összes lehetséges belépési és injection pontot** request és response elemzéssel:
- Minden HTTP paraméter (GET, POST, Cookie, Header)
- Rejtett form mezők
- Custom headerek
- RESTful endpoint-ok
- WebSocket kapcsolatok

### Miért fontos?

- **Ha nem találod meg az összes paramétert**, hiányos lesz a teszt
- **Rejtett paraméterek** (hidden fields) gyakran érzékeny adatokat tartalmaznak
- **Custom headerek** speciális funkciókat triggerelhetnek
- **Multi-step process-ek** különböző paramétereket fogadnak el

---

## Amit Keresünk

### Requests (Bejövő kérések):
1. **GET vs POST** használat
2. **Query string paraméterek** (URL-ben)
3. **POST body paraméterek**
4. **Hidden form fields** (rejtett mezők)
5. **Custom headers** (pl. `debug: false`)
6. **Cookies**
7. **HTTP methods** (PUT, DELETE, PATCH, OPTIONS)

### Responses (Kimenő válaszok):
1. **Set-Cookie header-ek** (új cookie-k)
2. **Redirects** (3xx status code)
3. **Error codes** (400, 403, 500)
4. **Custom headers** (pl. `Server: BIG-IP`)
5. **Response body-ban található paraméterek**

---

## Eszközök és Módszerek

### Tool #1: **Burp Suite - Intercepting Proxy**

#### Használat:
```
1. Indítsd el Burp Suite-ot
2. Proxy → Intercept → Intercept is on
3. Böngészőben állítsd be a proxy-t (127.0.0.1:8080)
4. Böngéssz végig az alkalmazáson
5. Proxy → HTTP history
6. Minden request/response megjelenik
```

#### Mit nézz:
- **Target → Site map** - Teljes alkalmazás térkép
- **Proxy → HTTP history** - Minden kérés részletesen
- **Intruder** - Paraméterek tesztelésére
- **Repeater** - Kérések újraküldésére

---

### Tool #2: **OWASP ZAP (Zed Attack Proxy)**

#### Használat:
```
1. Indítsd el ZAP-ot
2. Quick Start → Manual Explore
3. Launch Browser
4. Böngéssz végig az alkalmazáson
5. History tab - Minden kérés
```

#### Mit nézz:
- **Sites** - Site tree struktúra
- **History** - Minden HTTP transaction
- **Active Scan** - Később automatikus tesztelés
- **Spider** - Automatikus crawling

---

### Tool #3: **Chrome DevTools - Network Tab**

#### Használat:
```
1. F12 → Network tab
2. Preserve log checkbox ✓
3. Disable cache checkbox ✓
4. Böngéssz végig az alkalmazáson
5. Klikk bármely request-re → Headers, Payload, Response
```

#### Előnyök:
- Beépített, nincs extra tool
- WebSocket is látszik
- Performance metrics
- Preview és Response könnyen látható

---

### Tool #4: **curl - Manual Request Testing**

#### Parancsok:
```bash
# GET request részletes output
curl -v https://pelda.hu/api/users

# POST request JSON-nel
curl -X POST https://pelda.hu/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}'

# Custom header küldése
curl https://pelda.hu/api/data \
  -H "X-Debug: true" \
  -H "X-Custom-Header: value"

# Cookie küldése
curl https://pelda.hu/profile \
  -H "Cookie: SESSIONID=abc123; user=admin"

# PUT request
curl -X PUT https://pelda.hu/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Name"}'

# DELETE request
curl -X DELETE https://pelda.hu/api/users/1
```

**Mit csinál:**
- `-v` = verbose (header-eket is mutatja)
- `-X` = HTTP method megadása
- `-H` = Header hozzáadása
- `-d` = Data (POST/PUT body)

---

## Entry Points Mapping Workflow

### 1. Lépés: Teljes Alkalmazás Bejárás

**Proxy Setup:**
```
1. Burp Suite vagy ZAP elindítása
2. Browser proxy beállítása
3. MINDEN oldal meglátogatása
4. MINDEN funkció kipróbálása
5. MINDEN form kitöltése
```

**Mit csinálj:**
- Regisztráció
- Bejelentkezés
- Profil módosítás
- Keresés
- File upload
- Delete, update műveletek
- Admin funkciók (ha van hozzáférésed)

---

### 2. Lépés: Spreadsheet Készítés

**Excel/Google Sheets táblázat oszlopai:**

| Request # | URL/Endpoint | Method | Parameters | Hidden Fields | Cookies | Custom Headers | Auth? | HTTPS? | Notes |
|-----------|--------------|--------|------------|---------------|---------|----------------|-------|--------|-------|
| 1 | /login | POST | user, pass | token | SESSIONID | - | No | Yes | Login form |
| 2 | /api/users | GET | id, limit | - | SESSIONID | X-API-Key | Yes | Yes | REST API |
| 3 | /buy | POST | item, price | customerID | SESSIONID | - | Yes | Yes | Hidden field! |

**Miért fontos:**
- Rendszerezett dokumentáció
- Később könnyebb tesztelni
- Nem veszíted el az áttekintést

---

### 3. Lépés: Paraméter Azonosítás

#### GET Request Paraméterek (Query String):

**Példa:**
```
https://pelda.hu/search?q=test&category=books&sort=price&order=asc
```

**Azonosított paraméterek:**
- `q` = keresési kifejezés
- `category` = kategória
- `sort` = rendezés típusa
- `order` = asc/desc

**Különleges elválasztók:**
```
# Standard: &
/search?a=1&b=2&c=3

# Ritka elválasztók:
/search?a=1~b=2~c=3
/search?a=1:b=2:c=3
/search?a=1;b=2;c=3
```

---

#### POST Request Paraméterek (Body):

**Tool: Burp Suite Repeater**

**Példa POST body:**
```
user=admin&password=test123&remember=true&submit=Login
```

**Content-Type típusok:**

**1. application/x-www-form-urlencoded:**
```
username=admin&password=test&role=user
```

**2. application/json:**
```json
{
  "username": "admin",
  "password": "test123",
  "rememberMe": true
}
```

**3. multipart/form-data** (file upload):
```
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="test.jpg"
Content-Type: image/jpeg

[binary data]
------WebKitFormBoundary
Content-Disposition: form-data; name="description"

Test image upload
```

---

#### Hidden Form Fields (Rejtett mezők):

**HTML példa:**
```html
<form action="/buy" method="POST">
  <input type="text" name="item" value="laptop">
  <input type="number" name="quantity" value="1">
  
  <!-- REJTETT MEZŐK -->
  <input type="hidden" name="price" value="999.99">
  <input type="hidden" name="customerID" value="12345">
  <input type="hidden" name="isAdmin" value="false">
  <input type="hidden" name="discount" value="0">
  
  <button type="submit">Buy</button>
</form>
```

**Mit csinálj:**
1. View Page Source (Ctrl+U)
2. Keress: `type="hidden"`
3. Dokumentáld az összes rejtett mezőt
4. Később teszteld: manipulálhatók-e? (price, isAdmin, stb.)

**Burp-ban:**
- Proxy → HTTP History → Request → Params tab
- Látod az ÖSSZES paramétert (hidden is)

---

#### Cookies:

**Példa Cookie header:**
```
Cookie: SESSIONID=abc123def456; UserRole=user; isLoggedIn=true; Preferences=dark-mode
```

**Azonosított cookie-k:**
- `SESSIONID` = Session azonosító
- `UserRole` = Felhasználó szerepkör ⚠️
- `isLoggedIn` = Bejelentkezve-e ⚠️
- `Preferences` = UI beállítások

**Fontos:**
- `UserRole=user` → változtatható `admin`-re?
- `isLoggedIn=true` → változtatható-e?

**Browser DevTools:**
```
F12 → Application → Cookies → https://pelda.hu
```

---

#### Custom Headers:

**Példa request:**
```http
GET /api/data HTTP/1.1
Host: pelda.hu
X-API-Key: sk-abc123
X-Debug: false
X-User-IP: 192.168.1.100
X-Forwarded-For: 1.2.3.4
Authorization: Bearer eyJhbGc...
```

**Azonosított custom headerek:**
- `X-API-Key` = API kulcs ⚠️
- `X-Debug` = Debug mode kapcsoló ⚠️
- `X-User-IP` = User IP (manipulálható?)
- `X-Forwarded-For` = Proxy IP

**Mit tesztelsz később:**
- `X-Debug: true` → extra info?
- `X-User-IP` → IP-based access control bypass?
- `X-API-Key` → valid key? leaked?

---

### 4. Lépés: HTTP Methods Azonosítás

#### OPTIONS Request (Supported Methods):

**Parancs:**
```bash
curl -X OPTIONS https://pelda.hu/api/users -v
```

**Válasz példa:**
```http
HTTP/1.1 200 OK
Allow: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
```

**Mit jelent:**
- GET, POST = Standard
- PUT, DELETE = ⚠️ Érdekes! Teszteld, hogy működnek-e!
- PATCH = Partial update
- HEAD = Csak header, no body

---

#### Ritka HTTP Methods Tesztelése:

**Parancsok:**
```bash
# PUT request
curl -X PUT https://pelda.hu/api/users/1 -d '{"name":"hacked"}' -H "Content-Type: application/json"

# DELETE request
curl -X DELETE https://pelda.hu/api/users/1

# PATCH request
curl -X PATCH https://pelda.hu/api/users/1 -d '{"email":"new@email.com"}' -H "Content-Type: application/json"

# TRACE request (http method reflection)
curl -X TRACE https://pelda.hu/
```

**Mit keress:**
- PUT/DELETE engedélyezve van authentication nélkül?
- TRACE működik? (XSS lehetőség)

---

## Response Elemzés

### Set-Cookie Header Figyelése:

**Példa response:**
```http
HTTP/1.1 200 OK
Set-Cookie: SESSIONID=xyz789; Path=/; HttpOnly; Secure
Set-Cookie: UserPref=lang=en; Path=/
Set-Cookie: AdminToken=secret123; Path=/admin
```

**Mit figyelj:**
- Új cookie-k mikor jönnek létre
- HttpOnly, Secure flag-ek vannak-e
- Session cookie értéke predictable-e?
- Érzékeny cookie-k (AdminToken)

---

### Redirect Követés:

**Példa redirect:**
```http
HTTP/1.1 302 Found
Location: https://pelda.hu/dashboard
Set-Cookie: JustLoggedIn=true
```

**Mit figyelj:**
- Redirect chain (több redirect egymás után)
- Redirect URL-ben van-e paraméter? (open redirect?)
- Redirect body-ban van-e érzékeny info?

**curl-lel:**
```bash
# Redirect követése
curl -L https://pelda.hu/login

# Redirect NE kövesse
curl https://pelda.hu/login -v
```

---

### Error Code Elemzés:

**4xx Client Errors:**
```
400 Bad Request → Hibás paraméter formátum
401 Unauthorized → Nincs autentikáció
403 Forbidden → Nincs jogosultság (⚠️ van az endpoint, de nincs hozzáférés!)
404 Not Found → Nincs ilyen endpoint
```

**5xx Server Errors:**
```
500 Internal Server Error → Szerver oldali hiba (stack trace?)
503 Service Unavailable → Túlterhelt
```

**Mit csinálj:**
- **403 response** → Próbálj bypass-olni (HTTP method change, header manipulation)
- **500 error** → Nézd meg a response body-t (stack trace, debug info?)

---

## OWASP Attack Surface Detector

### Concept:
Az **Attack Surface Detector (ASD)** a **forráskódból** automatikusan kinyeri az összes endpointot és paramétert.

### Használat (CLI):

#### Parancs:
```bash
# Download JAR
wget https://github.com/secdec/attack-surface-detector-cli/releases/download/v1.3.5/attack-surface-detector-cli-1.3.5.jar

# Futtatás
java -jar attack-surface-detector-cli-1.3.5.jar /path/to/source-code

# JSON output generálás
java -jar attack-surface-detector-cli-1.3.5.jar /path/to/source-code -json
```

**Kimenet példa:**
```
[0] GET: /login (0 variants): PARAMETERS={url=name=url, paramType=QUERY_STRING, dataType=STRING}
[1] GET: /logout (0 variants): PARAMETERS={}
[2] POST: /forgot_password (0 variants): PARAMETERS={email=name=email, paramType=QUERY_STRING, dataType=STRING}
[3] POST: /password_resets (0 variants): PARAMETERS={password, user, confirm_password}
[4] DELETE: /sessions/{id} (0 variants): PARAMETERS={}
...
Generated 40 distinct endpoints with 0 variants
Generated 36 distinct parameters
```

**Mit ad:**
- **Összes endpoint** (GET, POST, PUT, DELETE, stb.)
- **Összes paraméter** típussal
- **Unlinked endpoints** (amiket spider nem talál)
- **Optional parameters** (amiket a frontend nem használ)

---

### ASD Burp/ZAP Plugin:

**Burp Suite:**
```
1. Extender → BApp Store → Attack Surface Detector
2. Import JSON (ASD CLI output-ból)
3. Target → Site map automatikusan frissül
```

**OWASP ZAP:**
```
1. Manage Add-ons → Attack Surface Detector
2. Import JSON
3. Sites tree automatikusan frissül
```

---

## Példák - Entry Point Azonosítás

### Példa #1: GET Request (Online Shopping)

**Request:**
```http
GET /shoppingApp/buyme.asp?CUSTOMERID=100&ITEM=z101a&PRICE=62.50&IP=x.x.x.x HTTP/1.1
Host: x.x.x.x
Cookie: SESSIONID=Z29vZCBqb2IgcGFkYXdhIG15IHVzZXJuYW1lIGlzIGZvbyBhbmQgcGFzc3dvcmQgaXMgYmFy
```

**Azonosított paraméterek:**

| Parameter | Location | Type | Potential Issue |
|-----------|----------|------|-----------------|
| CUSTOMERID | Query String | Integer | IDOR (Insecure Direct Object Reference) |
| ITEM | Query String | String | - |
| PRICE | Query String | Decimal | ⚠️ Price manipulation! |
| IP | Query String | IP Address | ⚠️ IP spoofing? |
| SESSIONID | Cookie | Base64 | Session prediction? |

**Mit tesztelsz később:**
- `CUSTOMERID=101` → Más customer adatai?
- `PRICE=0.01` → Árat lehet manipulálni?
- `IP=127.0.0.1` → IP-based access control bypass?

---

### Példa #2: POST Request (Login)

**Request:**
```http
POST /example/authenticate.asp?service=login HTTP/1.1
Host: x.x.x.x
Cookie: SESSIONID=dGhpcyBpcyBhIGJhZCBhcHAgdGhhdCBzZXRzIHByZWRpY3RhYmxlIGNvb2tpZXMgYW5kIG1pbmUgaXMgMTIzNA==;CustomCookie=00my00trusted00ip00is00x.x.x.x00

user=admin&pass=pass123&debug=true&fromtrustIP=true
```

**Azonosított paraméterek 3 helyen:**

**1. Query String:**
- `service=login`

**2. Cookie Header:**
- `SESSIONID` (predictable cookie!)
- `CustomCookie` (trusted IP info)

**3. POST Body:**
- `user=admin`
- `pass=pass123`
- `debug=true` ⚠️
- `fromtrustIP=true` ⚠️

**Mit tesztelsz később:**
- `debug=true` → Extra debug info?
- `fromtrustIP=true` → Access control bypass?
- `CustomCookie` IP manipulálása

---

## Spreadsheet Template

### Entry Points Documentation Template:

```
Request #: 1
URL: /api/users
Method: POST
Authenticated: Yes
HTTPS: Yes
Query Parameters: None
POST Parameters:
  - username (String, required)
  - email (String, required)
  - role (String, optional) ⚠️
Hidden Fields: None
Cookies:
  - SESSIONID (session token)
  - UserRole (user privilege) ⚠️
Custom Headers:
  - X-API-Key (API authentication)
Notes: role parameter might be exploitable for privilege escalation
Test Priority: HIGH

---

Request #: 2
URL: /buy
Method: POST
Authenticated: Yes
HTTPS: Yes
Query Parameters: None
POST Parameters:
  - item (String)
  - quantity (Integer)
Hidden Fields:
  - price (Decimal) ⚠️ CRITICAL
  - customerID (Integer) ⚠️
  - discount (Integer) ⚠️
Cookies:
  - SESSIONID
Notes: Hidden price and discount fields - test for manipulation!
Test Priority: CRITICAL
```

---

## Automatizált Mapping Script

### Teljes Entry Point Extraction:

```python
#!/usr/bin/env python3
import requests
from bs4 import BeautifulSoup
import json

def extract_entry_points(url):
    entry_points = []
    
    # GET request
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'html.parser')
    
    # Find all forms
    for form in soup.find_all('form'):
        endpoint = {
            'action': form.get('action'),
            'method': form.get('method', 'GET'),
            'fields': []
        }
        
        # Extract all input fields
        for input_field in form.find_all('input'):
            field = {
                'name': input_field.get('name'),
                'type': input_field.get('type'),
                'value': input_field.get('value'),
                'hidden': input_field.get('type') == 'hidden'
            }
            endpoint['fields'].append(field)
        
        entry_points.append(endpoint)
    
    return entry_points

# Használat
url = "https://pelda.hu/login"
results = extract_entry_points(url)
print(json.dumps(results, indent=2))
```

---

## Gyakorlati Cheat Sheet

| Feladat | Tool/Parancs |
|---------|--------------|
| Proxy setup | Burp Suite (127.0.0.1:8080) |
| View all requests | Burp → Proxy → HTTP History |
| View parameters | Burp → Request → Params tab |
| View cookies | Browser DevTools → Application → Cookies |
| Custom headers | `curl -H "X-Custom: value" URL` |
| POST request | `curl -X POST -d "user=test" URL` |
| Hidden fields | View Page Source → Ctrl+F "hidden" |
| Supported methods | `curl -X OPTIONS URL -v` |
| Spider application | ZAP → Spider / Burp → Spider |
| Source code analysis | Attack Surface Detector CLI |

---

## Fontos Toolok

### Intercepting Proxies:
- **Burp Suite** - Industry standard
- **OWASP ZAP** - Open source alternative
- **Fiddler** - Windows proxy tool
- **mitmproxy** - CLI-based proxy

### Browser Tools:
- **Chrome DevTools** - Network tab
- **Firefox Developer Tools**
- **Browser Extensions** - Cookie editors

### Command Line:
- **curl** - HTTP requests
- **wget** - Download
- **httpie** - User-friendly HTTP client

### Source Code Analysis:
- **OWASP Attack Surface Detector** - Endpoint discovery from source

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **Input Validation**
   - Minden paraméter validálása
   - Whitelist approach
   - Type checking

2. **Hidden Fields**
   - NE tárolj érzékeny adatokat hidden fieldben
   - Server-side session használata
   - Price, role stb. a backend-en tárold

3. **HTTP Methods**
   - Csak szükséges methods engedélyezése
   - TRACE, OPTIONS tiltása production-ban

4. **Custom Headers**
   - `X-Debug` headerek NE legyenek production-ban
   - Header-based authentication legyen biztonságos

5. **Cookies**
   - HttpOnly, Secure flag-ek
   - Secure session management
   - Unpredictable session ID-k

---

## Fontos Megjegyzések

✅ **Ez a MAPPING fázis** - még nem támadsz, csak térkép készítesz  
✅ **Minden paramétert dokumentálj** - spreadsheet vagy note taking app  
✅ **Proxy használata kötelező** - különben nem látod az összes paramétert  
✅ **Burp/ZAP HTTP History** = arany bánya  
❌ **NE ugorj a tesztelésbe** - előbb térképezd fel az EGÉSZ alkalmazást!  
⚠️ **Hidden fields** gyakran a legsebezhetőbbek (price, role manipulation)

---

**Összefoglalva:** Ez a fejezet az **alkalmazás támadási felületének feltérképezéséről** szól. Proxy tool-lal (Burp, ZAP) **minden HTTP request és response-t elemezz**, dokumentáld az összes paramétert (GET, POST, Cookie, Header), és készíts spreadsheet-et. **Rejtett form mezők, custom headerek, és ritka HTTP methods** mind injection pontok lehetnek. Ez a **legfontosabb előkészítő fázis** - ha ezt jól csinálod, a későbbi tesztelés sokkal hatékonyabb lesz!
