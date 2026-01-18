# WSTG-INFO-03: Webszerver Metafájlok Vizsgálata Információszivárgásra

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy hogyan **vizsgáld meg a különböző metafájlokat** (robots.txt, sitemap.xml, security.txt, stb.), amelyek **érzékeny információkat szivárogtathatnak** a weboldal struktúrájáról, rejtett útvonalakról, vagy technológiai részletekről.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM a HTML kód vizsgálata, NEM XSS keresés! Ez **speciális fájlok elemzése**, amelyek metainformációkat tartalmaznak a weboldalról és gyakran **véletlenül felfednek rejtett mappákat vagy funkciókat**.

---

## Mi a cél?

**Rejtett vagy eltakart útvonalak és funkciók azonosítása** metafájlok elemzésével, valamint:
- Információgyűjtés a rendszerekről
- Attack surface (támadási felület) feltérképezése
- Technológiai részletek kinyerése
- Social engineering alapok gyűjtése

### Mit keresünk?

- **Rejtett mappák és útvonalak** (amiket robotok nem indexelhetnek)
- **Admin panelek, backup könyvtárak**
- **Fejlesztői/teszt környezetek**
- **Technológiai stack** (CMS, frameworkök)
- **Kapcsolattartók, fejlesztők nevei**
- **Bug bounty programok**

---

## Vizsgálandó Fájlok

### 1. **robots.txt**
### 2. **META tagek** (HTML fejlécben)
### 3. **sitemap.xml**
### 4. **security.txt**
### 5. **humans.txt**
### 6. **Egyéb .well-known/ fájlok**

---

## 1. robots.txt Vizsgálata

### Concept:
A `robots.txt` fájl megmondja a keresőmotoroknak, hogy **mely oldalakat NE indexeljék**. Paradox módon pont **ezek az oldalak a legérdekesebbek**, mert ha valaki el akarja rejteni, valószínűleg érzékeny!

### Lokáció:
```
https://pelda.hu/robots.txt
```

### Tool #1: **curl**

#### Parancsok:
```bash
curl https://pelda.hu/robots.txt

curl -s https://pelda.hu/robots.txt | grep -i disallow

curl -s https://pelda.hu/robots.txt | grep -E "Disallow|Allow"
```

**Mit csinál:**
- Letölti a robots.txt fájlt
- `-s` = silent mode
- `grep` = csak a Disallow/Allow sorokat szűri

---

### Tool #2: **wget**

#### Parancsok:
```bash
wget https://pelda.hu/robots.txt

wget --no-verbose https://pelda.hu/robots.txt && cat robots.txt

wget -qO- https://pelda.hu/robots.txt | grep Disallow
```

**Mit csinál:**
- `--no-verbose` = kevesebb kimenet
- `-qO-` = quiet mode, kimenetet stdout-ra írja
- Letölti és megjeleníti a fájlt

---

### Tool #3: **robotstxt Parser (Python)**

#### Parancs:
```bash
python3 -c "
import urllib.robotparser
rp = urllib.robotparser.RobotFileParser()
rp.set_url('https://pelda.hu/robots.txt')
rp.read()
print(rp.can_fetch('*', '/admin'))
"
```

**Mit csinál:** Python scripttel ellenőrzi, hogy egy adott útvonal engedélyezett-e.

---

### robots.txt Példa (Google):

```text
User-agent: *
Disallow: /search
Allow: /search/about
Disallow: /sdch
Disallow: /admin
Disallow: /backup
Disallow: /private
```

**Elemzés:**
- `User-agent: *` = minden robotra vonatkozik
- `Disallow: /admin` = ⚠️ **ÉRDEKES!** Van admin panel!
- `Disallow: /backup` = ⚠️ **ÉRDEKES!** Van backup könyvtár!
- `Disallow: /private` = ⚠️ **ÉRDEKES!** Van privát terület!

**Fontos:** A robots.txt **NEM biztonsági mechanizmus**! A tiltás ellenére IS elérhető az útvonal!

---

### Google Webmaster Tools

**URL:** https://www.google.com/webmasters/tools

**Funkcionalitás:**
- "Analyze robots.txt" funkció
- Tesztelheted, hogy a Google hogyan látja a robots.txt-et
- Szimulálhatod a crawlinget

---

## 2. META Tagek Vizsgálata

### Concept:
HTML `<HEAD>` részben található META tagek **technológiai információkat** és **robot direktívákat** tartalmazhatnak.

### Tool #1: **curl + grep**

#### Parancsok:
```bash
curl -s https://pelda.hu | grep -i '<meta'

curl -s https://pelda.hu | grep -i 'meta name="robots"'

curl -s https://pelda.hu | grep -oP '<meta[^>]+>' | head -20
```

**Mit csinál:**
- Letölti a HTML-t és META tageket keres
- `-oP` = Perl regex, csak a matchelt részt írja ki
- `head -20` = első 20 találat

---

### Tool #2: **View Page Source (böngésző)**

1. Nyisd meg a weboldalt böngészőben
2. Jobb klikk → **View Page Source** (vagy Ctrl+U)
3. Keresd meg: `<meta` (Ctrl+F)

---

### Érdekes META Tagek:

#### Robots META Tag:
```html
<META NAME="ROBOTS" CONTENT="NOINDEX, NOFOLLOW">
```
**Jelentés:** Ez az oldal NE legyen indexelve!

---

#### Open Graph (Social Media):
```html
<meta property="og:title" content="The White House" />
<meta property="og:description" content="..." />
<meta property="og:image" content="https://example.com/image.png" />
<meta property="og:url" content="https://www.example.com/" />
<meta property="fb:app_id" content="1790466490985150" />
```

**Mit árul el:**
- Facebook App ID → Social media integráció
- Image URL → Lehet érdekes útvonal
- Site description → Kontextus

---

#### Twitter Card:
```html
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="@whitehouse" />
<meta name="twitter:creator" content="@whitehouse" />
```

**Mit árul el:**
- Twitter handle → Social engineering
- Creator info → Ki készítette

---

#### Technológiai META tagek:
```html
<meta name="generator" content="WordPress 5.8" />
<meta name="application-name" content="MyApp">
<meta name="theme-color" content="#f5f5f5">
<meta name="apple-mobile-web-app-title" content="MyApp">
```

**Mit árul el:**
- CMS típus és verzió (WordPress 5.8)
- Alkalmazás neve
- Mobile app konfiguráció

---

### Tool #3: **whatweb** (META info is)

#### Parancs:
```bash
whatweb -v https://pelda.hu | grep -i meta
```

**Mit csinál:** META tageket is detektál a technológiák mellett.

---

## 3. sitemap.xml Vizsgálata

### Concept:
A `sitemap.xml` fájl **térképet ad az összes oldalról**, amit a keresőmotorok indexelhetnek. Gyakran tartalmaz **rejtett vagy nem linkelt oldalakat** is!

### Lokáció:
```
https://pelda.hu/sitemap.xml
https://pelda.hu/sitemap_index.xml
https://pelda.hu/sitemap/sitemap.xml
```

### Tool #1: **wget**

#### Parancsok:
```bash
wget https://pelda.hu/sitemap.xml

wget --no-verbose https://pelda.hu/sitemap.xml && cat sitemap.xml

wget -qO- https://pelda.hu/sitemap.xml | grep -oP '<loc>\K[^<]+'
```

**Mit csinál:**
- Letölti a sitemap-et
- `grep -oP '<loc>\K[^<]+'` = csak az URL-eket nyeri ki a `<loc>` tagekből

---

### Tool #2: **curl + xmllint**

#### Parancsok:
```bash
curl -s https://pelda.hu/sitemap.xml | xmllint --format -

curl -s https://pelda.hu/sitemap.xml | grep '<loc>'

curl -s https://pelda.hu/sitemap.xml | grep -oP '(?<=<loc>).*?(?=</loc>)'
```

**Mit csinál:**
- `xmllint --format -` = szépen formázza az XML-t
- Regex-szel kiszedi az URL-eket

---

### sitemap.xml Példa:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="https://www.google.com/schemas/sitemap/0.84">
  <sitemap>
    <loc>https://www.google.com/gmail/sitemap.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://www.google.com/forms/sitemaps.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://www.google.com/admin/sitemap.xml</loc>
  </sitemap>
</sitemapindex>
```

**Elemzés:**
- `gmail/sitemap.xml` → Gmail aloldal sitemap
- `admin/sitemap.xml` → ⚠️ **ÉRDEKES!** Admin terület sitemap!

---

### Tool #3: **Python script sitemap parsing**

#### Parancs:
```bash
python3 -c "
import requests
from xml.etree import ElementTree as ET

r = requests.get('https://pelda.hu/sitemap.xml')
root = ET.fromstring(r.content)
for loc in root.iter('{http://www.sitemaps.org/schemas/sitemap/0.9}loc'):
    print(loc.text)
"
```

**Mit csinál:** Python-nal parse-olja a sitemap XML-t és kiírja az URL-eket.

---

## 4. security.txt Vizsgálata

### Concept:
A `security.txt` fájl **biztonsági kapcsolattartási információkat** tartalmaz (RFC 9116). Hasznos social engineering-hez és bug bounty infókhoz.

### Lokáció:
```
https://pelda.hu/security.txt
https://pelda.hu/.well-known/security.txt
```

### Tool #1: **curl**

#### Parancsok:
```bash
curl https://pelda.hu/.well-known/security.txt

curl https://pelda.hu/security.txt

curl -s https://pelda.hu/.well-known/security.txt | grep -E "Contact|Encryption|Policy"
```

**Mit csinál:** Letölti a security.txt-et és fontos mezőket szűr.

---

### security.txt Példa (LinkedIn):

```text
Contact: mailto:security@linkedin.com
Contact: https://www.linkedin.com/help/linkedin/answer/62924
Encryption: https://www.linkedin.com/help/linkedin/answer/79676
Canonical: https://www.linkedin.com/.well-known/security.txt
Policy: https://www.linkedin.com/help/linkedin/answer/62924
```

**Mit árul el:**
- **Contact:** Security csapat email címe → Social engineering
- **Encryption:** PGP public key → Key metadata (Name, Email)
- **Policy:** Bug bounty program → Lehet pénzt keresni :)

---

### OpenPGP Public Key Metadata:

Ha van `Encryption:` mező egy PGP kulccsal, azt elemezd:

```bash
curl -s https://pelda.hu/pgp-key.asc | gpg --import
gpg --list-keys
```

**Mit árulhat el:**
- Key Owner neve és email címe
- Key létrehozási dátum
- Key expiráció
- Key méret (biztonsági szint)

---

## 5. humans.txt Vizsgálata

### Concept:
A `humans.txt` fájl **emberekről szól**, akik a weboldalt készítették. Karrierinfók, fejlesztők nevei, stb.

### Lokáció:
```
https://pelda.hu/humans.txt
```

### Tool #1: **wget**

#### Parancsok:
```bash
wget https://pelda.hu/humans.txt

wget --no-verbose https://pelda.hu/humans.txt && cat humans.txt

curl -s https://pelda.hu/humans.txt
```

---

### humans.txt Példa (Google):

```text
Google is built by a large team of engineers, designers, researchers, robots, 
and others in many different sites across the globe. It is updated continuously, 
and built with more tools and technologies than we can shake a stick at. 
If you'd like to help us out, see careers.google.com.
```

**Mit árul el:**
- Karrieroldalak linkje → Technológiai stack infók
- Csapat összetétel → Social engineering

---

## 6. Egyéb .well-known/ Fájlok

### Concept:
A `.well-known/` könyvtár **szabványosított fájlokat** tartalmaz (RFC-k szerint).

### Érdekes .well-known/ fájlok:

| Fájl | Funkció |
|------|---------|
| `/.well-known/security.txt` | Biztonsági kapcsolattartók |
| `/.well-known/change-password` | Jelszóváltoztatás útvonal |
| `/.well-known/openid-configuration` | OpenID Connect konfig |
| `/.well-known/assetlinks.json` | Android app links |
| `/.well-known/apple-app-site-association` | iOS app links |

### Tool #1: **curl - .well-known fájlok**

#### Parancsok:
```bash
curl https://pelda.hu/.well-known/security.txt

curl https://pelda.hu/.well-known/change-password

curl https://pelda.hu/.well-known/openid-configuration
```

---

### Tool #2: **ffuf - .well-known fuzzing**

#### Parancs:
```bash
ffuf -w wordlist.txt -u https://pelda.hu/.well-known/FUZZ

# Saját wordlist létrehozása RFC alapján:
cat > well-known-list.txt << EOF
security.txt
change-password
openid-configuration
assetlinks.json
apple-app-site-association
EOF

ffuf -w well-known-list.txt -u https://pelda.hu/.well-known/FUZZ
```

**Mit csinál:** Végigpróbálja a lehetséges .well-known fájlokat.

---

### IANA lista:
**URL:** https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml

Hivatalos lista az összes .well-known/ fájlról.

---

## Google Dorking Metafájlokra

### Parancsok:

```bash
# robots.txt keresése
site:pelda.hu inurl:robots.txt

# sitemap keresése
site:pelda.hu inurl:sitemap.xml

# security.txt keresése
site:pelda.hu inurl:security.txt

# humans.txt keresése
site:pelda.hu inurl:humans.txt

# .well-known könyvtár
site:pelda.hu inurl:.well-known
```

---

## Automatizált Toolok

### Tool #1: **Burp Suite**

1. Indítsd el Burp Suite-ot
2. Állítsd be a Proxy-t
3. Navigate to **Target → Site map**
4. Keresd meg a metafájlokat (robots.txt, sitemap.xml)

**Spider funkció:** Automatikusan crawlolja a robots.txt Disallow útvonalakat is!

---

### Tool #2: **OWASP ZAP**

#### Használat:
1. Indítsd el ZAP-ot
2. Quick Start → Automated Scan
3. Add meg a target URL-t
4. ZAP automatikusan ellenőrzi:
   - robots.txt
   - sitemap.xml
   - META tageket

---

### Tool #3: **WhatWeb**

#### Parancs:
```bash
whatweb -v https://pelda.hu
```

**Mit csinál:** META tageket és metafájlokat is elemez.

---

### Tool #4: **Custom Python Script**

#### Teljes metafájl scraper:
```python
#!/usr/bin/env python3
import requests

domain = "https://pelda.hu"
files = [
    "/robots.txt",
    "/sitemap.xml",
    "/security.txt",
    "/.well-known/security.txt",
    "/humans.txt",
    "/.well-known/change-password"
]

for f in files:
    url = domain + f
    r = requests.get(url)
    if r.status_code == 200:
        print(f"[+] Found: {url}")
        print(r.text[:200])
    else:
        print(f"[-] Not found: {url}")
```

**Használat:**
```bash
chmod +x metafile_scanner.py
./metafile_scanner.py
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| robots.txt letöltés | `curl https://pelda.hu/robots.txt` |
| Disallow sorok szűrése | `curl -s https://pelda.hu/robots.txt \| grep Disallow` |
| sitemap.xml letöltés | `wget https://pelda.hu/sitemap.xml` |
| sitemap URL-ek kinyerése | `curl -s https://pelda.hu/sitemap.xml \| grep -oP '<loc>\K[^<]+'` |
| security.txt ellenőrzés | `curl https://pelda.hu/.well-known/security.txt` |
| humans.txt letöltés | `curl https://pelda.hu/humans.txt` |
| META tagek keresése | `curl -s https://pelda.hu \| grep -i '<meta'` |
| .well-known fuzzing | `ffuf -w list.txt -u https://pelda.hu/.well-known/FUZZ` |

---

## Komplex Workflow Példa

### 1. lépés: robots.txt elemzés
```bash
curl -s https://pelda.hu/robots.txt | grep Disallow > disallowed_paths.txt
cat disallowed_paths.txt
```

### 2. lépés: Tiltott útvonalak tesztelése
```bash
while read path; do
    url="https://pelda.hu${path}"
    status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    echo "$url - $status"
done < <(grep -oP 'Disallow: \K.*' disallowed_paths.txt)
```

### 3. lépés: sitemap elemzés
```bash
curl -s https://pelda.hu/sitemap.xml | grep -oP '<loc>\K[^<]+' > all_urls.txt
wc -l all_urls.txt
```

### 4. lépés: security.txt és kapcsolattartók
```bash
curl -s https://pelda.hu/.well-known/security.txt | grep Contact
```

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **Ne rakj érzékeny infót robots.txt-be**
   - Ha nem akarod, hogy indexeljék, használj autentikációt!

2. **Figyeld a META tageket**
   - Ne használj `<meta name="generator">` tageket
   - Távolítsd el a verzió információkat

3. **sitemap.xml gondos kezelése**
   - Ne legyen benne dev/admin útvonal
   - Csak publikus oldalak

4. **security.txt helyesen**
   - Használd, de ne árulj el túl sokat
   - PGP key metadata legyen minimal

5. **humans.txt óvatosan**
   - Ne írd ki a fejlesztők teljes nevét
   - Ne adj ki részletes tech stacket

---

## Fontos Megjegyzések

✅ **Ez passzív recon** - csak letöltöd a fájlokat  
✅ **Kombináld más technikákkal** - robots.txt → directory bruteforce  
✅ **Mindig nézd meg mind a 6 fájltípust**  
❌ **NE feltételezd, hogy léteznek** - nem minden oldalon van  
⚠️ **robots.txt != védelem** - a tiltott útvonalak IS elérhetők!

---

**Összefoglalva:** Ez a fejezet a **metafájlok vizsgálatáról** szól (robots.txt, sitemap.xml, security.txt, stb.). Ezek a fájlok gyakran **véletlenül felfednek rejtett útvonalakat, admin paneleket, vagy technológiai részleteket**. A robots.txt `Disallow` direktívái paradox módon a legérdekesebb oldalakat jelölik ki, mert pont ezeket akarják elrejteni!
