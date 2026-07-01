# Rozdział 12: XMLHttpRequest

## 1. Czym jest XMLHttpRequest

**XMLHttpRequest** (XHR) to API JavaScript do wykonywania asynchronicznych żądań HTTP bez przeładowania strony. Mimo nazwy obsługuje nie tylko XML — obsługuje wszystkie typy danych: JSON, tekst, binarnie, FormData.

XHR jest Web API (nie ECMAScript). Istnieje od 1999 roku (Microsoft w IE5 jako ActiveXObject, potem standaryzowany przez Mozilla), jest dostępny we wszystkich przeglądarkach i był podstawą "AJAX" (Asynchronous JavaScript and XML) przez ponad dekadę.

Choć Fetch API jest nowoczesnym zamiennikiem, XHR nadal dominuje w legacy code, starszych bibliotekach (jQuery's `$.ajax()` używał XHR) i specyficznych przypadkach (np. monitorowanie postępu uploadu pliku).

### API

```javascript
const xhr = new XMLHttpRequest();
xhr.open(method, url, async, user, password);
xhr.send(body);

// Właściwości
xhr.readyState;      // 0-4: UNSENT, OPENED, HEADERS_RECEIVED, LOADING, DONE
xhr.status;          // HTTP status code
xhr.statusText;      // "OK", "Not Found"...
xhr.responseText;    // odpowiedź jako string
xhr.responseXML;     // odpowiedź jako Document (XML/HTML)
xhr.response;        // odpowiedź wg responseType
xhr.responseType;    // ""|"arraybuffer"|"blob"|"document"|"json"|"text"
xhr.responseURL;     // finalny URL (po redirect)
xhr.timeout;         // timeout w ms
xhr.withCredentials; // odpowiednik credentials: "include" w Fetch

// Event handlery
xhr.onreadystatechange = function() { ... }; // każda zmiana readyState
xhr.onload = function() { ... };             // sukces
xhr.onerror = function() { ... };            // błąd sieci
xhr.ontimeout = function() { ... };          // timeout
xhr.onprogress = function(e) { ... };        // postęp pobierania
xhr.upload.onprogress = function(e) { ... }; // postęp wysyłania

// Nagłówki
xhr.setRequestHeader(name, value);
xhr.getResponseHeader(name);
xhr.getAllResponseHeaders(); // string z wszystkimi nagłówkami
```

---

## 2. Dlaczego powstał

### Problem: przeładowanie strony przy każdej interakcji

W 1999 roku każda interakcja ze stroną (kliknięcie linku, submit formularza) wymagała pełnego przeładowania. Outlook Web Access (Microsoft, 1996-1999) był pierwszym poważnym webapem — Microsoft stworzył XMLHTTP (ActiveXObject) właśnie po to, żeby budować dynamiczne interfejsy.

W 2004 Google Maps i Gmail spopularyzowały AJAX — były rewolucją w tym że działały jak aplikacje desktopowe, nie strony HTML. Cały świat zobaczył możliwości XHR.

### Problem: standartyzacja

ActiveXObject.XMLHTTP był microsoftowym wynalazkiem. Mozilla przeportowała go do Firefox 1.0 (2004) jako natywny `XMLHttpRequest`. W3C standaryzował XMLHttpRequest Level 1 w 2006, Level 2 (z CORS, binary data, progress events) w 2012.

---

## 3. Jak działa

### readyState Machine

XHR jest state machine'ą przechodzącą przez 5 stanów:

```
UNSENT (0)
    │ → xhr.open() wywołane
    ▼
OPENED (1)
    │ → xhr.send() wywołane
    ▼
HEADERS_RECEIVED (2)
    │ → nagłówki odpowiedzi odebrane
    ▼
LOADING (3)
    │ → body jest pobierane (onprogress events)
    ▼
DONE (4)
    │ → transfer zakończony (sukces lub błąd)
```

```javascript
xhr.onreadystatechange = function() {
    if (xhr.readyState === 2) {
        console.log("Nagłówki:", xhr.getAllResponseHeaders());
    }
    if (xhr.readyState === 3) {
        console.log("Pobieranie...", xhr.responseText.length, "bajtów");
    }
    if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            processResponse(xhr.responseText);
        } else {
            handleError(xhr.status);
        }
    }
};
```

### Krok po kroku

```
xhr.open("GET", "/api/data", true)  ← true = asynchroniczny (default)
        │
        ▼ readyState = 1 (OPENED)
        │
xhr.setRequestHeader("Authorization", "Bearer " + token)
        │
xhr.send()
        │
        ▼ przeglądarka wysyła request
        │ JavaScript kontynuuje (asynchroniczny)
        │
        ▼ [odpowiedź serwera]
        │ readyState = 2 (HEADERS_RECEIVED) → onreadystatechange
        │ readyState = 3 (LOADING) → onreadystatechange + onprogress
        │ readyState = 4 (DONE) → onreadystatechange + onload
```

### Synchroniczny XHR (deprecated)

```javascript
// xhr.open(method, url, async=false) → synchroniczny — BARDZO ZŁE!
xhr.open("GET", "/api/data", false); // blokuje główny wątek!
xhr.send();
// Wszystko stoi w miejscu dopóki serwer nie odpowie
// UI zamrożone → złe UX + potencjalny DoS
// Deprecated i usunięty w wielu kontekstach (nie w main thread)
```

### withCredentials — odpowiednik credentials:include

```javascript
const xhr = new XMLHttpRequest();
xhr.withCredentials = true; // wyślij cookies, auth cert cross-origin
xhr.open("GET", "https://api.example.com/data");
xhr.send();
// Wymaga Access-Control-Allow-Credentials: true po stronie serwera
// I Access-Control-Allow-Origin nie może być "*"
```

---

## 4. Co dzieje się wewnętrznie

### Relacja XHR a Fetch API

Oba używają tej samej warstwy sieciowej przeglądarki (network layer). CORS, cookies, SameSite — te same reguły stosują się do obu.

### Progress events — unikalność XHR

Fetch API nie ma natywnego monitorowania postępu uploadu (od 2023 jest eksperymentalne przez streaming). XHR od zawsze miał:

```javascript
xhr.upload.onprogress = function(e) {
    if (e.lengthComputable) {
        const percent = (e.loaded / e.total) * 100;
        updateProgressBar(percent);
    }
};

xhr.onprogress = function(e) { // pobieranie
    if (e.lengthComputable) {
        const percent = (e.loaded / e.total) * 100;
        updateDownloadBar(percent);
    }
};
```

### Nagłówki — ograniczenia bezpieczeństwa

Przeglądarka nie pozwoli ustawić pewnych nagłówków przez `setRequestHeader()`:

```
ZAKAZANE (przeglądarka je ignoruje lub rzuca błąd):
- Host
- Content-Length
- Transfer-Encoding
- Trailer
- Cookie
- Cookie2
- Referer
- Origin
- Accept-Charset
- Accept-Encoding
- TE
- DNT
- Connection
- Upgrade
- Via
- Headers zaczynające od "Proxy-" lub "Sec-"
```

To ograniczenie chroni przed request smuggling i innymi atakami na warstwie HTTP.

---

## 5. Analogiczny przykład z życia

XHR to stary model telegraficzny:

- **Telegram** (request) = wysyłasz wiadomość i czekasz na potwierdzenie
- **Punkt 0: gotowe** — telegram przygotowany (UNSENT)
- **Punkt 1: wysłany** — telegram przyjęty przez telegraf (OPENED/SEND)
- **Punkt 2: potwierdzenie** — odbiorca potwierdził odbiór (HEADERS_RECEIVED)
- **Punkt 3: treść płynie** — telegram jest przekazywany (LOADING)
- **Punkt 4: komplet** — cały telegram dostarczony (DONE)

Każdy krok można monitorować. Można też anulować telegram przed dostarczeniem (`xhr.abort()`).

---

## 6. Przykład kodu

```javascript
// Kompletny przykład XHR z obsługą błędów

function makeXHRRequest(url, options = {}) {
    return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest();
        
        // Otwórz połączenie (async = true domyślnie)
        xhr.open(
            options.method || "GET",
            url,
            true // async
        );
        
        // Ustaw nagłówki
        if (options.headers) {
            Object.entries(options.headers).forEach(([key, value]) => {
                xhr.setRequestHeader(key, value);
            });
        }
        
        // Cookies cross-origin (jak credentials: "include" w Fetch)
        xhr.withCredentials = options.withCredentials || false;
        
        // Timeout
        xhr.timeout = options.timeout || 30000;
        
        // Typ odpowiedzi
        xhr.responseType = options.responseType || "json";
        
        // Handler sukcesu (readyState 4)
        xhr.onload = function() {
            if (xhr.status >= 200 && xhr.status < 300) {
                resolve(xhr.response); // responseType: "json" auto-parsuje
            } else {
                reject(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
            }
        };
        
        // Handler błędu sieci
        xhr.onerror = function() {
            reject(new Error("Network error"));
        };
        
        // Handler timeout
        xhr.ontimeout = function() {
            reject(new Error("Request timed out"));
        };
        
        // Monitoring postępu uploadu
        if (options.onUploadProgress) {
            xhr.upload.onprogress = options.onUploadProgress;
        }
        
        // Wyślij
        xhr.send(options.body || null);
        
        // Zwróć metodę abort (dla anulowania)
        Promise.abort = () => xhr.abort();
    });
}

// Użycie:
try {
    const data = await makeXHRRequest("/api/users", {
        method: "GET",
        headers: { "Authorization": "Bearer token123" },
        timeout: 10000
    });
    console.log(data);
} catch (e) {
    console.error(e.message);
}
```

---

## 7. Przykład z prawdziwej aplikacji

### jQuery $.ajax() — wewnętrzna implementacja

jQuery przez lata opakowywało XHR:

```javascript
// Jak jQuery $.ajax() używa XHR pod spodem (uproszczone)
$.ajax({
    url: "/api/users",
    method: "GET",
    headers: { "X-Requested-With": "XMLHttpRequest" }, // marker AJAX
    success: function(data) { renderUsers(data); },
    error: function(xhr, status, error) { showError(error); }
});

// Nagłówek X-Requested-With: XMLHttpRequest
// Jest używany przez serwery do wykrywania requestów AJAX
// Ale NIE jest mechanizmem CSRF! Można go łatwo dodać w Fetch/XHR przez atakującego
// Jest ograniczony przez CORS (niestandardowy nagłówek → preflight) — to jest faktyczna ochrona
```

### Formularz uploadu pliku z progress

```javascript
const fileInput = document.getElementById("file");
const progressBar = document.getElementById("progress");

fileInput.addEventListener("change", async function() {
    const file = this.files[0];
    const formData = new FormData();
    formData.append("file", file);
    formData.append("name", file.name);
    
    const xhr = new XMLHttpRequest();
    
    xhr.upload.onprogress = function(e) {
        if (e.lengthComputable) {
            progressBar.value = (e.loaded / e.total) * 100;
        }
    };
    
    xhr.onload = function() {
        if (xhr.status === 200) {
            const result = JSON.parse(xhr.responseText);
            showSuccess(result.fileUrl);
        }
    };
    
    xhr.open("POST", "/api/upload");
    xhr.setRequestHeader("X-CSRF-Token", getCsrfToken());
    xhr.withCredentials = true;
    xhr.send(formData);
    // Uwaga: nie ustawiaj Content-Type! Browser ustawi go automatycznie
    // z poprawnym boundary dla multipart/form-data
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Ufanie X-Requested-With jako ochrona CSRF

```javascript
// Backend (Node.js) — błędna ochrona CSRF
app.post("/api/transfer", (req, res) => {
    if (req.headers["x-requested-with"] !== "XMLHttpRequest") {
        return res.status(403).json({ error: "Forbidden" });
    }
    // Atakujący może dodać ten nagłówek przez Fetch!
    // fetch(url, { headers: { "X-Requested-With": "XMLHttpRequest" } })
    // X-Requested-With to niestandardowy nagłówek → CORS preflight
    // Ale jeśli serwer pozwala na ten nagłówek w CORS...
});
```

### Błąd 2: Synchroniczny XHR

```javascript
// BARDZO ZŁE:
xhr.open("GET", url, false); // synchroniczny
xhr.send();
// Blokuje UI, deprecated, niedostępny w Service Workers i Web Workers
```

### Błąd 3: Parsowanie JSON bez try/catch

```javascript
// BŁĄD: serwer może zwrócić nie-JSON (np. error page w HTML)
xhr.onload = function() {
    const data = JSON.parse(xhr.responseText); // TypeError jeśli nie JSON!
    processData(data);
};

// POPRAWKA:
xhr.responseType = "json"; // automatyczne parsowanie
xhr.onload = function() {
    if (xhr.response) { // null jeśli parsowanie nie powiodło się
        processData(xhr.response);
    }
};
```

### Błąd 4: Brak walidacji status kodu

```javascript
// BŁĄD: onload wywołuje się też dla 4xx/5xx
xhr.onload = function() {
    processData(JSON.parse(xhr.responseText)); // może to być błąd!
};

// POPRAWKA:
xhr.onload = function() {
    if (xhr.status >= 200 && xhr.status < 300) {
        processData(xhr.response);
    } else {
        handleError(xhr.status, xhr.response);
    }
};
```

---

## 9. Znaczenie dla bezpieczeństwa

### XHR i CSRF

Historycznie XHR z `setRequestHeader("Content-Type", "application/json")` wymagał preflight OPTIONS (bo JSON nie jest "simple" content type). To utrudniało CSRF. Ale:

- Proste GET/POST z form-urlencoded/text mogą być cross-origin bez preflight
- `withCredentials: true` wymaga CORS po stronie serwera

### getAllResponseHeaders() — information disclosure

```javascript
xhr.onload = function() {
    const headers = xhr.getAllResponseHeaders();
    console.log(headers);
    // Może ujawniać: Server: Apache/2.4.41, X-Powered-By: PHP/7.4.3
    // Stack fingerprinting!
};
```

### XHR abort i timing attacks

Czas między `send()` a `onload` może ujawniać informacje o przetwarzaniu serwera:

```javascript
const start = Date.now();
xhr.onload = () => console.log(Date.now() - start + "ms");
```

### Zakazane nagłówki — BYPASS przez Flash/Silverlight (historycznie)

Stare pluginy Flash/Silverlight mogły ustawiać zakazane nagłówki przez XHR (CSRF przez Flash). To był powód wieloletnich ataków. Flash jest wycofany (2020).

### Powiązane CWE i zagrożenia

- **CWE-352** — CSRF (XHR bez CSRF token)
- **CWE-918** — SSRF (backend używa XHR/Fetch na URL od klienta)
- **CWE-200** — Information Exposure (nagłówki stack)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Network

Identycznie jak dla Fetch:
1. **Network** → filtr **XHR** (lub **Fetch/XHR**)
2. Sprawdź każdy request: nagłówki, body, response
3. Sprawdź `getAllResponseHeaders()` w `xhr.getResponseHeader()` — information disclosure

### Wykrywanie użycia XHR vs Fetch

```javascript
// W DevTools Console — sprawdź co używa aplikacja
// Monkey-patch oba:
const originalXHR = XMLHttpRequest.prototype.send;
XMLHttpRequest.prototype.send = function(...args) {
    console.log("XHR:", this._url || "unknown");
    return originalXHR.apply(this, args);
};

const originalOpen = XMLHttpRequest.prototype.open;
XMLHttpRequest.prototype.open = function(method, url, ...args) {
    this._url = url;
    console.log("XHR open:", method, url);
    return originalOpen.apply(this, [method, url, ...args]);
};
```

### Burp Suite — identyfikacja XHR

Requesty przez XHR zazwyczaj mają nagłówek:
```
X-Requested-With: XMLHttpRequest
```

Szukaj tego nagłówka w Burp → Proxy → HTTP History.

---

## 11. Jak testować bezpieczeństwo

```
□ Czy synchroniczny XHR (async=false) jest używany → blokada Event Loop?
□ Czy withCredentials: true jest używane cross-origin bez potrzeby?
□ Czy X-Requested-With jest jedyną ochroną CSRF (niewystarczające!)?
□ Czy getAllResponseHeaders() ujawnia wrażliwe nagłówki (Server, X-Powered-By)?
□ Czy odpowiedzi błędów XHR ujawniają stack traces lub wewnętrzne ścieżki?
□ Czy JSON.parse(xhr.responseText) jest używane bez try/catch?
□ Czy backend waliduje Content-Type response (nie tylko sprawdza XHR header)?
□ Czy upload endpoint pozwala na dowolne typy plików (brak MIME type validation)?
□ Czy timeout jest skonfigurowany (xhr.timeout)?
□ Czy XHR requesty z wrażliwymi danymi używają HTTPS?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: CSRF przy błędnej ochronie przez X-Requested-With

**Wymagania:** Serwer sprawdza tylko obecność `X-Requested-With: XMLHttpRequest`.

**Przebieg:**
```javascript
// evil.com — atakujący
fetch("https://victim.com/api/transfer", {
    method: "POST",
    headers: {
        "X-Requested-With": "XMLHttpRequest",  // łatwe do dodania!
        "Content-Type": "application/json"
    },
    credentials: "include",  // wysyła cookies ofiary
    body: JSON.stringify({ to: "attacker", amount: 10000 })
});
// Serwer widzi X-Requested-With → pozwala → CSRF!
```

### Scenariusz 2: Finger printing przez Response Headers

**Przebieg:**
```javascript
// Pentester w DevTools Console:
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/endpoint");
xhr.onload = () => {
    console.log("=== Server Info ===");
    console.log(xhr.getAllResponseHeaders());
    // Server: Apache/2.4.51 (Ubuntu)
    // X-Powered-By: PHP/8.0.12
    // X-AspNet-Version: 4.0.30319
};
xhr.send();
```

---

## 13. Jak się zabezpieczać

### Prawidłowa CSRF ochrona (nie X-Requested-With)

```javascript
// Prawidłowe podejście — CSRF token w niestandardowym nagłówku
function makeSecureXHR(url, data) {
    const xhr = new XMLHttpRequest();
    xhr.open("POST", url);
    
    // CSRF token z ciasteczka (podwójne ciasteczko wzorzec)
    // lub z meta tagu (lepsze)
    const csrfToken = document.querySelector('[name=csrf-token]').content;
    xhr.setRequestHeader("X-CSRF-Token", csrfToken);
    xhr.setRequestHeader("Content-Type", "application/json");
    
    xhr.send(JSON.stringify(data));
}
```

### Ukrywanie nagłówków serwera

```apache
# Apache — ukryj wersję
ServerTokens Prod
ServerSignature Off

# Nginx
server_tokens off;

# Express.js
app.disable('x-powered-by');
// lub z helmet:
const helmet = require('helmet');
app.use(helmet.hidePoweredBy());
```

### Migraj do Fetch API

```javascript
// Zamiast XHR — użyj Fetch z AbortController
async function fetchWithTimeout(url, options, timeoutMs = 10000) {
    const controller = new AbortController();
    const id = setTimeout(() => controller.abort(), timeoutMs);
    
    try {
        const response = await fetch(url, {
            ...options,
            signal: controller.signal
        });
        clearTimeout(id);
        return response;
    } catch (e) {
        clearTimeout(id);
        throw e;
    }
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. XHR to pierwotne AJAX API — używane od 1999, zdominowało web do ~2015
2. readyState machine: UNSENT→OPENED→HEADERS_RECEIVED→LOADING→DONE
3. `withCredentials = true` = odpowiednik `credentials: "include"` w Fetch
4. Synchroniczny XHR (`async=false`) jest deprecated i blokuje UI
5. X-Requested-With NIE jest mechanizmem CSRF protection — tylko wskazówką

**Najczęstsze nieporozumienia:**

- "XHR jest bezpieczniejszy niż Fetch" — mają identyczny model bezpieczeństwa (CORS, cookies)
- "X-Requested-With chroni przed CSRF" — NIE. Atakujący może go dodać przez Fetch/XHR. Ochrona pochodzi z CORS preflightu dla niestandardowych nagłówków.
- "XMLHttpRequest obsługuje tylko XML" — NIE. Obsługuje JSON, text, binary, FormData.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **Network** → filtr `XHR` → wszystkie stare AJAX requesty
2. Szukaj nagłówka `X-Requested-With: XMLHttpRequest` w requestach
3. **Sources** → szukaj `new XMLHttpRequest()`, `$.ajax(`, `$.get(`, `$.post(`
4. **Burp** → sprawdź czy CSRF token jest obecny w każdym modyfikującym stan requeście

---

## Powiązania

```
XMLHttpRequest
        │
        ├──► Fetch API (Rozdział 11)
        │         Fetch to nowoczesna alternatywa dla XHR
        │
        ├──► CORS (Rozdział 13)
        │         XHR podlega tym samym regułom CORS co Fetch
        │
        ├──► Cookies (Rozdział 14)
        │         withCredentials kontroluje wysyłanie cookies
        │
        └──► WebSocket (Rozdział 25)
                  WebSocket to alternatywa dla polling przez XHR
```
