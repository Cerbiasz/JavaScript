# Rozdział 30: URL API

## 1. Czym jest URL API

**URL API** to zestaw interfejsów JavaScript do parsowania, konstruowania i manipulowania URL (Uniform Resource Locators). Zapewnia standaryzowany, obiektowy dostęp do składowych URL bez potrzeby ręcznego parsowania stringów.

URL API składa się z:
- **`URL` class** — parsowanie i manipulacja URL
- **`URLSearchParams` class** — manipulacja query stringiem
- **`URL.createObjectURL()`** — tworzenie blob URL
- **`URL.revokeObjectURL()`** — zwalnianie blob URL

Przed URL API jedynym sposobem parsowania URL był DOM hack z `<a>` element:
```javascript
// Stary sposób (hack):
const a = document.createElement('a');
a.href = url;
a.hostname; // parsowany hostname
```

---

## 2. Dlaczego powstał

### Problem: brak standardowego parsera URL

Parsowanie URL przez manualne operacje na stringach jest podatne na błędy:
```javascript
// Manualne parsowanie — łatwo popełnić błąd
const url = 'https://user:pass@example.com:8080/path?query=value#hash';
const [protocol, rest] = url.split('://');
// ... kilkadziesiąt linii error-prone parsing
```

URL API (WHATWG URL Standard, 2016) dostarcza kompletny, standaryzowany parser zgodny ze specyfikacją używaną przez przeglądarki. Ważne: **URL API implementuje tę samą specyfikację parsowania co przeglądarki** — co oznacza że obsługuje IDN, percent-encoding, normalizację itd.

---

## 3. Jak działa

### URL class — parsowanie

```javascript
const url = new URL('https://user:pass@example.com:8080/path/to/page?q=hello&lang=pl#section');

url.href;         // "https://user:pass@example.com:8080/path/to/page?q=hello&lang=pl#section"
url.protocol;     // "https:" (z dwukropkiem!)
url.username;     // "user"
url.password;     // "pass"
url.hostname;     // "example.com"
url.port;         // "8080" (pusty string jeśli domyślny)
url.host;         // "example.com:8080" (hostname + port)
url.origin;       // "https://example.com:8080"
url.pathname;     // "/path/to/page"
url.search;       // "?q=hello&lang=pl" (z ?)
url.searchParams; // URLSearchParams object
url.hash;         // "#section" (z #)
```

### Relatywne URL

```javascript
// Drugi argument = base URL
const url = new URL('/api/users', 'https://example.com');
// → https://example.com/api/users

const url2 = new URL('../other', 'https://example.com/path/to/');
// → https://example.com/path/other

// Przydatne w kontekście Service Worker i relatywnych URL:
const base = self.registration.scope;
new URL('assets/image.png', base); // absolutny URL
```

### URLSearchParams

```javascript
const params = new URLSearchParams('q=hello&lang=pl&page=1');

// Odczyt
params.get('q');        // "hello"
params.getAll('lang');  // ["pl"] (tablica dla multipli)
params.has('page');     // true
params.keys();          // iterator: 'q', 'lang', 'page'
params.entries();       // iterator: ['q','hello'], ['lang','pl'], ...

// Modyfikacja
params.set('page', '2');    // nadpisz
params.append('tag', 'js'); // dodaj (może być multiple)
params.delete('lang');      // usuń

// Konwersja
params.toString(); // "q=hello&page=2&tag=js"

// Tworzenie z obiektu
const p2 = new URLSearchParams({ q: 'test', limit: '10' });

// Iteracja
for (const [key, value] of params) {
    console.log(key, '=', value);
}
```

### Modyfikacja URL

Właściwości URL są **settable**:

```javascript
const url = new URL('https://example.com/path?q=hello');

url.pathname = '/other/path';
url.searchParams.set('q', 'world');
url.hash = '#section2';

url.href; // "https://example.com/other/path?q=world#section2"
```

---

## 4. Co dzieje się wewnętrznie

### WHATWG URL Standard

URL API implementuje **WHATWG URL Standard** (nie RFC 3986!). Różnice:
- Bardziej liberalne parsowanie (toleruje niektóre błędy)
- Obsługa IDN (Internationalized Domain Names)
- Normalizacja (lowercase scheme i host, etc.)
- Obsługa `blob:`, `data:`, `javascript:` URLs

### Normalizacja

```javascript
new URL('HTTPS://EXAMPLE.COM/PATH').href;
// → "https://example.com/PATH" (scheme i host lowercase, path zachowany)

new URL('https://example.com/%41pple').pathname;
// → "/Apple" (percent-decoded gdy bezpieczne)
```

### Walidacja

`new URL()` rzuca `TypeError` dla niepoprawnych URL:
```javascript
try {
    new URL('not-a-url');
} catch (e) {
    e.name; // "TypeError"
    e.message; // "Failed to construct 'URL': Invalid URL"
}
```

---

## 5. Analogiczny przykład z życia

URL API to recepcjonista rozpisujący adres na składniki:

- Dostajesz adres: `https://user@bank.com:443/login?return=/dashboard#form`
- Recepcjonista (URL API) rozkłada go na: protokół (https), użytkownik (user), domena (bank.com), port (443), ścieżka (/login), parametry (?return=...), kotwica (#form)
- Możesz zapytać o każdy składnik osobno lub zmodyfikować i złożyć z powrotem

---

## 6. Przykład kodu

```javascript
// === Bezpieczna walidacja i parsowanie URL ===

function parseURL(rawUrl, baseUrl = window.location.href) {
    try {
        return new URL(rawUrl, baseUrl);
    } catch {
        return null;
    }
}

// Sprawdź czy URL jest same-origin
function isSameOrigin(url) {
    const parsed = parseURL(url);
    return parsed?.origin === window.location.origin;
}

// Bezpieczny redirect — tylko same-origin
function safeRedirect(returnUrl) {
    const parsed = parseURL(returnUrl);
    
    if (!parsed) return '/dashboard';
    if (parsed.origin !== location.origin) return '/dashboard';
    if (parsed.protocol !== 'https:' && parsed.protocol !== 'http:') return '/dashboard';
    
    return parsed.pathname + parsed.search + parsed.hash;
}

// Manipulacja query params
function addQueryParam(url, key, value) {
    const parsed = new URL(url);
    parsed.searchParams.set(key, value);
    return parsed.toString();
}

// Walidacja callback URL dla OAuth
function validateCallbackUrl(url, allowedOrigins) {
    const parsed = parseURL(url);
    if (!parsed) return false;
    if (!allowedOrigins.includes(parsed.origin)) return false;
    if (!['https:', 'http:'].includes(parsed.protocol)) return false;
    return true;
}

// Parsowanie query string bez URL API (stary sposób — BŁĘDNY)
// function parseQuery(qs) {
//     return Object.fromEntries(qs.slice(1).split('&').map(p => p.split('=')));
// }
// PROBLEM: brak dekodowania (%20 → ' '), brak obsługi +, duplikatów itd.

// POPRAWNY sposób:
function parseQuery(qs) {
    return Object.fromEntries(new URLSearchParams(qs));
}
```

---

## 7. Przykład z prawdziwej aplikacji

### URL confusion attacks

Różnice w parsowaniu URL między serwerem a klientem prowadzą do podatności:

```javascript
// Aplikacja blokuje ścieżki /admin/* przez middleware
// Ale parsuje URL inaczej niż przeglądarka

// Atakujący wysyła:
fetch('/admin%2fconfig')  // %2f = /
// Serwer (Python/PHP) widzi: /admin/config → blokuje
// Serwer (niektóre frameworki) widzi: /admin%2fconfig → nie blokuje!

// Albo: podwójne encoding
fetch('/admin%252fconfig')  // %25 = %, więc %252f = %2f = /
// Niektóre serwery podwójnie decodują → /admin/config

// URL API prawidłowo parsuje:
new URL('/admin%2fconfig', 'https://example.com').pathname;
// → "/admin/config" (zdekodowane)
```

### @-sign confusion

Cecha URL API często używana w atakach:

```javascript
// Atakujący wysyła do ofiary link:
const url = 'https://bank.com@evil.com/phishing';
new URL(url).hostname; // "evil.com"
// "bank.com" to username, "evil.com" to faktyczny host!

new URL(url).username; // "bank.com"
new URL(url).host;     // "evil.com"

// Ofiara widzi "bank.com" w linku → kliknięcie → evil.com
```

### javascript: URL

```javascript
// javascript: URL to wektor XSS
new URL('javascript:alert(1)').protocol; // "javascript:"
new URL('javascript:alert(1)').href;     // "javascript:alert(1)"

// NIEBEZPIECZNE użycie:
const userUrl = getUserInputUrl();
window.location.href = userUrl; // jeśli userUrl = 'javascript:...' → XSS!

// POPRAWKA: sprawdź protokół
const parsed = new URL(userUrl, location.href);
if (!['https:', 'http:'].includes(parsed.protocol)) {
    console.error('Niedozwolony protokół:', parsed.protocol);
    return;
}
window.location.href = parsed.href;
```

---

## 8. Typowe błędy programistów

### Błąd 1: Ręczne parsowanie URL

```javascript
// BŁĄD: manualne parsowanie przez split
const parts = url.split('/');
const hostname = url.split('/')[2]; // błędne dla http://user@host:port/path

// POPRAWKA: użyj URL API
const { hostname } = new URL(url);
```

### Błąd 2: Brak obsługi TypeError z new URL()

```javascript
// BŁĄD: URL może rzucić TypeError
const parsed = new URL(userInput); // TypeError jeśli niepoprawny URL!

// POPRAWKA:
let parsed;
try {
    parsed = new URL(userInput, location.origin); // base dla relatywnych
} catch {
    return null; // niepoprawny URL
}
```

### Błąd 3: Ignorowanie @-sign w URL

```javascript
// BŁĄD: sprawdzasz czy URL zaczyna się od 'https://bank.com'
if (url.startsWith('https://bank.com')) {
    redirect(url); // PODATNE! 'https://bank.com@evil.com' też przejdzie!
}

// POPRAWKA: użyj URL API i sprawdź hostname
const parsed = new URL(url);
if (parsed.origin === 'https://bank.com') {
    redirect(url); // sprawdza faktyczny origin
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### URL Parsing Differentials

**URL Parsing Differential** to sytuacja gdy serwer i klient parsują ten sam URL inaczej. To prowadzi do:

- **SSRF** — serwer wysyła request do innego hosta niż oczekiwał
- **ACL Bypass** — blokada `/admin` ominięta przez `/ADMIN` lub `/admin/../admin`
- **Cache Poisoning** — proxy i aplikacja różnie parsują URL

```javascript
// Przykład parsing differential:
// Python's urllib.parse:
urllib.parse.urlparse('http://foo@evil.com:80@bar.com/') 
// netloc: 'foo@evil.com:80@bar.com'
// hostname: 'bar.com'

// WHATWG URL (przeglądarka):
new URL('http://foo@evil.com:80@bar.com/').hostname
// → 'bar.com' (taki sam!)

// ALE: starsza wersja Ruby:
# URI.parse('http://foo@evil.com:80@bar.com/').host
# → 'evil.com' (RÓŻNE!)
```

### Open Redirect przez URL manipulation

```javascript
// Podatna aplikacja:
const returnUrl = new URLSearchParams(location.search).get('return');
location.href = returnUrl; // OPEN REDIRECT!

// Atakujący:
// /login?return=https://evil.com/phishing

// POPRAWKA:
const returnUrl = new URLSearchParams(location.search).get('return');
const parsed = (() => { try { return new URL(returnUrl, location.origin); } catch { return null; } })();
if (parsed?.origin !== location.origin) {
    location.href = '/dashboard';
} else {
    location.href = parsed.pathname;
}
```

### javascript: i data: URL injection

```javascript
// Jeśli user-input URL trafia do href, src, action itd.:
link.href = userInput; // XSS przez javascript: lub data:

// Bezpieczne — tylko http/https:
function setHref(element, url) {
    const parsed = (() => { try { return new URL(url, location.origin); } catch { return null; } })();
    if (!parsed || !['http:', 'https:'].includes(parsed.protocol)) {
        element.href = '#'; // fallback
        return;
    }
    element.href = parsed.href;
}
```

### URL normalizacja i WAF bypass

```javascript
// WAF blokuje /etc/passwd
// Atakujący próbuje:
fetch('/etc//passwd')      // double slash
fetch('/etc/./passwd')     // dot segment
fetch('/etc/%70asswd')     // percent encoded 'p'
fetch('/etc/p%61sswd')     // encoded 'a'

// URL API normalizuje:
new URL('/etc//passwd', 'https://example.com').pathname; // "/etc//passwd" (nie normalizuje double slash w ścieżce!)
// ale serwer może normalizować → bypass WAF który nie normalizuje
```

### Powiązane CWE

- **CWE-601** — URL Redirection to Untrusted Site (Open Redirect)
- **CWE-20** — Improper Input Validation (URL injection)
- **CWE-79** — XSS (javascript: / data: URL)
- **CWE-918** — SSRF (URL parsing differential)

---

## 10. Jak identyfikować podczas pentestu

### Szukaj wektorów URL injection

```javascript
// Miejsca gdzie URL params trafiają do lokacji/src/href:
location.href = params.get('return');
location.replace(params.get('redirect'));
element.src = params.get('url');
fetch(params.get('endpoint'));
window.open(params.get('target'));

// Testuj:
?return=javascript:alert(1)
?return=data:text/html,<script>alert(1)</script>
?return=//evil.com
?return=https://evil.com
?return=\evil.com (backslash normalization)
```

### URL parsing tricks do testowania

```javascript
// W konsoli przeglądarki:
new URL('https://evil.com@legitimate.com').hostname; // legitimate.com
new URL('https://legitimate.com%40evil.com').hostname; // evil.com (lub SyntaxError)

// Testuj różne encodingi:
new URL('/path%2ftraversal').pathname;
new URL('/PATH').pathname; // normalizacja case?
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy URL ze schowka, params, storage jest walidowany przed użyciem w location.href?
□ Czy javascript:/data: protokoły są blokowane?
□ Czy @-sign confusion jest możliwa (link z bank.com@evil.com)?
□ Czy server-side URL parsing różni się od client-side (parsing differential)?
□ Czy open redirect jest możliwy przez query params?
□ Czy URL encoding bypass (double-encoding, partial encoding) działa?
□ Czy fetch/XMLHttpRequest akceptuje dowolny URL z user input (SSRF)?
```

---

## 12. Jak się zabezpieczać

```javascript
// Biblioteka pomocnicza do bezpiecznego URL handling:

const UrlUtils = {
    parse(url, base = location.origin) {
        try { return new URL(url, base); }
        catch { return null; }
    },
    
    isSameOrigin(url) {
        return this.parse(url)?.origin === location.origin;
    },
    
    isSafeProtocol(url) {
        const p = this.parse(url)?.protocol;
        return p === 'https:' || p === 'http:';
    },
    
    // Bezpieczny redirect tylko same-origin
    safeRedirect(url, fallback = '/') {
        const parsed = this.parse(url);
        if (!parsed || parsed.origin !== location.origin) {
            return fallback;
        }
        return parsed.pathname + parsed.search + parsed.hash;
    },
    
    // Bezpieczny zewnętrzny link (http/https only)
    safeExternalUrl(url) {
        const parsed = this.parse(url);
        if (!parsed || !['https:', 'http:'].includes(parsed.protocol)) return null;
        return parsed.href;
    }
};
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. URL API = standaryzowany parser URL zgodny ze specyfikacją WHATWG
2. `new URL()` rzuca TypeError dla niepoprawnych URL — zawsze try/catch
3. `URLSearchParams` do bezpiecznego parsowania query string
4. @-sign confusion: `bank.com@evil.com` → hostname = `evil.com`
5. `javascript:` i `data:` protokoły → XSS przez URL injection

**Dla pentestera:**
- Testuj `?return=javascript:alert(1)`, `?return=//evil.com`
- Sprawdź @-sign confusion w linkach na stronie
- URL parsing differentials serwer/klient → SSRF lub ACL bypass
- `new URL(userInput).protocol` w konsoli do szybkiej weryfikacji

---

## Powiązania

```
URL API
    │
    ├──► History API (Rozdział 29)
    │         pushState(url) — URL walidowany przez URL API
    │
    ├──► Fetch API (Rozdział 11)
    │         fetch(url) — URL parsowany przez URL API
    │
    ├──► CORS (Rozdział 13)
    │         Origin = protocol + hostname + port (z URL API)
    │
    └──► File API (Rozdział 27)
              URL.createObjectURL() → blob: URL
```
