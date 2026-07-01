# Rozdział 43: COEP (Cross-Origin-Embedder-Policy)

## 1. Czym jest COEP

**Cross-Origin-Embedder-Policy (COEP)** to nagłówek HTTP odpowiedzi, który wymusza aby wszystkie zasoby ładowane przez dokument posiadały jawne zezwolenie na cross-origin embedding — poprzez nagłówki CORS lub CORP (Cross-Origin-Resource-Policy). COEP gwarantuje że dokumenty mogą wczytywać tylko zasoby które expressis verbis zgodziły się na bycie załadowanymi cross-origin.

Dwie główne wartości:

| Wartość | Zachowanie |
|---------|-----------|
| `unsafe-none` | Domyślne — żadnych ograniczeń na ładowane zasoby |
| `require-corp` | Każdy cross-origin zasób musi mieć CORS lub CORP nagłówek |
| `credentialless` | Cross-origin zasoby ładowane bez credentials (cookies, auth) |

```http
Cross-Origin-Embedder-Policy: require-corp
```

COEP + COOP = **cross-origin isolation**: stan w którym `crossOriginIsolated === true`, odblokowujący `SharedArrayBuffer` i precyzyjny timer, konieczne dla bezpieczeństwa wielowątkowego kodu.

---

## 2. Dlaczego powstał

### Problem: Spectre i subresource timing attacks

Po ujawnieniu Spectre (2018), przeglądarki wyłączyły `SharedArrayBuffer` i zmniejszyły precyzję timerów — bo umożliwiały timing side-channel attacks. Aby je bezpiecznie przywrócić, potrzeba było gwarancji **process isolation**.

Sama izolacja browsing context group (COOP) nie wystarczy — atakujący mógłby nadal ładować zasób cross-origin przez `<img>` lub `<script>` i mieć go w tym samym procesie:

```html
<!-- Na target.com z COOP: same-origin, ale bez COEP: -->
<img src="https://secret-api.com/data.png?token=SECRET">
<!-- Jeśli secret-api.com jest w tym samym procesie → Spectre może czytać! -->
```

COEP rozwiązuje to: wymusza że KAŻDY cross-origin zasób musi mieć zgodę (CORS/CORP). Jeśli zasób nie zgadza się na embedding → nie jest ładowany. Eliminuje niezamierzone cross-origin resources w procesie.

---

## 3. Jak działa

### require-corp

```http
Cross-Origin-Embedder-Policy: require-corp
```

Gdy COEP = `require-corp` jest aktywne, każdy cross-origin zasób ładowany przez dokument musi mieć:

**Opcja A: CORS nagłówek:**
```http
# Zasób (np. CDN): 
Access-Control-Allow-Origin: https://app.example.com
# lub:
Access-Control-Allow-Origin: *
```

**Opcja B: CORP nagłówek:**
```http
# Zasób który nie chce CORS ale zgadza się na embedding:
Cross-Origin-Resource-Policy: cross-origin
# lub:
Cross-Origin-Resource-Policy: same-site
```

Jeśli zasób nie ma żadnego z powyższych → **przeglądarka blokuje załadowanie**.

### credentialless

```http
Cross-Origin-Embedder-Policy: credentialless
```

Nowsza wartość (Chrome 96+): cross-origin zasoby są ładowane ale **bez cookies, certyfikatów i autoryzacji**. Eliminuje potrzebę CORS/CORP na każdym zasobie — kosztem braku credentials.

```javascript
// Z credentialless: <img src="https://cdn.com/image.png"> ładuje się
// Ale bez cookies/auth → anonimowy request
// Przydatne dla publicznych assets na CDN
```

### Hierarchia zasobów

COEP = `require-corp` dotyczy WSZYSTKICH zasobów:
- `<img>`, `<video>`, `<audio>`
- `<script src="...">` (zewnętrzne)
- `<link rel="stylesheet">`
- `<iframe src="...">`
- Zasoby przez `fetch()`, `XMLHttpRequest`

---

## 4. Co dzieje się wewnętrznie

### Weryfikacja przy ładowaniu zasobu

```
Przeglądarka ładuje zasób cross-origin dla dokumentu z COEP:require-corp:

1. Sprawdź czy zasób ma CORS header (Access-Control-Allow-Origin)?
   → TAK → ładuj zasób ✓

2. Sprawdź czy zasób ma CORP header (Cross-Origin-Resource-Policy)?
   → TAK (same-origin/same-site/cross-origin) → ładuj zasób ✓

3. Brak CORS i brak CORP?
   → ZABLOKUJ → Console error: "...has been blocked by COEP policy"
```

### COEP w iframach

Iframe wewnątrz dokumentu z COEP musi też mieć COEP:

```html
<!-- Dokument z COEP: require-corp chce załadować iframe: -->
<iframe src="https://partner.example.com/widget"></iframe>

<!-- partner.example.com musi też wysyłać COEP: require-corp! -->
<!-- Bez tego: iframe jest blokowany lub degradowany -->

<!-- Lub: attribute credentialless na iframe (Chrome 110+): -->
<iframe src="https://partner.example.com/widget" credentialless></iframe>
```

### crossOriginIsolated requirement

```javascript
// crossOriginIsolated = true tylko gdy:
// 1. COOP: same-origin (na dokumencie)
// 2. COEP: require-corp (na dokumencie)
// I: wszystkie zasoby na stronie mają CORS lub CORP!

// Praktyczny test:
crossOriginIsolated // true = SharedArrayBuffer dostępny
                    // false = SharedArrayBuffer undefined lub disabled
```

### COEP i Workers

Service Workers i Web Workers dziedziczą COEP od swojego kontekstu:

```javascript
// Worker w dokumencie z COEP: require-corp:
const worker = new Worker('/worker.js');
// Worker też ma COEP! Musi respektować te same ograniczenia ładowania zasobów.

// SharedArrayBuffer w Worker z crossOriginIsolated:
const sab = new SharedArrayBuffer(1024);
worker.postMessage({ buffer: sab }, [sab]);
// Działa gdy crossOriginIsolated === true
```

---

## 5. Analogiczny przykład z życia

COEP to **polityka "tylko zaproszeni goście"** na prywatnym przyjęciu:

- Organizator (strona z COEP) przyjmuje tylko gości którzy mają:
  - Oficjalne zaproszenie (CORS header: "pozwalam się załadować cross-origin")
  - Lub przepustkę (CORP header: "pozwalam na embedding")
- Gość bez zaproszenia ani przepustki → nie wchodzi (zasób blokowany)
- COOP (poprzedni rozdział) to oddzielna sala VIP — tylko sama-origin goście
- Razem (COOP+COEP): impreza jest zarówno VIP (COOP) jak i weryfikuje gości (COEP)

---

## 6. Przykład kodu

```javascript
// === Sprawdzenie COEP status ===

// Czy crossOriginIsolated jest aktywne?
console.log('Cross-Origin Isolated:', crossOriginIsolated);

// Jeśli false: który nagłówek brakuje?
// Sprawdź: response headers w Network tab → szukaj COOP i COEP
```

```http
# === Kompletna konfiguracja dla crossOriginIsolated ===

# Nagłówki odpowiedzi głównego dokumentu:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp

# Nagłówki własnych zasobów (np. API, media):
Cross-Origin-Resource-Policy: same-origin
# lub dla publicznych zasobów:
Cross-Origin-Resource-Policy: cross-origin

# Zewnętrzne zasoby (CDN, biblioteki) muszą mieć CORS lub CORP:
# Jeśli CDN nie ma CORS/CORP → trzeba hostować własną kopię
```

```javascript
// === Sprawdzenie jakie zasoby blokuje COEP ===

// DevTools Console:
// Szukaj błędów: "...has been blocked due to COEP"

// Programatycznie - zasób nie załadowany:
const img = document.querySelector('img.external');
img.addEventListener('error', (e) => {
    console.error('Zasób zablokowany przez COEP:', img.src);
    // Potrzebuje CORS lub CORP
});
```

```javascript
// === Praktyczne wdrożenie z fallback ===

// Na serwerze: obsługa CORS dla własnych zasobów:
app.use('/assets', (req, res, next) => {
    // Pozwól na cross-origin embedding naszych własnych zasobów:
    res.setHeader('Cross-Origin-Resource-Policy', 'same-site');
    // lub 'cross-origin' jeśli zasób ma być publiczny
    next();
});

// Dla API (requires CORS dla fetch z innych origins):
app.use('/api', cors({
    origin: ['https://app.example.com', 'https://admin.example.com'],
    credentials: true,
}));
// Access-Control-Allow-Origin: https://app.example.com
// COEP: require-corp będzie respektował CORS nagłówek jako "zezwolenie"
```

---

## 7. Przykład z prawdziwej aplikacji

### Wdrożenie COEP krok po kroku

Realistyczna aplikacja z cross-origin zasobami:

```
Mapa zasobów:
- /index.html (nasz dokument) → COEP: require-corp ✓
- /styles/app.css (same-origin) → CORP: same-origin ✓
- https://cdn.bootstrap.com/bootstrap.min.css → musi mieć CORS lub CORP!
- https://maps.google.com/api/embed → Google Maps API — czy ma CORS?
- https://fonts.googleapis.com/css → Google Fonts — ma CORS: * ✓
- https://api.stripe.com/v3/... → Stripe — czy ma CORP?
```

```javascript
// Narzędzie do audytu które zasoby brakuje CORS/CORP:
// (Uruchom w DevTools przed wdrożeniem COEP)

const resources = performance.getEntriesByType('resource');
const crossOrigin = resources.filter(r => {
    const url = new URL(r.name);
    return url.origin !== location.origin;
});

console.table(crossOrigin.map(r => ({
    url: r.name,
    type: r.initiatorType,
    hasCorpOrCors: r.responseStatus > 0, // uproszczone - sprawdź Network tab
})));
```

### WebAssembly z SharedArrayBuffer

```javascript
// Wielowątkowy WASM wymaga SharedArrayBuffer:
// → wymaga crossOriginIsolated = true
// → wymaga COOP + COEP

// Przykład: Emscripten (WASM) z pthread:
if (!crossOriginIsolated) {
    throw new Error(
        'Ta aplikacja wymaga cross-origin isolation. ' +
        'Serwer musi wysyłać COOP: same-origin i COEP: require-corp.'
    );
}

// Inicjalizacja wielowątkowego WASM:
const wasmModule = await WebAssembly.compileStreaming(fetch('/app.wasm'));
const sharedMemory = new WebAssembly.Memory({
    initial: 256,
    maximum: 256,
    shared: true,  // wymaga crossOriginIsolated!
});
```

### Google Maps i COEP — problem rzeczywisty

Google Maps API (Maps JavaScript API) historycznie nie miało CORP headera → blokowanie przez COEP: require-corp. Rozwiązania:

1. Użyj `COEP: credentialless` zamiast `require-corp`
2. Ładuj Maps przez iframe z `credentialless` atrybutem
3. Użyj Maps Static API (zwykłe obrazki z CORS *)

---

## 8. Typowe błędy programistów

### Błąd 1: COEP bez COOP

```http
# BŁĄD: samo COEP nie daje crossOriginIsolated
Cross-Origin-Embedder-Policy: require-corp
# Brak COOP → crossOriginIsolated === false

# POPRAWKA:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

### Błąd 2: Zapomnienie o iframach

```html
<!-- BŁĄD: strona ma COEP ale embedduje zewnętrzny iframe bez COEP -->
<iframe src="https://partner.com/widget"></iframe>
<!-- partner.com nie ma COEP → jest blokowany przez COEP na naszej stronie -->

<!-- POPRAWKA A: poproś partner.com o dodanie COEP -->
<!-- POPRAWKA B: użyj credentialless attribute: -->
<iframe src="https://partner.com/widget" credentialless></iframe>
<!-- POPRAWKA C: użyj COEP: credentialless zamiast require-corp -->
```

### Błąd 3: Brak CORP na własnych zasobach

```javascript
// BŁĄD: własne API bez CORP
// app.example.com ma COEP: require-corp
// Ładuje zasób z: https://cdn.example.com/image.png (same-site CDN)
// CDN nie ma Access-Control-Allow-Origin ani Cross-Origin-Resource-Policy
// → Blokowane przez COEP!

// POPRAWKA: dodaj CORP na CDN:
// Response header dla CDN zasobów:
// Cross-Origin-Resource-Policy: same-site
```

### Błąd 4: COEP i third-party scripts bez CORS

```html
<!-- BŁĄD: Google Analytics bez CORS na niektórych zasobach -->
<script src="https://www.google-analytics.com/analytics.js"></script>
<!-- Z COEP: require-corp: sprawdź czy analytics.js ma CORS/CORP -->
<!-- Jeśli nie → blokowane → tracking nie działa -->

<!-- POPRAWKA: użyj COEP: credentialless jeśli nie możesz kontrolować third-party -->
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przed Spectre

```
Spectre wymaga: shared memory (SharedArrayBuffer) + high-res timer

COEP: require-corp gwarantuje że:
1. Każdy zasób w procesie wyraził zgodę na cross-origin access
2. Brak "unexpected cross-origin resources" w pamięci procesu
3. SharedArrayBuffer bezpieczny → przeglądarka może go udostępnić

Bez COEP: atakujący mógłby załadować wrażliwy zasób (bez jego wiedzy) 
          do swojego procesu i użyć Spectre timing attack
```

### CORP jako dodatkowa warstwa

```http
# Zasób który nie chce być embeddowany cross-site:
Cross-Origin-Resource-Policy: same-origin
# Blokuje Spectre read przez cross-origin embedding

# Zasób który może być embeddowany przez same-site:
Cross-Origin-Resource-Policy: same-site

# Zasób publiczny (CDN font, obrazek):
Cross-Origin-Resource-Policy: cross-origin
```

### Ochrona przed cross-origin data leaks

```
Scenariusz (bez COEP):
1. Aplikacja bankowa bank.com nie ma CORP na /user/avatar
2. Atakujący na evil.com: <img src="https://bank.com/user/avatar?id=123">
3. Przeglądarka ładuje awatar (z ciasteczkami sesji bank.com!)
4. Spectre może odczytać piksele awatara z pamięci procesu
5. → Wyciek danych użytkownika!

Z CORP: same-origin na bank.com/user/avatar:
6. evil.com próbuje załadować → CORP blokuje → Spectre niemożliwy
```

### Powiązane CWE

- **CWE-200** — Information Exposure (Spectre leaks)
- **CWE-693** — Protection Mechanism Failure
- **CWE-922** — Insecure Storage of Sensitive Information

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź nagłówki

```bash
# COEP na dokumencie:
curl -I https://target.com | grep -i "cross-origin-embedder-policy"

# CORP na zasobach:
curl -I https://target.com/api/user-data | grep -i "cross-origin-resource-policy"
curl -I https://cdn.target.com/image.jpg | grep -i "cross-origin-resource-policy"

# Brak CORP na prywatnych zasobach (avatary, dane użytkownika) → Spectre risk
```

### Sprawdź crossOriginIsolated

```javascript
// Na target.com w konsoli:
console.log({
    crossOriginIsolated,
    coopValue: document.head.querySelector('meta[http-equiv="Cross-Origin-Opener-Policy"]')
    // (meta COOP jest rzadkie — COOP jest zwykle przez HTTP header)
});
```

### DevTools: COEP violations

```
1. Network tab → filtruj "blocked"
2. Console: szukaj "COEP" lub "Cross-Origin-Embedder-Policy" w błędach
3. Security tab → sprawdź czy crossOriginIsolated jest aktywne
```

### Test CORP protection

```javascript
// Sprawdź czy prywatne zasoby mają CORP:
const privateEndpoints = [
    '/user/avatar',
    '/api/profile-picture',
    '/media/private/',
];

for (const endpoint of privateEndpoints) {
    const resp = await fetch(endpoint);
    console.log(endpoint, resp.headers.get('Cross-Origin-Resource-Policy'));
    // null = brak CORP = potencjalnie podatny na Spectre embedding
}
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy COEP header jest ustawiony na dokumencie?
□ Czy COOP jest ustawiony razem z COEP?
□ Czy crossOriginIsolated === true gdy wymagany SharedArrayBuffer?
□ Czy własne zasoby mają CORP header?
□ Czy prywatne zasoby (avatary, dane) mają CORP: same-origin lub same-site?
□ Czy cross-site zasoby (CDN, Google Fonts) mają CORS lub CORP?
□ Czy iframy third-party są obsługiwane (credentialless lub własny COEP)?
□ Czy COEP-Report-Only było używane przed wdrożeniem?
□ Czy raporty COEP są zbierane i analizowane?
□ Czy po wdrożeniu COEP wszystkie funkcje aplikacji działają?
```

---

## 12. Jak się zabezpieczać

```http
# Kompletna konfiguracja dla crossOriginIsolated:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp

# Na własnych zasobach:
Cross-Origin-Resource-Policy: same-origin   # prywatne zasoby
Cross-Origin-Resource-Policy: same-site     # zasoby dla subdomen
Cross-Origin-Resource-Policy: cross-origin  # publiczne zasoby (CDN)
```

```javascript
// === Express.js: COEP + CORP ===

// Główna aplikacja:
app.use((req, res, next) => {
    res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
    res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
    next();
});

// Prywatne zasoby:
app.use('/user/', (req, res, next) => {
    res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');
    next();
});

// Publiczne zasoby (CDN-like):
app.use('/public/', (req, res, next) => {
    res.setHeader('Cross-Origin-Resource-Policy', 'cross-origin');
    res.setHeader('Access-Control-Allow-Origin', '*');
    next();
});
```

```javascript
// === Strategia migracji na COEP ===

// Krok 1: Report-Only
// Cross-Origin-Embedder-Policy-Report-Only: require-corp; report-to="coep"

// Krok 2: Zbierz raporty, zidentyfikuj blokowane zasoby

// Krok 3: Dodaj CORP na własne zasoby:
// Cross-Origin-Resource-Policy: same-origin/same-site/cross-origin

// Krok 4: Poproś third-party dostawców o CORS lub CORP
// (Google Fonts *, CDN biblioteki zazwyczaj mają CORS)

// Krok 5: Dla problemowych zasobów: switch to COEP: credentialless
// Cross-Origin-Embedder-Policy: credentialless

// Krok 6: Włącz COEP: require-corp (lub credentialless) na produkcji
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. COEP = `Cross-Origin-Embedder-Policy` — wymusza że każdy cross-origin zasób ma CORS lub CORP
2. `require-corp` = wszystkie zasoby muszą mieć zezwolenie; `credentialless` = brak credentials dla cross-origin
3. COEP + COOP = `crossOriginIsolated = true` → `SharedArrayBuffer` bezpieczny
4. Chroni przed Spectre: eliminuje niezamierzone cross-origin zasoby w pamięci procesu
5. Wymaga CORP headera (`Cross-Origin-Resource-Policy`) na własnych zasobach

**Dla pentestera:**
- Brak `Cross-Origin-Embedder-Policy` = brak cross-origin isolation = SharedArrayBuffer unsafe
- Sprawdź `Cross-Origin-Resource-Policy` na prywatnych zasobach — brak = Spectre embedding risk
- Test: `crossOriginIsolated` w konsoli — false gdy COEP/COOP nie wdrożone
- COEP violations widoczne w Console jako błędy ładowania zasobów

---

## Powiązania

```
COEP (Cross-Origin-Embedder-Policy)
    │
    ├──► COOP (Rozdział 42)
    │         COOP + COEP = crossOriginIsolated
    │         Oba wymagane dla SharedArrayBuffer
    │
    ├──► CORP (Rozdział 44)
    │         CORP: Cross-Origin-Resource-Policy header na zasobach
    │         COEP: wymusza obecność CORS lub CORP na zasobach
    │
    ├──► CORS (Rozdział 13)
    │         CORS: Access-Control-Allow-Origin
    │         COEP akceptuje CORS jako zezwolenie na cross-origin embedding
    │
    └──► Web Workers / WASM (Rozdziały 20, 50)
              SharedArrayBuffer wymaga crossOriginIsolated
              COEP niezbędne dla wielowątkowego WASM
```
