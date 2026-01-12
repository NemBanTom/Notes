# Weboldalak Tartalmának Vizsgálata Információszivárgásra

|ID          |
|------------|
|WSTG-INFO-05|

## Összefoglaló

A programozók gyakran hagynak HTML kommenteket, metaadatokat és debuggolási információkat a kódban. A modern frontend JavaScript alkalmazások (React, Angular, Vue) esetén pedig gyakori, hogy érzékeny adatok kerülnek a kliens oldali kódba: API kulcsok, internal IP címek, credential-ök, rejtett admin útvonalak. Source map fájlok további betekintést nyújtanak a forráskódba.

**Cél:** 
- HTML kommentek és metaadatok átvizsgálása érzékeny információkra
- JavaScript fájlok elemzése hardcoded credential-ökre, API kulcsokra
- Source map és debug fájlok azonosítása
- Redirect válaszokban található információszivárgás detektálása

## Tesztelési Módszertan

### 1. HTML Kommentek és Metaadatok Vizsgálata

**Manuális forrás megtekintés:**
```bash
# Teljes HTML forrás letöltése
curl -s https://target.com | less

# Csak HTML kommentek
curl -s https://target.com | grep -o '<!--.*-->'

# Többsoros kommentek is
curl -s https://target.com | awk '/<!--/,/-->/'
```

**Kommentek keresése wget-tel:**
```bash
# Teljes oldal letöltése
wget -r -l 1 https://target.com

# Minden HTML fájlban komment keresés
grep -r "<!--" target.com/

# SQL kommentek keresése
grep -r "SELECT\|INSERT\|UPDATE\|DELETE" target.com/ | grep "<!--"
```

**META tagek kinyerése:**
```bash
# Összes META tag
curl -s https://target.com | grep -i '<meta'

# Specifikus META információk
curl -s https://target.com | grep -i '<meta name="author"'
curl -s https://target.com | grep -i '<meta name="generator"'
curl -s https://target.com | grep -i '<meta name="keywords"'

# Robots META
curl -s https://target.com | grep -i '<meta name="robots"'
```

**Érzékeny információk keresése kommentekben:**
```bash
# Password, credential keresés
curl -s https://target.com | grep -i "password\|credential\|secret\|key" | grep "<!--"

# SQL query-k
curl -s https://target.com | grep -iE "select|insert|update|delete|from|where" | grep "<!--"

# IP címek
curl -s https://target.com | grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' | grep "<!--"

# Email címek kommentekben
curl -s https://target.com | grep -oE '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b' | grep "<!--"
```

### 2. JavaScript Fájlok Azonosítása és Letöltése

**JavaScript fájlok linkjeinek gyűjtése:**
```bash
# Script tag-ek src attribútuma
curl -s https://target.com | grep -o '<script[^>]*src="[^"]*"' | sed 's/.*src="\([^"]*\)".*/\1/'

# Teljes URL-ek készítése
curl -s https://target.com | grep -o '<script[^>]*src="[^"]*"' | sed 's/.*src="\([^"]*\)".*/\1/' | while read js; do
    if [[ $js == http* ]]; then
        echo $js
    else
        echo "https://target.com$js"
    fi
done
```

**Összes JavaScript fájl letöltése:**
```bash
# Egyszerű letöltés
wget -r -l 1 -A js https://target.com

# Vagy linkextractor-rel
curl -s https://target.com | grep -oP '(?<=src=")[^"]*\.js(?=")' | while read js; do
    wget "https://target.com$js"
done
```

**getJS - JavaScript collector tool:**
```bash
# Telepítés
go install github.com/003random/getJS@latest

# Használat
getJS --url https://target.com --output js_files.txt

# Complete option
getJS --url https://target.com --complete
```

**linkfinder - Endpoint discovery JS-ből:**
```bash
# Telepítés
git clone https://github.com/GerbenJavado/LinkFinder.git
cd LinkFinder

# Használat
python3 linkfinder.py -i https://target.com/app.js -o results.html

# Burp output-ból
python3 linkfinder.py -i burpfile -b
```

### 3. JavaScript Kód Analízis - Érzékeny Információk

**API kulcsok keresése:**
```bash
# Gyakori API kulcs minták
grep -r "api[_-]key\|apikey\|api[_-]secret" *.js
grep -r "AIza[0-9A-Za-z_-]{35}" *.js  # Google API key pattern
grep -r "sk_live_[0-9a-zA-Z]{24}" *.js  # Stripe Secret Key
grep -r "access_token\|accessToken" *.js

# AWS credentials
grep -r "AKIA[0-9A-Z]{16}" *.js  # AWS Access Key ID
grep -r "aws_secret_access_key" *.js
```

**Credential-ök keresése:**
```bash
# Password változók
grep -ri "password\s*[:=]\|passwd\s*[:=]\|pwd\s*[:=]" *.js

# Database connection stringek
grep -ri "postgresql://\|mysql://\|mongodb://" *.js
grep -ri "jdbc:\|Server=\|Database=" *.js

# Admin credential-ök
grep -ri "admin.*password\|root.*password" *.js
```

**Internal IP címek és útvonalak:**
```bash
# Private IP ranges
grep -roE '10\.\d{1,3}\.\d{1,3}\.\d{1,3}' *.js
grep -roE '192\.168\.\d{1,3}\.\d{1,3}' *.js
grep -roE '172\.(1[6-9]|2[0-9]|3[0-1])\.\d{1,3}\.\d{1,3}' *.js

# API endpoint-ok
grep -ri "api/\|/v1/\|/v2/\|/admin/\|/internal/" *.js

# Hidden routes
grep -ri "hidden\|secret\|admin\|internal" *.js | grep -i "route\|path\|url"
```

**Átfogó JS secret scanner:**
```bash
# truffleHog JS fájlokra
trufflehog filesystem ./js_files --only-verified

# Secretfinder
git clone https://github.com/m4ll0k/SecretFinder.git
python3 SecretFinder.py -i https://target.com/app.js -o results.html

# Egy könyvtár összes JS fájljára
find . -name "*.js" -exec grep -H -E "api[_-]?key|password|secret|token|auth" {} \;
```

**Regex minták érzékeny adatokra:**
```bash
# Komplex grep minden JS-ben
grep -rE "(api[_-]?key|api[_-]?secret|access[_-]?token|auth[_-]?token|secret[_-]?key|private[_-]?key)\s*[:=]\s*['\"][^'\"]{8,}['\"]" *.js

# Email és username
grep -roE '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b' *.js

# JWT tokenek
grep -roE 'eyJ[A-Za-z0-9-_=]+\.[A-Za-z0-9-_=]+\.?[A-Za-z0-9-_.+/=]*' *.js
```

### 4. Source Map Fájlok Azonosítása

**Source map fájlok keresése:**
```bash
# .map extension próbálgatása
curl -I https://target.com/static/js/main.js.map
curl -I https://target.com/static/js/app.js.map
curl -I https://target.com/assets/bundle.js.map

# Minden JS fájlhoz
curl -s https://target.com | grep -oP '(?<=src=")[^"]*\.js(?=")' | while read js; do
    echo "Checking: $js.map"
    curl -I "https://target.com${js}.map" 2>&1 | grep "HTTP"
done
```

**Source map letöltés és elemzés:**
```bash
# Source map letöltése
curl -s https://target.com/static/js/main.js.map -o main.js.map

# Érzékeny útvonalak a sources részben
cat main.js.map | jq -r '.sources[]' | grep -E "home|user|admin|config"

# Teljes fájlnevek és mappák
cat main.js.map | jq -r '.sources[]'

# Sourcemap vizualizáció
cat main.js.map | jq '.'
```

**Source map tartalom visszafejtése:**
```bash
# Forrás fájlok kinyerése source map-ből (mapcat tool)
npm install -g mapcat
mapcat main.js.map

# Vagy online: https://sokra.github.io/source-map-visualization/
```

**Automatizált source map scanner:**
```bash
# URL lista source map check-re
while read url; do
    js_files=$(curl -s "$url" | grep -oP '(?<=src=")[^"]*\.js(?=")')
    for js in $js_files; do
        map_url="${url}${js}.map"
        status=$(curl -s -o /dev/null -w "%{http_code}" "$map_url")
        if [ "$status" = "200" ]; then
            echo "[+] Found: $map_url"
        fi
    done
done < urls.txt
```

### 5. Inline JavaScript Vizsgálata

**Inline script-ek kinyerése:**
```bash
# <script> és </script> közötti tartalom
curl -s https://target.com | awk '/<script>/,/<\/script>/' | grep -v "src="

# JSON típusú script-ek (config-ok)
curl -s https://target.com | grep -A 50 'type="application/json"'
```

**Config objektumok keresése:**
```bash
# window.config, window.__INITIAL_STATE__ stb.
curl -s https://target.com | grep -E "window\.(config|__.*__|env|settings)"

# React/Redux initial state
curl -s https://target.com | grep "__INITIAL_STATE__\|__PRELOADED_STATE__"

# Angular config
curl -s https://target.com | grep "angular.module.*config"
```

### 6. Redirect Válaszok Vizsgálata

**3xx válaszok tartalmának ellenőrzése:**
```bash
# Redirect követés nélkül, teljes válasz
curl -i https://target.com/admin

# Csak 3xx válaszok
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://target.com/admin

# Redirect body tartalmának mentése
curl -i --max-redirs 0 https://target.com/admin > redirect_response.txt
```

**Burp Suite / ZAP használata:**
- Proxy bekapcsolása
- Target böngészése
- History-ban 301, 302, 303, 307, 308 válaszok szűrése
- Response body ellenőrzése minden redirect-nél

**Redirect chain követése:**
```bash
# Teljes redirect chain
curl -sL -o /dev/null -D - https://target.com

# Minden egyes redirect válasz body-ja
curl -s --max-redirs 0 https://target.com > step1.html
curl -s --max-redirs 0 https://redirect-url > step2.html
```

### 7. Automatizált Content Analysis

**Burp Suite Scanner:**
- Passive scan minden válaszra
- "Information disclosure" findings átnézése
- Comments, metadata, error messages

**ZAP Spider + Passive Scanner:**
```bash
# ZAP CLI használata
zap-cli quick-scan -s all https://target.com

# Spider + passive scan
zap-cli spider https://target.com
zap-cli active-scan https://target.com
```

**Grep Pattern Fájlok (Burp):**
```text
# Custom grep pattern hozzáadása Burp-ben:
- password
- api.?key
- secret
- SELECT.*FROM
- Exception
- Error
- Stack.?trace
```

**Wayback Machine - régi source code:**
```bash
# Wayback URLs JavaScript fájlokra
waybackurls target.com | grep "\.js$" | sort -u

# Régi JS fájlok letöltése és elemzése
waybackurls target.com | grep "\.js$" | while read url; do
    wget "$url"
done
```

### 8. API Kulcsok Validálása

**Google Maps API Key teszt:**
```bash
# API kulcs korlátozások ellenőrzése
curl "https://maps.googleapis.com/maps/api/staticmap?center=45,10&zoom=7&size=400x400&key=YOUR_API_KEY"

# gmapsapiscanner
git clone https://github.com/ozguralp/gmapsapiscanner.git
python3 maps_api_scanner.py -k YOUR_API_KEY
```

**KeyHacks - API kulcs exploitok:**
```bash
# GitHub repo: https://github.com/streaak/keyhacks
# Példák különböző API kulcsok tesztelésére

# AWS Access Key
aws s3 ls --profile test

# Slack Token
curl -X POST -H 'Authorization: Bearer xoxb-YOUR-TOKEN' https://slack.com/api/auth.test

# GitHub Token
curl -H "Authorization: token YOUR_TOKEN" https://api.github.com/user
```

**Stripe Key teszt:**
```bash
# Secret key teszt
curl https://api.stripe.com/v1/charges -u sk_test_YOUR_KEY:
```

### 9. Comprehensive Bash Script

**all_info_leakage.sh:**
```bash
#!/bin/bash
TARGET=$1
OUTPUT_DIR="info_leak_results"
mkdir -p $OUTPUT_DIR

echo "[+] Downloading HTML..."
curl -s $TARGET > $OUTPUT_DIR/index.html

echo "[+] Extracting comments..."
grep -o '<!--.*-->' $OUTPUT_DIR/index.html > $OUTPUT_DIR/comments.txt

echo "[+] Extracting META tags..."
grep -i '<meta' $OUTPUT_DIR/index.html > $OUTPUT_DIR/meta_tags.txt

echo "[+] Finding JavaScript files..."
grep -oP '(?<=src=")[^"]*\.js(?=")' $OUTPUT_DIR/index.html > $OUTPUT_DIR/js_files.txt

echo "[+] Downloading JavaScript files..."
while read js; do
    if [[ $js == http* ]]; then
        wget -q "$js" -P $OUTPUT_DIR/js/
    else
        wget -q "$TARGET$js" -P $OUTPUT_DIR/js/
    fi
done < $OUTPUT_DIR/js_files.txt

echo "[+] Searching for API keys in JS..."
grep -rh "api[_-]key\|apikey\|api[_-]secret" $OUTPUT_DIR/js/ > $OUTPUT_DIR/api_keys.txt

echo "[+] Searching for credentials in JS..."
grep -rh "password\|passwd\|secret" $OUTPUT_DIR/js/ > $OUTPUT_DIR/credentials.txt

echo "[+] Checking for source maps..."
while read js; do
    map_url="${js}.map"
    curl -s -o /dev/null -w "%{http_code} $map_url\n" "$map_url"
done < $OUTPUT_DIR/js_files.txt > $OUTPUT_DIR/sourcemaps.txt

echo "[+] Analysis complete. Results in $OUTPUT_DIR/"
```

### 10. Browser Developer Tools

**Manual inspection steps:**
1. F12 / Developer Tools megnyitása
2. **Elements/Inspector tab:**
   - Right-click → View Page Source
   - Search (Ctrl+F): password, api, key, secret, token
3. **Network tab:**
   - Clear, reload page
   - Filter: JS, XHR
   - Minden response preview és response body ellenőrzése
4. **Sources tab:**
   - Prettify minified JS ({}button)
   - Breakpoint-ok állítása
   - Watch expressions
5. **Console tab:**
   - `console.log` output-ok
   - Error messages
   - Global variables: `window`, `window.config`

## Lehetséges Hatások és Ellenintézkedések

**Hatás:** Hardcoded API kulcsok kihasználása limitek nélkül (költségek növelése). Database connection stringek → direct DB access. Admin útvonalak felfedése → privilege escalation. Source map-ek → teljes forráskód hozzáférés, könnyebb vulnerability discovery. Kommentekben SQL query-k → injection pontok azonosítása.

**Ellenintézkedések:**
- Kommentek eltávolítása production kódból (build process során)
- API kulcsok backend-en tárolása, frontend-en csak proxy endpoint-ok
- Environment változók használata, soha ne hardcode-olj credential-öket
- Source map fájlok NE kerüljenek production-be vagy védve legyenek (.htaccess, authentication)
- Minification és obfuscation használata (de ez nem helyettesíti a proper secret management-et)
- Content Security Policy (CSP) megfelelő beállítása
- API kulcsok korlátozása: IP whitelist, referrer check, rate limiting
- Automated secret scanning CI/CD pipeline-ban (gitleaks, trufflehog)
- Regular code review security szempontból
- Redirect válaszok ne tartalmazzanak sensitive információt
