# Rozdział 44: CORP (Cross-Origin-Resource-Policy)

## 1. Czym jest CORP

**Cross-Origin-Resource-Policy (CORP)** to nagłówek HTTP odpowiedzi, który pozwala zasobom (obrazkom, skryptom, API responses, itp.) deklarować kto może je wczytać cross-origin. Jest to mechanizm analogiczny do CORS, ale prostszy i bardziej ograniczający — działa na poziomie **zasobu** i nie wymaga negocjacji preflight.

Trzy wartości:

| Wartość | Kto może załadować zasób |
|---------|--------------------------|
| `same-origin` | Tylko ten sam origin (protokół + host + port) |
| `same-site` | Ten sam site (eTLD+1, włącznie z subdomenami) |
| `cross-origin` | Dowolny origin (publiczny zasób) |

```http
Cross-Origin-Resource-Policy: same-origin
```

CORP odpowiada pytanie: **"kto może wczytać ten zasób?"** na poziomie serwera, bez angażowania JavaScript po stronie klienta. Jest wymagany przez COEP (`require-corp`) i stanowi ochronę przed Spectre-based cross-origin data leaks.

---

## 2. Dlaczego powstał

### Problem: cross-origin subresource embedding

Przed CORP, każda strona mogła wczytać dowolny zasób przez `<img>`, `<script>`, `<link>`:

```html
<!-- Na evil.com: -->
<img src="https://intranet.company.com/secret-dashboard.png">
<!-- Przeglądarka wczyta obraz z cookies sesji użytkownika -->
<!-- Jeśli użytkownik jest zalogowany → obrazek załaduje się poprawnie -->
<!-- evil.com nie widzi pikseli (SOP) ale... Spectre może! -->
```

CORS rozwiązuje problem dla **JavaScript** (fetch/XHR). Ale nie dla prostych `<img>`, `<script>`, `<link>` — te mogą wczytać cross-origin zasób bez CORS.

CORP (W3C, 2018, Firefox 75+, Chrome 73+, Safari 12+) daje zasobom możliwość odmowy cross-origin embeddowania — bez potrzeby CORS.

Dwa powody dla CORP:
1. **Ochrona prywatnych zasobów** przed embeddowaniem przez cross-origin strony
2. **Wymaganie COEP** — COEP `require-corp` akceptuje CORP jako dowód na cross-origin consent

---

## 3. Jak działa

### same-origin

```http
Cross-Origin-Resource-Policy: same-origin
```

```html
<!-- Na bank.com/user/avatar: Cross-Origin-Resource-Policy: same-origin -->

<!-- Na bank.com: -->
<img src="/user/avatar"> <!-- OK — same-origin ✓ -->

<!-- Na evil.com: -->
<img src="https://bank.com/user/avatar">
<!-- Network request → przeglądarka widzi CORP: same-origin → BLOKUJE -->
<!-- Console: "Cross-Origin Resource Policy: blocked the request..." -->
```

### same-site

```http
Cross-Origin-Resource-Policy: same-site
```

```html
<!-- Na cdn.bank.com: Cross-Origin-Resource-Policy: same-site -->

<!-- Na bank.com: -->
<img src="https://cdn.bank.com/logo.png"> <!-- OK — same-site ✓ -->

<!-- Na sub.bank.com: -->
<img src="https://cdn.bank.com/logo.png"> <!-- OK — same-site ✓ -->

<!-- Na evil.com: -->
<img src="https://cdn.bank.com/logo.png"> <!-- BLOKOWANE — cross-site ✗ -->
```

### cross-origin

```http
Cross-Origin-Resource-Policy: cross-origin
```

```
Zasób publiczny — każdy może go wczytać.
Wymagane przez COEP: require-corp dla zewnętrznych CDN!
(CDN biblioteka, publiczny font → musi mieć CORS lub CORP: cross-origin)
```

### Brak CORP nagłówka

```
Domyślne zachowanie bez CORP:
- Zasób może być wczytany przez kogokolwiek (jak cross-origin)
- ALE: COEP: require-corp na dokumencie wczytującym → BLOKUJE brak-CORP zasoby!
```

---

## 4. Co dzieje się wewnętrznie

### Weryfikacja przy ładowaniu

```
Przeglądarka wczytuje zasób:

1. Sprawdź CORP na response zasób:
   - Brak CORP → sprawdź czy wczytujący dokument ma COEP:require-corp
     → TAK → blokuj zasób (brak zezwolenia)
     → NIE → wczytaj normalnie (stare zachowanie)
   
   - CORP: same-origin → sprawdź czy wczytujący i zasób mają ten sam origin
     → TAK → wczytaj ✓
     → NIE → blokuj ✗
   
   - CORP: same-site → sprawdź czy wczytujący i zasób są same-site
     → TAK → wczytaj ✓
     → NIE → blokuj ✗
   
   - CORP: cross-origin → wczytaj zawsze ✓
```

### CORP vs CORS — kiedy co używać?

```
CORS (Access-Control-Allow-Origin):
- Używane dla fetch/XHR (JavaScript)
- Mechanizm preflight dla non-simple requests
- JavaScript może odczytać odpowiedź
- Wymagane przez COEP (spełnia "zezwolenie")

CORP (Cross-Origin-Resource-Policy):
- Używane dla subresource embedding (<img>, <script>, itp.)
- Brak preflight
- JavaScript NIE może odczytać odpowiedzi (CORP nie daje dostępu JS!)
- Tylko mówi: "mogę/nie mogę być załadowany cross-origin"
- Wymagane przez COEP (spełnia "zezwolenie")
```

### CORP i opaque responses

W fetch z `mode: 'no-cors'`:

```javascript
// fetch z no-cors zwraca "opaque response" — JS nie widzi danych
const response = await fetch('https://cdn.example.com/image.png', { mode: 'no-cors' });
response.type; // 'opaque' — status=0, body=null

// CORP: same-origin na tym zasobie → nawet no-cors request jest blokowany!
```

---

## 5. Analogiczny przykład z życia

CORP to **etykieta "tylko do użytku wewnętrznego"** na dokumentach:

- Dokument z etykietą `same-origin`: może być kopiowany wyłącznie w tej samej firmie
- Dokument z `same-site`: może krążyć wewnątrz grupy firm (holding)
- Dokument bez etykiety: każdy może go skopiować
- `cross-origin`: oficjalnie przeznaczony do publicznego udostępnienia

Etykieta jest na dokumencie (zasób/serwer), nie na kopiarce (przeglądarka). Nie musisz konfigurować kopiarki — wystarczy etykieta na dokumentach.

---

## 6. Przykład kodu

```http
# === Konfiguracja nagłówków CORP w zależności od zasobu ===

# Prywatne dane użytkownika (tylko same-origin):
Cross-Origin-Resource-Policy: same-origin
Cache-Control: private, no-store

# CDN zasoby same-company (subdomenowe):
Cross-Origin-Resource-Policy: same-site
Cache-Control: public, max-age=86400

# Publiczne zasoby (npm CDN, biblioteki):
Cross-Origin-Resource-Policy: cross-origin
Access-Control-Allow-Origin: *
Cache-Control: public, max-age=31536000
```

```javascript
// === Express.js: CORP middleware ===

function corpMiddleware(policy = 'same-origin') {
    return (req, res, next) => {
        res.setHeader('Cross-Origin-Resource-Policy', policy);
        next();
    };
}

// Routing z różnymi politykami:
app.use('/user/', corpMiddleware('same-origin'));         // prywatne
app.use('/company/', corpMiddleware('same-site'));        // intranet
app.use('/public/', corpMiddleware('cross-origin'));      // publiczne CDN
app.use('/api/', corpMiddleware('same-origin'));          // API (+ CORS)

// API zwykle potrzebuje też CORS dla fetch:
app.use('/api/', cors({ origin: 'https://app.example.com', credentials: true }));
app.use('/api/', corpMiddleware('same-origin'));
```

```python
# === Flask: CORP na zasobach ===

from flask import send_from_directory
from functools import wraps

def corp_policy(policy):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            response = f(*args, **kwargs)
            response.headers['Cross-Origin-Resource-Policy'] = policy
            return response
        return decorated
    return decorator

@app.route('/user/avatar/<int:user_id>')
@login_required
@corp_policy('same-origin')
def user_avatar(user_id):
    return send_from_directory('avatars', f'{user_id}.png')

@app.route('/public/image/<filename>')
@corp_policy('cross-origin')
def public_image(filename):
    return send_from_directory('public_images', filename)
```

```javascript
// === Nginx: CORP konfiguracja ===
// nginx.conf fragment:

/*
location /api/ {
    add_header Cross-Origin-Resource-Policy "same-origin";
    add_header Access-Control-Allow-Origin "https://app.example.com";
    proxy_pass http://backend;
}

location /public/ {
    add_header Cross-Origin-Resource-Policy "cross-origin";
    add_header Access-Control-Allow-Origin "*";
    expires 1y;
}
*/
```

---

## 7. Przykład z prawdziwej aplikacji

### Ochrona prywatnych avatarów przed Spectre

Bez CORP:

```
1. Użytkownik zalogowany na company.com
2. Użytkownik odwiedza evil.com (w tej samej przeglądarce)
3. Evil.com: <img src="https://company.com/user/avatar.png">
4. Przeglądarka pobiera avatar z session cookie → sukces (HTTP 200)
5. JavaScript nie widzi obrazu (SOP) — ale evil.com wie że jesteś zalogowany!
6. Z Spectre: evil.com mógłby odczytać piksele obrazu z pamięci procesu
   → wyciek prywatnych danych (np. zdjęcie profileowe)
```

Z CORP: same-origin:

```
3. Evil.com próbuje: <img src="https://company.com/user/avatar.png">
4. Przeglądarka widzi CORP: same-origin → cross-origin request → BLOKUJE
5. Obrazek nigdy nie trafia do pamięci procesu evil.com
6. Spectre nie może odczytać niczego
```

### COEP i CDN integracja

Problem typowy przy wdrażaniu COEP: CDN nie ma CORP/CORS:

```javascript
// Audyt przed COEP:
async function auditExternalResources() {
    const resources = performance.getEntriesByType('resource');
    const issues = [];
    
    for (const resource of resources) {
        const url = new URL(resource.name);
        if (url.origin === location.origin) continue;  // same-origin OK
        
        // Sprawdź czy zasób ma CORP lub CORS:
        try {
            const response = await fetch(resource.name, { mode: 'cors' });
            if (!response.headers.get('Cross-Origin-Resource-Policy') &&
                !response.headers.get('Access-Control-Allow-Origin')) {
                issues.push({
                    url: resource.name,
                    type: resource.initiatorType,
                    issue: 'Brak CORP i CORS — będzie blokowane przez COEP',
                });
            }
        } catch (e) {
            issues.push({ url: resource.name, issue: 'Brak CORS' });
        }
    }
    
    return issues;
}
```

### Intranet resources ochrona

```
Problem: internal.company.com ma API bez CORP
Atakujący (external) → user odwiedza evil.com → JS na evil.com:
fetch('http://internal.company.com/sensitive-api')
→ Przez user's browser, z credentials intranetowych → możliwy data leak!

Z CORP: same-origin na internal.company.com:
→ Fetch z evil.com → CORP: same-origin → BLOKOWANE
→ JavaScript na evil.com nie ma dostępu
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak CORP na prywatnych zasobach

```javascript
// BŁĄD: prywatne pliki użytkownika bez CORP
app.get('/uploads/:userId/:filename', authenticate, (req, res) => {
    res.sendFile(path.join(UPLOADS_DIR, req.params.userId, req.params.filename));
    // Brak CORP header → zasób może być wczytany cross-origin!
});

// POPRAWKA:
app.get('/uploads/:userId/:filename', authenticate, (req, res) => {
    res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');
    res.sendFile(path.join(UPLOADS_DIR, req.params.userId, req.params.filename));
});
```

### Błąd 2: Mylenie CORP z CORS

```javascript
// BŁĄD: myślenie że CORP daje dostęp JavaScript do zasobu
// CORP: same-site → CDN zasób może być załadowany przez same-site strony
// ALE: JavaScript nadal nie może odczytać danych (to jest rola CORS!)

// CORP != CORS:
// CORP: "kto może wczytać mój zasób" (embedding)
// CORS: "kto może odczytać moje dane przez JS" (programmatic access)

// Dla API (JS access): CORS
// Dla obrazów/mediów (embedding): CORP
// Dla pełnej ochrony: CORP + CORS
```

### Błąd 3: CORP: cross-origin na prywatnych danych

```http
# BŁĄD: prywatne API z cross-origin CORP
Cross-Origin-Resource-Policy: cross-origin   # ← NIEBEZPIECZNE dla prywatnych danych!
# Każdy może teraz embeddować ten zasób!

# POPRAWKA:
Cross-Origin-Resource-Policy: same-origin   # dla prywatnych
Cross-Origin-Resource-Policy: same-site     # dla intranet/company-wide
```

### Błąd 4: Zapomnienie o CORP gdy COEP jest wdrożone

```javascript
// BŁĄD: wdrożenie COEP: require-corp bez aktualizacji własnych zasobów
// Dokumenty: Cross-Origin-Embedder-Policy: require-corp
// Zasoby same-site CDN: brak CORP
// Efekt: CDN zasoby blokowane na własnych stronach!

// POPRAWKA: dodaj CORP na wszystkich własnych zasobach
// CDN: Cross-Origin-Resource-Policy: same-site
// API: Cross-Origin-Resource-Policy: same-origin
// Publiczne: Cross-Origin-Resource-Policy: cross-origin
```

---

## 9. Znaczenie dla bezpieczeństwa

### Defense Against Spectre

```
Łańcuch ochrony:
1. Użytkownik zalogowany na target.com
2. Evil.com chce wczytać prywatny zasób przez Spectre:
   a. <img src="https://target.com/private/data"> 
   b. Bez CORP → przeglądarka pobiera → Spectre może odczytać z RAM!
   c. Z CORP: same-origin → przeglądarka blokuje → zasób nigdy nie trafia do RAM evil.com
```

### Cross-Site Leaks (XS-Leaks)

XS-Leaks to szeroka kategoria ataków side-channel przez cross-origin embedding:

```javascript
// XS-Leak przez timing:
const start = performance.now();
const img = new Image();
img.src = 'https://target.com/user/avatar?id=123';
img.onload = () => {
    const time = performance.now() - start;
    // Jeśli avatar istnieje → szybkie ładowanie (z cache serwera)
    // Jeśli nie → wolniejsze (404 + time)
    // → możliwe wykrycie istnienia użytkownika!
};
// Z CORP: same-origin → img.src blokowane → timing attack niemożliwy
```

### CORP jako wymaganie COEP

```
Ekosystem zabezpieczeń:
COOP: same-origin → izolacja browsing context
COEP: require-corp → wymaga CORP lub CORS na zasobach
CORP: same-origin/same-site → zasób wyraża zgodę

Razem: crossOriginIsolated = true → Spectre mitigation
```

### Prywatność użytkownika (login state oracle)

```javascript
// Bez CORP: evil.com może sprawdzić czy jesteś zalogowany przez:
const img = new Image();
img.src = 'https://bank.com/user/profile-pic';
img.onload = () => alert('Zalogowany w bank.com!');
img.onerror = () => alert('Niezalogowany lub nie masz zdjęcia');

// Z CORP: same-origin → request blokowany zawsze → evil.com nic nie wie
```

### Powiązane CWE

- **CWE-200** — Information Exposure (XS-Leaks, Spectre)
- **CWE-346** — Origin Validation Error
- **CWE-693** — Protection Mechanism Failure

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź CORP na prywatnych zasobach

```bash
# Sprawdź prywatne endpointy (wymagające auth):
curl -I https://target.com/api/user-data \
    -H "Authorization: Bearer TOKEN" | grep -i "cross-origin-resource-policy"

# Sprawdź avatary i prywatne media:
curl -I https://target.com/uploads/user123/photo.jpg \
    -H "Cookie: session=COOKIE" | grep -i "cross-origin-resource-policy"

# Brak CORP = potencjalnie podatny na XS-Leaks i Spectre
```

### Test cross-origin embeddowania

```javascript
// Z przeglądarki na evil.com (lub lokalny test):
// Sprawdź czy prywatne zasoby są ładowane cross-origin:

const img = new Image();
img.onload = () => console.log('CORP BRAK — zasób załadowany!');
img.onerror = () => console.log('CORP lub auth block');
img.src = 'https://target.com/private/resource';

// Jeśli onload → brak CORP (lub brak auth wymagane)
// Jeśli onerror z powodu CORP → sprawdź console: "Cross-Origin Resource Policy"
```

### XS-Leak test przez timing

```javascript
// Test login state oracle:
async function checkLoginState(target) {
    const start = performance.now();
    
    try {
        await fetch(`${target}/api/me`, { 
            mode: 'no-cors',
            credentials: 'include'
        });
        const time = performance.now() - start;
        // Jeśli CORP: same-origin → fetch blokowane → nie ma informacji
        // Jeśli brak CORP → timing może ujawnić stan logowania
        return time;
    } catch (e) {
        return null;
    }
}
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy prywatne zasoby (avatary, dokumenty, dane) mają CORP: same-origin?
□ Czy zasoby same-company CDN mają CORP: same-site?
□ Czy publiczne zasoby mają CORP: cross-origin (wymagane przez COEP)?
□ Czy API endpointy mają CORP i CORS (oba gdy potrzeba)?
□ Czy cross-origin embedding prywatnych zasobów jest blokowane?
□ Czy XS-Leak timing attacks są możliwe (brak CORP)?
□ Czy login state oracle jest możliwe przez <img> load?
□ Czy CORP jest wymagane przez COEP na stronie?
□ Czy brak CORP na własnych zasobach powoduje COEP failures?
```

---

## 12. Jak się zabezpieczać

```http
# Schematy CORP per typ zasobu:

# Prywatne dane (avatary, profile, docs):
Cross-Origin-Resource-Policy: same-origin
Content-Type: image/jpeg   # (poprawny Content-Type!)
Cache-Control: private, no-store

# Company CDN (same-site zasoby):  
Cross-Origin-Resource-Policy: same-site
Cache-Control: public, max-age=86400

# Publiczne zasoby / zewnętrzny CDN:
Cross-Origin-Resource-Policy: cross-origin
Access-Control-Allow-Origin: *
Cache-Control: public, max-age=31536000

# API (private + CORS dla JS):
Cross-Origin-Resource-Policy: same-origin
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

```python
# === Django: automatyczny CORP middleware ===

class CORPMiddleware:
    PRIVATE_PATHS = ['/user/', '/uploads/', '/api/private/']
    SITE_PATHS = ['/company/', '/cdn/', '/shared/']
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        response = self.get_response(request)
        
        path = request.path
        
        if any(path.startswith(p) for p in self.PRIVATE_PATHS):
            response['Cross-Origin-Resource-Policy'] = 'same-origin'
        elif any(path.startswith(p) for p in self.SITE_PATHS):
            response['Cross-Origin-Resource-Policy'] = 'same-site'
        else:
            response['Cross-Origin-Resource-Policy'] = 'cross-origin'
        
        return response
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. CORP = `Cross-Origin-Resource-Policy` header na zasobach — kto może je wczytać
2. `same-origin` = tylko ten sam origin; `same-site` = ten sam site; `cross-origin` = wszyscy
3. Chroni przed Spectre, XS-Leaks, login state oracles przez cross-origin embedding
4. Wymagany przez COEP (`require-corp`) jako "zezwolenie" zasobu na cross-origin access
5. Różni się od CORS: CORP = kto może embeddować; CORS = kto może odczytać przez JS

**Dla pentestera:**
- Sprawdź prywatne endpointy: czy `Cross-Origin-Resource-Policy` jest ustawione?
- Brak CORP = możliwe XS-Leaks (timing, load/error oracles)
- Test: wczytaj prywatny zasób jako `<img>` z cross-origin — jeśli ładuje się → brak CORP
- Sprawdź czy COEP na stronie nie jest blokowane przez brak CORP na zasobach

---

## Powiązania

```
CORP (Cross-Origin-Resource-Policy)
    │
    ├──► COEP (Rozdział 43)
    │         COEP: require-corp wymaga CORP lub CORS na zasobach
    │         CORP spełnia wymaganie COEP
    │
    ├──► CORS (Rozdział 13)
    │         CORS: JavaScript access; CORP: embedding
    │         Oba akceptowane przez COEP jako "zezwolenie"
    │
    ├──► SRI (Rozdział 38)
    │         SRI: weryfikuje hash zasobu
    │         CORP: kontroluje kto może załadować zasób
    │         Komplementarne zabezpieczenia
    │
    └──► Fetch Metadata (Rozdział 39)
              FM po stronie serwera sprawdza źródło requestu
              CORP po stronie zasobu deklaruje kto może go wczytać
```
