# WSTG-INFO-01: Keresőmotor Felfedezési Felderítés Információszivárgásra

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **keresőmotorok segítségével** (Google, Bing, stb.) hogyan lehet **passzívan** információt gyűjteni egy célalkalmazásról vagy szervezetről. A lényeg, hogy a keresőmotorok indexelnek mindent, amit találnak - **még olyan tartalmakat is, amiket nem kellene**.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM aktív támadás, NEM SQL injection, NEM XSS keresés! Ez **passzív információgyűjtés** - csak azt nézzük meg, amit a keresők már eltároltak.

---

## Mi a cél?

**Azonosítani érzékeny tervezési és konfigurációs információkat**, amelyek:
- Közvetlenül a cél weboldalán találhatók
- Közvetetten harmadik fél szolgáltatásokon keresztül elérhetők (fórumok, архívumok)

### Mit keresünk?

- Hálózati diagramok és konfigurációk
- Adminisztrátorok régi posztjai, emailjei
- Bejelentkezési eljárások, felhasználónév formátumok
- **Felhasználónevek, jelszavak, privát kulcsok**
- Felhőszolgáltatás konfigurációs fájlok
- Részletes hibaüzenetek
- Nem publikus alkalmazások (dev, test, UAT, staging verziók)

---

## Keresőmotorok

### Használandó keresők:

1. **Google** - Világszerte legnépszerűbb
2. **Bing** - Microsoft, második legnépszerűbb
3. **DuckDuckGo** - Adatvédelem-központú
4. **Baidu** - Kína legnépszerűbb keresője
5. **Shodan** - IoT eszközök és szolgáltatások keresője
6. **Internet Archive Wayback Machine** - Weboldalak archívuma 1996-tól
7. **Common Crawl** - Nyílt web crawl adatbázis

---

## Keresési Operátorok (Search Operators)

### Alapvető operátorok:

| Operátor | Funkció | Példa |
|----------|---------|-------|
| `site:` | Csak adott domainre korlátozza a keresést | `site:example.com` |
| `inurl:` | Kulcsszó az URL-ben | `inurl:admin` |
| `intitle:` | Kulcsszó az oldal címében | `intitle:"index of"` |
| `intext:` / `inbody:` | Kulcsszó az oldal törzsében | `intext:password` |
| `filetype:` | Specifikus fájltípus | `filetype:pdf` |

---

## Példa Parancsok (Google Dorking)

### 1. Alapvető domain felderítés
```
site:pelda.hu
```
**Mit csinál:** Minden indexelt oldalt megmutat a pelda.hu domainről.

---

### 2. Érzékeny fájlok keresése
```
site:pelda.hu filetype:pdf
site:pelda.hu filetype:xls "konfidenciális"
site:pelda.hu filetype:sql
```
**Mit csinál:** Konkrét fájltípusokat keres, amik tartalmazhatnak érzékeny adatokat.

---

### 3. Bejelentkezési oldalak keresése
```
site:pelda.hu inurl:login
site:pelda.hu inurl:admin
site:pelda.hu intitle:"admin panel"
```
**Mit csinál:** Admin és login oldalakat keres, amik esetleg rosszul védettek.

---

### 4. Hibaüzenetek keresése
```
site:pelda.hu intext:"sql syntax error"
site:pelda.hu intext:"warning: mysql"
site:pelda.hu "fatal error"
```
**Mit csinál:** Olyan oldalakat keres, ahol kikerült hibainformáció (stack trace, adatbázis info).

---

### 5. Konfigurációs fájlok
```
site:pelda.hu filetype:env
site:pelda.hu filetype:config
site:pelda.hu inurl:".git"
```
**Mit csinál:** Véletlenül publikált konfigurációs fájlokat keres.

---

### 6. Directory listing
```
site:pelda.hu intitle:"index of"
site:pelda.hu intitle:"index of" "parent directory"
```
**Mit csinál:** Olyan könyvtárakat keres, ahol nincs index fájl és látszik a tartalom.

---

## Speciális Toolok

### 1. **Internet Archive Wayback Machine**

#### Használat:
```
https://web.archive.org/web/*/pelda.hu
```
**Mit csinál:** Megmutatja a weboldal összes archivált verzióját 1996-tól.

**Mire jó:** Régi, már törölt oldalak megtekintése, régi sebezhetőségek azonosítása.

---

### 2. **Shodan**

#### Parancsok:
```
hostname:pelda.hu
org:"Pelda Kft"
port:3389 country:"HU"
```
**Mit csinál:** Internethez kapcsolt eszközöket keres (szerverek, routerek, kamerák).

**Figyelem:** Shodan aktívabban szkenneli a netet, de még mindig passzív információgyűjtés.

---

### 3. **Google Hacking Database (GHDB)**

Nem tool, hanem **adatbázis** előre elkészített "dork" lekérdezésekkel.

**Elérhető:** https://www.exploit-db.com/google-hacking-database

**Kategóriák:**
- Files containing usernames
- Files containing passwords
- Sensitive Directories
- Vulnerable Files
- Error Messages

---

## Kombinált Példák (Összetett Keresések)

### 1. Jelszó tartalmú fájlok keresése
```
site:pelda.hu (filetype:xls OR filetype:csv) intext:password
```

---

### 2. Dev/Test környezet keresése
```
site:pelda.hu (inurl:dev OR inurl:test OR inurl:staging)
```

---

### 3. Backup fájlok
```
site:pelda.hu (filetype:bak OR filetype:old OR filetype:backup)
```

---

## Védelem (Remediation)

### Hogyan védekezz ellene?

1. **robots.txt használata** - Megmondod a crawlereknek, mit NE indexeljenek
2. **HTML meta tagek** - `<meta name="robots" content="noindex">`
3. **Google Search Console** - Törölhetsz régi indexelt tartalmat
4. **Autentikáció** - Érzékeny oldalak mögé kötelező bejelentkezés
5. **Rendszeres ellenőrzés** - Időnként keresd meg saját magad

---

## Gyors Cheat Sheet

| Feladat | Parancs |
|---------|---------|
| Domain összes oldala | `site:pelda.hu` |
| PDF fájlok | `site:pelda.hu filetype:pdf` |
| Admin oldalak | `site:pelda.hu inurl:admin` |
| SQL hibák | `site:pelda.hu intext:"sql error"` |
| Directory listing | `site:pelda.hu intitle:"index of"` |
| Archív verzió | `https://web.archive.org/web/*/pelda.hu` |
| Backup fájlok | `site:pelda.hu ext:bak` |

---

## Fontos Megjegyzések

✅ **Ez passzív recon** - nem támadsz semmit, csak nézel  
✅ **Legális** - publikusan elérhető információkat gyűjtesz  
✅ **Első lépés** - Mindig ezzel kezdd a pentestet  
❌ **NE keverd** más technikákkal - ez csak információgyűjtés, nem exploitálás!

---

**Összefoglalva:** Ez a fejezet a **passzív információgyűjtés királya**. Keresőmotorok segítségével olyan adatokat találsz meg, amiket véletlenül indexeltek. Mindig ezt csináld először, mert sok értékes infót kapsz anélkül, hogy bármit is "megérintenél" a célrendszeren.
