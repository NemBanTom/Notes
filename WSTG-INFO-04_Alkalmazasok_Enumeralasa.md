# Webszerveren Lévő Alkalmazások Enumerálása

|ID          |
|------------|
|WSTG-INFO-04|

## Összefoglaló

Egy webszerveren gyakran több alkalmazás is fut egyidejűleg, különböző URL-eken, portokon vagy virtuális hostokon. A virtual hosting és a cloud szolgáltatások elterjedésével egy IP cím több tíz vagy akár több száz domain-t is kiszolgálhat. A teljes támadási felület feltérképezéséhez minden elérhető alkalmazást azonosítani kell.

**Három fő terület:**
1. **Nem standard URL-ek** - Rejtett alkalmazások pl. `/admin`, `/api/v2`
2. **Nem standard portok** - HTTP(S) szolgáltatások más portokon (8080, 8443, 3000, stb.)
3. **Virtuális hostok** - Egy IP címen több domain

**Cél:** A webszerveren elérhető összes alkalmazás azonosítása a teljes scope meghatározásához.

## Tesztelési Módszertan

### 1. Nem Standard URL-ek Felderítése

**Directory fuzzing - gobuster**
```bash
# Alapvető használat
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt

# Több extension
gobuster dir -u https://target.com -w wordlist.txt -x php,html,txt,js

# Status kódok szűrése
gobuster dir -u https://target.com -w wordlist.txt -b 404,403

# Részletes kimenet
gobuster dir -u https://target.com -w wordlist.txt -v -o results.txt

# User-Agent és cookie
gobuster dir -u https://target.com -w wordlist.txt -a "Mozilla/5.0" -c "session=abc123"
```

**ffuf - gyorsabb alternatíva**
```bash
# Alapvető directory fuzzing
ffuf -u https://target.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt

# Rekurzív scan
ffuf -u https://target.com/FUZZ -w wordlist.txt -recursion -recursion-depth 2

# Extension fuzzing
ffuf -u https://target.com/FUZZ -w wordlist.txt -e .php,.html,.txt,.bak

# Status kód szűrés és match
ffuf -u https://target.com/FUZZ -w wordlist.txt -mc 200,301,302 -fc 404

# Rate limiting
ffuf -u https://target.com/FUZZ -w wordlist.txt -rate 100

# Kimenet JSON-ban
ffuf -u https://target.com/FUZZ -w wordlist.txt -o results.json -of json
```

**dirsearch**
```bash
# Alapvető scan
dirsearch -u https://target.com

# Több extension
dirsearch -u https://target.com -e php,html,js,txt,bak

# Wordlist megadása
dirsearch -u https://target.com -w /path/to/wordlist.txt

# Rekurzív
dirsearch -u https://target.com -r

# Threads növelése
dirsearch -u https://target.com -t 50

# Exclude status codes
dirsearch -u https://target.com -x 403,404,500
```

**feroxbuster - rekurzív fuzzer**
```bash
# Alapvető használat
feroxbuster -u https://target.com

# Wordlist és extensions
feroxbuster -u https://target.com -w wordlist.txt -x php,txt

# Mélység limit
feroxbuster -u https://target.com --depth 3

# Threads és rate limit
feroxbuster -u https://target.com -t 50 --rate-limit 10

# Filter status codes
feroxbuster -u https://target.com -C 404,403
```

**Gyakori rejtett URL-ek wordlistje:**
```bash
# SecLists használata
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/big.txt
/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt

# Gyakori admin és API útvonalak
cat > common_hidden.txt << EOF
/admin
/administrator
/api
/api/v1
/api/v2
/backup
/dev
/test
/staging
/old
/new
/temp
/tmp
/webmail
/mail
/portal
/dashboard
/console
/cpanel
/phpmyadmin
/adminer
EOF
```

**Google Dorking rejtett URL-ekre:**
```text
site:target.com inurl:admin
site:target.com inurl:login
site:target.com inurl:dashboard
site:target.com inurl:api
site:target.com inurl:dev
site:target.com inurl:test
site:target.com inurl:backup
```

### 2. Nem Standard Portok Vizsgálata

**Nmap - teljes port scan**
```bash
# Minden TCP port + service detection
nmap -p- -sV target.com

# Gyorsabb scan (top 1000 port)
nmap -sV target.com

# Agresszív scan + script
nmap -A -p- target.com

# HTTP szolgáltatások keresése minden porton
nmap -p- --open -sV target.com | grep -i http

# Gyakori web portok
nmap -p 80,443,8080,8443,8000,8888,3000,5000 -sV target.com

# UDP portok (lassabb)
nmap -sU --top-ports 100 target.com
```

**Masscan - nagyon gyors port scanner**
```bash
# Teljes port range gyorsan
masscan -p1-65535 target.com --rate=1000

# HTTP szolgáltatások
masscan -p80,443,8080,8443,8000,8888 target.com --rate=1000

# Banner grabbing
masscan -p1-65535 target.com --banners --rate=1000

# Output file
masscan -p1-65535 target.com --rate=1000 -oL output.txt
```

**Gyakori nem standard web portok:**
```
80, 443      # Standard HTTP/HTTPS
8080, 8443   # Alternatív HTTP/HTTPS
8000, 8888   # Development szerverek
3000         # Node.js (Express default)
4200         # Angular dev server
5000         # Flask default
8081, 8082   # Proxy, admin portok
9090, 9091   # Monitoring (Prometheus, stb.)
10000        # Webmin
2222         # Alternatív SSH (de HTTP is lehet)
```

**RustScan - gyors port scanner + nmap integráció**
```bash
# Gyors scan majd nmap
rustscan -a target.com -- -sV

# Több target
rustscan -a target1.com,target2.com

# Port range
rustscan -a target.com --range 1-10000
```

**Specifikus port ellenőrzés curl-el:**
```bash
# HTTP szolgáltatás tesztelése
curl -I http://target.com:8080
curl -I http://target.com:3000

# HTTPS
curl -Ik https://target.com:8443

# Banner grabbing netcat-tel
nc -v target.com 8080
GET / HTTP/1.1
Host: target.com
```

### 3. Virtuális Hostok Felderítése

#### DNS Zone Transfer

```bash
# Nameserver meghatározás
host -t ns target.com
dig ns target.com

# Zone transfer kísérlet
host -l target.com ns1.target.com
dig axfr @ns1.target.com target.com

# Minden nameserverre próbálkozás
for ns in $(host -t ns target.com | awk '{print $4}'); do
    echo "Trying $ns"
    dig axfr @$ns target.com
done
```

#### Subdomain Enumeration

**Amass**
```bash
# Passzív enumeration
amass enum -passive -d target.com

# Aktív enumeration
amass enum -d target.com

# Brute force is
amass enum -brute -d target.com

# Output fájlba
amass enum -d target.com -o subdomains.txt

# Több domain
amass enum -d target1.com,target2.com
```

**Subfinder**
```bash
# Alapvető használat
subfinder -d target.com

# Több domain
subfinder -dL domains.txt

# Output
subfinder -d target.com -o subdomains.txt

# Silent mode (csak eredmények)
subfinder -d target.com -silent

# Csak aktív subdomainok
subfinder -d target.com -nW
```

**assetfinder**
```bash
# Subdomain keresés
assetfinder target.com

# Csak subdomainok
assetfinder --subs-only target.com

# Output pipe
assetfinder target.com | tee subdomains.txt
```

**knockpy**
```bash
# Wordlist alapú brute force
knockpy target.com

# Custom wordlist
knockpy -w wordlist.txt target.com
```

**Sublist3r**
```bash
# Alapvető használat
sublist3r -d target.com

# Brute force is
sublist3r -d target.com -b

# Threads növelése
sublist3r -d target.com -t 50

# Output
sublist3r -d target.com -o subdomains.txt
```

#### Certificate Transparency Logs

**crt.sh**
```bash
# cURL használatával
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sort -u

# Wildcard keresés
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u | grep target.com
```

**ctfr**
```bash
# Certificate transparency keresés
ctfr -d target.com

# Output
ctfr -d target.com -o subdomains.txt
```

#### Reverse DNS Lookup

**Nmap**
```bash
# Reverse DNS egy IP-re
nmap -sL target.com

# IP range reverse DNS
nmap -sL 192.168.1.0/24
```

**DNSRecon**
```bash
# Reverse lookup
dnsrecon -r 192.168.1.0/24

# Standard enumeration
dnsrecon -d target.com

# Brute force
dnsrecon -d target.com -D wordlist.txt -t brt
```

#### Virtual Host Brute Forcing

**ffuf vhost discovery**
```bash
# Virtual host fuzzing
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u https://target.com -H "Host: FUZZ.target.com" -fs 1234

# IP címmel
ffuf -w wordlist.txt -u http://192.168.1.100 -H "Host: FUZZ.target.com" -mc 200

# Filter by size
ffuf -w wordlist.txt -u http://target.com -H "Host: FUZZ.target.com" -fs 0
```

**gobuster vhost mode**
```bash
# Vhost discovery
gobuster vhost -u https://target.com -w wordlist.txt

# Append domain
gobuster vhost -u https://target.com -w wordlist.txt --append-domain

# Követési opciók
gobuster vhost -u https://target.com -w wordlist.txt -k -r
```

**VHostScan**
```bash
# Virtual host scan
VHostScan -t target.com

# Wordlist megadása
VHostScan -t target.com -w wordlist.txt

# SSL support
VHostScan -t target.com --ssl
```

### 4. Web-based Szolgáltatások

**Reverse IP Lookup - online**
```bash
# curl használatával API-k
curl -s "https://api.hackertarget.com/reverseiplookup/?q=192.168.1.100"

# DNSlytics
curl -s "https://dnslytics.com/reverse-ip/192.168.1.100"
```

**Shodan**
```bash
# Hostname alapján
shodan host target.com

# IP alapján
shodan host 192.168.1.100

# Domain keresés
shodan search hostname:target.com

# Port specifikus
shodan search hostname:target.com port:8080
```

**SecurityTrails (API)**
```bash
# Subdomain lista (API key szükséges)
curl -s "https://api.securitytrails.com/v1/domain/target.com/subdomains" \
  -H "APIKEY: YOUR_API_KEY" | jq .
```

### 5. SSL Certificate Információk

**OpenSSL**
```bash
# Certificate lekérés és Subject Alternative Names
openssl s_client -connect target.com:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep -E 'DNS:|Subject:'

# Csak CN
openssl s_client -connect target.com:443 </dev/null 2>/dev/null | openssl x509 -noout -subject

# Certificate chain
openssl s_client -connect target.com:443 -showcerts </dev/null
```

**sslscan**
```bash
# SSL/TLS scan + certificate info
sslscan target.com

# Csak certificate
sslscan --show-certificate target.com
```

### 6. Web Application Fingerprinting

**WhatWeb**
```bash
# Alapvető használat
whatweb target.com

# Agresszív mód
whatweb -a 3 target.com

# URL lista
whatweb -i urls.txt

# Verbose
whatweb -v target.com
```

**Wappalyzer CLI**
```bash
# Technológia detektálás
wappalyzer https://target.com
```

**Nikto**
```bash
# Alapvető web scan
nikto -h target.com

# Port megadása
nikto -h target.com -p 8080

# SSL
nikto -h https://target.com -ssl
```

### 7. Content Discovery Combination

**Komplex workflow bash scriptben:**
```bash
#!/bin/bash
TARGET=$1

echo "=== Subdomain Enumeration ==="
subfinder -d $TARGET -silent | tee subdomains.txt
amass enum -passive -d $TARGET | tee -a subdomains.txt
cat subdomains.txt | sort -u > subdomains_unique.txt

echo -e "\n=== Port Scanning ==="
nmap -iL subdomains_unique.txt -p- --open -oN nmap_results.txt

echo -e "\n=== Virtual Host Discovery ==="
while read subdomain; do
    ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
         -u http://$subdomain -H "Host: FUZZ.$subdomain" -fs 0 -mc 200
done < subdomains_unique.txt

echo -e "\n=== Directory Fuzzing ==="
while read subdomain; do
    echo "Fuzzing $subdomain"
    gobuster dir -u https://$subdomain -w /usr/share/wordlists/dirb/common.txt -q
done < subdomains_unique.txt
```

### 8. Automatizált Reconnaissance Frameworkok

**recon-ng**
```bash
recon-ng
[recon-ng][default] > workspaces create target_recon
[recon-ng][target_recon] > db insert domains target.com
[recon-ng][target_recon] > modules search
[recon-ng][target_recon] > modules load recon/domains-hosts/brute_hosts
[recon-ng][target_recon] > run
```

**Spiderfoot**
```bash
# Web interface indítás
python3 sf.py -l 127.0.0.1:5001

# CLI mód
python3 sfcli.py -s target.com -t DOMAIN_NAME
```

**OWASP Amass teljes workflow:**
```bash
# Intelligence gathering
amass intel -d target.com -whois

# Asset discovery
amass enum -d target.com -config config.ini

# Visualizáció
amass viz -d3 -d target.com
```

### 9. Screenshot és Összegzés

**Aquatone - screenshot készítés**
```bash
# Subdomainokról screenshot
cat subdomains.txt | aquatone

# Port scan eredményekből
cat nmap_results.xml | aquatone -nmap
```

**httpx - HTTP probe**
```bash
# Élő hostok szűrése
cat subdomains.txt | httpx -silent

# Status kód, title, tech
cat subdomains.txt | httpx -title -tech-detect -status-code

# Screenshot
cat subdomains.txt | httpx -screenshot
```

**EyeWitness**
```bash
# URL listából screenshot és report
eyewitness -f urls.txt --web

# Nmap XML-ből
eyewitness -x nmap.xml --web
```

## Lehetséges Hatások és Ellenintézkedések

**Hatás:** Rejtett alkalmazások, admin felületek, API endpoint-ok, staging/dev környezetek felderítése. Nem standard portokon gyakran kevésbé védett szolgáltatások futnak. Virtual host enumeration teljes infrastruktúra feltárása. Több támadási vektor azonosítása.

**Ellenintézkedések:**
- Nem használt portok lezárása tűzfalon
- Development/staging környezetek VPN mögé helyezése vagy IP whitelist
- Virtual hostok megfelelő konfigurálása, wildcard DNS kerülése ahol nem szükséges
- Nem használt alkalmazások eltávolítása vagy letiltása
- Web Application Firewall (WAF) használata rate limiting-gel
- Directory listing kikapcsolása minden könyvtárban
- Certificate transparency monitoring - figyeld milyen tanúsítványok kerülnek kiállításra
- Rejtett URL-ek helyett authentication használata
- Intrusion Detection System (IDS) a port scanning és fuzzing detektálására
