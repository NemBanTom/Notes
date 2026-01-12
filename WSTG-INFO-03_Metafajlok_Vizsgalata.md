# Webszerver Metafájlok Vizsgálata Információszivárgásra

|ID          |
|------------|
|WSTG-INFO-03|

## Összefoglaló

A webszerverek különböző metafájlokat használnak a crawlerek, robotok számára történő útmutatáshoz, vagy információk megosztásához. Ezek a fájlok gyakran felfednek rejtett útvonalakat, funkciókat, technológiai részleteket, amelyek hasznos támadási felületet jelenthetnek.

**Cél:** Rejtett vagy obfuszkált útvonalak, funkciók azonosítása metafájlok elemzésével, valamint további információk gyűjtése a rendszer mélyebb megértéséhez.

## Tesztelési Módszertan

### 1. robots.txt Vizsgálata

A `robots.txt` meghatározza, mely tartalmakat ne indexeljék a crawlerek. Gyakran éppen a legérzékenyebb területeket listázzák itt.

**Manuális lekérés:**
```bash
# cURL
curl -s https://target.com/robots.txt

# wget
wget -q https://target.com/robots.txt -O -

# HTTP header-rel együtt
curl -i https://target.com/robots.txt
```

**Elemzés és URL-ek kinyerése:**
```bash
# Összes Disallow útvonal listázása
curl -s https://target.com/robots.txt | grep -i "disallow" | awk '{print $2}'

# Allow és Disallow egyaránt
curl -s https://target.com/robots.txt | grep -iE "allow|disallow"

# Egyedi útvonalak (rendezve, duplikáltak nélkül)
curl -s https://target.com/robots.txt | grep -iE "allow|disallow" | awk '{print $2}' | sort -u
```

**Robots.txt parser script:**
```python
#!/usr/bin/env python3
import requests
import sys

def parse_robots(url):
    try:
        r = requests.get(f"{url}/robots.txt")
        if r.status_code == 200:
            print(f"[+] robots.txt megtalálva: {url}/robots.txt\n")
            disallow = []
            allow = []
            
            for line in r.text.split('\n'):
                if line.lower().startswith('disallow:'):
                    path = line.split(':', 1)[1].strip()
                    if path and path != '/':
                        disallow.append(path)
                elif line.lower().startswith('allow:'):
                    path = line.split(':', 1)[1].strip()
                    if path:
                        allow.append(path)
            
            if disallow:
                print("[!] Tiltott útvonalak (Disallow):")
                for path in set(disallow):
                    print(f"  {url}{path}")
            
            if allow:
                print("\n[+] Engedélyezett útvonalak (Allow):")
                for path in set(allow):
                    print(f"  {url}{path}")
        else:
            print(f"[-] robots.txt nem található: {r.status_code}")
    except Exception as e:
        print(f"[!] Hiba: {e}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Használat: {sys.argv[0]} https://target.com")
        sys.exit(1)
    parse_robots(sys.argv[1])
```

**Automatizált ellenőrzés több target-en:**
```bash
# URL lista feldolgozása
while read url; do
    echo "=== $url ==="
    curl -s "$url/robots.txt" | grep -iE "disallow|allow"
    echo ""
done < urls.txt
```

**Google Webmaster Tools használata:**
- Google Search Console-ban "robots.txt Tester" funkció
- Syntax ellenőrzés és tesztelés különböző user-agent-ekkel

### 2. META Tag-ek Elemzése

**HTML META robotok direktíva:**
```bash
# META robots tag keresése
curl -s https://target.com | grep -i '<meta name="robots"'

# Minden META tag
curl -s https://target.com | grep -i '<meta'
```

**Részletes META információk gyűjtése:**
```bash
# OpenGraph (og:) meta tag-ek
curl -s https://target.com | grep -i 'property="og:'

# Twitter card meta tag-ek
curl -s https://target.com | grep -i 'name="twitter:'

# Összes meta tag strukturált formában
curl -s https://target.com | grep -oP '<meta[^>]+>' | sort -u
```

**Python script META adatok kinyeréséhez:**
```python
#!/usr/bin/env python3
import requests
from bs4 import BeautifulSoup
import sys

def extract_meta(url):
    try:
        r = requests.get(url)
        soup = BeautifulSoup(r.text, 'html.parser')
        
        print(f"[+] META tag-ek elemzése: {url}\n")
        
        # Robots meta
        robots = soup.find('meta', {'name': 'robots'})
        if robots:
            print(f"[!] Robots direktíva: {robots.get('content')}")
        
        # Generator (CMS/framework)
        generator = soup.find('meta', {'name': 'generator'})
        if generator:
            print(f"[!] Generator: {generator.get('content')}")
        
        # OpenGraph meta tag-ek
        og_tags = soup.find_all('meta', property=lambda x: x and x.startswith('og:'))
        if og_tags:
            print("\n[+] OpenGraph meta tag-ek:")
            for tag in og_tags:
                print(f"  {tag.get('property')}: {tag.get('content')}")
        
        # Twitter meta tag-ek
        twitter_tags = soup.find_all('meta', attrs={'name': lambda x: x and x.startswith('twitter:')})
        if twitter_tags:
            print("\n[+] Twitter meta tag-ek:")
            for tag in twitter_tags:
                print(f"  {tag.get('name')}: {tag.get('content')}")
        
        # Theme color, application name, stb.
        interesting = ['theme-color', 'application-name', 'apple-mobile-web-app-title', 'msapplication-TileColor']
        for name in interesting:
            tag = soup.find('meta', {'name': name})
            if tag:
                print(f"  {name}: {tag.get('content')}")
                
    except Exception as e:
        print(f"[!] Hiba: {e}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Használat: {sys.argv[0]} https://target.com")
        sys.exit(1)
    extract_meta(sys.argv[1])
```

### 3. sitemap.xml Elemzése

A sitemap teljes struktúrát fedhet fel az oldalról, beleértve a nem linkelt URL-eket is.

**Sitemap lekérés:**
```bash
# Alapértelmezett helyek
curl -s https://target.com/sitemap.xml
curl -s https://target.com/sitemap_index.xml

# Gyakori változatok
curl -s https://target.com/sitemap-index.xml
curl -s https://target.com/sitemap1.xml
```

**URL-ek kinyerése sitemap-ből:**
```bash
# Egyszerű grep
curl -s https://target.com/sitemap.xml | grep -oP '(?<=<loc>)[^<]+'

# XML parsing xmllint-tel
curl -s https://target.com/sitemap.xml | xmllint --xpath '//loc/text()' -

# Nested sitemap-ek követése
curl -s https://target.com/sitemap.xml | grep -oP '(?<=<loc>)[^<]+' | while read sitemap; do
    echo "=== $sitemap ==="
    curl -s "$sitemap" | grep -oP '(?<=<loc>)[^<]+'
done
```

**Python sitemap parser:**
```python
#!/usr/bin/env python3
import requests
import xml.etree.ElementTree as ET
import sys

def parse_sitemap(url):
    try:
        r = requests.get(url)
        root = ET.fromstring(r.content)
        
        # XML namespace kezelés
        ns = {'sm': 'http://www.sitemaps.org/schemas/sitemap/0.9'}
        
        # Sitemap index-e?
        sitemaps = root.findall('.//sm:sitemap/sm:loc', ns)
        if sitemaps:
            print(f"[+] Sitemap index található, {len(sitemaps)} sitemap:\n")
            for sitemap in sitemaps:
                print(f"  {sitemap.text}")
                # Rekurzív elemzés
                parse_sitemap(sitemap.text)
        
        # URL-ek
        urls = root.findall('.//sm:url/sm:loc', ns)
        if urls:
            print(f"\n[+] {len(urls)} URL találva:")
            for url in urls:
                print(f"  {url.text}")
    
    except Exception as e:
        print(f"[!] Hiba: {e}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Használat: {sys.argv[0]} https://target.com/sitemap.xml")
        sys.exit(1)
    parse_sitemap(sys.argv[1])
```

### 4. security.txt Vizsgálata

Az RFC 9116 szerinti security.txt kapcsolattartási és biztonsági információkat tartalmaz.

**Lekérés:**
```bash
# Két lehetséges hely
curl -s https://target.com/security.txt
curl -s https://target.com/.well-known/security.txt

# Automatizált check mindkét helyre
for path in "/security.txt" "/.well-known/security.txt"; do
    echo "=== Próbálkozás: $path ==="
    curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://target.com$path"
    curl -s "https://target.com$path"
done
```

**Információk kinyerése:**
```bash
curl -s https://target.com/.well-known/security.txt | grep -iE "contact|encryption|policy|hiring|canonical"
```

**PGP kulcs lekérés és elemzés:**
```bash
# Ha van Encryption URL
curl -s https://target.com/.well-known/security.txt | grep "Encryption:" | awk '{print $2}' | while read pgp_url; do
    echo "=== PGP kulcs: $pgp_url ==="
    curl -s "$pgp_url" | gpg --import
    curl -s "$pgp_url" | gpg --with-fingerprint
done
```

**Bug bounty program check:**
```bash
# Security.txt policy URL-je gyakran bounty programra mutat
curl -s https://target.com/.well-known/security.txt | grep -i "policy"
```

### 5. humans.txt

A humans.txt a fejlesztőkről, használt technológiákról adhat információt.

```bash
# Lekérés
curl -s https://target.com/humans.txt

# Érdekes információk keresése
curl -s https://target.com/humans.txt | grep -iE "team|technolog|framework|cms|career|site"
```

### 6. .well-known Directory Felderítés

Az IANA által standardizált .well-known útvonalak számos hasznos információt tartalmazhatnak.

**Gyakori .well-known fájlok:**
```bash
# Lista készítés
WELL_KNOWN=(
    "security.txt"
    "change-password"
    "openid-configuration"
    "assetlinks.json"
    "apple-app-site-association"
    "dnt-policy.txt"
    "matrix/client"
    "nodeinfo"
    "webfinger"
    "host-meta"
    "host-meta.json"
)

# Ellenőrzés
for file in "${WELL_KNOWN[@]}"; do
    echo -n "Checking /.well-known/$file ... "
    status=$(curl -s -o /dev/null -w "%{http_code}" "https://target.com/.well-known/$file")
    if [ "$status" = "200" ]; then
        echo "FOUND ($status)"
        curl -s "https://target.com/.well-known/$file" | head -20
    else
        echo "$status"
    fi
done
```

**OpenID Configuration:**
```bash
# OAuth/OIDC endpoint-ok felderítése
curl -s https://target.com/.well-known/openid-configuration | jq .

# Endpoints kinyerése
curl -s https://target.com/.well-known/openid-configuration | jq -r '.authorization_endpoint, .token_endpoint, .userinfo_endpoint'
```

**Change Password URL:**
```bash
# RFC 8615 szerinti jelszóváltoztatás URL
curl -I https://target.com/.well-known/change-password
```

### 7. Egyéb Metafájlok

**crossdomain.xml (Flash/Silverlight)**
```bash
curl -s https://target.com/crossdomain.xml
curl -s https://target.com/clientaccesspolicy.xml
```

**browserconfig.xml (Windows tile config)**
```bash
curl -s https://target.com/browserconfig.xml
```

**manifest.json (PWA)**
```bash
curl -s https://target.com/manifest.json | jq .
```

**ads.txt / app-ads.txt**
```bash
# Advertising bizalmi információk
curl -s https://target.com/ads.txt
curl -s https://target.com/app-ads.txt
```

**donate.txt**
```bash
# Donation információk
curl -s https://target.com/donate.txt
```

### 8. Automatizált Metafájl Felderítés

**Komplex script minden metafájl ellenőrzésére:**
```bash
#!/bin/bash
TARGET=$1

echo "=== Metafájlok felderítése: $TARGET ==="

# Metafájlok listája
FILES=(
    "robots.txt"
    "sitemap.xml"
    "sitemap_index.xml"
    "humans.txt"
    "security.txt"
    ".well-known/security.txt"
    ".well-known/change-password"
    ".well-known/openid-configuration"
    "crossdomain.xml"
    "clientaccesspolicy.xml"
    "browserconfig.xml"
    "manifest.json"
    "ads.txt"
    "app-ads.txt"
)

for file in "${FILES[@]}"; do
    URL="${TARGET}/${file}"
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL")
    
    if [ "$STATUS" = "200" ]; then
        echo -e "\n[+] FOUND: $URL (HTTP $STATUS)"
        echo "--- Tartalom (első 10 sor) ---"
        curl -s "$URL" | head -10
        echo "-------------------------------"
    fi
done
```

**ffuf használata metafájl fuzzing-hoz:**
```bash
# Wordlist létrehozása
cat > meta_files.txt << EOF
robots.txt
sitemap.xml
sitemap_index.xml
humans.txt
security.txt
.well-known/security.txt
crossdomain.xml
manifest.json
ads.txt
EOF

# Fuzzing
ffuf -u https://target.com/FUZZ -w meta_files.txt -mc 200,301,302 -v
```

**Burp Suite / ZAP Spider:**
- Mindkét eszköz automatikusan ellenőrzi a robots.txt-et
- Sitemap.xml import lehetőség
- Scope-ba vehető a .well-known directory

### 9. Diff Analízis Időbeli Változásokra

**robots.txt változás követés:**
```bash
# Mentés baseline-ként
curl -s https://target.com/robots.txt > robots_baseline.txt

# Később összehasonlítás
curl -s https://target.com/robots.txt > robots_current.txt
diff robots_baseline.txt robots_current.txt
```

**Wayback Machine historikus robots.txt:**
```bash
# waybackurls használata
waybackurls target.com | grep "robots.txt"

# Vagy közvetlenül
curl -s "http://web.archive.org/cdx/search/cdx?url=target.com/robots.txt&output=json" | jq .
```

### 10. Google Dorks Metafájlokra

```text
site:target.com filetype:xml sitemap
site:target.com inurl:robots.txt
site:target.com inurl:.well-known
site:target.com "security.txt"
inurl:/.well-known/security.txt
```

## Lehetséges Hatások és Ellenintézkedések

**Hatás:** Metafájlok felfednek rejtett admin felületeket, API endpoint-okat, development környezeteket, backup fájlokat. A robots.txt-ben tiltott útvonalak gyakran a legérzékenyebb területek. Security.txt bug bounty információkat és támadási felületet mutat. Sitemap.xml teljes URL struktúrát fed fel, beleértve nem linkelt oldalakat is.

**Ellenintézkedések:**
- robots.txt: Ne listázz valóban kritikus útvonalakat, használj authentication-t helyette
- Érzékeny útvonalak ne kerüljenek robots.txt-be, mert az felhívja rá a figyelmet
- META tag-ek: Ne adj ki felesleges technológiai információkat (generator, version)
- sitemap.xml: Ne tartalmazza az admin/belső URL-eket, csak publikus tartalmat
- security.txt: Helyes, naprakész kapcsolattartási információk (pozitív gyakorlat)
- .well-known: Csak szükséges fájlok legyenek elérhetőek, megfelelő access control
- Rendszeres audit: Ellenőrizd mit feksz fel metafájlokban
- Metafájlok ne tartalmazzanak sensitive path-eket vagy internal endpoint-okat
