# Keresőmotoros Felderítés és Információszivárgás

|ID          |
|------------|
|WSTG-INFO-01|

## Összefoglaló

A keresőmotorok robotjai rendszeresen indexelik a weboldalakat. Ha nem megfelelően van konfigurálva a `robots.txt` vagy hiányoznak a meta tagek, érzékeny információk kerülhetnek az indexekbe: backup fájlok, development környezetek, konfigurációs fájlok, dokumentációk, hibás hibaüzenetek.

**Cél:** Megtalálni az indexelt érzékeny adatokat és információkat, amelyek segíthetik a további támadási vektorok azonosítását.

## Tesztelési Módszertan

### 1. Google Dorking

**Alapvető operátorok:**
```text
site:target.com                    # Domain szűrés
inurl:admin                        # URL-ben keresés
intitle:"index of"                 # Címben keresés
filetype:pdf                       # Fájltípus
intext:password                    # Tartalom szűrés
-www                               # Kizárás
```

**Gyakori dork-ok:**

```text
# Érzékeny fájlok
site:target.com filetype:sql
site:target.com filetype:env
site:target.com filetype:log
site:target.com ext:bak
site:target.com filetype:conf intext:password

# Alvéldák és dev környezetek
site:*.target.com -www
site:dev.target.com OR site:test.target.com OR site:staging.target.com
site:target.com inurl:dev
site:target.com inurl:staging

# Hibás konfigurációk
site:target.com inurl:phpinfo.php
site:target.com intitle:"Apache Status"
site:target.com intitle:"Index of" intext:".git"
site:target.com intitle:"directory listing"

# Admin felületek és login oldalak
site:target.com inurl:admin
site:target.com inurl:login
site:target.com intitle:"dashboard"
site:target.com inurl:wp-admin

# Dokumentáció és hibák
site:target.com intext:"sql syntax error"
site:target.com intext:"warning" intext:"mysql"
site:target.com filetype:txt intext:username

# Felhő és API
site:target.com inurl:api
site:target.com "*.s3.amazonaws.com"
site:target.com site:pastebin.com
```

### 2. Automatizált Dorking Eszközök

**Pagodo**
```bash
# Google Hacking Database automatizált keresés
pip3 install pagodo
pagodo -d target.com -g dorks.txt -l 50 -s -e 1.0
```

**dorkScanner**
```bash
git clone https://github.com/madhavmehndiratta/dorkScanner
cd dorkScanner
python3 dorkScanner.py -t target.com
```

**GooFuzz**
```bash
# Szótár alapú fuzzing Google-lel
git clone https://github.com/m3n0sd0n4ld/GooFuzz
python3 GooFuzz.py -t target.com -w wordlist.txt
```

### 3. Archív Oldalak Vizsgálata

**Wayback Machine**
```bash
# waybackurls - URL-ek gyűjtése
go install github.com/tomnomnom/waybackurls@latest
waybackurls target.com > urls.txt

# Érzékeny végpontok szűrése
cat urls.txt | grep -E "admin|config|backup|api|key"
```

**Waybackpack - Archív letöltés**
```bash
pip3 install waybackpack
waybackpack target.com -d ./archive --from-date 20200101 --to-date 20231231
```

**gau (GetAllUrls)**
```bash
# Több forrásból gyűjt URL-eket (Wayback, Common Crawl, etc.)
go install github.com/lc/gau/v2/cmd/gau@latest
gau target.com | tee urls.txt
```

### 4. GitHub és Kód Tárolók

**GitDorker**
```bash
git clone https://github.com/obheda12/GitDorker
python3 GitDorker.py -tf tokens.txt -q target.com -d dorks.txt
```

**GitHub keresés manuálisan:**
```text
target.com password
target.com api_key
target.com "BEGIN RSA PRIVATE KEY"
target.com extension:pem
org:targetorg password
org:targetorg token
```

**truffleHog - Git repo titkos adatok keresése**
```bash
trufflehog git https://github.com/target/repo --only-verified
```

**GitLeaks**
```bash
gitleaks detect --source . -v
```

### 5. Paste Site-ok és Leak Adatbázisok

**Manuális keresés:**
- pastebin.com
- paste.ubuntu.com
- slexy.org
- ghostbin.com
- justpaste.it

**psbdmp - Pastebin monitoring**
```bash
# API kulccsal keresés
curl "https://psbdmp.ws/api/v3/search/target.com"
```

### 6. Certificate Transparency Logs

**crt.sh**
```bash
# Subdomain felderítés SSL tanúsítványokból
curl "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u
```

**certspotter**
```bash
go install github.com/SSLMate/certspotter/cmd/certspotter@latest
certspotter -domain target.com
```

### 7. Shodan és IoT Keresők

**Shodan**
```bash
shodan search "hostname:target.com"
shodan search "ssl:target.com"
shodan search "org:'Target Organization'"
```

**Gyakori Shodan dork-ok:**
```text
hostname:target.com
ssl.cert.subject.cn:target.com
http.title:"Dashboard" hostname:target.com
port:3306 hostname:target.com
```

### 8. Social Media és Professional Networks

**LinkedIn**
- Munkatársak email formátumának megállapítása
- Használt technológiák azonosítása (skill-ek alapján)
- Szervezeti struktúra feltérképezése

**TheHarvester**
```bash
theHarvester -d target.com -b all -l 500
```

**h8mail - Email OSINT és breach checks**
```bash
h8mail -t admin@target.com -bc ";" -sk
```

### 9. Metadata Elemzés

**ExifTool**
```bash
# Letöltött fájlok metadata vizsgálata
exiftool document.pdf
exiftool -r ./downloaded_files/
```

**FOCA (Windows)**
- Metaadatok automatikus gyűjtése és elemzése
- Felhasználónevek, szoftververziók, belső útvonalak

### 10. DNS és Subdomain Keresés

**Amass**
```bash
amass enum -d target.com -o domains.txt
amass intel -d target.com -whois
```

**Subfinder**
```bash
subfinder -d target.com -o subdomains.txt
```

**assetfinder**
```bash
assetfinder --subs-only target.com
```

### 11. Egyéb OSINT Eszközök

**recon-ng**
```bash
recon-ng
marketplace install all
workspaces create target
db insert domains target.com
modules search
modules load recon/domains-hosts/google_site_web
run
```

**SpiderFoot**
```bash
# Web-based OSINT automatizálás
python3 sf.py -l 127.0.0.1:5001
# Browse: http://127.0.0.1:5001
```

**OSINT Framework**
- Böngésző-alapú: https://osintframework.com/
- Strukturált OSINT források kategorizálva

## Lehetséges Hatások és Ellenintézkedések

### Google Dorking találatok
**Hatás:** Érzékeny fájlok (SQL dump, .env, backup) közvetlen elérése credential-ök, API kulcsok, adatbázis kapcsolatok megszerzéséhez. Admin felületek, dev környezetek felfedése, amelyek gyengébb védelemmel rendelkezhetnek.

**Ellenintézkedés:**
- `robots.txt` és meta tagek (`<meta name="robots" content="noindex">`) megfelelő konfigurálása
- Rendszeres monitoring a nyilvános indexekben (`site:target.com` ellenőrzés)
- Google Search Console használata nemkívánatos indexelések eltávolítására
- Érzékeny fájlok hitelesítés mögé helyezése vagy teljes törlése

### Archív oldalak (Wayback Machine)
**Hatás:** Régi sebezhetőségek, már nem létező de indexelt végpontok, korábbi verziók kódjainak, konfigurációinak feltárása. Eltávolított vagy megváltozott admin felületek újbóli felfedezése.

**Ellenintézkedés:**
- Archive.org-nál kérvényezhető az archív tartalmak eltávolítása
- Érzékeny fájlok soha ne kerüljenek publikálásra, még átmenetileg sem
- Rendszeres audit a már nem használt tartalmakról

### GitHub és kód tárolók
**Hatás:** Hardcoded jelszavak, API kulcsok, privát SSH kulcsok, AWS credentials, belső infrastruktúra részletei. Egy kiszivárgott kulcs teljes rendszer kompromittálásához vezethet.

**Ellenintézkedés:**
- Git repository-k tisztítása titkos adatoktól (`.gitignore`, BFG Repo-Cleaner, git-secrets)
- Pre-commit hook-ok használata szenzitív adatok detektálásához
- Secret management rendszerek használata (Vault, AWS Secrets Manager)
- GitHub Secret Scanning és Dependabot alerts bekapcsolása

### Paste site-ok és leak adatbázisok
**Hatás:** Korábbi adatszivárgások, munkatársak által véletlenül megosztott információk, credential-ök. Aktív credential-ök használatával közvetlen hozzáférés a rendszerekhez.

**Ellenintézkedés:**
- Folyamatos monitoring szolgáltatások használata (HaveIBeenPwned monitoring)
- Azonnali jelszócsere szivárgás észlelésekor
- Kétfaktoros hitelesítés kényszerítése minden kritikus rendszeren

### Certificate Transparency Logs
**Hatás:** Összes subdomain felderítése, beleértve a rejtett, nem linkelt alvéldákat is (staging, admin, internal). Növelt támadási felület.

**Ellenintézkedés:**
- Wildcard tanúsítványok megfontolt használata
- Belső subdomainok külön, nem publikus CA-val történő aláírása
- Network szegmentálás és tűzfalszabályok megfelelő konfigurálása

### Shodan/Censys találatok
**Hatás:** Publikusan elérhető, rosszul konfigurált szolgáltatások (MongoDB, Redis, Elasticsearch) megtalálása. Azonnal kihasználható sebezhetőségek, adatok azonnali kiszűrése hitelesítés nélkül.

**Ellenintézkedés:**
- Minden szolgáltatás alapértelmezett hitelesítésének megváltoztatása
- Firewall szabályok: csak szükséges portok, csak megbízható IP-kről
- Rendszeres port scan saját infrastruktúrán külső nézőpontból

### Metadata PDF/dokumentumokban
**Hatás:** Belső felhasználónevek, email formátumok, szoftververziók, fájlrendszer útvonalak kiszivárgása. Phishing és social engineering kampányok támogatása.

**Ellenintézkedés:**
- Automatizált metadata tisztítás minden publikált dokumentumon
- Dokumentum generálási folyamatok átgondolása
- ExifTool vagy más tisztító eszközök beépítése a publication workflow-ba

### Social media reconnaissance
**Hatás:** Email formátumok, használt technológiák, rendszer architektúra, szervezeti hierarchia részletes térképe. Célzott spear-phishing és social engineering támadások előkészítése.

**Ellenintézkedés:**
- Munkatársak képzése a social media biztonságról
- Közösségi média policy kialakítása érzékeny információk megosztásáról
- LinkedIn és egyéb profilok review-ja biztonsági szempontból

### Development/staging környezetek
**Hatás:** Gyengébb védelem, debug módok bekapcsolva, alapértelmezett credential-ök. Ezeken keresztül éles környezet kompromittálása, source code hozzáférés.

**Ellenintézkedés:**
- Development/staging környezetek teljes izolálása VPN vagy IP whitelist mögé
- Különböző credential-ök használata minden környezetben
- Staging adatbázisok anonimizálása, valós production adat tilalma
