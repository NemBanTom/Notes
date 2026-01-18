# WSTG-INFO-05: Weboldal Tartalom Vizsgálata Információszivárgásra

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy a **HTML forráskód, JavaScript fájlok, és egyéb frontend kódban gyakran véletlenül érzékeny információk maradnak**. Fejlesztők kommenteket, debug információkat, API kulcsokat, jelszavakat hagynak a kódban, ami **támadók számára értékes információt jelenthet**.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM backend vulnerability scanning! Ez a **frontend forráskód elemzése** - HTML, CSS, JavaScript vizsgálata **statikusan**, amit a böngésző is lát.

---

## Mi a cél?

**Információszivárgás azonosítása** a weboldal tartalmában:
- HTML kommentek és metadata
- JavaScript kódban hardcoded sensitive data
- Source map fájlok (debug fájlok)
- Redirect response body-k

### Mit keresünk?

- **API kulcsok** (Google Maps, AWS, stb.)
- **Belső IP címek** (10.x.x.x, 192.168.x.x)
- **Jelszavak, credentials**
- **Adatbázis connection stringek**
- **Rejtett admin útvonalak**
- **SQL query-k a kommentekben**
- **Fejlesztői jegyzetek, TODO-k**
- **Debug információk**
- **Source map fájlok** (.js.map)

---

## 3 Fő Terület

### 1. **HTML Kommentek és Metadata**
### 2. **JavaScript Kód Elemzés**
### 3. **Source Map Fájlok**

---

## 1. HTML Kommentek és Metadata Vizsgálata

### Concept:
Fejlesztők gyakran hagynak **HTML kommenteket** `<!-- -->` a kódban debug célból, és elfelejtik eltávolítani production-ból.

### Tool #1: **Browser View Source**

#### Használat:
```
1. Nyisd meg a weboldalt böngészőben
2. Jobb klikk → "View Page Source" (vagy Ctrl+U)
3. Keress rá: <!-- (Ctrl+F)
4. Nézd végig az összes kommentet
```

**Mit keress:**
- SQL query-k
- Jelszavak
- Internal IP címek
- TODO jegyzetek
- Debug üzenetek

---

### Tool #2: **curl**

#### Parancsok:
```bash
# HTML letöltése
curl https://pelda.hu -o page.html

# Kommentek keresése
curl -s https://pelda.hu | grep -oP '<!--.*?-->' 

# Kommentek szépen formázva
curl -s https://pelda.hu | grep '<!--' -A 5

# Többsoros kommentek
curl -s https://pelda.hu | sed -n '/<!--/,/-->/p'
```

**Mit csinál:**
- `-s` = silent mode
- `-oP` = Perl regex (greedy match)
- `sed -n '/<!--/,/-->/p'` = Többsoros kommentek kinyerése

---

### Tool #3: **wget**

#### Parancsok:
```bash
# Teljes weboldal letöltése
wget -r -l 1 https://pelda.hu

# Csak HTML fájlok
wget -r -l 1 -A html https://pelda.hu

# Kommentek keresése minden letöltött fájlban
grep -r "<!--" pelda.hu/
```

**Mit csinál:**
- `-r` = recursive
- `-l 1` = depth 1
- `-A html` = Accept only HTML files

---

### Gyakori HTML Komment Példák

#### 1. SQL Query a kommentben:
```html
<div class="users">
  <div>User: Mary</div>
  <div>User: Peter</div>
  
  <!-- Query: SELECT id, name FROM app.users WHERE active='1' -->
</div>
```

**Mit árul el:** Adatbázis struktúra (táblák, oszlopok)

---

#### 2. Jelszó a kommentben:
```html
<!-- Use the DB admin password for testing: f@keP@a$$w0rD -->
<!-- TODO: Remove this before production! -->
<!-- Default login: admin / Admin123! -->
```

**Mit árul el:** ⚠️ **KRITIKUS** - Credentials!

---

#### 3. Belső IP cím:
```html
<!-- API endpoint: http://10.10.10.5/api/v1/users -->
<!-- Contact internal server: 192.168.1.100 -->
```

**Mit árul el:** Belső hálózat struktúra

---

#### 4. Debug információk:
```html
<!-- DEBUG: User role = admin, permissions = [read, write, delete] -->
<!-- Version: 2.3.1-beta, Build: 20230415 -->
```

**Mit árul el:** Verzió info, debug mode engedélyezett

---

### META Tag Vizsgálat

#### Tool: **curl + grep**

#### Parancsok:
```bash
# Összes META tag
curl -s https://pelda.hu | grep -i '<meta'

# Specifikus META tagek
curl -s https://pelda.hu | grep -i 'meta name="author"'
curl -s https://pelda.hu | grep -i 'meta name="keywords"'
curl -s https://pelda.hu | grep -i 'meta http-equiv="refresh"'
```

---

#### Érdekes META Tagek:

**Author META:**
```html
<META name="Author" content="Andrew Muller">
<META name="Company" content="SecretCorp Inc.">
```

**Mit árul el:** Ki fejlesztette, melyik cég → Social engineering

---

**Keywords META:**
```html
<META name="keywords" content="admin, dashboard, internal, vpn">
```

**Mit árul el:** Oldal funkciók, admin területek létezése

---

**Refresh META:**
```html
<META http-equiv="Refresh" content="15;URL=https://admin.pelda.hu/login">
```

**Mit árul el:** ⚠️ Rejtett admin URL!

---

**Robots META:**
```html
<META name="robots" content="noindex, nofollow">
```

**Mit árul el:** Ez az oldal ne legyen indexelve → Lehet érzékeny

---

## 2. JavaScript Kód Elemzés

### Concept:
Frontend JavaScript kódban fejlesztők gyakran **hardcode-olnak érzékeny adatokat** (API kulcsok, credentials, belső útvonalak).

### Tool #1: **Browser DevTools**

#### Használat:
```
1. Nyisd meg a weboldalt
2. F12 (DevTools)
3. Sources tab
4. Böngészd végig a .js fájlokat
5. Keress rá (Ctrl+F): API, key, password, secret, token, config
```

---

### Tool #2: **curl - JavaScript fájlok letöltése**

#### Parancsok:
```bash
# JavaScript fájlok azonosítása HTML-ből
curl -s https://pelda.hu | grep -oP 'src="[^"]*\.js"'

# Egy specifikus JS fájl letöltése
curl https://pelda.hu/static/js/main.js -o main.js

# Összes JS fájl letöltése
curl -s https://pelda.hu | grep -oP 'src="\K[^"]*\.js' | while read js; do
    wget "https://pelda.hu$js"
done
```

---

### Tool #3: **grep - Sensitive Data Keresés JS-ben**

#### Parancsok:
```bash
# API kulcsok keresése
grep -rn "api.?key\|apikey" *.js
grep -rn "secret" *.js
grep -rn "password" *.js

# AWS credentials
grep -rn "AKIA" *.js  # AWS Access Key ID pattern
grep -rn "aws_secret" *.js

# Google API key
grep -rn "AIza" *.js  # Google API Key pattern

# Belső IP címek
grep -rn "192\.168\|10\.\|172\.16" *.js

# Database connection strings
grep -rn "mongodb://\|postgres://\|mysql://" *.js
```

---

### Gyakori JavaScript Szivárgások

#### 1. API Key hardcoding:
```javascript
const GOOGLE_MAP_API_KEY = "AIzaSyDUEBnKgwiqMNpDplT6ozE4Z0XxuAbqDi4";
const RECAPTCHA_KEY = "6LcPscEUiAAAAHOwwM3fGvIx9rsPYUq62uRhGjJ0";

// AWS Credentials
const myS3Credentials = {
  accessKeyId: "AKIAIOSFODNN7EXAMPLE",
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
};
```

**Mit csinálj:** Teszteld az API kulcsokat!

---

#### 2. Database Connection String:
```javascript
var conString = "tcp://postgres:1234@localhost/postgres";
var mongoUrl = "mongodb://admin:password123@10.10.10.5:27017/mydb";
```

**Mit árul el:** ⚠️ **KRITIKUS** - DB credentials és IP

---

#### 3. Internal Routes és Admin URL-ek:
```javascript
const config = {
  "BASE_URL_API": "https://api.pelda.hu",
  "BASE_BACKOFFICE_API": "https://10.10.10.2/api",
  "ADMIN_PAGE": "/hidden_administrator",
  "SECRET_ENDPOINT": "/api/internal/debug"
};
```

**Mit csinálj:** Próbáld meg elérni ezeket az URL-eket!

---

#### 4. Credentials és Secrets:
```javascript
// TODO: Remove before production!
const testUser = {
  username: "admin",
  password: "Admin123!"
};

const JWT_SECRET = "my-super-secret-key-123";
```

**Mit csinálj:** Próbálj meg bejelentkezni!

---

### Tool #4: **LinkFinder - Endpoint Discovery**

#### Telepítés és használat:
```bash
# Telepítés
git clone https://github.com/GerbenJavado/LinkFinder.git
cd LinkFinder
pip3 install -r requirements.txt

# Használat
python3 linkfinder.py -i https://pelda.hu -o output.html

# Egy JS fájl elemzése
python3 linkfinder.py -i https://pelda.hu/static/js/main.js -o results.html
```

**Mit csinál:** 
- Automatikusan végigpásztázza a JS kódot
- Endpointokat, URL-eket keres
- HTML reportot generál

---

### Tool #5: **JSParser**

#### Parancs:
```bash
# Telepítés
pip3 install jsparser

# Használat
python3 jsparser.py -u https://pelda.hu
```

**Mit csinál:** JavaScript fájlokat elemez és URL-eket, API endpointokat talál.

---

### Tool #6: **Burp Suite - JS Analysis**

#### Használat:
1. Indítsd el Burp Suite Proxy-t
2. Böngéssz végig a weboldalon
3. **Proxy → HTTP history → Filter: Show only JS**
4. Minden JS fájlra jobb klikk → **Send to Repeater**
5. Search funkció (Ctrl+F) → keress rá: `api, key, secret, password`

---

### Tool #7: **retire.js - JavaScript Library Scanner**

#### Parancsok:
```bash
# Telepítés
npm install -g retire

# Használat
retire --js --jspath https://pelda.hu

# Lokális JS fájlok scan-elése
retire --path /path/to/js/files
```

**Mit csinál:** 
- Elavult JavaScript library-ket keres
- Ismert sebezhetőségeket azonosít a JS library-kben

---

## 3. Source Map Fájlok Vizsgálata

### Concept:
Source map fájlok (.js.map) **debug célból** vannak, hogy a minified/uglified kód **olvasható legyen**. Production-ban **NEM kellene** létezniük!

### Lokáció:
```
Ha van: https://pelda.hu/static/js/main.chunk.js
Akkor: https://pelda.hu/static/js/main.chunk.js.map
```

### Tool #1: **curl - Source Map Detektálás**

#### Parancsok:
```bash
# JS fájl végén szokott lenni: //# sourceMappingURL=
curl -s https://pelda.hu/static/js/main.js | tail -5

# Ha találtál source map referenciát:
curl https://pelda.hu/static/js/main.js.map -o main.js.map

# Source map automatikus keresés
curl -s https://pelda.hu | grep -oP 'src="\K[^"]*\.js' | while read js; do
    echo "Checking: $js"
    curl -s "https://pelda.hu$js.map" -o "${js##*/}.map" 2>/dev/null && echo "Found: ${js}.map"
done
```

---

### Tool #2: **Browser DevTools**

#### Használat:
```
1. F12 → Sources tab
2. Nézd meg, vannak-e "webpack://" kezdetű fájlok
3. Ha vannak → source map betöltődött
4. Böngészd végig az eredeti forráskódot
```

---

### Source Map Fájl Példa:

```json
{
  "version": 3,
  "file": "static/js/main.chunk.js",
  "sources": [
    "/home/sysadmin/cashsystem/src/actions/index.js",
    "/home/sysadmin/cashsystem/src/actions/reportAction.js",
    "/home/sysadmin/cashsystem/src/actions/userAction.js",
    "/home/sysadmin/cashsystem/src/config/database.js",
    "/home/sysadmin/cashsystem/src/utils/secrets.js"
  ],
  "sourcesContent": [
    "const API_KEY = 'sk-1234567890abcdef';\nconst DB_PASSWORD = 'SuperSecret123';",
    "..."
  ]
}
```

**Mit árul el:**
- **Teljes file path** → `/home/sysadmin/cashsystem/` (username, project név)
- **Project struktúra** → Mik a fájlok, mappák
- **Eredeti forráskód** → `sourcesContent` mező tartalmazza!
- **Hardcoded secrets** → API kulcsok, jelszavak

---

### Tool #3: **sourcemapper**

#### Telepítés és használat:
```bash
# Telepítés
npm install -g sourcemapper

# Használat
sourcemapper -url https://pelda.hu/static/js/main.js

# Output mentése
sourcemapper -url https://pelda.hu/static/js/main.js -output extracted/
```

**Mit csinál:**
- Automatikusan letölti a source map-et
- Kibontja az eredeti forrásfájlokat
- Mappába menti őket

---

## 4. Redirect Response Body Vizsgálata

### Concept:
**3xx redirect response-ok** body-ja is tartalmazhat érzékeny információt, amit a böngésző nem renderel, de **elküldésre kerül**.

### Tool #1: **curl -i (include headers)**

#### Parancsok:
```bash
# Redirect követése és body megtekintése
curl -i https://pelda.hu/admin

# Ne kövesse a redirectet
curl -I https://pelda.hu/admin

# Redirect body mentése
curl -s -D headers.txt https://pelda.hu/admin -o body.txt
```

**Mit keress:**
- Hibaüzenetek a redirect body-ban
- Érzékeny debug info
- Internal path-ek

---

### Tool #2: **Burp Suite - Intercept Redirect**

#### Használat:
```
1. Burp Proxy → Options → Intercept Client Requests
2. Böngéssz egy védett oldalra (pl. /admin)
3. Ha redirect jön (302), nézd meg a Response body-t
4. Lehet benne érzékeny info, amit a böngésző nem renderel
```

---

## Automatizált Workflow

### Teljes Information Leakage Scan Script:

```bash
#!/bin/bash

TARGET="https://pelda.hu"
OUTPUT_DIR="info_leak_scan"

mkdir -p $OUTPUT_DIR

echo "[*] Downloading HTML source..."
curl -s $TARGET > $OUTPUT_DIR/page.html

echo "[*] Extracting HTML comments..."
grep -oP '<!--.*?-->' $OUTPUT_DIR/page.html > $OUTPUT_DIR/comments.txt

echo "[*] Extracting META tags..."
grep -i '<meta' $OUTPUT_DIR/page.html > $OUTPUT_DIR/meta_tags.txt

echo "[*] Downloading JavaScript files..."
grep -oP 'src="\K[^"]*\.js' $OUTPUT_DIR/page.html | while read js; do
    echo "Downloading: $js"
    curl -s "${TARGET}${js}" > "$OUTPUT_DIR/$(basename $js)"
done

echo "[*] Searching for API keys in JS files..."
grep -rn "api.?key\|apikey\|secret\|password" $OUTPUT_DIR/*.js > $OUTPUT_DIR/sensitive_findings.txt

echo "[*] Searching for AWS credentials..."
grep -rn "AKIA\|aws_secret" $OUTPUT_DIR/*.js >> $OUTPUT_DIR/sensitive_findings.txt

echo "[*] Searching for internal IPs..."
grep -rn "192\.168\|10\.\|172\.16" $OUTPUT_DIR/*.js >> $OUTPUT_DIR/sensitive_findings.txt

echo "[*] Checking for source maps..."
for js in $OUTPUT_DIR/*.js; do
    if curl -s "${TARGET}/$(basename $js).map" -o "$OUTPUT_DIR/$(basename $js).map" 2>/dev/null; then
        echo "Found source map: $(basename $js).map"
    fi
done

echo "[*] Scan complete! Results in $OUTPUT_DIR/"
```

**Használat:**
```bash
chmod +x info_leak_scan.sh
./info_leak_scan.sh
```

---

## API Key Tesztelés

### Ha találtál API kulcsot, teszteld!

#### Google Maps API Key:
```bash
# Alap teszt
curl "https://maps.googleapis.com/maps/api/geocode/json?address=Budapest&key=YOUR_API_KEY"

# Ha működik, próbáld más API-kon is
curl "https://www.googleapis.com/geolocation/v1/geolocate?key=YOUR_API_KEY"
```

#### AWS Access Key:
```bash
# AWS CLI-vel
aws sts get-caller-identity --aws-access-key-id AKIAXXXXXX --aws-secret-access-key SECRET

# Ha működik, próbáld ki mit tudsz elérni
aws s3 ls --aws-access-key-id AKIAXXXXXX --aws-secret-access-key SECRET
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| HTML letöltés | `curl https://pelda.hu -o page.html` |
| HTML kommentek | `grep -oP '<!--.*?-->' page.html` |
| META tagek | `curl -s https://pelda.hu \| grep '<meta'` |
| JS fájlok listázása | `curl -s https://pelda.hu \| grep -oP 'src="\K[^"]*\.js'` |
| API key keresés JS-ben | `grep -rn "api.?key\|secret" *.js` |
| AWS key pattern | `grep -rn "AKIA" *.js` |
| Google API key pattern | `grep -rn "AIza" *.js` |
| Internal IP keresés | `grep -rn "192\.168\|10\." *.js` |
| Source map check | `curl https://pelda.hu/main.js.map` |
| LinkFinder | `python3 linkfinder.py -i https://pelda.hu` |

---

## Fontos Toolok

### Manual Analysis:
- **Browser View Source** - HTML forráskód
- **Browser DevTools** - JavaScript debugging
- **curl** - HTTP requests
- **wget** - Recursive download
- **grep** - Pattern searching

### Automated Tools:
- **Burp Suite** - Proxy és analysis
- **ZAP (OWASP ZAP)** - Security testing
- **LinkFinder** - Endpoint discovery from JS
- **JSParser** - JavaScript parsing
- **retire.js** - JavaScript library vulnerability scanner
- **sourcemapper** - Source map extraction

### API Key Testing:
- **Google Maps API Scanner** - https://github.com/ozguralp/gmapsapiscanner/
- **KeyHacks** - https://github.com/streaak/keyhacks (API key teszt guide)

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **HTML Comments**
   - NE hagyj kommenteket production kódban
   - Automatizált check build process-ben
   - Code review-ban figyelj rá

2. **META Tags**
   - Ne adj ki felesleges információt
   - Author, keywords tagek legyenek generic-ek

3. **JavaScript**
   - SOHA ne hardcode-olj API kulcsokat, jelszavakat
   - Használj environment variable-öket
   - Backend-en tárold a sensitive data-t

4. **Source Maps**
   - **NE** deploy-old production-ba!
   - Build process automatikusan távolítsa el
   - Ha muszáj, authentication mögé tedd

5. **API Keys**
   - Használj API key restriction-öket
   - IP whitelist
   - HTTP referrer check
   - Minimal permissions

---

## Fontos Megjegyzések

✅ **Ez statikus forráskód elemzés** - amit a böngésző is lát  
✅ **Mindig nézd meg a source-ot** - ne csak a renderelt oldalt  
✅ **Teszteld az API kulcsokat** - nem elég megtalálni!  
❌ **NE feltételezd, hogy nincs** - mindig ellenőrizd  
⚠️ **Source map = teljes forráskód** - ha van production-ban, KRITIKUS!

---

**Összefoglalva:** Ez a fejezet a **frontend forráskód elemzéséről** szól. HTML kommentekben, JavaScript kódban, és source map fájlokban **gyakran maradnak érzékeny információk** (API kulcsok, jelszavak, belső IP-k, rejtett útvonalak). Ezeket **statikus elemzéssel** kell megtalálni - view source, grep, és automated toolok használatával!
