# WSTG-INFO-08: Webalkalmazás Framework Fingerprinting (Azonosítás)

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **azonosítsd a weboldal mögött álló framework-öt vagy CMS-t** (WordPress, Drupal, Laravel, Django, stb.). Mivel ezek **nyílt forráskódú projektek**, ismert sebezhetőségeik vannak, és **specifikus strukturájuk**, amiből felismerhetők. A fingerprinting **drasztikusan csökkenti a tesztelési időt**, mert ismert támadási vektorokat használhatsz.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM webszerver fingerprinting (Apache vs nginx)! Ez az **alkalmazás framework/CMS azonosítás** - WordPress, Joomla, Django, Laravel, stb.

---

## Mi a cél?

**Azonosítani a webalkalmazás által használt komponenseket**:
- Framework (Laravel, Django, Ruby on Rails, stb.)
- CMS (WordPress, Drupal, Joomla, stb.)
- Verzió számok
- Plugin-ek és modulok

### Miért fontos?

- **Ismert sebezhetőségek** - Ha WordPress 4.7.0, kereshetsz rá CVE-kre
- **Default struktúra** - Tudod, hol vannak admin panelek (`/wp-admin/`)
- **Specifikus exploitok** - Framework-specifikus támadások
- **Plugin exploitok** - Elavult plugin-ek gyakran sebezhetők
- **Gyorsabb teszt** - Nem kell mindent trial-and-error alapon

---

## 6 Fő Fingerprinting Módszer

### 1. **HTTP Headers**
### 2. **Cookies**
### 3. **HTML Source Code**
### 4. **Specific Files and Folders**
### 5. **File Extensions**
### 6. **Error Messages**

---

## 1. HTTP Headers Vizsgálata

### Concept:
A **`X-Powered-By`** és más custom header-ek gyakran **elárulják a framework-öt**.

### Tool #1: **curl**

#### Parancsok:
```bash
# Alapvető header check
curl -I https://pelda.hu

# Csak a X-Powered-By header
curl -I https://pelda.hu | grep -i "x-powered-by"

# Minden X- kezdetű header
curl -I https://pelda.hu | grep -i "^x-"

# Verbose mode (minden header)
curl -v https://pelda.hu 2>&1 | grep -i "< "
```

---

### Példák különböző framework-ökre:

#### PHP Framework (pl. Laravel):
```http
HTTP/1.1 200 OK
Server: nginx/1.18.0
X-Powered-By: PHP/7.4.3
X-Frame-Options: SAMEORIGIN
```

**Mit árul el:**
- PHP backend
- Verzió: 7.4.3
- Lehet Laravel, Symfony, CodeIgniter, stb.

---

#### ASP.NET:
```http
HTTP/1.1 200 OK
Server: Microsoft-IIS/10.0
X-Powered-By: ASP.NET
X-AspNet-Version: 4.0.30319
```

**Mit árul el:**
- ASP.NET framework
- IIS webserver
- .NET verzió

---

#### Node.js (Express):
```http
HTTP/1.1 200 OK
X-Powered-By: Express
```

**Mit árul el:**
- Express.js framework (Node.js)

---

#### Custom/Obfuscated Header:
```http
HTTP/1.1 200 OK
Server: nginx/1.0.14
X-Powered-By: Blood, sweat and tears
X-Generator: Swiftlet
```

**Mit árul el:**
- `X-Powered-By` rejtett (humor)
- De `X-Generator: Swiftlet` elárulta a framework-öt!

---

### Tool #2: **netcat (nc)**

#### Parancs:
```bash
nc pelda.hu 80
HEAD / HTTP/1.0

# HTTPS-hez (openssl)
echo -e "HEAD / HTTP/1.0\n\n" | openssl s_client -connect pelda.hu:443 -quiet
```

**Mit csinál:** Raw HTTP request küldése és header-ek olvasása.

---

## 2. Cookies Vizsgálata

### Concept:
Framework-specifikus **cookie nevek** árulkodóak.

### Tool #1: **Browser DevTools**

#### Használat:
```
1. F12 → Application → Cookies
2. Nézd meg a cookie neveket
3. Keresd meg a framework-specifikus neveket
```

---

### Framework-specifikus Cookie Nevek:

| Framework/CMS | Cookie Név |
|---------------|------------|
| **WordPress** | `wp-settings`, `wordpress_logged_in_` |
| **Drupal** | `SESS[hash]` |
| **Joomla** | `[hash]=...` (random hash) |
| **CakePHP** | `CAKEPHP` |
| **Laravel** | `laravel_session` |
| **Django** | `sessionid`, `csrftoken` |
| **phpBB** | `phpbb3_` |
| **Kohana** | `kohanasession` |
| **TYPO3** | `fe_typo_user` |
| **DotNetNuke** | `DotNetNukeAnonymous` |
| **Bitrix** | `BITRIX_` |
| **MODx** | `SN4[12symb]` |
| **Wix** | `Domain=.wix.com` |

---

### Tool #2: **curl + grep**

#### Parancsok:
```bash
# Set-Cookie header-ek
curl -I https://pelda.hu | grep -i "set-cookie"

# Összes cookie részletesen
curl -c cookies.txt https://pelda.hu
cat cookies.txt

# Egy request során létrejött cookie-k
curl -v https://pelda.hu 2>&1 | grep -i "set-cookie"
```

---

### Példa: WordPress Cookie:

```http
HTTP/1.1 200 OK
Set-Cookie: wordpress_test_cookie=WP+Cookie+check; path=/
Set-Cookie: wp-settings-1=libraryContent%3Dbrowse; path=/
```

**Azonosítás:** ✅ WordPress!

---

### Példa: Laravel Cookie:

```http
HTTP/1.1 200 OK
Set-Cookie: laravel_session=eyJpdiI6...; path=/; httponly
Set-Cookie: XSRF-TOKEN=eyJpdiI6...; path=/
```

**Azonosítás:** ✅ Laravel!

---

## 3. HTML Source Code Elemzése

### Concept:
**META tagek, kommentek, specifikus script/CSS path-ek** árulkodnak.

### Tool #1: **View Page Source + Ctrl+F**

#### Használat:
```
1. Jobb klikk → View Page Source (Ctrl+U)
2. Ctrl+F keresés:
   - "generator"
   - "powered by"
   - "built upon"
   - Framework nevére (wordpress, drupal, joomla)
```

---

### META Generator Tag Példák:

#### WordPress:
```html
<meta name="generator" content="WordPress 6.2.2" />
```

#### Drupal:
```html
<meta name="Generator" content="Drupal 9 (https://drupal.org)" />
```

#### Joomla:
```html
<meta name="generator" content="Joomla! - Open Source Content Management" />
```

#### MediaWiki:
```html
<meta name="generator" content="MediaWiki 1.39.0" />
```

---

### HTML Kommentek:

#### Adobe ColdFusion:
```html
<!-- START headerTags.cfm -->
```

#### ZK Framework:
```html
<!-- ZK 8.0.0 -->
<script type="text/javascript" src="/zkau/web/js/zk.wpd"></script>
```

#### Business Catalyst:
```html
<!-- BC_OBNW -->
```

---

### Specifikus Path-ek és Script-ek:

#### WordPress:
```html
<link rel="stylesheet" href="/wp-content/themes/twentytwentythree/style.css">
<script src="/wp-includes/js/jquery/jquery.min.js"></script>
```

**Azonosítás:** ✅ WordPress!
- `/wp-content/` path
- `/wp-includes/` path

---

#### Drupal:
```html
<script src="/core/misc/drupal.js"></script>
<script src="/sites/default/files/js/js_..."></script>
```

**Azonosítás:** ✅ Drupal!
- `/core/` path
- `/sites/default/` path

---

#### Joomla:
```html
<script src="/media/jui/js/jquery.min.js"></script>
<script src="/media/system/js/core.js"></script>
```

**Azonosítás:** ✅ Joomla!
- `/media/jui/` path
- `/media/system/` path

---

### ASP.NET ViewState:

```html
<input type="hidden" name="__VIEWSTATE" id="__VIEWSTATE" value="/wEPDwUKMTY3...">
<input type="hidden" name="__VIEWSTATEGENERATOR" id="__VIEWSTATEGENERATOR" value="CA0B0334">
```

**Azonosítás:** ✅ ASP.NET!

---

### Tool #2: **curl + grep**

#### Parancsok:
```bash
# META generator tag keresése
curl -s https://pelda.hu | grep -i '<meta name="generator"'

# Powered by keresése
curl -s https://pelda.hu | grep -i 'powered by'

# WordPress specifikus
curl -s https://pelda.hu | grep -i 'wp-content\|wp-includes'

# Drupal specifikus
curl -s https://pelda.hu | grep -i '/core/\|/sites/'

# Joomla specifikus
curl -s https://pelda.hu | grep -i '/media/jui/'
```

---

## 4. Specific Files and Folders (Dirbusting)

### Concept:
Minden framework-nek **sajátos file és mappa struktúrája** van. **Directory brute-forcing** révén ezeket megtalálhatod.

### Tool #1: **gobuster**

#### Parancsok:
```bash
# WordPress specifikus fájlok/mappák
gobuster dir -u https://pelda.hu -w wordpress-paths.txt

# Általános CMS paths
gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirb/common.txt

# Csak 200, 301, 302 status code-ok
gobuster dir -u https://pelda.hu -w wordlist.txt -s 200,301,302
```

---

### WordPress Default Paths:

```
/wp-admin/          → 302 Redirect (WordPress admin login)
/wp-content/        → 403 Forbidden (directory listing disabled)
/wp-includes/       → 403 Forbidden
/wp-login.php       → 200 OK (Login page)
/xmlrpc.php         → 200 OK (XML-RPC interface)
/readme.html        → 200 OK (Version info!)
/license.txt        → 200 OK
```

**Tipp:** `readme.html` és `license.txt` gyakran **verzió információt** tartalmaz!

---

### Drupal Default Paths:

```
/core/              → 403 Forbidden
/modules/           → 403 Forbidden
/sites/             → 403 Forbidden
/CHANGELOG.txt      → 200 OK (⚠️ Verzió info!)
/README.txt         → 200 OK
/INSTALL.txt        → 200 OK
/user/login         → 200 OK (Login page)
```

---

### Joomla Default Paths:

```
/administrator/     → 200 OK (Admin panel!)
/components/        → 403 Forbidden
/modules/           → 403 Forbidden
/plugins/           → 403 Forbidden
/templates/         → 403 Forbidden
/README.txt         → 200 OK
```

---

### Laravel Default Paths:

```
/public/            → Általában web root
/.env               → ⚠️ KRITIKUS! (Ha elérhető)
/storage/           → 403 Forbidden
/vendor/            → 403 Forbidden
/artisan            → 403 vagy 500
```

**Figyelem:** Ha `.env` elérhető → **KRITIKUS szivárgás!** (DB credentials, API keys)

---

### Tool #2: **wfuzz**

#### Parancsok:
```bash
# WordPress paths fuzzing
wfuzz -c -w wordpress-paths.txt --hc 404 https://pelda.hu/FUZZ

# Hide 404 responses
wfuzz -w wordlist.txt --hc 404 https://pelda.hu/FUZZ

# Multiple payloads (file + extension)
wfuzz -w files.txt -w extensions.txt https://pelda.hu/FUZZ.FUZ2Z
```

---

### robots.txt Ellenőrzés ELŐBB!

**Parancs:**
```bash
curl https://pelda.hu/robots.txt
```

**WordPress robots.txt példa:**
```
User-agent: *
Disallow: /wp-admin/
Disallow: /wp-includes/
Disallow: /wp-content/plugins/
Disallow: /wp-content/cache/
Disallow: /wp-content/themes/
```

**Azonosítás:** ✅ WordPress! (és látsz rejtett path-eket is)

---

### Plugin/Module Version Discovery

**Drupal plugin CHANGELOG példa:**
```
https://pelda.hu/sites/all/modules/botcha/CHANGELOG.txt

Botcha 7.x-3.1, 2014-03-15
--------------------------
- Fixed XSS vulnerability
...
```

**Mit árul el:**
- Drupal használatban van
- Botcha plugin telepítve
- Verzió: 7.x-3.1
- ⚠️ Lehet, hogy sebezhető (XSS volt 3.0-ban)

---

## 5. File Extensions Azonosítása

### Concept:
**URL file extension-ök** árulkodnak a backend technológiáról.

### Gyakori File Extension-ök:

| Extension | Technológia |
|-----------|-------------|
| `.php` | PHP (WordPress, Drupal, Laravel, stb.) |
| `.aspx` | ASP.NET (Microsoft) |
| `.asp` | Classic ASP |
| `.jsp` | Java Server Pages |
| `.do` | Java (Struts framework) |
| `.action` | Java (Struts2) |
| `.py` | Python (Django, Flask ritkán látható) |
| `.rb` | Ruby (Ruby on Rails) |
| `.cfm` | ColdFusion |
| `.pl` | Perl |

---

### Példák URL-ekből:

```
https://pelda.hu/index.php?page=home
→ PHP backend

https://pelda.hu/login.aspx
→ ASP.NET

https://pelda.hu/search.jsp?q=test
→ Java Server Pages

https://pelda.hu/admin.do
→ Java Struts framework
```

---

### Tool: **curl + grep extension**

#### Parancs:
```bash
# PHP fájlok keresése
curl -s https://pelda.hu | grep -oP 'href="[^"]*\.php"'

# ASPX fájlok
curl -s https://pelda.hu | grep -oP 'href="[^"]*\.aspx"'

# JSP fájlok
curl -s https://pelda.hu | grep -oP 'href="[^"]*\.jsp"'
```

---

## 6. Error Messages Elemzése

### Concept:
**Hibaüzenetek** gyakran elárulják a framework-öt, verzió számot, és file path-eket.

### Tool #1: **Trigger Errors Manually**

#### Módszerek:
```bash
# Nem létező oldal
curl https://pelda.hu/nonexistent-page-xyz123

# Hibás paraméter
curl https://pelda.hu/index.php?id=abc

# SQL injection karakterek (error trigger)
curl "https://pelda.hu/product.php?id=1'"

# Path traversal
curl https://pelda.hu/file.php?path=../../../etc/passwd
```

---

### WordPress Error Példa:

```
Parse error: syntax error, unexpected ';' in 
/var/www/html/wp-content/themes/mytheme/functions.php on line 42
```

**Mit árul el:**
- ✅ WordPress
- ✅ PHP backend
- ✅ File path: `/var/www/html/wp-content/`
- ✅ Theme name: `mytheme`
- ⚠️ Debug mode enabled (production-ban NEM kellene!)

---

### Laravel Error Példa:

```
Whoops, looks like something went wrong.

1/1 ErrorException in Kernel.php line 123:
Undefined variable: user

in Kernel.php line 123
at HandleExceptions->handleError(...)
```

**Mit árul el:**
- ✅ Laravel framework (Whoops error handler)
- ✅ File: `Kernel.php` (Laravel core)
- ⚠️ Debug mode enabled

---

### Django Error Példa:

```html
<h1>OperationalError at /admin/</h1>
<pre class="exception_value">no such table: django_session</pre>

Python: 3.9.5
Django: 3.2.4
```

**Mit árul el:**
- ✅ Django framework
- ✅ Python 3.9.5
- ✅ Django 3.2.4
- ⚠️ Database issue + Debug mode

---

## Automatizált Toolok

### Tool #1: **WhatWeb**

**Telepítés:**
```bash
# Debian/Ubuntu
apt-get install whatweb

# Kali-ban már benne van
```

#### Parancsok:
```bash
# Alapvető scan
whatweb https://pelda.hu

# Verbose output
whatweb -v https://pelda.hu

# Aggressive mode (több request)
whatweb -a 3 https://pelda.hu

# Multiple targets
whatweb -i targets.txt

# Output JSON-be
whatweb --log-json=results.json https://pelda.hu
```

**Kimenet példa:**
```
https://pelda.hu [200 OK] 
  Country[HUNGARY][HU], 
  WordPress[6.2.2], 
  Apache[2.4.41], 
  PHP[7.4.3], 
  Title[My WordPress Site], 
  MetaGenerator[WordPress 6.2.2]
```

**Mit csinál:**
- **90+ plugin** különböző fingerprinting-re
- Text strings, regex, MD5 hash matching
- HTML tag patterns
- Cookie detection
- Nagyon gyors és pontos

---

### Tool #2: **Wappalyzer** (Browser Extension)

**Telepítés:**
- Chrome: Web Store → Wappalyzer
- Firefox: Add-ons → Wappalyzer

#### Használat:
```
1. Telepítsd a browser extension-t
2. Látogass el a target weboldara
3. Klikk a Wappalyzer ikonra
4. Lásd az azonosított technológiákat
```

**Mit mutat:**
- CMS (WordPress, Drupal, Joomla)
- Framework (Laravel, Django, React)
- Analytics (Google Analytics, Matomo)
- JavaScript libraries (jQuery, Bootstrap)
- Web servers (nginx, Apache)
- CDN (Cloudflare, Fastly)

**Előny:**
- ✅ Egyszerű használat
- ✅ Azonnal látod böngészés közben
- ✅ Ikon-alapú megjelenítés

**Hátrány:**
- ❌ Csak amit a browser lát (client-side)
- ❌ False positive-ok előfordulhatnak

---

### Tool #3: **BuiltWith** (Online Service)

**URL:** https://builtwith.com

**Használat:**
```
1. Menj a builtwith.com-ra
2. Írd be a domain nevet
3. Részletes report minden technológiáról
```

**Mit ad:**
- CMS és framework
- Hosting provider
- Analytics és tracking
- CDN és caching
- SSL certificate info
- **Historical data** (régi technológiák)

---

### Tool #4: **Nikto**

#### Parancsok:
```bash
# Alapvető scan (CMS detection is)
nikto -h https://pelda.hu

# Specific tests
nikto -h https://pelda.hu -Tuning 1,2,3

# Output fájlba
nikto -h https://pelda.hu -o nikto_report.html -Format html
```

**Mit csinál:**
- Vulnerability scanner
- CMS/framework detection
- Default file detection
- Version disclosure

---

### Tool #5: **CMSeek**

**Telepítés:**
```bash
git clone https://github.com/Tuhinshubhra/CMSeeK
cd CMSeeK
pip3 install -r requirements.txt
```

#### Parancsok:
```bash
# WordPress detection
python3 cmseek.py -u https://pelda.hu

# Deep scan
python3 cmseek.py -u https://pelda.hu --follow-redirect

# Multiple targets
python3 cmseek.py --list targets.txt
```

**Támogatott CMS-ek:**
- WordPress
- Joomla
- Drupal
- Magento
- vBulletin
- 170+ CMS

---

### Tool #6: **WPScan** (WordPress Specific)

#### Parancsok:
```bash
# Alapvető WordPress scan
wpscan --url https://pelda.hu

# Enumerate plugins
wpscan --url https://pelda.hu --enumerate p

# Enumerate themes
wpscan --url https://pelda.hu --enumerate t

# Enumerate users
wpscan --url https://pelda.hu --enumerate u

# Aggressive detection
wpscan --url https://pelda.hu --enumerate ap,at,u --plugins-detection aggressive
```

**Mit ad:**
- WordPress verzió
- Telepített plugin-ek + verziók
- Telepített theme-ek
- User enumeration
- Ismert sebezhetőségek (CVE-k)

---

## Framework-Specific Fingerprinting

### WordPress Fingerprinting:

**Gyors check parancsok:**
```bash
# Version from generator meta tag
curl -s https://pelda.hu | grep -i 'generator.*wordpress'

# wp-json REST API endpoint
curl https://pelda.hu/wp-json/

# readme.html (version info)
curl https://pelda.hu/readme.html | grep -i version
```

**Eredmény:**
```json
{
  "name": "My WordPress Site",
  "description": "Just another WordPress site",
  "url": "https://pelda.hu",
  "home": "https://pelda.hu",
  "namespaces": ["oembed/1.0", "wp/v2"]
}
```

---

### Drupal Fingerprinting:

**Gyors check:**
```bash
# CHANGELOG.txt (version)
curl https://pelda.hu/CHANGELOG.txt

# X-Generator header
curl -I https://pelda.hu | grep -i x-generator

# Drupal-specific path
curl https://pelda.hu/core/misc/drupal.js
```

---

### Joomla Fingerprinting:

**Gyors check:**
```bash
# Joomla META tag
curl -s https://pelda.hu | grep -i 'joomla'

# Administrator path
curl -I https://pelda.hu/administrator/

# Joomla-specific JS
curl https://pelda.hu/media/system/js/core.js
```

---

### Laravel Fingerprinting:

**Gyors check:**
```bash
# Laravel cookie
curl -I https://pelda.hu | grep -i 'laravel_session'

# Debug page (if debug=true)
curl https://pelda.hu/intentional-error

# .env exposure (CRITICAL if accessible!)
curl https://pelda.hu/.env
```

---

## Komplex Fingerprinting Script

### Teljes Automated Framework Detection:

```bash
#!/bin/bash

TARGET="https://pelda.hu"
OUTPUT="framework_detection.txt"

echo "[*] Framework Fingerprinting for $TARGET" > $OUTPUT
echo "==========================================" >> $OUTPUT

# 1. HTTP Headers
echo -e "\n[+] HTTP Headers:" >> $OUTPUT
curl -I $TARGET | grep -i "server\|x-powered-by\|x-generator\|x-aspnet" >> $OUTPUT

# 2. Cookies
echo -e "\n[+] Cookies:" >> $OUTPUT
curl -I $TARGET | grep -i "set-cookie" >> $OUTPUT

# 3. META Generator
echo -e "\n[+] META Generator:" >> $OUTPUT
curl -s $TARGET | grep -i '<meta name="generator"' >> $OUTPUT

# 4. Common Paths Check
echo -e "\n[+] Path Detection:" >> $OUTPUT

# WordPress
if curl -s $TARGET/wp-login.php | grep -q "WordPress"; then
    echo "  [!] WordPress detected!" >> $OUTPUT
    curl -s $TARGET/readme.html | grep -i "version" >> $OUTPUT
fi

# Drupal
if curl -s $TARGET/CHANGELOG.txt | grep -q "Drupal"; then
    echo "  [!] Drupal detected!" >> $OUTPUT
    curl -s $TARGET/CHANGELOG.txt | head -5 >> $OUTPUT
fi

# Joomla
if curl -s $TARGET/administrator/ | grep -q "Joomla"; then
    echo "  [!] Joomla detected!" >> $OUTPUT
fi

# Laravel
if curl -I $TARGET | grep -q "laravel_session"; then
    echo "  [!] Laravel detected!" >> $OUTPUT
fi

# 5. File Extensions
echo -e "\n[+] File Extensions:" >> $OUTPUT
curl -s $TARGET | grep -oP 'href="[^"]*\.(php|aspx|jsp|do)"' | head -10 >> $OUTPUT

# 6. WhatWeb scan
echo -e "\n[+] WhatWeb Scan:" >> $OUTPUT
whatweb $TARGET >> $OUTPUT

echo -e "\n[*] Scan complete! Results in $OUTPUT"
cat $OUTPUT
```

**Használat:**
```bash
chmod +x framework_detect.sh
./framework_detect.sh
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| HTTP headers | `curl -I https://pelda.hu` |
| X-Powered-By | `curl -I https://pelda.hu \| grep -i x-powered-by` |
| Cookies | `curl -I https://pelda.hu \| grep -i set-cookie` |
| META generator | `curl -s https://pelda.hu \| grep -i generator` |
| WordPress check | `curl -s https://pelda.hu/wp-login.php` |
| Drupal CHANGELOG | `curl https://pelda.hu/CHANGELOG.txt` |
| WhatWeb scan | `whatweb https://pelda.hu` |
| Wappalyzer | Browser extension → Click icon |
| WPScan | `wpscan --url https://pelda.hu` |
| Directory brute-force | `gobuster dir -u https://pelda.hu -w wordlist.txt` |

---

## Framework Detection Reference Table

| Framework/CMS | Cookie | META Tag | Path | Extension |
|---------------|--------|----------|------|-----------|
| WordPress | `wp-settings` | `WordPress X.X` | `/wp-admin/` | `.php` |
| Drupal | `SESS[hash]` | `Drupal X` | `/core/` | `.php` |
| Joomla | Random hash | `Joomla!` | `/administrator/` | `.php` |
| Laravel | `laravel_session` | - | `/public/` | `.php` |
| Django | `sessionid` | - | `/admin/` | `.py` (ritkán) |
| ASP.NET | - | - | - | `.aspx` |
| Ruby on Rails | `_session_id` | - | - | `.rb` (ritkán) |
| Express.js | `connect.sid` | - | - | Node.js |

---

## Védelem (Remediation)

### Hogyan védekezz a fingerprinting ellen?

1. **Remove/Hide Headers**
   ```nginx
   # nginx config
   server_tokens off;
   more_clear_headers 'X-Powered-By';
   ```

2. **Custom Cookie Names**
   ```php
   // WordPress config
   ini_set('session.name', 'MY_CUSTOM_SESSION');
   ```

3. **Remove META Generator Tags**
   ```php
   // WordPress functions.php
   remove_action('wp_head', 'wp_generator');
   ```

4. **Block Default Paths**
   ```nginx
   location ~ /(readme|license|changelog)\.txt {
       deny all;
   }
   ```

5. **Custom Error Pages**
   - Ne mutass stack trace-t
   - Generic error messages

**AZONBAN:**
- **Security through obscurity ≠ Biztonság!**
- Ezek csak **lassítják** az atackoló-t
- **Fontosabb:** Tartsd frissen a rendszert, patcheld a sebezhetőségeket!

---

## Fontos Megjegyzések

✅ **Framework fingerprinting = legális** recon  
✅ **Ismert framework → ismert sebezhetőségek**  
✅ **Kombináld a módszereket** - header + cookie + path  
✅ **WhatWeb + Wappalyzer** = legjobb kombó  
❌ **NE elégedj meg egy módszerrel** - lehet obfuscated  
⚠️ **Version info = arany** - keress CVE-ket hozzá!

---

**Összefoglalva:** Ez a fejezet a **webalkalmazás framework/CMS azonosításáról** szól. **6 módszer** van: HTTP headers (`X-Powered-By`), Cookies (`wp-settings`), HTML source (`<meta generator>`), Specific files (`/wp-admin/`), File extensions (`.php`, `.aspx`), és Error messages. **Automatizált toolok** (WhatWeb, Wappalyzer, WPScan) jelentősen meggyorsítják a folyamatot. Az **azonosított framework + verzió** alapján **kereshetsz ismert sebezhetőségeket** (CVE-k) és **specifikus exploitokat** használhatsz!
