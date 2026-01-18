# WSTG-INFO-07: Végrehajtási Útvonalak Feltérképezése az Alkalmazásban

## Mi a fejezet lényege?

Ez a fejezet arról szól, hogy **meg kell érteni az alkalmazás struktúráját és működési folyamatait** (workflow-kat) MIELŐTT tesztelnéd. Fel kell térképezni, hogy **milyen útvonalakon lehet végigmenni** az alkalmazásban, milyen döntési pontok vannak, és hogyan kapcsolódnak össze a különböző funkciók.

⚠️ **NE KEVERD ÖSSZE:** Ez NEM konkrét sebezhetőség keresés! Ez **strukturális feltérképezés** - meg kell érteni, hogy az alkalmazás hogyan működik, milyen workflow-k léteznek, mielőtt tesztelnéd őket.

---

## Mi a cél?

**Az alkalmazás feltérképezése és a fő workflow-k megértése**:
- Minden link/URL dokumentálása
- Döntési pontok azonosítása (decision points)
- User journey-k feltérképezése
- Code coverage dokumentálása

### Miért fontos?

- **Black-box testing-ben** nem látod a kódot → térképezned kell
- **Teljes kódbázist tesztelni lehetetlen** → priorizálni kell
- **Workflow megértés** nélkül hiányos lesz a teszt
- **Dokumentáció** bizonyítja, hogy mit teszteltél és mit nem

---

## 3 Fő Megközelítés a Code Coverage-hez

### 1. **Path Testing** (Útvonal Tesztelés)
**Concept:** Teszteld **minden lehetséges útvonalat** az alkalmazáson keresztül.

**Példa:**
```
Login → 
  ├─ Success → Dashboard → 
  │             ├─ View Profile
  │             ├─ Edit Profile
  │             └─ Logout
  └─ Fail → Error Message → Retry
```

**Problémák:**
- **Exponenciális növekedés** - Minden döntési pont megduplázza az útvonalakat
- 10 döntési pont = 1024 lehetséges útvonal
- **Időigényes és bonyolult**

**Mikor használd:** 
- Kritikus funkciók (payment, authentication)
- Kis, zárt alkalmazásrészek

---

### 2. **Data Flow (Taint Analysis)** (Adatáramlás Elemzés)
**Concept:** Kövesd az **user input útját** az alkalmazáson keresztül.

**Példa:**
```
User Input: "John<script>alert(1)</script>"
   ↓
Form Submit → Server Processing → Database Storage → Output Rendering
   ↓              ↓                    ↓                  ↓
GET Param     Validation?         Sanitization?      Escaping?
```

**Mit figyelsz:**
- Honnan jön a user input? (GET, POST, Cookie, Header)
- Hogyan dolgozza fel a szerver?
- Hol tárolódik? (DB, Session, File)
- Hogyan renderelődik vissza?
- **Van-e sanitization/validation bármely ponton?**

**Mikor használd:**
- Input validation teszteléshez
- SQL injection, XSS felderítéshez
- IDOR (Insecure Direct Object Reference) kereséshez

---

### 3. **Race Condition Testing** (Versenyhelyzet Tesztelés)
**Concept:** **Több párhuzamos kérés** ugyanarra az erőforrásra.

**Példa:**
```
User egyenlege: 100 USD

Thread 1: Vásárlás 80 USD → Ellenőrzés (100 >= 80) ✓ → Levonás
Thread 2: Vásárlás 80 USD → Ellenőrzés (100 >= 80) ✓ → Levonás

Végeredmény: -60 USD (mindkét tranzakció sikerült!)
```

**Mit tesztelsz:**
- Párhuzamos order-ek
- Egyidejű account módosítások
- File upload race conditions
- Coupon code többszöri használata

**Mikor használd:**
- E-commerce alkalmazások
- Banking/financial apps
- Multi-user rendszerek

---

## Mapping Workflow

### 1. Lépés: Spidering (Automatikus Feltérképezés)

#### Tool #1: **OWASP ZAP Spider**

**Használat:**
```
1. Indítsd el ZAP-ot
2. Quick Start → Automated Scan
3. URL to attack: https://pelda.hu
4. Attack gomb
5. Spider tab → Követi az összes linket
```

**Spider Types:**

**A) Traditional Spider:**
- HTML linkeket követi (`<a href>`)
- Form-okat azonosít
- Robots.txt-et tiszteletben tartja (opciósan)
- **Gyors, de JavaScript-et NEM futtatja**

**B) AJAX Spider:**
- JavaScript-et futtatja (headless browser)
- AJAX call-okat is követi
- SPA (Single Page Application) támogatás
- **Lassabb, de modern appok-hoz kell**

**C) OpenAPI Support:**
- Swagger/OpenAPI spec import
- REST API endpoint-ok automatikus feltérképezése
- **API testing-hez ideális**

---

#### Tool #2: **Burp Suite Spider**

**Használat:**
```
1. Burp Suite → Target → Site map
2. Jobb klikk a target-re → Spider this host
3. Spider Options:
   - Check Robots.txt
   - Maximum link depth
   - Forms: prompt/auto-submit
4. Spider Status → Látod a progresst
```

**Spider eredmények:**
- **Site map** - Tree view minden URL-ről
- **Target scope** - Definiálhatod, mit spiderelj
- **Discovered items** - Hidden directories, parameters

---

#### Tool #3: **Katana (CLI Spider)**

**Telepítés és használat:**
```bash
# Telepítés (Go required)
go install github.com/projectdiscovery/katana/cmd/katana@latest

# Alapvető használat
katana -u https://pelda.hu

# JavaScript is futtatása
katana -u https://pelda.hu -js-crawl

# Output fájlba
katana -u https://pelda.hu -o urls.txt

# Depth limit
katana -u https://pelda.hu -d 5

# Form auto-fill
katana -u https://pelda.hu -form-fill
```

**Mit csinál:**
- Gyors, modern spider
- JavaScript rendering
- Headless Chrome support
- CLI-based (scriptelhető)

---

### 2. Lépés: Manual Crawling (Manuális Bejárás)

**Miért kell manuálisan is?**
- Spider nem minden funkciót talál meg
- Protected areas (authentication)
- Complex workflows (multi-step processes)
- Business logic understanding

**Mit csinálj:**
```
1. Regisztráció teljes folyamata
2. Bejelentkezés
3. Profile management
   - View profile
   - Edit profile
   - Change password
   - Upload avatar
4. Main functionality
   - Search
   - Filter/Sort
   - Add to cart
   - Checkout process
5. Admin panel (ha van access)
6. Error states
   - 404 pages
   - Access denied
   - Form validation errors
```

---

### 3. Lépés: Spreadsheet Dokumentáció

**Excel/Google Sheets template:**

| URL | HTTP Method | Parameters | Auth Required | Workflow Step | Decision Points | Tested? | Notes |
|-----|-------------|------------|---------------|---------------|-----------------|---------|-------|
| /register | GET | - | No | Step 1 | - | Yes | Registration form |
| /register | POST | username, email, password | No | Step 2 | Email validation | Yes | Redirects to /login |
| /login | POST | username, password | No | Step 3 | Credentials check | Yes | Success → /dashboard |
| /dashboard | GET | - | Yes | Step 4 | - | Yes | Main page |
| /profile/edit | POST | name, bio, avatar | Yes | Step 5 | File upload | Yes | Avatar size check |
| /admin | GET | - | Yes | Admin | Role check | Yes | 403 if not admin |

---

### 4. Lépés: Workflow Diagram Készítése

**Tool: draw.io, Lucidchart, vagy egyszerű Markdown:**

```
Registration Workflow:
┌─────────────┐
│   /register │  GET
│   (form)    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   /register │  POST (username, email, password)
└──────┬──────┘
       │
    ┌──┴──┐
    │ Valid? │
    └──┬──┘
   Yes │ No
       │  │
       ▼  ▼
   /login  Error message → Retry


Login Workflow:
┌─────────────┐
│    /login   │  POST (username, password)
└──────┬──────┘
       │
    ┌──┴──────┐
    │ Correct?│
    └──┬──────┘
   Yes │ No
       │  │
       ▼  ▼
  /dashboard  Error → Retry
       │
       ├─→ /profile
       ├─→ /settings
       └─→ /logout
```

---

### 5. Lépés: Decision Points Azonosítása

**Decision point = ahol az alkalmazás dönt** (if/else, switch/case)

**Példák:**

**1. Authentication check:**
```
Request: /admin
  ├─ Logged in? 
  │   ├─ Yes → Role check
  │   │   ├─ Admin? → Show admin panel
  │   │   └─ User? → 403 Forbidden
  │   └─ No → Redirect to /login
```

**2. Payment processing:**
```
POST /checkout
  ├─ Cart not empty?
  │   ├─ Yes → Calculate total
  │   │   ├─ Coupon code valid?
  │   │   │   ├─ Yes → Apply discount
  │   │   │   └─ No → Full price
  │   │   └─ Process payment
  │   │       ├─ Success → Order confirmation
  │   │       └─ Fail → Error page
  │   └─ No → Error: Empty cart
```

**3. File upload:**
```
POST /upload
  ├─ File provided?
  │   ├─ Yes → Check file type
  │   │   ├─ Allowed extension?
  │   │   │   ├─ Yes → Check file size
  │   │   │   │   ├─ < 5MB? → Save file
  │   │   │   │   └─ > 5MB? → Error: Too large
  │   │   │   └─ No → Error: Invalid type
  │   └─ No → Error: No file
```

---

## Code Coverage Dokumentálása

### Coverage Spreadsheet Template:

```
Feature: User Registration
Total Paths: 12
Tested Paths: 10
Coverage: 83%

Tested Paths:
✓ Valid registration
✓ Duplicate username
✓ Duplicate email
✓ Weak password
✓ Missing required field
✓ XSS in username field
✓ SQL injection in email
✓ Email format validation
✓ Password length validation
✓ Special characters in name

Untested Paths:
✗ Unicode characters in username (Low priority)
✗ Extremely long input (>1000 chars) (Medium priority)

Notes:
- Password validation seems robust
- XSS protection working
- TODO: Test race condition (simultaneous registration)
```

---

## Automatizált vs. Manuális Mapping

### Automatizált Előnyök:
✅ Gyors (1000+ URL percek alatt)
✅ Nem fárad el
✅ Minden linket követ
✅ Exportálható eredmények

### Automatizált Hátrányok:
❌ JavaScript heavy apps problémásak
❌ Authentication mögött nem lát
❌ Business logic-ot nem érti
❌ False positives

### Manuális Előnyök:
✅ Complex workflows megértése
✅ Business logic discovery
✅ Edge cases találása
✅ Context awareness

### Manuális Hátrányok:
❌ Időigényes
❌ Emberihiba lehetséges
❌ Nem skálázódik nagy appokra

**Best practice:** **KOMBINÁLD A KETTŐT!**
```
1. Automatizált spider → Gyors coverage
2. Manuális walkthrough → Mélyebb megértés
3. Dokumentáció → Mit találtál
4. Prioritizálás → Mit tesztelsz először
```

---

## Advanced Spidering Techniques

### 1. Authenticated Spidering

**ZAP:**
```
1. Manuálisan jelentkezz be
2. ZAP → Tools → Options → Authentication
3. Session Management → Cookie-based
4. Spider → Use current session
```

**Burp Suite:**
```
1. Proxy → Intercept a login request
2. Session Handling Rules → Add
3. Cookie Jar → Add cookies from response
4. Spider with authenticated session
```

---

### 2. Form Auto-Submission

**ZAP Spider Options:**
```
Spider → Options → Form Handling:
- Process forms: ✓
- POST forms: ✓
- Default form values:
  - Text: "test"
  - Email: "test@test.com"
  - Number: 100
```

**Burp Suite:**
```
Spider → Options → Form Submission:
- Prompte for guidance
- Automatically submit
- Don't submit forms
- Use predefined values
```

---

### 3. Scope Definition

**Csak a target domain-t spider-ezd:**

**ZAP:**
```
1. Sites → Right-click target → Include in Context
2. Spider → Only in scope
```

**Burp:**
```
1. Target → Scope → Add
2. Protocol: https
3. Host: pelda.hu
4. Spider → Options → Only spider in-scope items ✓
```

---

## Workflow Examples

### E-commerce Site Workflow:

```
Homepage (/)
  │
  ├─→ Product Listing (/products)
  │     ├─→ Product Detail (/product/123)
  │     │     ├─→ Add to Cart (POST /cart/add)
  │     │     └─→ Reviews (/product/123/reviews)
  │     └─→ Category Filter (/products?category=electronics)
  │
  ├─→ Search (/search?q=laptop)
  │     └─→ Search Results
  │           └─→ Product Detail
  │
  ├─→ Cart (/cart)
  │     ├─→ Update Quantity (POST /cart/update)
  │     ├─→ Remove Item (DELETE /cart/item/123)
  │     └─→ Checkout (/checkout)
  │           ├─→ Shipping Info (POST /checkout/shipping)
  │           ├─→ Payment (POST /checkout/payment)
  │           └─→ Order Confirmation (/order/456)
  │
  └─→ User Account
        ├─→ Register (/register)
        ├─→ Login (/login)
        ├─→ Profile (/profile)
        │     ├─→ Edit Profile (POST /profile/edit)
        │     └─→ Change Password (POST /profile/password)
        └─→ Order History (/orders)
              └─→ Order Details (/order/456)
```

**Decision Points:**
1. Login: Credentials valid?
2. Cart: Items in cart?
3. Checkout: Address valid?
4. Payment: Card valid?
5. Admin: User role = admin?

---

## Python Script: Automated Mapping

### URL Collector + Workflow Generator:

```python
#!/usr/bin/env python3
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse
import networkx as nx
import matplotlib.pyplot as plt

class WebsiteMapper:
    def __init__(self, base_url):
        self.base_url = base_url
        self.visited = set()
        self.graph = nx.DiGraph()
        
    def spider(self, url, depth=0, max_depth=3):
        if depth > max_depth or url in self.visited:
            return
        
        self.visited.add(url)
        print(f"[{depth}] Visiting: {url}")
        
        try:
            r = requests.get(url, timeout=5)
            soup = BeautifulSoup(r.text, 'html.parser')
            
            # Find all links
            for link in soup.find_all('a', href=True):
                next_url = urljoin(self.base_url, link['href'])
                
                # Only follow links from same domain
                if urlparse(next_url).netloc == urlparse(self.base_url).netloc:
                    self.graph.add_edge(url, next_url)
                    self.spider(next_url, depth + 1, max_depth)
        
        except Exception as e:
            print(f"Error visiting {url}: {e}")
    
    def generate_report(self):
        print("\n=== Website Map Report ===")
        print(f"Total URLs discovered: {len(self.visited)}")
        print(f"\nAll URLs:")
        for url in sorted(self.visited):
            print(f"  - {url}")
        
        print(f"\nURL relationships:")
        for source, target in self.graph.edges():
            print(f"  {source} → {target}")
    
    def visualize(self, output_file='website_map.png'):
        plt.figure(figsize=(15, 10))
        pos = nx.spring_layout(self.graph)
        nx.draw(self.graph, pos, with_labels=True, 
                node_color='lightblue', 
                node_size=500, 
                font_size=8, 
                arrows=True)
        plt.savefig(output_file)
        print(f"\nVisualization saved to {output_file}")

# Használat
mapper = WebsiteMapper("https://pelda.hu")
mapper.spider("https://pelda.hu", max_depth=2)
mapper.generate_report()
mapper.visualize()
```

**Használat:**
```bash
python3 website_mapper.py
```

---

## Gyakorlati Cheat Sheet

| Feladat | Tool/Parancs |
|---------|--------------|
| Automatikus spider | ZAP → Quick Start → Automated Scan |
| AJAX spider | ZAP → AJAX Spider → Target URL |
| Burp spider | Target → Right-click → Spider this host |
| CLI spider | `katana -u https://pelda.hu` |
| Authenticated spider | Login first → Spider with session |
| Form auto-submit | Spider Options → Form Handling |
| Scope definition | Target → Scope → Add |
| Workflow diagram | draw.io, Mermaid, Markdown |
| Coverage tracking | Excel/Google Sheets |

---

## Fontos Toolok

### Spider Tools:
- **OWASP ZAP** - Traditional + AJAX Spider
- **Burp Suite** - Professional spider
- **Katana** - Modern CLI spider
- **Gospider** - Go-based spider
- **Hakrawler** - Simple web crawler

### Diagramming:
- **draw.io** - Free diagramming tool
- **Lucidchart** - Professional diagrams
- **Mermaid** - Markdown-based diagrams
- **PlantUML** - Text-to-diagram

### Documentation:
- **Excel/Google Sheets** - Coverage tracking
- **Notion** - Rich documentation
- **Obsidian** - Markdown notes
- **Joplin** - Open source notes

---

## Védelem (Remediation)

### Hogyan védekezz a mapping ellen?

1. **robots.txt**
   - Azonban: NEM biztonság! Spider-ek ignorálják

2. **Rate limiting**
   - Lassítsd az automatikus spider-eket
   - IP-based vagy session-based limits

3. **CAPTCHA**
   - Kritikus endpoint-okon (login, register)
   - Bot detekció

4. **WAF (Web Application Firewall)**
   - Suspicious patterns detekció
   - Spider user-agent blokkolás

5. **Authentication**
   - Sensitive areas mögé

**Azonban:** **Mapping NEM támadás!**
- Legitim pentester is mappinget csinál
- Crawlerek (Google bot) is mappingelnek
- **Defense in depth** kell, ne csak obscurity!

---

## Fontos Megjegyzések

✅ **Ez strukturális megértés** - workflow-k és decision points  
✅ **Kombináld auto + manual** - egyik sem elég önmagában  
✅ **Dokumentáld részletesen** - spreadsheet + diagram  
✅ **Decision points = kritikus tesztpontok**  
❌ **Ne csak spider-ezz** - manuális walkthrough is kell!  
⚠️ **Authentication-nel protected areas** külön figyelmet igényelnek

---

**Összefoglalva:** Ez a fejezet az **alkalmazás végrehajtási útvonalainak feltérképezéséről** szól. Használj **automatikus spider-t** (ZAP, Burp, Katana) gyors URL discovery-hez, de **manuális walkthrough** is kell a complex workflow-k megértéséhez. **Dokumentáld minden útvonalat spreadsheet-ben**, azonosítsd a **decision point-okat**, és készíts **workflow diagramot**. Ez az alapja annak, hogy megértsd az alkalmazás működését és **priorizáld a tesztelendő területeket**!
