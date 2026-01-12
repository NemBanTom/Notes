# Webszerver Fingerprinting

|ID          |
|------------|
|WSTG-INFO-02|

## Összefoglaló

A webszerver fingerprinting célja a célszerver típusának és verziójának azonosítása. A pontos verzióazonosítás lehetővé teszi az ismert, verzió-specifikus sebezhetőségek felderítését. A régebbi, nem frissített szerverek különösen sebezhetőek a publikusan ismert exploitokra.

**Cél:** A webszerver típusának és verziójának meghatározása a további sebezhetőség-elemzéshez.

## Tesztelési Módszertan

### 1. Banner Grabbing - Manuális

**Telnet (HTTP)**
```bash
telnet target.com 80
GET / HTTP/1.1
Host: target.com

# vagy
echo -e "HEAD / HTTP/1.1\r\nHost: target.com\r\n\r\n" | nc target.com 80
```

**OpenSSL (HTTPS)**
```bash
openssl s_client -connect target.com:443
HEAD / HTTP/1.1
Host: target.com

# vagy egysorosban
echo -e "HEAD / HTTP/1.1\r\nHost: target.com\r\n\r\n" | openssl s_client -connect target.com:443 -quiet
```

**cURL**
```bash
curl -I target.com
curl -I -k https://target.com

# Részletes információk
curl -v target.com 2>&1 | grep -i server
```

**HTTPie**
```bash
http HEAD target.com
https HEAD target.com
```

**Wget**
```bash
wget --server-response --spider target.com
```

### 2. HTTP Header Elemzés

**Gyakori Server Header-ök:**
```http
Server: Apache/2.4.41 (Unix)
Server: nginx/1.17.3
Server: Microsoft-IIS/10.0
Server: lighttpd/1.4.54
Server: LiteSpeed
Server: Cloudflare
```

**Rejtett fingerprint-ek a header sorrendben:**

Apache jellemző sorrend:
- Date, Server, Last-Modified, ETag, Accept-Ranges, Content-Length

nginx jellemző sorrend:
- Server, Date, Content-Type, Content-Length

**Header elemző script:**
```python
import requests

def analyze_headers(url):
    response = requests.get(url)
    print("=== HTTP Headers ===")
    for header, value in response.headers.items():
        print(f"{header}: {value}")
    
    # Egyedi header-ök keresése
    interesting = ['Server', 'X-Powered-By', 'X-AspNet-Version', 'X-AspNetMvc-Version']
    print("\n=== Érdekes Headers ===")
    for h in interesting:
        if h in response.headers:
            print(f"{h}: {response.headers[h]}")

analyze_headers("http://target.com")
```

### 3. Malformált Kérések

**Érvénytelen HTTP verzió**
```bash
echo -e "GET / INVALID/1.1\r\nHost: target.com\r\n\r\n" | nc target.com 80
```

**Érvénytelen metódus**
```bash
echo -e "INVALID / HTTP/1.1\r\nHost: target.com\r\n\r\n" | nc target.com 80
```

**Hiányzó Host header**
```bash
echo -e "GET / HTTP/1.1\r\n\r\n" | nc target.com 80
```

**HTTP/0.9 próba**
```bash
echo -e "GET /\r\n\r\n" | nc target.com 80
```

### 4. Hibaoldalak Elemzése

**404 oldalak vizsgálata**
```bash
curl -i http://target.com/nonexistent$(date +%s)
curl -i http://target.com/$(uuidgen)
```

**Különböző HTTP hibakódok provokálása**
```bash
# 400 Bad Request
curl -H "Host: " target.com

# 405 Method Not Allowed
curl -X INVALID target.com

# 414 Request-URI Too Large
curl "target.com/$(python3 -c 'print("A"*10000)')"

# 500 Internal Server Error (óvatosan!)
curl "target.com/test.php?param=$(python3 -c 'print("x"*10000)')"
```

**Alapértelmezett hibaoldalak gyűjteménye:**
- https://0xdf.gitlab.io/cheatsheets/404
- https://github.com/jothatron/Default-Error-Pages

### 5. Automatizált Eszközök

**Nmap**
```bash
# Alapvető scan
nmap -sV -p 80,443 target.com

# Részletes verzió detekció
nmap -sV --version-intensity 9 -p 80,443 target.com

# HTTP-specifikus NSE scriptek
nmap -p 80,443 --script http-server-header target.com
nmap -p 80,443 --script http-headers target.com
nmap -p 80 --script http-methods target.com

# Banner grabbing
nmap -p 80,443 --script banner target.com
```

**Nikto**
```bash
# Alapvető scan
nikto -h target.com

# SSL/TLS támogatással
nikto -h https://target.com

# Port megadással
nikto -h target.com -p 8080

# Teljes scan minden plugin-nel
nikto -h target.com -Plugins '@@ALL'

# Kimenet fájlba
nikto -h target.com -o results.html -Format htm
```

**WhatWeb**
```bash
# Alapvető használat
whatweb target.com

# Agresszív mód (több kérés)
whatweb -a 3 target.com

# Verbose kimenet
whatweb -v target.com

# URL lista
whatweb -i urls.txt

# JSON kimenet
whatweb --log-json=output.json target.com
```

**Wappalyzer CLI**
```bash
npm install -g wappalyzer

wappalyzer analyze target.com
```

**httprint**
```bash
# Signature alapú fingerprinting
httprint -h target.com -s signatures.txt
```

**wafw00f - WAF detekció**
```bash
# WAF azonosítás
wafw00f target.com

# Minden támogatott WAF ellen tesztel
wafw00f -a target.com
```

**curl + jq kombináció (API-k vizsgálatához)**
```bash
curl -s -I target.com | grep -i "server\|x-powered"
curl -s -D - target.com -o /dev/null | grep -iE "server|powered|version"
```

### 6. SSL/TLS Certificate Információk

**OpenSSL certificate lekérés**
```bash
# Tanúsítvány részletek
openssl s_client -connect target.com:443 < /dev/null 2>/dev/null | openssl x509 -text

# Csak issuer és subject
echo | openssl s_client -connect target.com:443 2>/dev/null | openssl x509 -noout -issuer -subject

# Támogatott cipher suite-ok
nmap --script ssl-enum-ciphers -p 443 target.com
```

**testssl.sh**
```bash
git clone https://github.com/drwetter/testssl.sh.git
cd testssl.sh
./testssl.sh target.com
```

### 7. Technológia Stack Felderítés

**Wappalyzer (böngészős vagy CLI)**
```bash
# CLI verzió
npm install -g wappalyzer
wappalyzer http://target.com
```

**BuiltWith (online)**
- https://builtwith.com/target.com

**Netcraft Site Report**
- https://sitereport.netcraft.com/?url=target.com

**retire.js - JavaScript könyvtárak verzió check**
```bash
retire --js --jspath http://target.com
```

### 8. Speciális Fingerprinting Technikák

**HTTP/2 támogatás ellenőrzése**
```bash
curl -I --http2 https://target.com
nghttp -nv https://target.com
```

**WebSocket vizsgálat**
```bash
# wscat használatával
wscat -c ws://target.com/socket
```

**Időzítés alapú fingerprinting**
```python
import requests
import time

def timing_fingerprint(url):
    times = []
    for i in range(10):
        start = time.time()
        requests.get(url)
        elapsed = time.time() - start
        times.append(elapsed)
    
    avg_time = sum(times) / len(times)
    print(f"Átlagos válaszidő: {avg_time:.3f}s")
    # A válaszidők mintázata utalhat a szerver típusára

timing_fingerprint("http://target.com")
```

**Cookie elemzés**
```bash
# Cookie-k részletes vizsgálata
curl -I target.com 2>&1 | grep -i "set-cookie"

# Session cookie nevek (gyakori minták):
# PHPSESSID - PHP
# JSESSIONID - Java/Tomcat
# ASP.NET_SessionId - ASP.NET
# connect.sid - Express.js (Node)
```

### 9. Specifikus Webszerverek Jellemzői

**Apache jellemzők:**
- ETag formátum: "inode-size-timestamp"
- Alapértelmezett 404: Apache branding
- mod_status: /server-status endpoint

**nginx jellemzők:**
- ETag formátum: egyszerűbb hash
- Karakterisztikus hibaoldalak
- Alapértelmezett 404: "nginx" branding

**IIS jellemzők:**
- X-Powered-By: ASP.NET
- X-AspNet-Version header
- Specifikus hibakódok és oldalak

**LiteSpeed jellemzők:**
- Alt-Svc header jelenléte
- X-LiteSpeed-Cache header

### 10. Összesített Script

**all_fingerprint.sh**
```bash
#!/bin/bash

TARGET=$1

echo "=== Banner Grabbing ==="
curl -I -s $TARGET | grep -i server

echo -e "\n=== Nmap Service Detection ==="
nmap -sV -p 80,443 $TARGET | grep -i "service"

echo -e "\n=== WhatWeb ==="
whatweb $TARGET

echo -e "\n=== Certificate Info ==="
echo | openssl s_client -connect ${TARGET#http://}:443 2>/dev/null | openssl x509 -noout -subject -issuer 2>/dev/null

echo -e "\n=== WAF Detection ==="
wafw00f $TARGET 2>/dev/null

echo -e "\n=== HTTP Headers ==="
curl -s -I $TARGET

echo -e "\n=== Malformed Request ==="
echo -e "GET / INVALID/1.1\r\nHost: $TARGET\r\n\r\n" | nc ${TARGET#http://} 80
```

## Lehetséges Hatások és Ellenintézkedések

**Hatás:** A szerver típusának és verziójának ismerete lehetővé teszi a támadó számára a verzió-specifikus exploitok keresését. A CVE adatbázisokban közvetlenül kereshetők a sebezhetőségek. Elavult szerverek esetén gyakran vannak publikus exploitok.

**Ellenintézkedések:**
- Server header elrejtése vagy módosítása (`ServerTokens Prod` Apache-nál, `server_tokens off` nginx-nél)
- Alapértelmezett hibaoldalak testreszabása
- Reverse proxy használata (Cloudflare, nginx) köztes rétegként
- Rendszeres patch management és verziófrissítések
- WAF (Web Application Firewall) telepítése
- Security through obscurity nem elég: a tényleges frissítés a prioritás
