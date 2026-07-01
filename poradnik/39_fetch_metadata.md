# Rozdział 39: Fetch Metadata Headers

## 1. Czym są Fetch Metadata Headers

**Fetch Metadata Headers** to zestaw nagłówków HTTP automatycznie dołączanych przez przeglądarkę do każdego requestu, dostarczających serwerowi kontekstowych informacji o tym skąd pochodzi request i jaki jest jego cel. Pozwalają serwerowi podejmować decyzje bezpieczeństwa — np. odrzucić request który wygląda podejrzanie (np. cross-origin request do endpointu który powinien być używany tylko same-origin).

Cztery główne nagłówki Fetch Metadata:

| Nagłówek | Opis |
|----------|------|
| `Sec-Fetch-Site` | Relacja między requestem a witryną (same-origin, same-site, cross-site, none) |
| `Sec-Fetch-Mode` | Tryb requestu (navigate, cors, no-cors, same-origin, websocket) |
| `Sec-Fetch-Dest` | Przeznaczenie odpowiedzi (document, script, image, style, fetch, iframe, ...) |
| `Sec-Fetch-User` | Czy request jest wynikiem interakcji użytkownika (?1) |

Nagłówki są poprzedzone przedrostkiem `Sec-` — co oznacza że są **"forbidden header names"** i nie mogą być modyfikowane przez JavaScript (ani nadpisane przez `fetch()`, `XMLHttpRequest`, ani `<meta>` tagiem). Są ustawiane wyłącznie przez przeglądarkę.

---

## 2. Dlaczego powstał

### Problem: serwer nie wiedział skąd pochodzi request

Przed Fetch Metadata serwer widząc request nie mógł łatwo odróżnić:
- Legalnego requestu nawigacyjnego użytkownika klikającego link
- Cross-origin requestu z atakującej strony (CSRF)
- Requestu zainicjowanego przez skrypt vs przez nawigację przeglądarki

Tradycyjne metody (tokeny CSRF, weryfikacja `Origin`/`Referer`) mają ograniczenia — `Referer` może być brak, `Origin` nie jest zawsze wysyłany. 

Fetch Metadata (W3C Working Draft 2019, Chrome 76, Firefox 90, Safari 16.4) rozwiązuje to przez dostarczenie serwerowi rzetelnych, browser-verified informacji o kontekście requestu — których atakujący nie może sfałszować po stronie klienta (nagłówki `Sec-*` są zakazane dla JavaScript).

---

## 3. Jak działa

### Sec-Fetch-Site

```
same-origin  → request pochodzi z tej samej strony (origin = origin)
same-site    → request pochodzi z tego samego site (different subdomain)
cross-site   → request pochodzi z całkowicie innej domeny
none         → request nawigacyjny (np. wpisany URL, bookmark)
```

```
Przykłady:
https://app.example.com/api/data
    ← request z https://app.example.com/page    → Sec-Fetch-Site: same-origin
    ← request z https://sub.example.com/page    → Sec-Fetch-Site: same-site
    ← request z https://evil.com/attack         → Sec-Fetch-Site: cross-site
    ← wpisany URL w pasek adresu                 → Sec-Fetch-Site: none
```

### Sec-Fetch-Mode

```
navigate    → request nawigacyjny (np. kliknięcie <a href>, wpisanie URL, redirect)
cors        → request z trybem CORS (fetch() z mode: 'cors')
no-cors     → request bez CORS (np. <img src>, <script src>)
same-origin → fetch() z mode: 'same-origin'
websocket   → WebSocket handshake
```

### Sec-Fetch-Dest

```
document    → pełna strona (nawigacja)
iframe      → ładowanie w iframe
script      → tag <script> lub import()
style       → arkusz CSS
image       → tag <img>
font        → czcionka (@font-face)
media       → <video> lub <audio>
fetch       → fetch() API
xhr         → XMLHttpRequest
worker      → Web Worker, Service Worker
embed       → <embed>
object      → <object>
serviceworker → rejestracja Service Worker
```

### Sec-Fetch-User

```
?1          → request jest wynikiem interakcji użytkownika (kliknięcie, submit formularza)
(brak)      → automatyczny request (skrypt, prefetch)
```

### Przykładowe requesty

```http
# Nawigacja (kliknięcie linku):
GET /page HTTP/1.1
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
Sec-Fetch-User: ?1

# Fetch API (same-origin AJAX):
GET /api/data HTTP/1.1
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: fetch

# Cross-origin image load (na malicious stronie):
GET /avatar.png HTTP/1.1
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image

# CSRF attack (form submit z evil.com):
POST /transfer HTTP/1.1
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

---

## 4. Co dzieje się wewnętrznie

### Browser-enforced, nie manipulowalne

Nagłówki `Sec-*` są w liście **forbidden request headers** przeglądarki. JavaScript nie może ich ustawić:

```javascript
// Te próby są cicho ignorowane przez przeglądarkę:
fetch('/api', {
    headers: {
        'Sec-Fetch-Site': 'same-origin',     // ZIGNOROWANE
        'Sec-Fetch-Mode': 'no-cors',          // ZIGNOROWANE
    }
});

// Przeglądarka nadpisze je własnymi wartościami automatycznie
```

Curl i narzędzia takie jak Burp Suite MOGĄ wysyłać dowolne nagłówki — ale jest to wyłącznie narzędziem testowym. Prawdziwy browser zawsze ustawia je zgodnie ze stanem.

### Resource Isolation Policy (RIP)

Najczęstszy wzorzec użycia — **Resource Isolation Policy**:

```python
def resource_isolation_policy(request):
    """
    Odrzuć cross-site requesty do endpointów które nie powinny
    być wywoływane cross-site.
    """
    # Pozwól nawigację (Sec-Fetch-Mode: navigate) do dokumentów
    if request.headers.get('Sec-Fetch-Mode') == 'navigate':
        if request.headers.get('Sec-Fetch-Dest') not in ['document', 'iframe']:
            return 403  # np. navigate do endpointu JSON — podejrzane
    
    # Pozwól requesty same-origin
    fetch_site = request.headers.get('Sec-Fetch-Site')
    if fetch_site in ('same-origin', 'same-site', 'none'):
        return None  # OK
    
    # Cross-site → zablokuj (chyba że to zasób publiczny)
    if fetch_site == 'cross-site':
        return 403

# Prosta wersja (tzw. "fetch metadata for dummies"):
def simple_policy(request):
    site = request.headers.get('Sec-Fetch-Site', '')
    if site not in ('same-origin', 'same-site', 'none', ''):
        return 403  # cross-site → odrzuć
```

### Sec-Fetch-Dest i Content-Type negocjacja

Serwer może używać `Sec-Fetch-Dest` do weryfikacji czy odpowiedź pasuje do przeznaczenia:

```python
def verify_destination(request):
    dest = request.headers.get('Sec-Fetch-Dest', '')
    
    # /api/data jest endpointem JSON — tylko fetch/xhr powinny go używać
    if request.path.startswith('/api/'):
        if dest not in ('', 'fetch', 'empty'):  # empty = fetch() z no body
            return 403  # ktoś próbuje ładować API endpoint jako script/image
```

---

## 5. Analogiczny przykład z życia

Fetch Metadata to **identyfikator pracownika z kodem dostępu** przy wejściu do biura:

- Identyfikator mówi nie tylko KTO (origin) ale CO robi (mode, dest):
  - "Pracownik z tego samego działu, idzie do stołówki" (`same-origin, navigate, document`)
  - "Osoba z zewnątrz, chce do serwerowni" (`cross-site, cors, fetch`) → ALARM!
- Identyfikatora nie można sfałszować — jest wystawiany przez system budynku (przeglądarka), nie przez samego pracownika
- Ochrona (serwer) sprawdza etykietę przed wejściem do każdego pomieszczenia (endpointu)

---

## 6. Przykład kodu

```javascript
// === Klient (przeglądarka) — Fetch Metadata jest automatyczne ===
// Nie musisz nic robić — przeglądarka doda nagłówki

// Te requesty automatycznie dostaną odpowiednie nagłówki:
fetch('/api/data');                           // Sec-Fetch-Site: same-origin
fetch('https://api.other.com/data');          // Sec-Fetch-Site: cross-site

// Fetch z credentials:
fetch('/api/auth/profile', { credentials: 'include' });
// Sec-Fetch-Site: same-origin, Sec-Fetch-Mode: cors
```

```javascript
// === Serwer Node.js: Resource Isolation Policy Middleware ===

function fetchMetadataMiddleware(req, res, next) {
    const site = req.headers['sec-fetch-site'];
    const mode = req.headers['sec-fetch-mode'];
    const dest = req.headers['sec-fetch-dest'];
    
    // Jeśli nagłówki nie istnieją → stara przeglądarka lub Curl → zdecyduj czy OK
    if (!site) {
        // Option A: Allow (backward compat)
        // Option B: Require Fetch Metadata dla API endpoints
        return next();
    }
    
    // Nawigacja do strony — zawsze OK
    if (mode === 'navigate' && dest === 'document') {
        return next();
    }
    
    // Same-origin / same-site → OK
    if (['same-origin', 'same-site', 'none'].includes(site)) {
        return next();
    }
    
    // Cross-site → sprawdź czy to publiczny endpoint
    if (site === 'cross-site') {
        // Publiczne API endpointy (CORS dozwolony)
        if (req.path.startsWith('/api/public/')) {
            return next();
        }
        // Statyczne assety (obrazki, CSS, JS) — mogą być embeddowane cross-site
        if (dest === 'image' || dest === 'style' || dest === 'script') {
            return next();
        }
        
        // Prywatne endpointy nie powinny być wywoływane cross-site
        console.warn(`[FetchMetadata] Blocked cross-site request to ${req.path}`);
        return res.status(403).json({ error: 'Cross-site requests not allowed' });
    }
    
    next();
}

// Specyficzne reguły per endpoint:
function apiAuthMiddleware(req, res, next) {
    const dest = req.headers['sec-fetch-dest'];
    
    // /api/login powinien być wywoływany tylko przez fetch/xhr (formularz AJAX)
    // nie przez nawigację lub embedding
    if (dest && !['fetch', 'empty', ''].includes(dest)) {
        return res.status(400).json({ error: 'Invalid request destination' });
    }
    
    next();
}
```

```python
# === Serwer Python/Flask: implementacja RIP ===

from functools import wraps
from flask import request, abort

def resource_isolation_policy(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        site = request.headers.get('Sec-Fetch-Site', '')
        mode = request.headers.get('Sec-Fetch-Mode', '')
        dest = request.headers.get('Sec-Fetch-Dest', '')
        
        # Brak nagłówków → stara przeglądarka lub curl
        if not site:
            return f(*args, **kwargs)
        
        # Nawigacja do dokumentu → OK
        if mode == 'navigate' and dest in ('document', ''):
            return f(*args, **kwargs)
        
        # Same-origin requests → OK
        if site in ('same-origin', 'same-site', 'none'):
            return f(*args, **kwargs)
        
        # Cross-site → zablokuj
        abort(403, description='Cross-site request blocked by Resource Isolation Policy')
    
    return decorated_function

@app.route('/api/transfer', methods=['POST'])
@resource_isolation_policy
def transfer():
    # Tylko same-origin fetch może tu dotrzeć
    return jsonify({'status': 'ok'})
```

---

## 7. Przykład z prawdziwej aplikacji

### CSRF przez form submit — blokowanie przez Fetch Metadata

Klasyczny CSRF:

```html
<!-- Na evil.com: -->
<form action="https://bank.com/transfer" method="POST">
    <input name="to" value="attacker_account">
    <input name="amount" value="1000">
    <input type="submit" value="Kliknij tutaj!">
</form>
```

Bez ochrony — bank.com wykona przelew. Z Fetch Metadata:

```
POST /transfer HTTP/1.1
Host: bank.com
Sec-Fetch-Site: cross-site       ← ALARM!
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
```

Bank.com odczyta `Sec-Fetch-Site: cross-site` + `Sec-Fetch-Mode: navigate` → form submit z obcej strony → CSRF → odrzuć (403).

### Speculative loading i prywatność

```javascript
// Atakujący chce sprawdzić czy użytkownik jest zalogowany przez embeddowanie:
const img = new Image();
img.src = 'https://bank.com/api/avatar';
img.onload = () => console.log('zalogowany');   // zdjęcie załadowane
img.onerror = () => console.log('niezalogowany'); // 403
```

Request dostanie:
```http
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: no-cors
Sec-Fetch-Dest: image
```

Serwer bankowy odrzuca `no-cors` requesty z `cross-site` → timeline attack niemożliwy.

### Spectre/timing attacks mitygacja

```python
# Odrzucaj cross-site requests do endpointów z wrażliwymi danymi:
# Uniemożliwia ataki Spectre przez cross-origin reads

@resource_isolation_policy
def get_user_data():
    # Jeśli user dane zawierają PII → nie chcemy żeby cross-site strony 
    # mogły je nawet próbować załadować (timing, error oracle, itp.)
    return jsonify(user.private_data)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Ignorowanie brakujących nagłówków

```javascript
// BŁĄD: wymóg nagłówków bez fallback
function checkFetchMetadata(req) {
    if (req.headers['sec-fetch-site'] !== 'same-origin') {
        throw new Error('Forbidden');
    }
    // PROBLEM: curl, wget, Postman — brak nagłówków → błąd dla legitymnych klientów
}

// POPRAWKA: graceful fallback
function checkFetchMetadata(req) {
    const site = req.headers['sec-fetch-site'];
    if (!site) return true; // stary klient lub non-browser → allow (lub reject, zależy od API)
    return ['same-origin', 'same-site', 'none'].includes(site);
}
```

### Błąd 2: Poleganie wyłącznie na Fetch Metadata (bez CSRF tokenów)

```javascript
// BŁĄD: zastąpienie CSRF tokenów przez Fetch Metadata
// Fetch Metadata jest defense-in-depth, nie jedyną ochroną
// Starsze przeglądarki (Safari < 16.4) nie wysyłają Fetch Metadata

// POPRAWKA: Fetch Metadata + CSRF token jako warstwa defense-in-depth
function validateRequest(req) {
    // Layer 1: Fetch Metadata (dla wspieranych przeglądarek)
    const site = req.headers['sec-fetch-site'];
    if (site && site === 'cross-site') return false;
    
    // Layer 2: CSRF token (dla wszystkich przeglądarek)
    const csrfToken = req.headers['x-csrf-token'];
    if (!validateCsrfToken(csrfToken, req.session)) return false;
    
    return true;
}
```

### Błąd 3: Nadmierne blokowanie same-site

```javascript
// BŁĄD: blokowanie same-site requestów dla strony z subdomenami
// same-site oznacza że subdomena (sub.example.com) dostaje do example.com → Sec-Fetch-Site: same-site
// To może być legalne

// Sprawdzaj cel: czy to API które nie powinno być cross-subdomain?
function checkSubdomainPolicy(req) {
    const site = req.headers['sec-fetch-site'];
    
    // Dla bardzo wrażliwych endpointów: tylko same-origin
    if (req.path.startsWith('/api/admin/')) {
        return site === 'same-origin' || site === 'none' || !site;
    }
    
    // Dla normalnych API: same-origin + same-site OK
    return ['same-origin', 'same-site', 'none'].includes(site) || !site;
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przed CSRF

Fetch Metadata jest dodatkową warstwą ochrony przed CSRF bez potrzeby tokenów sesji:

```
Atakujący (evil.com) → POST /transfer na bank.com
Request: Sec-Fetch-Site: cross-site, Sec-Fetch-Mode: navigate

Serwer widzi cross-site + navigate → form submit z zewnątrz → CSRF → odrzuć
```

Ważne: to nie zastępuje CSRF tokenów — jest **defense in depth**.

### Ochrona przed clickjacking (częściowa)

```
Atakujący embedduje bank.com w iframe:
Request: Sec-Fetch-Site: cross-site, Sec-Fetch-Mode: navigate, Sec-Fetch-Dest: iframe

Serwer odrzuca navigate + dest=iframe z cross-site → clickjacking utrudniony
(lepiej używać X-Frame-Options lub CSP frame-ancestors)
```

### Ochrona przed Spectre/cross-origin reads

```
Atakujący próbuje załadować dane przez cross-origin image load:
Sec-Fetch-Site: cross-site, Sec-Fetch-Mode: no-cors, Sec-Fetch-Dest: image

Serwer widzi cross-site no-cors image → odrzuca → brak timing oracle
```

### Ochrona przed SSRF (Server-Side Request Forgery)

```javascript
// Na serwerze aplikacyjnym: jeśli request pochodzi od klienta "proxy" przez backend:
// Backend powinien sprawdzać czy jest inicjatorem requestu

// Fetch Metadata chroni zasoby WEWNĄTRZ sieci przed embeddowaniem z zewnątrz
```

### Powiązane CWE

- **CWE-352** — Cross-Site Request Forgery (CSRF)
- **CWE-346** — Origin Validation Error
- **CWE-200** — Information Exposure

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź czy serwer waliduje Fetch Metadata

```bash
# Request bez nagłówków Fetch Metadata (jak Curl):
curl -X POST https://target.com/api/transfer \
    -H "Content-Type: application/json" \
    -d '{"to": "attacker", "amount": 1000}' \
    -b "session=STOLEN_SESSION"

# Symuluj cross-site request (Burp Suite):
# Dodaj nagłówki Sec-Fetch-Site: cross-site, Sec-Fetch-Mode: navigate
# Sprawdź czy serwer odrzuca czy akceptuje
```

### Burp Suite — modyfikacja nagłówków Fetch Metadata

W Burp Suite (i innych proxy) można ręcznie ustawiać nagłówki `Sec-Fetch-*`:

```
1. Przechwycony request w Burp
2. Zmień Sec-Fetch-Site na: cross-site
3. Zmień Sec-Fetch-Mode na: navigate
4. Forward request
5. Jeśli serwer akceptuje → brak walidacji Fetch Metadata
```

Uwaga: przeglądarka nie może tego zrobić (nagłówki `Sec-*` są forbidden) — ale Burp tak.

### Sprawdzenie implementacji po stronie serwera

```bash
# Sprawdź headers w requests (DevTools → Network → Request Headers)
# Szukaj: Sec-Fetch-Site, Sec-Fetch-Mode, Sec-Fetch-Dest, Sec-Fetch-User

# Sprawdź czy server loguje/waliduje te nagłówki (przez błędy lub różne odpowiedzi):
# Request z Sec-Fetch-Site: same-origin vs cross-site → czy odpowiedź jest inna?
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy serwer waliduje nagłówek Sec-Fetch-Site dla wrażliwych endpointów?
□ Czy requesty z Sec-Fetch-Site: cross-site są odrzucane dla prywatnych API?
□ Czy brak nagłówków Fetch Metadata (curl) jest obsługiwany gracefully?
□ Czy Fetch Metadata jest używany jako defense-in-depth (nie jedyna ochrona)?
□ Czy endpoint /api/... jest odrzucany gdy Sec-Fetch-Dest != 'fetch'?
□ Czy cross-site navigate (form submit) do API jest blokowany?
□ Czy serwer loguje próby cross-site requestów?
□ Czy Fetch Metadata działa poprawnie z CDN/reverse proxy (nie stripuje nagłówków)?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Express.js: kompletna implementacja Resource Isolation Policy ===

const FETCH_METADATA_POLICY = {
    // Endpointy publiczne (CORS API, public assets)
    public: ['/api/public/', '/static/', '/.well-known/'],
    
    // Endpointy tylko same-origin
    sameOriginOnly: ['/api/admin/', '/api/user/'],
    
    // Endpointy nawigacyjne (HTML pages)
    navigation: ['/', '/login', '/dashboard'],
};

function resourceIsolationPolicy(req, res, next) {
    const site = req.headers['sec-fetch-site'];
    const mode = req.headers['sec-fetch-mode'];
    const dest = req.headers['sec-fetch-dest'];
    
    // Brak nagłówków → stara przeglądarka, curl, itp.
    if (!site) {
        return next();
    }
    
    // Same-origin/same-site/none → OK
    if (['same-origin', 'same-site', 'none'].includes(site)) {
        return next();
    }
    
    // Cross-site request
    if (site === 'cross-site') {
        // Sprawdź czy to publiczny endpoint
        const isPublic = FETCH_METADATA_POLICY.public.some(p => req.path.startsWith(p));
        if (isPublic) return next();
        
        // Nawigacja do HTML strony z zewnątrz — OK (np. kliknięcie linku)
        if (mode === 'navigate' && dest === 'document') {
            return next();
        }
        
        // Wszystko inne cross-site → odrzuć
        res.set('Vary', 'Sec-Fetch-Site, Sec-Fetch-Mode');
        return res.status(403).json({
            error: 'Cross-site request blocked',
            code: 'FETCH_METADATA_VIOLATION'
        });
    }
    
    next();
}

app.use(resourceIsolationPolicy);
```

```python
# === Django middleware: Fetch Metadata Isolation ===

class FetchMetadataMiddleware:
    PUBLIC_PATHS = ['/api/public/', '/static/', '/media/']
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        site = request.META.get('HTTP_SEC_FETCH_SITE', '')
        mode = request.META.get('HTTP_SEC_FETCH_MODE', '')
        dest = request.META.get('HTTP_SEC_FETCH_DEST', '')
        
        if not site:
            return self.get_response(request)
        
        if site in ('same-origin', 'same-site', 'none'):
            return self.get_response(request)
        
        if site == 'cross-site':
            # Publiczne ścieżki
            if any(request.path.startswith(p) for p in self.PUBLIC_PATHS):
                return self.get_response(request)
            
            # Nawigacja do dokumentu
            if mode == 'navigate' and dest == 'document':
                return self.get_response(request)
            
            from django.http import JsonResponse
            return JsonResponse(
                {'error': 'Cross-site request blocked'},
                status=403
            )
        
        return self.get_response(request)
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Fetch Metadata = 4 nagłówki HTTP automatycznie dodawane przez przeglądarkę: `Sec-Fetch-Site`, `Sec-Fetch-Mode`, `Sec-Fetch-Dest`, `Sec-Fetch-User`
2. Nagłówki `Sec-*` są **forbidden** dla JavaScript — nie mogą być sfałszowane przez stronę (ale mogą przez Burp/curl)
3. `Sec-Fetch-Site: cross-site` dla wrażliwych endpointów → odrzuć → ochrona przed CSRF i Spectre
4. Resource Isolation Policy: pozwól `same-origin/same-site/none`, sprawdź pozostałe
5. Nie zastępuje CSRF tokenów — to **defense in depth**

**Dla pentestera:**
- Użyj Burp Suite aby ustawić `Sec-Fetch-Site: cross-site` i sprawdź czy serwer waliduje
- Sprawdź czy endpoint API akceptuje request z `Sec-Fetch-Mode: navigate` (form submit z innej strony)
- Brak walidacji Fetch Metadata = brak dodatkowej warstwy ochrony CSRF
- Porównaj response dla `same-origin` vs `cross-site` — czy serwer reaguje inaczej?

---

## Powiązania

```
Fetch Metadata Headers
    │
    ├──► CORS (Rozdział 13)
    │         CORS: kontrola dostępu do zasobów cross-origin
    │         Fetch Metadata: informacja o kontekście requestu po stronie serwera
    │
    ├──► CSRF (Rozdział 14)
    │         Fetch Metadata jako dodatkowa ochrona przed CSRF
    │         (uzupełnienie CSRF tokenów, nie zastępstwo)
    │
    ├──► COOP (Rozdział 42)
    │         Oba chronią przed cross-origin atakami
    │         COOP: izolacja browsing context; Fetch Metadata: walidacja requestów
    │
    └──► CSP (Rozdział 36)
              CSP kontroluje zasoby klienta
              Fetch Metadata kontroluje requesty po stronie serwera
```
