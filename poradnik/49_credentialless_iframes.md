# Rozdział 49: Credentialless iframes

## 1. Czym są Credentialless iframes

**Credentialless iframes** to mechanizm embeddowania stron w iframie w trybie anonimowym — bez cookies, certyfikatów klienta ani innych danych uwierzytelniających. Iframe ładuje zewnętrzną treść tak, jakby był niezalogowanym, nowym użytkownikiem — niezależnie od tego czy użytkownik jest zalogowany na tej stronie.

Implementacja przez atrybut `credentialless` lub `COEP: credentialless`:

```html
<!-- Credentialless iframe (Chrome 110+): -->
<iframe src="https://third-party.com/widget" credentialless></iframe>
```

```http
<!-- Alternatywnie: COEP w trybie credentialless: -->
Cross-Origin-Embedder-Policy: credentialless
```

Kluczowe właściwości:
- Brak third-party cookies
- Brak storage (localStorage, sessionStorage) z poprzednich wizyt
- Brak cache (lub izolowany cache)
- Nowe, tymczasowe storage per iframe lifecycle
- Normalna komunikacja z parent przez postMessage (w przeciwieństwie do Fenced Frames)

---

## 2. Dlaczego powstał

### Problem: COEP: require-corp blokuje third-party iframes

Przy wdrożeniu `COEP: require-corp` (rozdział 43), każdy iframe musi mieć COEP. Ale wiele third-party frameworków i widgetów (Google Maps, YouTube embed, chat widgety) nie wysyła `COEP: require-corp` → blokowane przez COEP.

```html
<!-- Strona z COEP: require-corp próbuje załadować YouTube: -->
<iframe src="https://www.youtube.com/embed/VIDEO_ID"></iframe>
<!-- YouTube nie ma COEP: require-corp → blokowane! -->
<!-- YouTube embed przestaje działać -->
```

Dwa problemy z `COEP: require-corp`:
1. Third-party iframes bez COEP → blokowane
2. Third-party images/scripts bez CORP/CORS → blokowane

`COEP: credentialless` rozwiązuje problem nr 1 i 2:
- Images/scripts bez CORP/CORS → ładowane ale bez credentials (anonimowo)
- Iframes bez COEP → ładowane ale bez credentials (anonimowo)
- Efekt: `crossOriginIsolated = true` **bez** wymagania COEP na zasobach!

---

## 3. Jak działa

### COEP: credentialless

```http
<!-- Na dokumencie głównym: -->
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: credentialless

<!-- Efekt: crossOriginIsolated = true -->
<!-- Zasoby cross-origin ładowane BEZ credentials (cookies, client certs) -->
<!-- Nie wymaga CORP ani CORS na zasobach! -->
```

```javascript
// Sprawdź:
crossOriginIsolated;  // true z COOP: same-origin + COEP: credentialless!
```

### Atrybut credentialless na iframe

```html
<!-- Iframe z credentialless: -->
<iframe src="https://youtube.com/embed/VIDEO_ID" credentialless></iframe>
<!-- YouTube załaduje się bez cookies YouTube'a -->
<!-- User nie jest "zalogowany" w tym iframe -->
<!-- Ale iframe MOŻE kommunikować przez postMessage! -->
```

```javascript
// Wewnątrz credentialless iframe:
document.cookie;  // puste (brak cookies!)
localStorage.getItem('key');  // null (izolowane storage)
// fetch() wysyłane BEZ credentials

// Ale: postMessage do parent działa!
window.parent.postMessage('data', 'https://parent.com');  // OK (w odróżnieniu od Fenced Frames)
```

### Porównanie z require-corp i Fenced Frames

```
COEP: require-corp:
- Zasoby cross-origin MUSZĄ mieć CORS lub CORP
- Jeśli brak → blokowane całkowicie
- Potrzeba COEP na embedded iframach
- Najbezpieczniejsze ale najtrudniejsze we wdrożeniu

COEP: credentialless:
- Zasoby cross-origin ładowane BEZ credentials
- Nie potrzeba CORP/CORS na zasobach
- Iframes ładowane bez credentials (nie potrzeba COEP na third-party!)
- Łatwiejsze wdrożenie, mniej restrykcyjne

Fenced Frames:
- Pełna izolacja (brak komunikacji bidirectional)
- Opaque URL
- Specjalny element (<fencedframe>)
- Dla Privacy Sandbox use cases (reklamy)
```

---

## 4. Co dzieje się wewnętrznie

### Storage i cache isolation

```
Credentialless iframe:
- Cookies: storage key = (origin, "credentialless", top-level-site)
  → Nowy, izolowany "credentialless cookie jar"
  → Brak dostępu do "normalnych" cookies origin
  
- localStorage/sessionStorage: izolowane
  → Mogą istnieć dane ale są odcięte od zwykłego localStorage

- Cache: przeglądarki implementują izolację przez network partition key
  → Brak cache hitów z poprzednich sesji cross-origin
```

### Network requests z credentialless

```javascript
// Request z credentialless iframe:
fetch('https://api.example.com/data');
// BRAK nagłówka Cookie (chyba że Set-Cookie w ramach credentialless session)
// BRAK Authorization header (client cert)
// Brak SameSite=Lax cookies (bo: cross-origin + credentialless)

// Analogiczne do:
fetch('https://api.example.com/data', {
    credentials: 'omit'  // brak credentials
});
```

### crossOriginIsolated z credentialless

```javascript
// Wymagania dla crossOriginIsolated = true:
// Option A (bezpieczniejszy): COOP: same-origin + COEP: require-corp
// Option B (łatwiejszy): COOP: same-origin + COEP: credentialless

// Oba dają:
crossOriginIsolated;  // true
new SharedArrayBuffer(1024);  // dozwolone!
```

---

## 5. Analogiczny przykład z życia

Credentialless iframe to **poczekalni z kartą gościa (anonimową)**:

- Pracownik (user) ma kluczową kartę dostępu (cookies) do wszystkich pomieszczeń
- Zewnętrzny gość (third-party iframe) dostaje kartę gościa (brak credentials) — może wejść do lobby (załadować się) ale nie ma dostępu do pomieszczeń wymagających uwierzytelnienia
- Gość może rozmawiać z pracownikiem przez interkom (postMessage) — w odróżnieniu od Fenced Frames gdzie interkom jest wyłączony
- Gdyby gość miał pełną kartę (cookies), mógłby wejść do wszystkich pomieszczeń i wykraść dane

---

## 6. Przykład kodu

```html
<!-- === Credentialless iframes — użycie === -->

<!DOCTYPE html>
<html>
<head>
    <title>Strona z COEP: credentialless</title>
</head>
<body>

<!-- YouTube bez potrzeby COEP na YouTube: -->
<iframe 
    src="https://www.youtube.com/embed/VIDEO_ID"
    credentialless
    width="560" 
    height="315"
    allowfullscreen>
</iframe>

<!-- Google Maps bez potrzeby COEP/CORP na Google: -->
<iframe
    src="https://www.google.com/maps/embed?pb=..."
    credentialless
    width="600"
    height="450">
</iframe>

<!-- Third-party widget z postMessage komunikacją: -->
<iframe
    id="widget"
    src="https://widget.example.com/chat"
    credentialless>
</iframe>

<script>
// postMessage DZIAŁA (w odróżnieniu od Fenced Frames):
const widgetFrame = document.getElementById('widget');
widgetFrame.addEventListener('load', () => {
    widgetFrame.contentWindow.postMessage(
        { action: 'init', userId: 'user-123' },  
        'https://widget.example.com'
    );
});

window.addEventListener('message', (e) => {
    if (e.origin === 'https://widget.example.com') {
        console.log('Widget message:', e.data);
    }
});
</script>

</body>
</html>
```

```http
# === COEP: credentialless dla crossOriginIsolated ===

Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: credentialless

# Teraz: SharedArrayBuffer dostępny!
# I: YouTube, Google Maps itp. działają BEZ potrzeby ich COEP!
```

```javascript
// === Feature detection ===

// Sprawdź czy credentialless jest obsługiwane:
const frame = document.createElement('iframe');
const supportsCredentialless = 'credentialless' in HTMLIFrameElement.prototype;

if (supportsCredentialless) {
    frame.credentialless = true;
    console.log('Credentialless iframes obsługiwane');
} else {
    // Chrome 110+, Firefox: nie obsługiwane
    console.warn('Credentialless iframes niedostępne');
}

// Sprawdź COEP mode:
// (brak bezpośredniego API, sprawdź przez crossOriginIsolated)
console.log('Cross-origin isolated:', crossOriginIsolated);
```

---

## 7. Przykład z prawdziwej aplikacji

### WebAssembly wielowątkowy z YouTube embed

Problem: aplikacja wymagająca SharedArrayBuffer (np. video editor) chce też embedować YouTube'a:

```javascript
// Bez credentialless:
// COEP: require-corp → YouTube blokowany → albo SharedArrayBuffer albo YouTube

// Z COEP: credentialless:
// Oba działają!
```

```http
# Response headers strony:
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: credentialless

# crossOriginIsolated = true → SharedArrayBuffer dostępny
# YouTube iframe z atrybutem credentialless → YouTube ładuje się (bez YouTube cookies)
```

### Anonimowe ładowanie third-party

```javascript
// Przypadek: chcemy embeddować third-party widget ale:
// - nie chcemy żeby widget widział cookies user → credentialless!
// - chcemy komunikować się z widgetem → postMessage OK (inaczej niż Fenced Frames)
// - chcemy zbierać dane o interakcji → addEventListener('message')

// Widget chat bez logowania użytkownika jako użytkownika chat serwisu:
const chatFrame = document.createElement('iframe');
chatFrame.src = 'https://chat-service.com/widget';
chatFrame.credentialless = true;
// Widget.com nie otrzyma swoich cookies → anonymous sesja w widgecie
// Możemy przekazać własny token przez postMessage po załadowaniu
chatFrame.addEventListener('load', () => {
    chatFrame.contentWindow.postMessage(
        { token: app.userToken, userId: app.userId },
        'https://chat-service.com'
    );
});
```

### Security consideration: sensitive state w credentialless

```
UWAGA na odwrotny problem:
- Strona ładuje payment form w credentialless iframe
- Payment form (stripe.com) NIE ma swoich cookies → user musi wpisać dane karty od nowa!
- Jeśli Stripe chce prefillować dane z konta → nie może (brak cookies!)

Wniosek: credentialless jest odpowiednie gdy:
✓ Third-party content jest STATYCZNY (YouTube video, mapa bez logowania)
✓ Widget używa tokenów przekazanych przez postMessage (nie własnych cookies)
✗ Third-party potrzebuje własnej sesji (payment, SSO widget z saved credentials)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Credentialless zamiast Fenced Frames gdy potrzeba izolacji

```html
<!-- BŁĄD: credentialless iframe dla reklam (brak pełnej izolacji) -->
<iframe src="https://ads.com/ad" credentialless></iframe>
<!-- postMessage nadal działa → ad może przesyłać tracking data do parent! -->

<!-- POPRAWKA: dla reklam użyj Fenced Frame (pełna izolacja): -->
<fencedframe></fencedframe>  <!-- + Protected Audience API -->
```

### Błąd 2: COEP: credentialless gdy COEP: require-corp jest lepszy

```http
# BŁĄD: używanie credentialless "bo łatwiejsze" gdy:
# - Wszystkie zasoby są kontrolowane przez nas (same-site CDN)
# - Zasoby mają CORP/CORS

Cross-Origin-Embedder-Policy: credentialless
# Nieco mniej bezpieczne niż require-corp (zasoby bez CORS/CORP ładowane bez credentials)

# POPRAWKA dla w pełni kontrolowanego środowiska:
Cross-Origin-Embedder-Policy: require-corp
```

### Błąd 3: Przekazywanie wrażliwych danych przez postMessage do credentialless iframe

```javascript
// PROBLEMATYCZNE: postMessage działa z credentialless iframe
// Jeśli iframe jest skompromitowany → może exfiltrować dane

const iframe = document.createElement('iframe');
iframe.src = 'https://third-party.com/widget';
iframe.credentialless = true;
iframe.addEventListener('load', () => {
    // UWAGA: third-party może nasłuchiwać na window.addEventListener('message')
    // i wysyłać te dane dalej!
    iframe.contentWindow.postMessage({ 
        sensitiveData: userPrivateInfo  // NIEBEZPIECZNE!
    }, 'https://third-party.com');
});

// POPRAWKA: nie wysyłaj wrażliwych danych do untrusted third-party iframes
// Używaj tylko niezbędnych danych (np. anonimowy ID sesji, kategoria, nie PII)
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona cookies przez izolację

```
Bez credentialless iframe:
- Iframe ładuje się z cookies third-party (SameSite=None)
- Atakujący (przez XSS na parent) może:
  document.querySelector('iframe').contentWindow.document.cookie
  → Dostęp do cookies iframe! (jeśli same-origin lub CORS pozwala)

Z credentialless:
- Iframe nie ma "prawdziwych" cookies
- Skradziony dostęp do iframe → puste/izolowane cookies
- Mniejszy impact
```

### Spectre mitigation przez COEP: credentialless

```
COEP: credentialless aktywuje crossOriginIsolated:
→ Proces izolacja (Site Isolation)
→ Cross-origin zasoby bez credentials = nie ma co "kraść" przez Spectre
  (dane i tak są anonimowe/puste)
→ SharedArrayBuffer bezpieczny
```

### Prywatność użytkownika

```
Third-party widget bez cookies:
+ Widget nie może budować profilu użytkownika przez cookies
+ Widget nie ma persistent session między stronami
+ Widget nie może korelować użytkownika ze swoją bazą (bez logu przez cookies)

- Widget może nadal budować fingerprint przez JS (User-Agent, screen size, itp.)
- Widget może używać first-party data przekazanego przez postMessage
```

### Powiązane CWE

- **CWE-359** — Privacy Violation (credentialless redukuje tracking)
- **CWE-200** — Information Exposure (izolacja cookies zmniejsza leak surface)
- **CWE-693** — Protection Mechanism Failure (wymagane przez COEP dla crossOriginIsolated)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź atrybut credentialless

```javascript
// W DevTools konsoli:
document.querySelectorAll('iframe[credentialless]');
// Jeśli wynik ma elementy → strona używa credentialless iframes

// Sprawdź COEP mode:
// Network tab → główny dokument → Response Headers
// Cross-Origin-Embedder-Policy: credentialless (lub require-corp)
```

### Test izolacji cookies

```javascript
// Wewnątrz credentialless iframe (przez DevTools → iframe context):
document.cookie;     // puste lub izolowane
localStorage;        // izolowane

// Sprawdź czy iframe może wysyłać cookies:
// Network tab → requesty z iframe → czy Cookies są wysyłane?
// Z credentialless: brak cookies w requestach!
```

### Test komunikacji postMessage

```javascript
// Test: czy postMessage działa (odróżnienie od Fenced Frames):
const iframe = document.querySelector('iframe[credentialless]');
try {
    iframe.contentWindow?.postMessage('test', '*');
    console.log('postMessage działa (credentialless, nie Fenced Frame)');
} catch (e) {
    console.log('postMessage zablokowane (Fenced Frame?)');
}
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy credentialless iframes nie mają dostępu do prawdziwych cookies third-party?
□ Czy COEP: credentialless aktywuje crossOriginIsolated?
□ Czy wrażliwe dane nie są przekazywane przez postMessage do credentialless iframes?
□ Czy credentialless jest użyte zamiast Fenced Frames gdy wymagana pełna izolacja?
□ Czy third-party widgets z credentialless mogą budować fingerprints?
□ Czy payment widgets działają poprawnie (mogą wymagać cookies → nie credentialless)?
□ Czy credentialless iframe może eksfiltrować dane przez postMessage?
□ Czy X-Frame-Options lub CSP frame-ancestors chroni credentialless embedded stronę?
□ Czy stara przeglądarka (bez credentialless) jest obsługiwana gracefully?
```

---

## 12. Jak się zabezpieczać

```html
<!-- === Bezpieczne credentialless iframe === -->

<!-- Tylko dla statycznych lub token-based third-party: -->
<iframe
    src="https://third-party.com/widget"
    credentialless
    sandbox="allow-scripts allow-same-origin"
    loading="lazy">
</iframe>

<!-- Nie wysyłaj PII przez postMessage: -->
<script>
iframe.addEventListener('load', () => {
    iframe.contentWindow.postMessage(
        { 
            sessionId: 'anonymous-' + crypto.randomUUID(),
            // NIE: userId, email, imię, itp.
        },
        'https://third-party.com'
    );
});

// Waliduj wiadomości z powrotem:
window.addEventListener('message', (e) => {
    if (e.origin !== 'https://third-party.com') return;
    if (typeof e.data !== 'object') return;
    // Obsłuż tylko oczekiwane wiadomości
});
</script>
```

```http
# Konfiguracja dla crossOriginIsolated z credentialless (łatwiejsze wdrożenie):
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: credentialless

# Dla stron które muszą embeddować YouTube/Maps/inne third-party bez COEP:
# → COEP: credentialless zamiast require-corp
```

```javascript
// === Migracja z require-corp na credentialless gdy third-party blokuje ===

// Krok 1: Sprawdź które iframy są blokowane:
// Console: "...blocked by COEP"

// Krok 2: Zmień COEP na credentialless:
// Cross-Origin-Embedder-Policy: credentialless

// Krok 3: Dodaj credentialless atrybut na iframach gdzie chcesz jawnej izolacji:
document.querySelectorAll('iframe').forEach(frame => {
    if (isThirdPartyIframe(frame)) {
        frame.credentialless = true;
    }
});

// Krok 4: Weryfikacja:
console.log('crossOriginIsolated:', crossOriginIsolated);  // powinno być true
console.log('SharedArrayBuffer:', typeof SharedArrayBuffer !== 'undefined');
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Credentialless iframe = `<iframe credentialless>` — ładuje się bez cookies i storage third-party
2. `COEP: credentialless` = alternatywa dla `require-corp` — umożliwia `crossOriginIsolated` bez wymagania COEP/CORS na third-party zasobach
3. W odróżnieniu od Fenced Frames: postMessage z parent DZIAŁA
4. Przydatne gdy third-party embed (YouTube, Maps) nie ma COEP, ale chcemy SharedArrayBuffer
5. Cookies są izolowane — widget widzi "anonimowego" użytkownika

**Dla pentestera:**
- Sprawdź `<iframe credentialless>` — cookies third-party są izolowane → zmniejszona attack surface
- Test: czy `postMessage` działa z iframe → tak (inaczej niż Fenced Frames)
- Sprawdź `COEP: credentialless` header → czy `crossOriginIsolated` jest true?
- Testuj czy wrażliwe dane są przekazywane przez postMessage do credentialless iframe (ryzyko exfiltracji!)
- Brak `credentialless` przy third-party → third-party dostaje swoje cookies → potencjalny tracking

---

## Powiązania

```
Credentialless iframes
    │
    ├──► COEP (Rozdział 43)
    │         COEP: credentialless = tryb bez credentials
    │         Alternatywa dla COEP: require-corp
    │
    ├──► Fenced Frames (Rozdział 48)
    │         Fenced Frames: pełna izolacja (brak postMessage)
    │         Credentialless: izolacja cookies ale postMessage działa
    │
    ├──► CHIPS (Rozdział 41)
    │         CHIPS: partitioned third-party cookies
    │         Credentialless: brak jakichkolwiek third-party cookies
    │
    └──► postMessage (Rozdział 23)
              postMessage działa w credentialless iframach
              Wymagana walidacja origin po obu stronach
```
