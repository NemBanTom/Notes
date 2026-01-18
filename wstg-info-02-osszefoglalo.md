# WSTG-INFO-02: Webszerver Fingerprinting (Azonosítás)

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy hogyan **azonosítsd a webszerver típusát és verzióját**. A cél megállapítani, hogy milyen szoftver fut a célrendszeren (Apache, nginx, IIS, stb.) és milyen verziószámmal, mert a **régi, nem frissített szerverek ismert sebezhetőségekkel rendelkezhetnek**.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM a weboldal tartalmának elemzése, NEM SQL injection! Ez **a webszerver szoftverének azonosítása** - tehát az infrastruktúrával foglalkozik, nem az alkalmazással.

---

## Mi a cél?

**Meghatározni a futó webszerver verzióját és típusát**, hogy:
- Ismert sebezhetőségeket találj hozzá
- Verzió-specifikus exploitokat azonosíts
- Jobb támadási felületet kapj

### Mit szeretnénk megtudni?

- **Szerver típusa**: Apache, nginx, IIS, lighttpd, stb.
- **Verzió**: pl. Apache/2.4.41, nginx/1.17.3
- **Operációs rendszer**: Unix, Windows
- **Modulok és konfigurációk**: mod_ssl, mod_security, stb.

---

## Technikák

### 1. **Banner Grabbing** (Fejléc információk gyűjtése)
### 2. **Malformed Requests** (Hibás kérések küldése)
### 3. **Automatizált Toolok** (Nmap, Nikto, Netcraft)

---

## 1. Banner Grabbing (Fejléc Elemzés)

### Concept:
HTTP válasz fejléceket vizsgálod, mert sok szerver **kiírja saját magát** a `Server:` headerbe.

### Tool #1: **curl**

#### Parancsok:
```bash
curl -I http://pelda.hu
curl -I https://pelda.hu
curl -sI http://pelda.hu | grep -i server
```

**Kimenet példa (Apache):**
```
Server: Apache/2.4.41 (Unix)
```

**Kimenet példa (nginx):**
```
Server: nginx/1.17.3
```

**Kimenet példa (rejtett):**
```
Server: Website.com
```

**Mit csinál:**
- `-I` = csak a fejléceket kéri le (HEAD request)
- `-s` = silent mode (nincs progress bar)
- `grep -i server` = csak a Server sort szűri ki

---

### Tool #2: **telnet**

#### Parancsok:
```bash
telnet pelda.hu 80
GET / HTTP/1.1
Host: pelda.hu

telnet pelda.hu 443
# HTTPS-hez nem ideális, használj inkább openssl-t
```

**Mit csinál:** Manuálisan HTTP kérést küldesz és olvasod a választ.

**Figyelem:** Enter-t kétszer kell ütni a GET után!

---

### Tool #3: **openssl** (HTTPS-hez)

#### Parancsok:
```bash
openssl s_client -connect pelda.hu:443
GET / HTTP/1.1
Host: pelda.hu

echo -e "GET / HTTP/1.1\nHost: pelda.hu\n\n" | openssl s_client -connect pelda.hu:443 -quiet
```

**Mit csinál:** TLS/SSL kapcsolaton keresztül HTTP kérést küld.

**Tipp:** A `-quiet` flag eltünteti a certificate infókat.

---

### Tool #4: **nc (netcat)**

#### Parancsok:
```bash
nc pelda.hu 80
GET / HTTP/1.1
Host: pelda.hu

echo -e "GET / HTTP/1.1\nHost: pelda.hu\n\n" | nc pelda.hu 80
```

**Mit csinál:** Egyszerű TCP kapcsolatot nyit és HTTP kérést küld.

---

## Header Field Order Fingerprinting

Ha a `Server:` header rejtett vagy módosított, a **header mezők sorrendje** is árulkodó lehet!

### Apache tipikus sorrend:
```
Date
Server
Last-Modified
ETag
Accept-Ranges
Content-Length
Connection
Content-Type
```

### nginx tipikus sorrend:
```
Server
Date
Content-Type
```

**Parancs a sorrend vizsgálatához:**
```bash
curl -I http://pelda.hu | cat -n
```

---

## 2. Malformed Requests (Hibás Kérések)

### Concept:
**Szándékosan rossz HTTP kérést** küldesz, és a **hibaüzenet alapján** azonosítod a szervert. Minden szerver másképp kezeli a hibákat!

### Tool #1: **curl hibás kéréssel**

#### Parancsok:
```bash
curl -X GET http://pelda.hu -H "Host: pelda.hu" --http1.1 --request-target "/ SANTA_CLAUS/1.1"

curl http://pelda.hu --head --http1.0

echo -e "GET / SANTA CLAUS/1.1\n\n" | nc pelda.hu 80
```

**Mit csinál:** Nem létező HTTP verziót kér (SANTA CLAUS/1.1), és figyeli a hibaüzenetet.

---

### Apache válasz hibás kérésre:
```html
HTTP/1.1 400 Bad Request
Server: Apache/2.4.41 (Unix)

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>400 Bad Request</title>
</head><body>
<h1>Bad Request</h1>
<p>Your browser sent a request that this server could not understand.</p>
</body></html>
```

---

### nginx válasz hibás kérésre:
```html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.17.3</center>
</body>
</html>
```

**Figyelj:** nginx 404-et ad 400 helyett!

---

### lighttpd válasz:
```html
HTTP/1.0 400 Bad Request
Server: lighttpd/1.4.54

<?xml version="1.0" encoding="iso-8859-1"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"...
```

---

### Tool #2: **Default error page requestek**

#### Parancsok:
```bash
curl http://pelda.hu/nonexistent_page_12345
curl http://pelda.hu/test.php
curl http://pelda.hu/../../../etc/passwd
```

**Mit csinál:** Nem létező oldalakat kér le, hogy lásd az alapértelmezett hibaoldalakat.

**Hasznos resource:** https://0xdf.gitlab.io/cheatsheets/404 - default error page fingerprinting

---

## 3. Automatizált Toolok

### Tool #1: **Nmap**

#### Parancsok:
```bash
nmap -sV -p 80,443 pelda.hu

nmap -sV --script=http-server-header pelda.hu

nmap -p 80 --script http-enum,http-headers pelda.hu
```

**Mit csinál:**
- `-sV` = Service version detection (megállapítja a szolgáltatás verzióját)
- `--script=http-server-header` = Specifikus script a szerver header kiolvasására
- `http-enum` = Felsorol web tartalmat
- `http-headers` = HTTP headereket gyűjt

**Kimenet példa:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Unix))
```

---

### Tool #2: **Nikto**

#### Parancsok:
```bash
nikto -h http://pelda.hu

nikto -h https://pelda.hu -ssl

nikto -h pelda.hu -p 80,443
```

**Mit csinál:**
- `-h` = target host
- `-ssl` = HTTPS scan
- `-p` = portok megadása
- Automatikusan fingerprinting + vulnerability scanning

**Kimenet részlet:**
```
+ Server: Apache/2.4.41 (Unix)
+ Retrieved x-powered-by header: PHP/7.4.3
```

---

### Tool #3: **WhatWeb**

#### Parancsok:
```bash
whatweb http://pelda.hu

whatweb -v http://pelda.hu

whatweb --aggression 3 http://pelda.hu
```

**Mit csinál:**
- Default: Passzív felderítés
- `-v` = Verbose output
- `--aggression 3` = Aktívabb scan (1-4 skála)

**Kimenet példa:**
```
http://pelda.hu [200 OK] Apache[2.4.41], Country[HUNGARY][HU], HTML5, HTTPServer[Apache/2.4.41 (Unix)]
```

---

### Tool #4: **httprint**

#### Parancsok:
```bash
httprint -h pelda.hu -s signatures.txt

httprint -h pelda.hu -P0

httprint -h lista.txt
```

**Mit csinál:**
- `-s` = signature file megadása
- `-P0` = No port scan (csak a megadott porton)
- `-h` = host vagy fájl hostokkal

---

### Tool #5: **Netcraft** (Online)

**URL:** https://toolbar.netcraft.com/site_report

**Használat:** Beírod a domain nevet a webes felületen.

**Mit ad vissza:**
- Szerver típus és verzió
- OS
- Hoszting provider
- IP address history
- SSL/TLS információk

---

### Tool #6: **Wappalyzer** (Browser Extension)

**Telepítés:** Chrome/Firefox kiegészítő

**Mit csinál:** Automatikusan detektálja a technológiákat a böngészőben:
- Webszerver
- CMS (WordPress, Joomla)
- JavaScript könyvtárak
- Analytics toolok

---

## Egyéb Fingerprinting Technikák

### 1. HTTP Method Testing
```bash
curl -X OPTIONS http://pelda.hu -v
curl -X TRACE http://pelda.hu -v
```

**Különböző szerverek különböző metódusokat támogatnak.**

---

### 2. ETag Analízis
```bash
curl -I http://pelda.hu | grep -i etag
```

**Apache ETag példa:**
```
ETag: "75-591d1d21b6167"
```

**nginx ETag példa:**
```
ETag: "5d71489a-75"
```

---

### 3. Cookie Analízis
```bash
curl -I http://pelda.hu | grep -i set-cookie
```

**Például:** 
- `PHPSESSID` → PHP backend
- `JSESSIONID` → Java (Tomcat)
- `ASP.NET_SessionId` → ASP.NET

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **Header obfuscation (elrejtés)**
   - Apache: `mod_headers` modul használata
   - nginx: `server_tokens off;` direktíva

2. **Reverse Proxy használata**
   - Extra biztonsági réteg
   - Elrejti a valódi webszervert

3. **Szoftver naprakészen tartása**
   - Mindig a legfrissebb verzió
   - Security patch-ek telepítése

4. **Custom error pages**
   - Ne mutass default hibaoldalakat
   - Egyedi 404, 500 stb. oldalak

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Alapvető banner grab | `curl -I http://pelda.hu` |
| Csak Server header | `curl -sI http://pelda.hu \| grep -i server` |
| HTTPS banner grab | `openssl s_client -connect pelda.hu:443` |
| Nmap version scan | `nmap -sV -p 80,443 pelda.hu` |
| Nikto scan | `nikto -h http://pelda.hu` |
| WhatWeb scan | `whatweb http://pelda.hu` |
| Malformed request | `echo -e "GET / SANTA/1.1\n\n" \| nc pelda.hu 80` |
| HTTP methods test | `curl -X OPTIONS http://pelda.hu -v` |
| 404 error page | `curl http://pelda.hu/xyz123` |

---

## Fontos Különbségek a Szerverek Között

| Szerver | Server Header | Tipikus Viselkedés |
|---------|---------------|-------------------|
| **Apache** | `Apache/2.4.41 (Unix)` | 400 Bad Request, részletes HTML error |
| **nginx** | `nginx/1.17.3` | Gyakran 404 ad 400 helyett, egyszerű HTML |
| **lighttpd** | `lighttpd/1.4.54` | XML formátumú error page |
| **IIS** | `Microsoft-IIS/10.0` | ASP.NET specifikus headerek |
| **Tomcat** | `Apache-Coyote/1.1` | JSESSIONID cookie |

---

## Összefoglalás - Workflow

### 1. lépés: Passzív banner grab
```bash
curl -I http://pelda.hu
```

### 2. lépés: Ha rejtett, próbálj malformed requestet
```bash
echo -e "GET / SANTA/1.1\n\n" | nc pelda.hu 80
```

### 3. lépés: Automatizált tool
```bash
nmap -sV -p 80,443 pelda.hu
nikto -h http://pelda.hu
```

### 4. lépés: Header order és ETag elemzés
```bash
curl -I http://pelda.hu | cat -n
```

---

## Fontos Megjegyzések

✅ **Ez infrastruktúra felderítés** - a szerver szoftverét vizsgálod, nem a weboldalt  
✅ **Kombinálj technikákat** - ha az egyik nem működik, próbáld a másikat  
✅ **Passzív és aktív** - banner grab passzív, malformed request aktívabb  
❌ **NE keverd össze** alkalmazás szintű tesztekkel - ez alacsonyabb réteg!  
⚠️ **Óvatosan** - túl agresszív scanelés WAF/IDS triggerelheti!

---

**Összefoglalva:** Ez a fejezet a **webszerver azonosításáról** szól. Célja megállapítani, hogy Apache, nginx, IIS vagy más szerver fut-e, és milyen verziószámmal. Ezt banner grabbinggel, hibás kérésekkel vagy automatizált toolokkal éred el. Ez fontos, mert a régi verziók ismert sebezhetőségekkel rendelkeznek!
