# WSTG-INFO-04: Webalkalmazások Felsorolása Webszerveren

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **egyetlen IP címen vagy domain néven gyakran TÖBB webalkalmazás is fut**, és ezeket mind fel kell deríteni! A probléma, hogy ezek az alkalmazások **rejtve maradhatnak**, ha nem tudod a pontos URL-t, portot, vagy virtuális host nevet.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM az alkalmazás sebezhetőségeinek tesztelése! Ez az **alkalmazások FELDERÍTÉSE** - vagyis meg kell találni az összes alkalmazást, ami az adott infrastruktúrán fut, MIELŐTT tesztelnéd őket.

---

## Mi a cél?

**Felsorolni az összes webalkalmazást**, amely egy adott webszerveren vagy IP címen fut, hogy:
- Teljes scope-ot kapj a pentesthez
- Ne maradjon rejtett alkalmazás tesztetlenül
- Minden attack surface-t feltérképezz

### Miért fontos?

- Egy IP cím mögött **10-20 különböző alkalmazás** is lehet
- Ezek **különböző biztonsági szinttel** rendelkezhetnek
- A rejtett admin interfészek gyakran **nem frissítettek**
- Virtual hosting miatt egy IP → sok DNS név

---

## 3 Fő Probléma, Amit Meg Kell Oldani

### 1. **Nem-standard URL-ek** (Different Base URL)
Alkalmazások nem a gyökérben vannak:
```
https://www.example.com/app1
https://www.example.com/app2
https://www.example.com/webmail
https://www.example.com/admin-secret-panel
```

### 2. **Nem-standard portok** (Non-standard Ports)
Alkalmazások nem 80/443 porton futnak:
```
https://www.example.com:8080/
https://www.example.com:8443/
https://www.example.com:3000/
```

### 3. **Virtuális Hostok** (Virtual Hosts)
Egy IP cím → több DNS név:
```
192.168.1.100 →
    www.example.com
    webmail.example.com
    helpdesk.example.com
    admin.example.com
```

---

## 1. Probléma: Nem-standard URL-ek Felderítése

### Concept:
Az alkalmazások rejtett alkönyvtárakban vannak, amiket **nem linkelt sehol** a főoldal.

### Tool #1: **Directory Brute-forcing - gobuster**

#### Parancsok:
```bash
gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirb/common.txt

gobuster dir -u https://pelda.hu -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

gobuster dir -u https://pelda.hu -w wordlist.txt -x php,html,txt
```

**Mit csinál:**
- `dir` = directory brute-force mode
- `-w` = wordlist megadása
- `-x` = file extension-ok hozzáadása
- Végigpróbálja a wordlist-et és megnézi, mi létezik

**Kimenet példa:**
```
/admin                (Status: 200)
/webmail              (Status: 301)
/backup               (Status: 403)
/test                 (Status: 200)
```

---

### Tool #2: **ffuf (Faster fuzzer)**

#### Parancsok:
```bash
ffuf -u https://pelda.hu/FUZZ -w /usr/share/wordlists/dirb/common.txt

ffuf -u https://pelda.hu/FUZZ -w wordlist.txt -mc 200,301,302,403

ffuf -u https://pelda.hu/FUZZ -w wordlist.txt -t 50 -fc 404
```

**Mit csinál:**
- `FUZZ` = placeholder a wordlist elemeknek
- `-mc` = match codes (csak ezeket a status code-okat mutassa)
- `-fc` = filter code (ezt NE mutassa, pl. 404)
- `-t` = threads száma (gyorsabb)

---

### Tool #3: **dirb**

#### Parancsok:
```bash
dirb https://pelda.hu

dirb https://pelda.hu /usr/share/wordlists/dirb/big.txt

dirb https://pelda.hu wordlist.txt -X .php,.html
```

**Mit csinál:**
- Alapértelmezett wordlist-tel indul
- `-X` = file extension-ök megadása

---

### Tool #4: **Google Dorking**

#### Parancsok (böngészőben):
```
site:pelda.hu inurl:admin
site:pelda.hu inurl:webmail
site:pelda.hu inurl:test
site:pelda.hu inurl:backup
site:pelda.hu inurl:dev
```

**Mit csinál:** Google már crawlolta az oldalakat, lehet rejtett URL-eket találni.

---

### Tool #5: **Burp Suite - Content Discovery**

1. Proxy → HTTP history elemzése
2. Target → Site map nézet
3. Intruder → Directory brute-force
4. Scanner → Passive crawling

---

## 2. Probléma: Nem-standard Portok Felderítése

### Concept:
Webalkalmazások **nem csak 80/443 porton** futhatnak, hanem bármilyen TCP porton (0-65535).

### Tool #1: **Nmap - Service Detection**

#### Parancsok:
```bash
nmap -sV -p- pelda.hu

nmap -Pn -sT -sV -p0-65535 pelda.hu

nmap -sV -p 80,443,8000,8080,8443,3000,5000,9000 pelda.hu
```

**Mit csinál:**
- `-sV` = Service version detection (megállapítja a szolgáltatást)
- `-p-` = All ports (0-65535)
- `-Pn` = Skip ping (assume host is up)
- `-sT` = TCP connect scan

**Kimenet példa:**
```
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 7.4
80/tcp    open  http        Apache httpd 2.4.41
443/tcp   open  ssl/http    nginx 1.18.0
8000/tcp  open  http-alt    SimpleHTTPServer
8080/tcp  open  http        Apache Tomcat 9.0
3000/tcp  open  http        Node.js Express
9000/tcp  open  http        PHP built-in server
```

---

### Tool #2: **masscan (Gyorsabb port scanner)**

#### Parancsok:
```bash
masscan -p1-65535 pelda.hu --rate=1000

masscan pelda.hu -p80,443,8080,8443,3000,5000,8000 --rate=10000

masscan 192.168.1.0/24 -p80,443,8080 --rate=1000
```

**Mit csinál:**
- Sokkal gyorsabb mint nmap
- `--rate` = packets per second
- Teljes port range gyors scan-elése

**Figyelem:** Utána nmap -sV-vel nézd meg részletesen!

---

### Tool #3: **Manual Testing - telnet/curl**

#### Parancsok:
```bash
# Tesztelés különböző portokon
curl http://pelda.hu:8080
curl http://pelda.hu:8000
curl http://pelda.hu:3000

# Telnet-tel
telnet pelda.hu 8080
GET / HTTP/1.1
Host: pelda.hu

# Openssl HTTPS portokon
openssl s_client -connect pelda.hu:8443
```

---

### Tool #4: **Nessus (Vulnerability Scanner)**

#### Használat:
1. Indítsd el Nessus-t
2. New Scan → Basic Network Scan
3. Targets: `pelda.hu`
4. Settings → Port scanning: **All TCP ports**
5. Launch scan

**Mit csinál:**
- Automatikusan detektálja a HTTP/HTTPS szolgáltatásokat
- Vulnerability scan is fut
- Részletes reportot ad

---

## 3. Probléma: Virtual Hosts Felderítése

### Concept:
**Egy IP cím több DNS névhez tartozhat**, és mindegyik DNS név külön webalkalmazást szolgálhat ki. Ezt **HTTP Host header** alapján különbözteti meg a webszerver.

---

### Technika #1: DNS Zone Transfer

#### Tool: **host / dig**

#### Parancsok:
```bash
# Name server-ek megkeresése
host -t ns pelda.hu
dig ns pelda.hu

# Zone transfer kísérlet
host -l pelda.hu ns1.pelda.hu
dig axfr @ns1.pelda.hu pelda.hu

# Több name server próbálása
for ns in $(host -t ns pelda.hu | awk '{print $4}'); do
    echo "Trying $ns"
    dig axfr @$ns pelda.hu
done
```

**Mit csinál:**
- DNS Zone Transfer-t kér
- Ha sikerül → MINDEN DNS bejegyzést megkapsz
- Manapság **ritkán működik**, de megéri próbálni

**Kimenet példa (ha sikeres):**
```
pelda.hu.           86400   IN      A       192.168.1.100
www.pelda.hu.       86400   IN      A       192.168.1.100
webmail.pelda.hu.   86400   IN      A       192.168.1.100
admin.pelda.hu.     86400   IN      A       192.168.1.100
dev.pelda.hu.       86400   IN      A       192.168.1.100
```

---

### Technika #2: DNS Inverse (PTR) Query

#### Tool: **host / dig**

#### Parancsok:
```bash
# PTR record lekérdezése
host 192.168.1.100
dig -x 192.168.1.100

# Reverse DNS lookup
nslookup 192.168.1.100
```

**Mit csinál:** 
- IP címből DNS nevet kér vissza
- Nem garantált, hogy működik
- Ha működik, lehet egy DNS nevet kapsz

**Kimenet példa:**
```
100.1.168.192.in-addr.arpa domain name pointer www.pelda.hu.
```

---

### Technika #3: Web-based DNS Search

#### Tool: **Netcraft SearchDNS**

**URL:** https://searchdns.netcraft.com

**Használat:**
1. Írj be egy domain nevet: `pelda.hu`
2. Netcraft megmutatja az összes ismert subdomain-t

**Kimenet példa:**
```
www.pelda.hu
mail.pelda.hu
webmail.pelda.hu
ftp.pelda.hu
admin.pelda.hu
```

---

### Technika #4: Reverse-IP Services

#### Online Toolok:

**1. MxToolBox Reverse IP:**
- **URL:** https://mxtoolbox.com/ReverseLookup.aspx
- Beírod az IP címet → megmutatja a DNS neveket

**2. ViewDNS.info:**
- **URL:** https://viewdns.info/reverseip/
- Reverse IP lookup

**3. DNSDumpster:**
- **URL:** https://dnsdumpster.com/
- Teljes DNS mapping

---

### Technika #5: Certificate Transparency Logs

#### Tool: **crt.sh**

**URL:** https://crt.sh

**Használat:**
```
# Böngészőben:
https://crt.sh/?q=%.pelda.hu

# Curl-lel:
curl "https://crt.sh/?q=%.pelda.hu&output=json" | jq '.[].name_value' | sort -u
```

**Mit csinál:**
- SSL Certificate Transparency log-okat keres
- **MINDEN SSL cert-ben szereplő domain-t megtalálja**
- Nagyon hatékony!

**Kimenet példa:**
```json
*.pelda.hu
www.pelda.hu
mail.pelda.hu
admin.pelda.hu
api.pelda.hu
dev.pelda.hu
```

---

### Technika #6: Subdomain Brute-forcing

#### Tool #1: **sublist3r**

#### Parancsok:
```bash
sublist3r -d pelda.hu

sublist3r -d pelda.hu -b -v

sublist3r -d pelda.hu -t 100 -v
```

**Mit csinál:**
- Automatikusan több forrásból gyűjt subdomain-okat
- `-b` = brute-force mode
- `-t` = threads
- `-v` = verbose

---

#### Tool #2: **amass**

#### Parancsok:
```bash
amass enum -d pelda.hu

amass enum -d pelda.hu -active

amass enum -d pelda.hu -brute -w wordlist.txt
```

**Mit csinál:**
- Nagyon átfogó subdomain enumeration
- Több forrást használ (DNS, crt.sh, stb.)
- `-active` = aktív DNS enumeration

---

#### Tool #3: **gobuster dns mode**

#### Parancsok:
```bash
gobuster dns -d pelda.hu -w /usr/share/wordlists/subdomains.txt

gobuster dns -d pelda.hu -w subdomains-top5000.txt -t 50

gobuster dns -d pelda.hu -w dns-wordlist.txt --wildcard
```

**Mit csinál:**
- DNS brute-forcing wordlist-tel
- `-t` = threads
- `--wildcard` = wildcard DNS detektálás

---

### Technika #7: Virtual Host Brute-forcing

#### Tool: **ffuf vhost mode**

#### Parancsok:
```bash
ffuf -u http://pelda.hu -H "Host: FUZZ.pelda.hu" -w subdomains.txt

ffuf -u http://192.168.1.100 -H "Host: FUZZ.pelda.hu" -w wordlist.txt -mc 200

ffuf -u https://pelda.hu -H "Host: FUZZ.pelda.hu" -w subdomains.txt -fs 1234
```

**Mit csinál:**
- Host header-t változtatja
- `-H "Host: FUZZ.pelda.hu"` = Host header fuzzolása
- `-fs` = filter by size (kiszűri a default response-t)

**Miért fontos:** Előfordul, hogy a subdomain DNS-ben nincs bejegyzés, de a webszerver válaszol rá!

---

### Technika #8: SSL Certificate Inspection

#### Tool: **openssl**

#### Parancsok:
```bash
# SSL cert CN és SAN mezők kiolvasása
openssl s_client -connect pelda.hu:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep DNS

openssl s_client -connect 192.168.1.100:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep -E 'DNS:|Subject:'

# Rövid verzió
echo | openssl s_client -connect pelda.hu:443 2>&1 | openssl x509 -noout -text | grep "DNS:"
```

**Mit csinál:**
- SSL certificate-ből kinyeri a DNS neveket
- CN (Common Name) és SAN (Subject Alternative Name) mezők
- Gyakran **több domain is szerepel egy cert-ben**

**Kimenet példa:**
```
Subject: CN = www.pelda.hu
DNS:www.pelda.hu, DNS:pelda.hu, DNS:mail.pelda.hu, DNS:admin.pelda.hu
```

---

### Technika #9: Google Dorking

#### Parancsok (böngészőben):
```
site:pelda.hu
site:*.pelda.hu

# Specifikus subdomainekre
inurl:admin site:pelda.hu
inurl:webmail site:pelda.hu
inurl:dev site:pelda.hu
```

**Mit csinál:** Google crawler találhatott subdomain-okat, amiket indexelt.

---

## Komplex Workflow: Teljes Alkalmazás Enumeration

### 1. lépés: Port Scanning
```bash
nmap -sV -p- pelda.hu -oA nmap_full_scan
```

### 2. lépés: Subdomain Enumeration
```bash
amass enum -d pelda.hu -o subdomains.txt
sublist3r -d pelda.hu >> subdomains.txt
curl "https://crt.sh/?q=%.pelda.hu&output=json" | jq -r '.[].name_value' | sort -u >> subdomains.txt
sort -u subdomains.txt -o subdomains_unique.txt
```

### 3. lépés: Virtual Host Fuzzing
```bash
ffuf -u http://192.168.1.100 -H "Host: FUZZ" -w subdomains_unique.txt -mc 200,301,302
```

### 4. lépés: Directory Brute-forcing (minden talált hoston)
```bash
while read host; do
    echo "Scanning $host"
    gobuster dir -u http://$host -w /usr/share/wordlists/dirb/common.txt
done < subdomains_unique.txt
```

### 5. lépés: Non-standard URL keresés
```bash
while read host; do
    ffuf -u http://$host/FUZZ -w app-paths.txt -mc 200,301,302
done < subdomains_unique.txt
```

---

## Összesítő Python Script

### Teljes Application Enumeration Script:

```python
#!/usr/bin/env python3
import subprocess
import requests

domain = "pelda.hu"
ip = "192.168.1.100"

print("[*] Port Scanning...")
subprocess.run(["nmap", "-sV", "-p-", ip, "-oA", "nmap_scan"])

print("[*] Subdomain Enumeration...")
subprocess.run(["sublist3r", "-d", domain, "-o", "subdomains.txt"])

print("[*] Certificate Transparency Search...")
r = requests.get(f"https://crt.sh/?q=%.{domain}&output=json")
if r.status_code == 200:
    certs = r.json()
    with open("crt_subdomains.txt", "w") as f:
        for cert in certs:
            f.write(cert['name_value'] + "\n")

print("[*] Virtual Host Fuzzing...")
subprocess.run(["ffuf", "-u", f"http://{ip}", "-H", f"Host: FUZZ.{domain}", 
                "-w", "subdomains.txt", "-mc", "200,301,302"])

print("[*] Directory Brute-forcing...")
with open("subdomains.txt", "r") as f:
    for sub in f:
        sub = sub.strip()
        print(f"[*] Scanning {sub}")
        subprocess.run(["gobuster", "dir", "-u", f"http://{sub}", 
                        "-w", "/usr/share/wordlists/dirb/common.txt"])

print("[*] Done!")
```

**Használat:**
```bash
chmod +x app_enum.py
./app_enum.py
```

---

## Gyakorlati Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Port scan (all ports) | `nmap -sV -p- pelda.hu` |
| Directory brute-force | `gobuster dir -u http://pelda.hu -w wordlist.txt` |
| Subdomain enum | `amass enum -d pelda.hu` |
| DNS zone transfer | `dig axfr @ns1.pelda.hu pelda.hu` |
| Reverse IP | `host 192.168.1.100` |
| SSL cert inspection | `openssl s_client -connect pelda.hu:443 \| grep DNS` |
| Virtual host fuzzing | `ffuf -u http://IP -H "Host: FUZZ.pelda.hu" -w subs.txt` |
| Certificate Transparency | `curl "https://crt.sh/?q=%.pelda.hu&output=json"` |
| Non-standard ports | `nmap -p- -sV pelda.hu` |

---

## Fontos Toolok Listája

### Port Scanning:
- **nmap** - Standard port scanner
- **masscan** - Gyors port scanner

### Directory Brute-forcing:
- **gobuster** - Go-based directory brute-forcer
- **ffuf** - Fast fuzzer
- **dirb** - Classic directory brute-forcer
- **dirsearch** - Python-based scanner

### Subdomain Enumeration:
- **amass** - Legjobb subdomain tool
- **sublist3r** - Egyszerű és hatékony
- **subfinder** - Go-based subdomain finder
- **assetfinder** - Passive subdomain finder

### DNS Tools:
- **dig** - DNS lookup
- **host** - DNS query tool
- **nslookup** - DNS query (Windows is)

### Online Services:
- **crt.sh** - Certificate Transparency
- **Netcraft SearchDNS** - Web-based DNS search
- **DNSDumpster** - DNS recon
- **MxToolBox** - Reverse IP lookup

### Vulnerability Scanners:
- **Nessus** - Commercial vulnerability scanner
- **Nikto** - Web server scanner
- **Burp Suite** - Web app security testing

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **Port Management**
   - Csak szükséges portok legyenek nyitva
   - Firewall szabályok
   - Nem használt szolgáltatások leállítása

2. **Virtual Host Security**
   - Default vhost konfiguráció (nem értelmes oldalt ad vissza)
   - Csak ismert Host header-eket fogadjon el

3. **DNS Security**
   - DNS Zone Transfer tiltása
   - DNSSEC használata
   - Reverse DNS proper konfigurálása

4. **SSL/TLS**
   - Wildcard certificate helyett specifikus cert-ek
   - Ne legyen a cert-ben minden subdomain

5. **Directory Listing**
   - Tiltsd le a directory listing-et
   - Minden könyvtárban legyen index fájl

6. **Security by Obscurity is NOT Security**
   - Rejtett alkalmazások is frissítettek legyenek
   - Adminisztrációs interfészek ne legyenek publikusan elérhetők

---

## Fontos Megjegyzések

✅ **Ez a DISCOVERY fázis** - találd meg az összes alkalmazást  
✅ **Kombináld a technikákat** - egyik sem 100%-os  
✅ **Automatizáld** - sok alkalmazásnál manuálisan lassú  
❌ **NE feltételezd**, hogy csak egy app van!  
⚠️ **Scope check** - lehet, hogy egyes alkalmazások nem részei a tesztnek!

---

**Összefoglalva:** Ez a fejezet az **alkalmazások felderítéséről** szól. Egy IP cím vagy domain mögött **TÖBB alkalmazás rejtőzhet** nem-standard URL-eken, portokon, vagy virtual host neveken. A teljes scope megtalálásához **port scanning, subdomain enumeration, directory brute-forcing, és virtual host fuzzing** mind szükséges!
