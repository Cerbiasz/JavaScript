# Rozdział 34: Permissions Policy

## 1. Czym jest Permissions Policy

**Permissions Policy** (dawniej **Feature Policy**) to mechanizm HTTP umożliwiający witrynom kontrolowanie dostępu do potencjalnie wrażliwych funkcji przeglądarki — zarówno dla siebie, jak i dla embeddowanych iframes. Pozwala na granularną kontrolę nad takimi funkcjami jak kamera, mikrofon, geolokalizacja, fullscreen, oraz setkami innych API.

Permissions Policy jest implementowane przez:
1. **HTTP Header:** `Permissions-Policy: camera=(), microphone=('self')`
2. **HTML atrybut `allow`:** `<iframe allow="camera 'none'"></iframe>`

Kluczowa rola w bezpieczeństwie:
- Ograniczenie powierzchni ataku przez wyłączenie niepotrzebnych API
- Ochrona użytkowników embeddowanych stron przed nadużyciami
- Wymuszenie zasady least-privilege dla funkcji przeglądarki

---

## 2. Dlaczego powstał

### Problem: brak kontroli nad API przeglądarki

Przed Permissions Policy:
- Każda strona (i każdy iframe) miała domyślnie dostęp do wszystkich API
- Złośliwy iframe mógł używać kamery, mikrofonu bez wiedzy użytkownika
- Brak mechanizmu dla hosta do ograniczenia iframes

Feature Policy (2018) → Permissions Policy (2020, Chrome 90):
- Przemianowanie z Feature Policy
- Nowa składnia (lista zamiast łańcuch)
- Lepsza granularność

---

## 3. Jak działa

### Składnia HTTP Header

```
Permissions-Policy: <feature>=(<allowlist>), <feature>=(<allowlist>)
```

**Allowlist wartości:**
- `()` — wyłącz dla wszystkich (strona + iframes)
- `(self)` — tylko same-origin (strona i same-origin iframes)
- `(*)` — wszystkie origins (permissive)
- `("https://partner.com")` — konkretny origin
- `(self "https://partner.com")` — same-origin + partner

```http
Permissions-Policy: 
    camera=(),
    microphone=(),
    geolocation=(self),
    payment=(self "https://payment-provider.com"),
    fullscreen=(*),
    clipboard-read=(self),
    clipboard-write=(self)
```

### Atrybut `allow` na iframe

```html
<!-- Wyłącz wszystko dla tego iframe: -->
<iframe src="..." allow=""></iframe>

<!-- Pozwól tylko na fullscreen: -->
<iframe src="..." allow="fullscreen"></iframe>

<!-- Pozwól na geolokalizację (tylko same-origin w iframe): -->
<iframe src="..." allow="geolocation 'self'"></iframe>

<!-- Wiele uprawnień: -->
<iframe src="..." allow="camera; microphone; autoplay 'none'"></iframe>
```

### Sprawdzanie uprawnień w JavaScript

```javascript
// Sprawdź czy API jest dozwolone:
document.featurePolicy?.allowsFeature('camera'); // true/false
document.featurePolicy?.allowsFeature('geolocation', 'https://maps.example.com');

// Lub przez Permissions API:
const result = await navigator.permissions.query({ name: 'camera' });
console.log(result.state); // 'granted', 'denied', 'prompt'
```

### Lista ważnych funkcji (features)

```
Kategoria media:
- camera
- microphone
- speaker-selection
- display-capture
- screen-wake-lock

Kategoria lokalizacja:
- geolocation
- accelerometer
- gyroscope
- magnetometer
- ambient-light-sensor

Kategoria nawigacja:
- fullscreen
- picture-in-picture
- autoplay

Kategoria płatności i auth:
- payment
- publickey-credentials-get
- usb
- bluetooth

Kategoria prywatność:
- interest-cohort (FLoC — usunięte)
- attribution-reporting
- private-aggregation
- shared-storage

Kategoria bezpieczeństwo:
- cross-origin-isolated
- document-domain
- sync-xhr
- unload (nowe)
```

---

## 4. Co dzieje się wewnętrznie

### Inheritance

Permissions Policy jest dziedziczona w hierarchii dokumentów:

```
Strona główna (allowed: camera=self)
    ↓
iframe A (same-origin) → może używać camera
    ↓
iframe B (cross-origin, allow="camera") → może używać camera (jeśli parent pozwolił)
    ↓
iframe C (cross-origin, bez allow) → NIE może używać camera
```

Zasada: **iframe może mieć co najwyżej tyle uprawnień ile parent**. Parent nie może nadać iframeowi więcej uprawnień niż sam posiada.

### Strict vs permissive defaults

Domyślne zachowanie różni się per-feature:
- `autoplay` — domyślnie `(self)` (Chrome)
- `camera`, `microphone` — domyślnie `(self)` (wymaga permission prompt)
- `fullscreen` — domyślnie `(self)`
- `geolocation` — domyślnie `(self)`

---

## 5. Analogiczny przykład z życia

Permissions Policy to lista kontroli dostępu do pokojów w biurze:

- Dyrektor (strona główna) decyduje kto ma dostęp do których pomieszczeń
- Gościa (iframe) można wpuścić do konkretnych pokojów (allow="camera")
- Ale gość nie może mieć dostępu do pokojów niedostępnych dla dyrektora
- Brak Permissions Policy = brak kontroli dostępu = każdy gość ma dostęp wszędzie

---

## 6. Przykład kodu

```javascript
// === Sprawdzanie Permissions Policy w JavaScript ===

// Sprawdź jakie funkcje są dozwolone na stronie:
async function checkPermissions() {
    const features = ['camera', 'microphone', 'geolocation', 'payment'];
    const results = {};
    
    for (const feature of features) {
        try {
            // document.featurePolicy (stare API — Feature Policy)
            const allowed = document.featurePolicy?.allowsFeature(feature);
            results[feature] = allowed;
        } catch {
            results[feature] = 'unknown';
        }
    }
    
    return results;
}

// Sprawdzenie przed użyciem API (defensive programming):
async function requestCamera() {
    // Sprawdź Permissions Policy
    if (document.featurePolicy?.allowsFeature('camera') === false) {
        throw new Error('Camera disabled by Permissions Policy');
    }
    
    // Sprawdź uprawnienie użytkownika
    const perm = await navigator.permissions.query({ name: 'camera' });
    if (perm.state === 'denied') {
        throw new Error('Camera permission denied');
    }
    
    return navigator.mediaDevices.getUserMedia({ video: true });
}
```

```http
# Przykładowy nagłówek dla banku (strict security):
Permissions-Policy: 
    camera=(),
    microphone=(),
    geolocation=(),
    payment=(self),
    fullscreen=(self),
    clipboard-read=(self),
    clipboard-write=(self),
    autoplay=(),
    usb=(),
    bluetooth=(),
    interest-cohort=()
```

---

## 7. Przykład z prawdziwej aplikacji

### Clickjacking + brak Permissions Policy

Jeśli strona embedduje zewnętrzne iframes bez `allow=""` ograniczeń, malicious iframe może próbować użyć kamery lub mikrofonu (wymagają permission prompt — ale sam prompt może być zmanipulowany):

```html
<!-- Strona bez Permissions Policy: -->
<iframe src="https://ads.example.com/ad" width="300" height="250">
<!-- Brak: allow="" -->
<!-- ads.example.com może próbować wywołać getUserMedia() → permission prompt pojawi się -->
<!-- Użytkownik może przypadkowo udzielić dostępu do kamery dla ads.example.com -->

<!-- POPRAWKA: -->
<iframe src="https://ads.example.com/ad" width="300" height="250"
    allow="fullscreen 'none'; camera 'none'; microphone 'none'">
```

### document.domain a Permissions Policy

Stary mechanizm `document.domain` umożliwiał rozluźnienie SOP dla subdomain:
```javascript
document.domain = 'example.com'; // łączyło bank.example.com i app.example.com
```

Permissions Policy może wyłączyć to: `document-domain=()` blokuje użycie `document.domain`. To jest zalecane przez COOP/COEP (rozdziały 42-43).

---

## 8. Typowe błędy programistów

### Błąd 1: Stara składnia Feature Policy

```http
# STARA składnia (Feature Policy) — nie używaj:
Feature-Policy: camera 'none'; microphone 'none'

# NOWA składnia (Permissions Policy):
Permissions-Policy: camera=(), microphone=()
```

### Błąd 2: Brak Permissions Policy na iframes

```html
<!-- BŁĄD: brak kontroli nad iframe -->
<iframe src="https://third-party.com/widget"></iframe>

<!-- POPRAWKA: restrykcyjna polityka dla iframes -->
<iframe src="https://third-party.com/widget"
    allow="fullscreen"
    sandbox="allow-scripts allow-same-origin">
```

### Błąd 3: Zbyt permissive polityka

```http
# BŁĄD: pozwolenie na wszystko
Permissions-Policy: camera=(*), microphone=(*), geolocation=(*)
# Każdy iframe może używać tych API!

# POPRAWKA: only what's needed
Permissions-Policy: camera=(self), microphone=(), geolocation=()
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ograniczenie powierzchni ataku

Permissions Policy na zasadzie **deny by default** drastycznie redukuje co złośliwy kod (przez XSS, supply chain) może zrobić:

```http
Permissions-Policy: 
    camera=(),              # XSS nie może aktywować kamery
    microphone=(),          # XSS nie może nagrywać audio
    geolocation=(),         # XSS nie może śledzić lokalizacji
    clipboard-read=(),      # XSS nie może czytać schowka
    payment=()              # XSS nie może inicjować płatności
```

### Ochrona iframes przed nadużyciami

Reklamowe iframes, widgety third-party — wszystkie mogą być ograniczone:

```html
<iframe src="https://widget.example.com" 
    allow=""    <!-- wyłącz WSZYSTKIE funkcje -->
    sandbox="allow-scripts allow-same-origin allow-forms">
```

### `interest-cohort=()` — Google FLoC

Google FLoC (Federated Learning of Cohorts) grupowało użytkowników dla targetowania reklam. Można było je wyłączyć: `Permissions-Policy: interest-cohort=()`. Wiele stron dodało to jako privacy best practice. FLoC został ostatecznie anulowany przez Google.

### Permissions Policy a `sync-xhr`

```http
Permissions-Policy: sync-xhr=()
```

Wyłącza synchroniczne XMLHttpRequest — blokuje główny wątek. To zmusza do asynchronicznego kodu, co jest i lepszą praktyką i wymaga Fetch API.

### Powiązane standardy

- **CSP** (Rozdział 36) — kontroluje jakie zasoby mogą być ładowane
- **Permissions Policy** — kontroluje jakie funkcje API mogą być używane
- **`sandbox`** attribute na iframe — kombonowanie z Permissions Policy

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź nagłówki HTTP

```bash
curl -I https://target.com | grep -i "permissions-policy\|feature-policy"
```

### DevTools → Application → Permissions Policy

1. **Application** → **Frames** → wybierz frame
2. Widoczna lista dozwolonych i zabronionych funkcji

### Console

```javascript
// Sprawdź jakie funkcje są dostępne:
document.featurePolicy?.features();      // lista wszystkich znanych features
document.featurePolicy?.allowedFeatures(); // lista dozwolonych
document.featurePolicy?.allowsFeature('camera'); // true/false
document.featurePolicy?.allowsFeature('camera', 'https://example.com'); // per-origin
```

### Burp Suite

Sprawdź response headers na każdym endpoint — `Permissions-Policy` powinien być w nagłówkach odpowiedzi.

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Permissions-Policy header jest ustawiony?
□ Czy nieużywane funkcje (camera, microphone, geolocation) są wyłączone?
□ Czy iframes third-party mają restrykcyjny allow="" atrybut?
□ Czy document-domain jest wyłączone (document-domain=())?
□ Czy sync-xhr jest wyłączone?
□ Czy payment API jest ograniczone (payment=(self))?
□ Czy po XSS Permissions Policy ogranicza co można zrobić?
□ Czy stara Feature-Policy jest zamieniona na Permissions-Policy?
```

---

## 12. Jak się zabezpieczać

```http
# Minimalistyczna Permissions-Policy dla aplikacji bankowej:
Permissions-Policy: 
    accelerometer=(),
    ambient-light-sensor=(),
    autoplay=(),
    battery=(),
    bluetooth=(),
    camera=(),
    display-capture=(),
    document-domain=(),
    encrypted-media=(self),
    fullscreen=(self),
    geolocation=(),
    gyroscope=(),
    interest-cohort=(),
    magnetometer=(),
    microphone=(),
    midi=(),
    payment=(self),
    picture-in-picture=(),
    publickey-credentials-get=(self),
    screen-wake-lock=(),
    sync-xhr=(),
    usb=(),
    web-share=(),
    xr-spatial-tracking=()
```

```javascript
// W aplikacji — defensywne sprawdzenie przed użyciem API:
async function useGeolocation() {
    if (!document.featurePolicy?.allowsFeature('geolocation')) {
        throw new Error('Geolocation blocked by policy');
    }
    return new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(resolve, reject);
    });
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Permissions Policy = HTTP header kontrolujący dostęp do browser API (kamera, GPS, itp.)
2. Składnia: `Permissions-Policy: camera=(), geolocation=(self)`
3. `()` = wyłączone dla wszystkich; `(self)` = tylko same-origin
4. Dziedziczy się w iframe hierarchii — parent nie może dać więcej niż ma
5. Redukuje powierzchnię ataku po XSS (złośliwy kod nie może użyć wyłączonych API)

**Dla pentestera:**
- Sprawdź obecność Permissions-Policy header w response
- Brak = brak ograniczeń → XSS ma dostęp do kamery, mikrofonu itd.
- DevTools → Application → Frames dla visual overview

---

## Powiązania

```
Permissions Policy
    │
    ├──► CSP (Rozdział 36)
    │         CSP: zasoby; PP: funkcje API
    │
    ├──► Referrer Policy (Rozdział 35)
    │         Oba są security HTTP headers
    │
    ├──► COOP/COEP (Rozdziały 42-43)
    │         document-domain=() związany z cross-origin isolation
    │
    └──► iframe sandbox
              Kombinacja sandbox + allow dla maksymalnej izolacji iframes
```
