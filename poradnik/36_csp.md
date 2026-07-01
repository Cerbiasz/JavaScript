# Rozdział 36: Content Security Policy (CSP)

## 1. Czym jest Content Security Policy

**Content Security Policy (CSP)** to mechanizm bezpieczeństwa HTTP umożliwiający twórcom stron precyzyjną kontrolę nad tym jakie zasoby (skrypty, style, obrazy, czcionki, ramki, połączenia sieciowe) mogą być ładowane przez stronę. CSP jest fundamentalną warstwą obrony przed XSS, clickjackingiem, code injection i innymi atakami opartymi na wstrzykiwaniu treści.

CSP działa przez:
1. HTTP Header: `Content-Security-Policy: <dyrektyw>`
2. Meta tag (ograniczone możliwości): `<meta http-equiv="Content-Security-Policy" content="...">`

CSP jest **allowlist** — domyślnie wszystko zablokowane, jawnie pozwalasz na konkretne źródła.

---

## 2. Dlaczego powstał

### Problem: brak kontroli nad ładowanymi zasobami

Przed CSP (2010):
- Raz wstrzyknięty skrypt (XSS) miał pełen dostęp do wykonania dowolnego JS
- Aplikacja nie mogła kontrolować skąd ładowane są skrypty
- Clickjacking przez `<iframe>` — brak mechanizmu blokowania framowania
- Inline event handlers i `eval()` były wszechobecne

CSP (W3C Specyfikacja, implementacja Chrome 2011, v1; v2 2015; v3 trwająca) rozwiązuje:
- Whitelist zaufanych źródeł skryptów
- Blokowanie inline scripts
- Blokowanie `eval()` i podobnych
- Raportowanie naruszeń
- Blokowanie framowania (zastępuje X-Frame-Options)

---

## 3. Jak działa

### Dyrektywy CSP — przegląd

```
Dyrektywy source:
default-src    — domyślna dla wszystkich nie określonych
script-src     — skrypty JavaScript
style-src      — style CSS
img-src        — obrazy
font-src       — czcionki
connect-src    — XMLHttpRequest, Fetch, WebSocket, EventSource
media-src      — audio, video
object-src     — <object>, <embed>, <applet>
frame-src      — <iframe>
frame-ancestors — kto może framować tę stronę (anty-clickjacking)
worker-src     — Web Workers, Service Workers
manifest-src   — Web App Manifests
base-uri       — <base> tag URL
form-action    — akcja formularzy

Dyrektywy specjalne:
report-uri     — endpoint do raportowania naruszeń (deprecated → report-to)
report-to      — nowoczesne raportowanie
upgrade-insecure-requests — upgrade http: → https:
block-all-mixed-content   — blokuje mieszane treści
sandbox        — sandbox dla strony (jak atrybut sandbox na iframe)
```

### Source Expressions (wartości)

```
'none'         — nic nie jest dozwolone
'self'         — tylko same-origin
'unsafe-inline' — inline scripts/styles (NIE używaj!)
'unsafe-eval'  — eval(), Function(), setTimeout(string) (NIE używaj!)
'unsafe-hashes' — inline event handlers z hash (ograniczone)
'wasm-unsafe-eval' — WebAssembly eval
'strict-dynamic' — zaufanie przez nonce/hash propaguje na dynamicznie ładowane skrypty
https:         — dowolne HTTPS URL
https://cdn.example.com — konkretna domena
https://cdn.example.com/scripts/ — konkretna ścieżka
'nonce-BASE64' — konkretny losowy nonce dla inline script
'sha256-HASH'  — hash konkretnego inline script
```

### Przykład pełnej polityki

```http
Content-Security-Policy:
    default-src 'none';
    script-src 'self' 'nonce-RANDOM_NONCE' https://cdn.trusted.com;
    style-src 'self' https://fonts.googleapis.com;
    img-src 'self' data: https:;
    font-src 'self' https://fonts.gstatic.com;
    connect-src 'self' https://api.example.com wss://ws.example.com;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';
    report-to csp-endpoint
```

---

## 4. Co dzieje się wewnętrznie

### Inline scripts i nonces

Jednym z najpotężniejszych mechanizmów CSP v2+ jest **nonce**:

```html
<!-- Serwer generuje losowy nonce per-request: -->
<!-- CSP header: script-src 'nonce-abc123xyz' -->

<script nonce="abc123xyz">
    // Ten skrypt zostanie wykonany (nonce się zgadza)
    app.init();
</script>

<script>
    // Ten skrypt NIE zostanie wykonany (brak nonce lub zły nonce)
    evil.code();
</script>
```

Kluczowe: nonce musi być **kryptograficznie losowy** i **per-request** (nie per-page load). Każde żądanie HTTP powinno generować nowy nonce.

### Hash-based CSP

```html
<!-- Zamiast nonce, możesz użyć hash zawartości: -->
<!-- sha256 z 'console.log("Hello")' → base64 encode = ... -->

<!-- CSP: script-src 'sha256-abc123==' -->
<script>console.log("Hello")</script>
<!-- Wykonany jeśli hash się zgadza -->
```

### strict-dynamic

`'strict-dynamic'` propaguje zaufanie:
```javascript
// Jeśli skrypt z nonce/hash dynamicznie ładuje inny skrypt:
const script = document.createElement('script');
script.src = 'https://example.com/module.js';
document.head.appendChild(script);

// Bez strict-dynamic: module.js musi być w whitelist
// Z strict-dynamic: skrypt załadowany przez zaufany skrypt → też zaufany
// (ignoruje host-based whitelist!)
```

Zalecana "strict" polityka CSP:
```
script-src 'nonce-{random}' 'strict-dynamic';
object-src 'none';
base-uri 'self';
```

### Report-Only mode

Testuj CSP bez blokowania:
```http
Content-Security-Policy-Report-Only: default-src 'self'; report-to csp-reports
```
Naruszenia są raportowane ale nie blokowane. Idealne do wdrożenia.

---

## 5. Analogiczny przykład z życia

CSP to lista gości na imprezie firmowej:

- Domyślnie wstęp ZABRONIONY dla wszystkich (`default-src 'none'`)
- Pracownicy firmy mają wejście (`'self'`)
- Zaproszeni partnerzy z konkretnych firm — mają zaproszenie (`https://partner.com`)
- Każdy gość z zaproszeniem (nonce na imprezie) dostaje jednorazowy identyfikator
- Ochrona weryfikuje identyfikator przy każdym wejściu

---

## 6. Przykład kodu

```javascript
// === Generowanie nonce w Node.js/Express ===
const crypto = require('crypto');

app.use((req, res, next) => {
    // Generuj nowy nonce dla każdego request
    res.locals.nonce = crypto.randomBytes(16).toString('base64');
    
    res.setHeader('Content-Security-Policy', [
        `default-src 'none'`,
        `script-src 'self' 'nonce-${res.locals.nonce}' 'strict-dynamic'`,
        `style-src 'self' 'nonce-${res.locals.nonce}'`,
        `img-src 'self' data: https:`,
        `font-src 'self'`,
        `connect-src 'self' https://api.example.com`,
        `frame-ancestors 'none'`,
        `base-uri 'self'`,
        `form-action 'self'`,
        `object-src 'none'`
    ].join('; '));
    
    next();
});

// W szablonie (Handlebars/EJS):
// <script nonce="{{nonce}}">...</script>
```

```javascript
// === CSP Reporting endpoint ===
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
    const report = req.body['csp-report'];
    
    // Loguj naruszenia
    console.log('CSP Violation:', {
        blockedUri: report['blocked-uri'],
        violatedDirective: report['violated-directive'],
        documentUri: report['document-uri'],
        referrer: report['referrer']
    });
    
    // Alert dla krytycznych naruszeń (script-src)
    if (report['violated-directive']?.startsWith('script-src')) {
        alertSecurityTeam(report);
    }
    
    res.status(204).send();
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Bypassy CSP — JSONP

```javascript
// Podatna polityka: script-src 'self' https://api.example.com
// Problem: api.example.com ma JSONP endpoint!

// Atakujący może wstrzyknąć:
<script src="https://api.example.com/data?callback=alert"></script>
// → serwer zwraca: alert({"data": "..."})
// → przeglądarka wykonuje alert() !
// → CSP pozwoliło (api.example.com w whitelist)
```

### Bypassy CSP — Angular/Vue template injection

```javascript
// Jeśli CSP ma 'unsafe-eval' LUB pozwala na framework z własnym sandbox escape:
// AngularJS < 1.6 umożliwiał escape sandbox przez template injection:

// Podatne: {{'a'.constructor.prototype.charAt=[].join;$eval('x=1} } };alert(1)//');}}
// Jeśli CSP nie blokuje Angular (np. przez cdn.jsdelivr.net) → bypass!
```

### Bypassy przez open redirect

```javascript
// CSP: script-src 'self' https://accounts.google.com
// Problem: Google OAuth ma open redirect:
// https://accounts.google.com/AccountChooser?continue=https://evil.com/malicious.js

// Wstrzyknięty tag:
<script src="https://accounts.google.com/AccountChooser?continue=https://evil.com/malicious.js">
// CSP pozwala (accounts.google.com w whitelist)
// Serwer Google redirectuje do evil.com/malicious.js
// Przeglądarka: request do evil.com → ale CSP blokuje! (evil.com nie w whitelist)
// ALE: jeśli redirect jest po stronie klienta lub CORS... edge cases exist
```

### `base-uri` bypass

```html
<!-- Podatna polityka: brak base-uri dyrektywy -->
<!-- Atakujący wstrzykuje: -->
<base href="https://evil.com/">
<!-- Teraz wszystkie relatywne URL (skrypty, obrazy) wskazują na evil.com! -->
<!-- <script src="app.js"> → https://evil.com/app.js ← MALICIOUS! -->

<!-- OBRONA: base-uri 'self' lub base-uri 'none' -->
```

---

## 8. Typowe błędy programistów

### Błąd 1: `unsafe-inline` niweluje CSP

```http
# BŁĄD: 'unsafe-inline' pozwala na wszystkie inline scripts → XSS nadal możliwy!
Content-Security-Policy: script-src 'self' 'unsafe-inline'

# Nonce i strict-dynamic ZAMIAST unsafe-inline:
Content-Security-Policy: script-src 'nonce-xxx' 'strict-dynamic'
```

### Błąd 2: Zbyt szerokie whitelist domeny

```http
# BŁĄD: '*' lub szerokie domeny
Content-Security-Policy: script-src https: *

# Problem: każde HTTPS URL można załadować → atakujący kontroluje externe JS
# POPRAWKA: konkretne domeny lub nonce
```

### Błąd 3: Brak frame-ancestors (clickjacking)

```http
# BŁĄD: brak frame-ancestors → aplikacja może być osadzona w iframe przez kogokolwiek
# POPRAWKA:
Content-Security-Policy: ...; frame-ancestors 'none'
# lub:
Content-Security-Policy: ...; frame-ancestors 'self' https://trusted-parent.com
```

### Błąd 4: `default-src *` jako "domyślne"

```http
# BŁĄD: zezwala na wszystko jako fallback
Content-Security-Policy: default-src *; script-src 'self'
# Inne dyrektywy (img-src, style-src) fallback do * → permissive!

# POPRAWKA: restrictive default
Content-Security-Policy: default-src 'none'; script-src 'self'; img-src 'self'
```

---

## 9. Znaczenie dla bezpieczeństwa

### CSP Level 1 vs 2 vs 3

| Feature | CSP1 | CSP2 | CSP3 |
|---------|------|------|------|
| Nonces | ✗ | ✓ | ✓ |
| Hashes | ✗ | ✓ | ✓ |
| `strict-dynamic` | ✗ | ✗ | ✓ |
| `worker-src` | ✗ | ✓ | ✓ |
| `navigate-to` | ✗ | ✗ | ✓ (experimental) |

### Google's "strict" CSP

Zalecana przez Google research:
```
Content-Security-Policy:
    script-src 'nonce-{random}' 'strict-dynamic' https: 'unsafe-inline';
    object-src 'none';
    base-uri 'self';
    report-uri /csp-report
```

`https:` i `'unsafe-inline'` są ignorowane przez przeglądarki wspierające nonce i strict-dynamic — ale fallback dla starszych.

### Raportowanie CSP naruszeń

CSP naruszenia są cennym źródłem informacji:
- Próby XSS (blocked-uri = dane wstrzykiętego skryptu)
- Misconfigured zasoby (blocked-uri = CDN nie w whitelist)
- Ataki na użytkowników (blocked-uri = external evil URL)

```javascript
// Analiza raportu CSP:
{
    "csp-report": {
        "document-uri": "https://app.com/dashboard",
        "referrer": "https://app.com/",
        "violated-directive": "script-src-elem",
        "effective-directive": "script-src-elem",
        "original-policy": "script-src 'self'; ...",
        "blocked-uri": "inline",      // lub URL zablokowanego zasobu
        "line-number": 45,
        "column-number": 3,
        "source-file": "https://app.com/dashboard",
        "status-code": 200
    }
}
```

### CSP Evaluators

- **Google CSP Evaluator**: `csp-evaluator.withgoogle.com`
- Analizuje politykę i wskazuje słabości

### Powiązane CWE

- **CWE-79** — XSS (CSP jako mitygacja)
- **CWE-1021** — Improper Restriction of Rendered UI Layers (clickjacking, frame-ancestors)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź politykę CSP

```bash
curl -I https://target.com | grep -i "content-security-policy"
```

### Ocena jakości CSP

Szukaj:
1. **`'unsafe-inline'`** — CSP jest słaby dla XSS
2. **`'unsafe-eval'`** — eval() jest dozwolone
3. **Szerokie wildcards** (`*`, `https:`) — zbyt permissive
4. **Brak `default-src`** — nie wszystkie zasoby są kontrolowane
5. **Brak `frame-ancestors`** — clickjacking możliwy
6. **Brak `base-uri`** — base tag injection możliwy
7. **JSONP endpoints w whitelist** — bypass przez JSONP
8. **Angular CDN w whitelist bez hash** — potencjalny bypass

### Narzędzia

```bash
# CSP Evaluator Google (API):
curl "https://csp-evaluator.withgoogle.com/getCSPEvaluation?csp=ENCODED_CSP"

# Zachowaj politykę i wklej do csp-evaluator.withgoogle.com
```

### DevTools → Console

```
CSP violations są widoczne w konsoli jako błędy:
"Refused to execute inline script because it violates the following Content Security Policy directive..."
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy CSP header jest obecny?
□ Czy polityka zawiera 'unsafe-inline' (słaby CSP)?
□ Czy polityka zawiera 'unsafe-eval'?
□ Czy są szerokie whitelist (https:, *)?
□ Czy frame-ancestors jest skonfigurowane (clickjacking)?
□ Czy base-uri jest zablokowane?
□ Czy form-action jest ograniczone?
□ Czy object-src jest 'none'?
□ Czy CSP-Report-Only jest używane do testowania?
□ Czy whitelist domeny mają JSONP lub angular endpoints?
□ Czy nonces są losowe i per-request?
□ Czy strict-dynamic jest używane?
□ Czy raportowanie CSP jest skonfigurowane?
```

---

## 12. Jak się zabezpieczać

```http
# Minimalna skuteczna polityka (nowoczesna):
Content-Security-Policy:
    default-src 'none';
    script-src 'nonce-{per-request-random}' 'strict-dynamic';
    style-src 'self' 'nonce-{per-request-random}';
    img-src 'self' data: https:;
    font-src 'self';
    connect-src 'self' https://api.example.com;
    frame-src 'none';
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';
    object-src 'none';
    report-to csp-endpoint

# Report-To configuration:
Report-To: {"group":"csp-endpoint","max_age":10886400,"endpoints":[{"url":"https://app.com/csp-report"}]}
```

```javascript
// Wdrożenie krok po kroku:
// 1. Zacznij od Report-Only
// 2. Zbierz raporty przez tydzień
// 3. Dodaj do whitelist brakujące zasoby
// 4. Przełącz na enforcement (Content-Security-Policy)
// 5. Monitor reports w produkcji
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. CSP = HTTP header definiujący whitelist dozwolonych źródeł zasobów i funkcji
2. `'unsafe-inline'` i `'unsafe-eval'` neutralizują ochronę XSS — unikaj
3. Nonce + `strict-dynamic` = nowoczesna, silna ochrona
4. `frame-ancestors 'none'` = ochrona przed clickjackingiem
5. `base-uri 'self'` i `object-src 'none'` — zawsze ustawiaj

**Dla pentestera:**
- Sprawdź obecność i jakość CSP header
- `'unsafe-inline'` = XSS nadal możliwy
- JSONP endpoints w whitelist = potencjalny CSP bypass
- Google CSP Evaluator dla szybkiej oceny
- Brak `frame-ancestors` = clickjacking możliwy

---

## Powiązania

```
CSP
    │
    ├──► Trusted Types (Rozdział 37)
    │         require-trusted-types-for 'script' → Trusted Types enforcement
    │
    ├──► Subresource Integrity (Rozdział 38)
    │         SRI + CSP = silna ochrona zasobów
    │
    ├──► Permissions Policy (Rozdział 34)
    │         PP: API features; CSP: zasoby
    │
    ├──► DOM/XSS (Rozdział 9)
    │         CSP jako główna mitygacja XSS
    │
    └──► Web Workers (Rozdział 20)
              worker-src dyrektywa CSP
```
