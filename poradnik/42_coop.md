# Rozdział 42: COOP (Cross-Origin-Opener-Policy)

## 1. Czym jest COOP

**Cross-Origin-Opener-Policy (COOP)** to nagłówek HTTP odpowiedzi, który kontroluje jak dokument współdzieli **browsing context group** z innymi dokumentami otwartymi przez `window.open()` lub linki `target="_blank"`. COOP izoluje dokument w oddzielonym, samodzielnym browsing context — uniemożliwiając cross-origin stronom uzyskanie referencji `window` do dokumentu i odwrotnie.

Główne wartości:

| Wartość | Zachowanie |
|---------|-----------|
| `unsafe-none` | Domyślne — brak izolacji; cross-origin `window.opener` jest dostępne |
| `same-origin-allow-popups` | Izoluje od cross-origin; pozwala dokumentowi otwierać popupy cross-origin |
| `same-origin` | Pełna izolacja — tylko same-origin dokumenty mogą być w tym samym browsing context group |

```http
Cross-Origin-Opener-Policy: same-origin
```

COOP jest kluczowy dla włączenia **cross-origin isolation** — co z kolei odblokowuje dostęp do `SharedArrayBuffer` i wysokiej rozdzielczości timera `performance.now()`, wymaganych przez np. WebAssembly SIMD czy wielowątkowe operacje.

---

## 2. Dlaczego powstał

### Problem: Spectre i cross-origin leaks przez SharedArrayBuffer

W 2018 ujawniono podatności **Spectre** i **Meltdown** — ataki side-channel oparte na CPU cache timing. W kontekście przeglądarki:

```javascript
// Spectre attack przez SharedArrayBuffer:
// Atakujący i ofiara w tym samym process → Spectre może czytać pamięć ofiary

const sharedBuffer = new SharedArrayBuffer(1024);
// Dzięki SharedArrayBuffer i wysokiej rozdzielczości timera:
// Atakujący może mierzyć timing dostępu do pamięci
// I przez cache side-channel odczytać dane ofiary z pamięci procesu
```

Przeglądarki **dezaktywowały** `SharedArrayBuffer` i zmniejszyły dokładność `performance.now()` globalnie (2018).

Aby je bezpiecznie przywrócić — potrzebna była **cross-origin isolation**: gwarancja że dokumenty z różnych originów NIE są w tym samym procesie OS. COOP + COEP (Rozdział 43) razem tworzą cross-origin isolation.

### Problem: window.opener atak

```javascript
// Na evil.com:
const victim = window.open('https://shop.com');
// victim.location.href = 'https://phishing.com'; // możliwe jeśli brak COOP!

// Na shop.com (bez COOP):
window.opener // referencja do evil.com! → evil.com może manipulować shop.com
window.opener.location = 'https://phishing.com'; // redirect evil.com
```

COOP rozwiązuje oba problemy przez izolację browsing context group.

---

## 3. Jak działa

### same-origin (pełna izolacja)

```http
Cross-Origin-Opener-Policy: same-origin
```

```javascript
// Na shop.com z COOP: same-origin:

// Jeśli evil.com otwiera shop.com przez window.open():
const shopWin = window.open('https://shop.com');
shopWin // null lub closed window — brak dostępu!
shopWin.document // SecurityError: nie możesz uzyskać dostępu

// Na shop.com:
window.opener // null — shop.com nie widzi kto go otworzył!
```

### same-origin-allow-popups

```http
Cross-Origin-Opener-Policy: same-origin-allow-popups
```

```javascript
// Na shop.com z same-origin-allow-popups:
// shop.com może otwierać cross-origin popupy (np. OAuth popup):
const oauthPopup = window.open('https://accounts.google.com/oauth');
// shop.com może komunikować się z tym popupem (jeśli popup na to pozwala)

// ALE: cross-origin strony które otwierają shop.com → nie mają dostępu do shop.com window
```

### unsafe-none (domyślne)

```http
Cross-Origin-Opener-Policy: unsafe-none
# lub brak nagłówka
```

```javascript
// Klasyczne zachowanie — wszystkie otwarte strony w tym samym browsing context group
// Możliwość window.opener ataków
```

### Cross-origin isolation status

```javascript
// Sprawdź czy cross-origin isolation jest aktywne:
crossOriginIsolated // true jeśli COOP=same-origin + COEP=require-corp

// Jeśli crossOriginIsolated === true:
new SharedArrayBuffer(1024);           // dozwolone!
performance.now();                     // wysoka rozdzielczość!
Atomics.wait(sharedBuffer, 0, 0, 100); // działa!
```

---

## 4. Co dzieje się wewnętrznie

### Browsing Context Group

Przeglądarka grupuje dokumenty w **Browsing Context Groups** — dokumenty w tej samej grupie mogą:
- Mieć referencje `window.opener` do siebie
- Synchronicznie komunikować się przez `postMessage`
- Potencjalnie być w tym samym procesie OS (site isolation)

```
Bez COOP:
┌─────────────────────────────────────┐
│ Browsing Context Group              │
│ ┌──────────┐ opener ┌────────────┐  │
│ │ evil.com │ ←────→ │ shop.com   │  │
│ └──────────┘        └────────────┘  │
│                     window.opener   │
│                     dostępne!       │
└─────────────────────────────────────┘

Z COOP: same-origin na shop.com:
┌──────────────┐  ┌──────────────────┐
│ BCG 1        │  │ BCG 2            │
│ ┌──────────┐ │  │ ┌────────────┐   │
│ │ evil.com │ │  │ │ shop.com   │   │
│ └──────────┘ │  │ └────────────┘   │
└──────────────┘  └──────────────────┘
    Brak window.opener między grupami!
```

### Process isolation (Site Isolation)

Z COOP + COEP (cross-origin isolation), przeglądarka może umieścić dokumenty w oddzielnych procesach OS:

```
Z cross-origin isolation:
Process A: evil.com
Process B: shop.com (COOP: same-origin, COEP: require-corp)
    → Spectre nie może przeskoczyć między procesami!
    → SharedArrayBuffer bezpieczny
```

### COOP-Report-Only

```http
# Testuj wpływ COOP bez faktycznego blokowania:
Cross-Origin-Opener-Policy-Report-Only: same-origin; report-to="coop-endpoint"
```

### Raportowanie naruszeń

```javascript
// Konfiguracja report endpoint:
// Nagłówek Report-To:
Report-To: {"group":"coop-endpoint","max_age":86400,"endpoints":[{"url":"https://example.com/reports"}]}
```

---

## 5. Analogiczny przykład z życia

COOP to **odrębne wejścia do biurowców**:

- Bez COOP: jeden budynek wspólny — każdy kto wchodzi przez te same drzwi (window.open) może widzieć innych lokatorów i chodzić po korytarzach (window.opener)
- Z COOP=same-origin: osobne budynki z oddzielnymi wejściami — tylko firmy z tej samej grupy (same-origin) są w jednym budynku
- cross-origin firma otwierająca drzwi do twojego budynku przez `window.open` dostaje osobne wejście do oddzielnego budynku — nie ma dostępu do twoich pomieszczeń
- SharedArrayBuffer to superszybka winda wewnątrz budynku — może działać tylko jeśli budynek jest bezpieczny (cross-origin isolated)

---

## 6. Przykład kodu

```javascript
// === Sprawdzenie stanu izolacji ===

// Czy cross-origin isolation jest aktywna?
console.log('Cross-origin isolated:', crossOriginIsolated);

// Wymagania dla crossOriginIsolated = true:
// 1. COOP: same-origin (na dokumencie)
// 2. COEP: require-corp (na dokumencie i wszystkich zasobach)

// Tylko gdy crossOriginIsolated === true:
if (crossOriginIsolated) {
    const buffer = new SharedArrayBuffer(1024 * 1024); // 1MB
    const view = new Int32Array(buffer);
    
    // Wysoka rozdzielczość czasu:
    const t0 = performance.now();
    // ... operacja ...
    const t1 = performance.now();
    console.log(`Czas: ${t1 - t0}ms`); // sub-millisecond precision
    
    // Worker z SharedArrayBuffer:
    const worker = new Worker('worker.js');
    worker.postMessage({ buffer }, [buffer]); // transfer
} else {
    console.warn('SharedArrayBuffer niedostępny bez cross-origin isolation');
}
```

```javascript
// === window.opener — sprawdzenie dostępu ===

// Na stronie bez COOP:
window.addEventListener('load', () => {
    if (window.opener) {
        console.log('Strona otwarta przez:', window.opener.location?.href);
        // Potencjalny tab-napping!
    }
    
    // Ochrona przed opener attacks bez COOP:
    window.opener = null; // Wyczyść referencję (nie zawsze skuteczne)
});

// Na stronie z COOP: same-origin:
// window.opener === null automatycznie dla cross-origin openerów
```

```javascript
// === Bezpieczne otwieranie linków — rel="noopener" jako fallback ===

// Bez COOP na serwerze, rel="noopener" na linkach:
// <a href="https://external.com" target="_blank" rel="noopener noreferrer">Link</a>

// Z JavaScript:
function safeOpen(url) {
    const popup = window.open(url, '_blank', 'noopener,noreferrer');
    // noopener: popup nie ma dostępu do window.opener
    // noreferrer: implicitly adds noopener
    return popup;
}
```

```http
# === Kompletna konfiguracja COOP + raportowanie ===

Cross-Origin-Opener-Policy: same-origin; report-to="coop"
Report-To: {"group":"coop","max_age":86400,"endpoints":[{"url":"https://app.example.com/csp-report"}]}
```

---

## 7. Przykład z prawdziwej aplikacji

### OAuth popup z COOP

Typowy problem: OAuth wymaga popupu cross-origin (np. Google login).

```javascript
// Problem: COOP: same-origin blokuje komunikację między parentem a OAuth popup

// Strona główna:
const oauthPopup = window.open(
    'https://accounts.google.com/oauth?client_id=...&redirect_uri=...',
    'oauth',
    'width=500,height=600'
);

// Bez COOP: oauthPopup === null jeśli Google nie ma COOP? Nie —
// Problem jest odwrotny: jeśli NASZA strona ma COOP=same-origin,
// a OAuth popup jest cross-origin → popup nie ma dostępu do naszego window
// → popup nie może wywołać window.opener.handleOAuth(token)!
```

Rozwiązanie:

```javascript
// Opcja 1: same-origin-allow-popups (pozwala naszej stronie otwierać cross-origin popup):
// Cross-Origin-Opener-Policy: same-origin-allow-popups

// Opcja 2: Użyj postMessage zamiast window.opener:
// Popup po auth robi:
window.opener.postMessage({ type: 'oauth_success', token: TOKEN }, 'https://oursite.com');
// Parent nasłuchuje:
window.addEventListener('message', (e) => {
    if (e.origin === 'https://oauth-provider.com' && e.data.type === 'oauth_success') {
        handleAuthToken(e.data.token);
    }
});
```

### Tab-napping bez COOP

Klasyczny tab-napping attack:

```html
<!-- Na evil.com, link: -->
<a href="https://evil.com/fake-bank" target="_blank">Przeczytaj artykuł o bezpieczeństwie</a>
```

```javascript
// Na evil.com/fake-bank (otwarty w nowej karcie):
window.opener.location.href = 'https://phishing-bank.com';
// Przekieruj oryginalną kartę użytkownika na phishing!
```

Z COOP na evil.com → shop.com: shop.com ma `window.opener = null` → atak niemożliwy.

### Spectre simulation

```javascript
// Atak Spectre wymaga SharedArrayBuffer i wysokiej rozdzielczości timera:
// Bez cross-origin isolation → SharedArrayBuffer niedostępny

// Z cross-origin isolation:
// crossOriginIsolated = true → przeglądarka gwarantuje process isolation
// → Spectre nie może czytać danych cross-origin

// Sprawdzenie:
if (!crossOriginIsolated) {
    console.error('NIEBEZPIECZNE: brak izolacji! SharedArrayBuffer może umożliwić Spectre!');
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Wdrożenie COOP bez COEP

```http
# BŁĄD: samo COOP nie aktywuje crossOriginIsolated
Cross-Origin-Opener-Policy: same-origin
# Brak: Cross-Origin-Embedder-Policy: require-corp

# crossOriginIsolated === false! SharedArrayBuffer nadal niedostępny!

# POPRAWKA: oba nagłówki razem:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

### Błąd 2: Pomylenie COOP z CORS

```javascript
// BŁĄD: myślenie że COOP kontroluje CORS
// COOP: kontroluje browsing context group i window.opener
// CORS: kontroluje dostęp do zasobów cross-origin przez fetch/XHR

// COOP nie wpływa na CORS nagłówki!
// CORS nie wpływa na window.opener!
```

### Błąd 3: Zbyt wczesne włączenie same-origin (brak testowania)

```http
# BŁĄD: wdrożenie na prod bez testowania
Cross-Origin-Opener-Policy: same-origin

# Konsekwencje:
# - OAuth popupy mogą przestać działać
# - window.open() cross-origin zwraca null-like window
# - Zewnętrzne widget integracje mogą się psuć

# POPRAWKA: Use Report-Only first!
Cross-Origin-Opener-Policy-Report-Only: same-origin; report-to="coop"
# Zbierz raporty → zidentyfikuj problemy → wdróż COOP
```

### Błąd 4: Poleganie na rel="noopener" zamiast COOP

```html
<!-- CZĘŚCIOWA ochrona: rel="noopener" chroni ten konkretny link -->
<a href="..." rel="noopener">Link</a>

<!-- Ale nie chroni jeśli ktoś inny otwiera TWOJĄ stronę przez window.open() -->
<!-- COOP: same-origin chroni całą stronę — bez względu na to jak jest otwierana -->
```

---

## 9. Znaczenie dla bezpieczeństwa

### Tab-napping ochrona

```
Bez COOP:
evil.com otwiera shop.com → shop.com ma window.opener = evil.com
evil.com może: window.open('https://shop.com').opener → dostęp! (odwrotnie)
lub shop.com może: window.opener → referencja do evil.com → redirect evil.com

Z COOP: same-origin:
evil.com otwiera shop.com → RÓŻNE browsing context groups
window.opener === null na shop.com
Brak cross-reference!
```

### SharedArrayBuffer i Spectre

```
SharedArrayBuffer wymagany dla:
- WebAssembly threads
- Emscripten multi-threaded apps (np. gry, video processing)
- AudioWorklet z shared memory
- Biblioteki kryptograficzne (np. sodium.js z WASM)

Bez crossOriginIsolated (COOP+COEP): SharedArrayBuffer = undefined
Z crossOriginIsolated: SharedArrayBuffer działa bezpiecznie
```

### Information leakage przez timing

```javascript
// Bez crossOriginIsolated: performance.now() zaokrąglone do 100µs lub więcej
// Uniemożliwia Spectre timing attacks ale też utrudnia profiling

// Z crossOriginIsolated: performance.now() z pełną rozdzielczością
// Bezpieczne bo process isolation uniemożliwia Spectre
```

### Powiązane CWE

- **CWE-346** — Origin Validation Error
- **CWE-1021** — Improper Restriction of Rendered UI Layers or Frames (tab-napping powiązane)
- **CWE-209** — Information Exposure Through an Error Message (Spectre powiązane)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź nagłówki HTTP

```bash
# Sprawdź COOP:
curl -I https://target.com | grep -i "cross-origin-opener-policy"

# Brak = unsafe-none (domyślne) = potencjalny tab-napping risk
# same-origin = pełna izolacja
# same-origin-allow-popups = częściowa izolacja

# Sprawdź też COEP (wymóg dla crossOriginIsolated):
curl -I https://target.com | grep -i "cross-origin-embedder-policy"
```

### Testowanie tab-napping

```javascript
// W konsoli przeglądarki — otwórz docelową stronę i sprawdź:
window.open('https://target.com', 'target');
// Sprawdź: czy target.window.opener jest dostępne?
// (z DevTools na target.com): window.opener !== null?

// Lub otwórz target.com i sprawdź:
// Jeśli window.opener !== null → podatny na tab-napping
```

### Sprawdź czy crossOriginIsolated jest wymagane

```javascript
// Na target.com w konsoli:
console.log('crossOriginIsolated:', crossOriginIsolated);
// false = brak COOP+COEP = SharedArrayBuffer niedostępny (lub unsafe)
// true = bezpieczna izolacja
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy COOP header jest ustawiony?
□ Czy wartość to same-origin (nie unsafe-none)?
□ Czy COEP jest ustawiony razem z COOP dla crossOriginIsolated?
□ Czy window.opener jest null dla cross-origin stron?
□ Czy OAuth popupy działają poprawnie z COOP?
□ Czy COOP-Report-Only było używane przed włączeniem?
□ Czy raporty COOP są zbierane i analizowane?
□ Czy rel="noopener" jest na linkach target="_blank" (defense in depth)?
□ Czy crossOriginIsolated jest true gdy wymagany SharedArrayBuffer?
```

---

## 12. Jak się zabezpieczać

```http
# Zalecana konfiguracja dla aplikacji wymagającej cross-origin isolation:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp

# Dla aplikacji z OAuth popupami:
Cross-Origin-Opener-Policy: same-origin-allow-popups
Cross-Origin-Embedder-Policy: require-corp

# Raportowanie (deploy etap):
Cross-Origin-Opener-Policy-Report-Only: same-origin; report-to="coop-reports"
Report-To: {"group":"coop-reports","max_age":86400,"endpoints":[{"url":"/reports/coop"}]}
```

```javascript
// === Bezpieczne linki bez COOP na serwerze ===

// Jeśli nie możesz wdrożyć COOP: zabezpiecz linki
// W React/HTML:
function ExternalLink({ href, children }) {
    return (
        <a href={href} target="_blank" rel="noopener noreferrer">
            {children}
        </a>
    );
}

// W JavaScript:
function openSafe(url) {
    const win = window.open();
    win.opener = null;  // Wyczyść przed nawigacją
    win.location = url;
}
```

```javascript
// === Migracja krok po kroku ===

// Krok 1: Zbierz raporty (Report-Only)
// Krok 2: Zidentyfikuj cross-origin integracje z problemami
// Krok 3: Napraw OAuth przez postMessage zamiast window.opener
// Krok 4: Dodaj COEP nagłówki
// Krok 5: Włącz COOP w trybie enforcing
// Krok 6: Weryfikacja: crossOriginIsolated === true
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. COOP = `Cross-Origin-Opener-Policy` header — izoluje browsing context group
2. `same-origin` = pełna izolacja, `window.opener = null` dla cross-origin
3. `same-origin-allow-popups` = izolacja ale NASZA strona może otwierać cross-origin popupy
4. COOP + COEP = `crossOriginIsolated = true` → `SharedArrayBuffer` i Spectre protection
5. Chroni przed tab-napping (`window.opener` attacks)

**Dla pentestera:**
- Brak `Cross-Origin-Opener-Policy` = potencjalny tab-napping
- Test: otwórz stronę przez `window.open()` i sprawdź `window.opener` na target
- Sprawdź `crossOriginIsolated` — czy wymagana izolacja jest wdrożona?
- COOP + COEP razem = cross-origin isolation; samo COOP ≠ crossOriginIsolated

---

## Powiązania

```
COOP (Cross-Origin-Opener-Policy)
    │
    ├──► COEP (Rozdział 43)
    │         COOP + COEP = crossOriginIsolated
    │         Oba wymagane dla SharedArrayBuffer
    │
    ├──► CORP (Rozdział 44)
    │         CORP: kontrola embeddowania zasobu
    │         Wymagany przez COEP (require-corp)
    │
    ├──► Web Workers (Rozdział 20)
    │         SharedArrayBuffer używany przez Workers
    │         Wymaga crossOriginIsolated
    │
    └──► postMessage (Rozdział 23)
              Alternatywna komunikacja zamiast window.opener
              Używana gdy COOP blokuje bezpośredni dostęp
```
