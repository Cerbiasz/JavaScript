# Rozdział 41: CHIPS (Cookies Having Independent Partitioned State)

## 1. Czym jest CHIPS

**CHIPS** (Cookies Having Independent Partitioned State) to mechanizm przeglądarki umożliwiający stronom trzecim (third-party) ustawianie cookies które są **partycjonowane** — czyli izolowane per top-level site. Cookie ustawione przez `widget.com` gdy jest embeddowane na `shop.com` jest dostępne tylko gdy `widget.com` jest na `shop.com` — a nie gdy jest embeddowane na `news.com`. Eliminuje to cross-site tracking przy zachowaniu legitimate use cases.

CHIPS jest implementowane przez atrybut `Partitioned` na ciasteczku:

```http
Set-Cookie: __Host-session=TOKEN; SameSite=None; Secure; Partitioned
```

CHIPS stanowi odpowiedź na deprecację klasycznych third-party cookies (blokowanych od Chrome 2024+) i część Privacy Sandbox Google. Pozwala embedded widgets, CDN session tokens i podobnym scenariuszom nadal działać — ale bez możliwości śledzenia użytkownika między domenami.

---

## 2. Dlaczego powstał

### Problem: third-party cookies = cross-site tracking

Klasyczne third-party cookie:

```
Użytkownik odwiedza shop.com → iframe ads.com ustawia cookie tracking_id=USER123
Użytkownik odwiedza news.com → iframe ads.com widzi ten sam cookie tracking_id=USER123
Użytkownik odwiedza bank.com → iframe ads.com: tracking_id=USER123

Wynik: ads.com zna pełną historię przeglądania użytkownika!
```

To jest fundamentalny mechanizm cross-site tracking i budowania profili użytkownika. Regulacje (GDPR, CCPA) i naciski użytkowników sprawiły że przeglądarki zaczęły blokować third-party cookies:
- Safari: ITP (Intelligent Tracking Prevention) od 2017
- Firefox: Enhanced Tracking Protection od 2019
- Chrome: Phase-out 2024+

Jednak całkowita blokada niszczy **legitimate** use cases:
- Customer support chat widget (potrzebuje sesji gdy embedowany na różnych stronach klienta)
- Mapy embeddowane (sesja dla zalogowanych użytkowników)
- CDN load balancing cookies
- Federated login (OAuth, SSO)
- Embedded payment forms

CHIPS (2022, Chrome 114+) rozwiązuje ten dylemat: cookie jest izolowane do top-level site gdzie zostało ustawione.

---

## 3. Jak działa

### Partycjonowanie cookies

Bez CHIPS (klasyczny third-party cookie):
```
Cookie key: (domain=widget.com, name=session)
Wartość: USER123

Dostęp z shop.com: widget.com → session=USER123 ✓
Dostęp z news.com: widget.com → session=USER123 ✓ (ŚLEDZENIE!)
```

Z CHIPS (partitioned cookie):
```
Cookie key: (domain=widget.com, partition-key=shop.com, name=session)
Wartość: SHOP_SESSION_XYZ

Cookie key: (domain=widget.com, partition-key=news.com, name=session)
Wartość: NEWS_SESSION_ABC

Dostęp z shop.com: widget.com → session=SHOP_SESSION_XYZ ✓
Dostęp z news.com: widget.com → session=NEWS_SESSION_ABC ✓ (INNE cookie!)
→ Widget.com nie może połączyć aktywności na shop.com i news.com!
```

### Atrybut Partitioned

```http
# Ustawianie partitioned cookie:
Set-Cookie: __Host-widget_session=TOKEN; 
            SameSite=None; 
            Secure; 
            Path=/;
            Partitioned

# WAŻNE: Partitioned WYMAGA:
# - SameSite=None (cross-site cookie)
# - Secure (tylko HTTPS)
# - Zalecane: __Host- prefix (wymusza Path=/ i brak Domain)
```

### Partition key

Partition key to top-level site (schemat + eTLD+1) strony w której jest embeddowany widget:

```
Użytkownik na https://shop.example.com
  → embed: https://widget.com/chat
  → Partition key: https://example.com

Użytkownik na https://news.example.org
  → embed: https://widget.com/chat  
  → Partition key: https://example.org
```

### Same-origin i partitioned

Jeśli widget i strona główna są same-site, CHIPS nie jest potrzebne (SameSite=Lax/Strict wystarczy). CHIPS jest dla **cross-site** embedded zasobów.

---

## 4. Co dzieje się wewnętrznie

### Cookie jar z partycją

Przeglądarka przechowuje cookies w "cookie jar" z dodatkowym kluczem partycji:

```
Cookie Store:
┌─────────────────────────────────────────────────────┐
│ (widget.com, shop.com) → __Host-session = SHOP_ABC  │
│ (widget.com, news.com) → __Host-session = NEWS_XYZ  │
│ (widget.com, null)     → __Host-old_session = TRACK │ ← unpartitioned (BLOKOWANE!)
└─────────────────────────────────────────────────────┘
```

Przeglądarka wysyłając request do `widget.com` z kontekstu `shop.com` filtruje tylko cookies z partition key = `shop.com`.

### Walidacja przez przeglądarkę

```
Jeśli ciasteczko ma Partitioned ale brak SameSite=None lub Secure:
    → przeglądarka odrzuca Set-Cookie (ignoruje)

Jeśli ciasteczko ma Partitioned i jest same-origin:
    → przeglądarka może przyjąć (same-origin partitioned cookie też możliwe)
```

### JavaScript i partitioned cookies

```javascript
// JavaScript w iframe widget.com na shop.com:
document.cookie  // widzi tylko cookies z partition key=shop.com

// Fetch z credentials:
fetch('/api/session', { credentials: 'include' });
// Wysyła tylko partitioned cookies dla tego kontekstu
```

---

## 5. Analogiczny przykład z życia

CHIPS to **karta lojalnościowa** w centrum handlowym:

- W centrum handlowym (top-level site) każdy sklep (third-party widget) może wydać ci kartę lojalnościową (cookie)
- Karta działa TYLKO w tym centrum handlowym — nie możesz jej użyć w innym centrum
- Posiadanie karty w jednym centrum nie ujawnia twoich zakupów drugiemu centrum
- Sklep (widget) może znać twoją historię w tym konkretnym centrum — ale nie we wszystkich centrach na świecie (brak global tracking)

---

## 6. Przykład kodu

```javascript
// === Iframe widget: ustawianie partitioned cookie przez JS ===
// (tylko za pomocą Set-Cookie header, nie document.cookie dla Partitioned!)

// Server-side: widget.com serwer ustawia cookie:
// Ten header musi pojawić się w response z widget.com
```

```python
# === Backend widget.com (Python/Flask): ===

@app.route('/widget/init')
def widget_init():
    response = make_response(render_template('widget.html'))
    
    # Partitioned cookie — izolowane do top-level site gdzie widget jest embedowany
    response.headers.add(
        'Set-Cookie',
        '__Host-widget_session={}; SameSite=None; Secure; Path=/; Partitioned'.format(
            generate_session_token()
        )
    )
    return response
```

```javascript
// === Node.js/Express: ustawianie partitioned cookie ===

app.get('/widget/session', (req, res) => {
    const sessionToken = generateSecureToken();
    
    // Ręcznie konstruuj header bo Express nie ma natywnego 'partitioned' option
    res.setHeader(
        'Set-Cookie',
        `__Host-widget_session=${sessionToken}; SameSite=None; Secure; Path=/; Partitioned`
    );
    
    res.json({ status: 'session initialized' });
});
```

```html
<!-- === Top-level site (shop.com): embeddowanie widgetu === -->
<!DOCTYPE html>
<html>
<head><title>Shop</title></head>
<body>
    <!-- Widget customer support: -->
    <iframe 
        src="https://widget.com/chat"
        allow="storage-access"
        style="width: 300px; height: 400px;">
    </iframe>
    
    <!-- Widget.com może teraz ustawiać __Host-widget_session z Partitioned -->
    <!-- Cookie będzie izolowane do partition key = shop.com -->
</body>
</html>
```

```javascript
// === Sprawdzenie wsparcia CHIPS w JavaScript ===

// Sprawdź czy przeglądarka wspiera partitioned cookies:
async function checkChipsSupport() {
    // Metoda 1: User-Agent parsing (mało eleganckie)
    // Metoda 2: Feature detection przez Storage Access API
    
    try {
        // Storage Access API (powiązane z CHIPS):
        const hasAccess = await document.hasStorageAccess();
        console.log('Storage access:', hasAccess);
    } catch (e) {
        console.log('Storage Access API not supported');
    }
    
    // Sprawdź cookie po ustawieniu:
    // Jeśli partitioned cookie zostało przyjęte, będzie widoczne w tym kontekście
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Customer Support Chat Widget

```
Bez CHIPS (cross-site tracking):
shop.com → chat.widget.com: session_id=USER_PROFILE_123
news.com → chat.widget.com: session_id=USER_PROFILE_123  ← TE SAME DANE!
Widget widzi że USER_PROFILE_123 odwiedza shop.com i news.com

Z CHIPS:
shop.com → chat.widget.com: session_id=SHOP_SESSION_XYZ
news.com → chat.widget.com: session_id=NEWS_SESSION_ABC  ← RÓŻNE!
Widget nie może połączyć aktywności między domenami
```

### CDN Load Balancing

```http
# CDN (np. Cloudflare) ustawia cookie do sticky sessions:
Set-Cookie: __cflb=BACKEND_ID; SameSite=None; Secure; Partitioned

# Bez Partitioned: BACKEND_ID widoczny na wszystkich stronach używających Cloudflare
# Z Partitioned: BACKEND_ID izolowany per top-level site
```

### Embedded Maps / Analytics

```javascript
// Google Maps lub inna usługa map embeddowana:
// Zalogowany użytkownik Google może widzieć swoje "saved places"

// Bez CHIPS: Google wie że ten sam użytkownik jest na shop.com i news.com
// Z CHIPS: sesja Googlee izolowana do top-level kontekstu
// (choć Google może użyć FPS — First-Party Sets dla własnych domen)
```

### Third-party Login / SSO

```
Problem z CHIPS dla SSO:
1. user.com embedduje auth.sso.com w iframe
2. SSO ustawia partitioned cookie
3. Gdy user.com redirectuje do sso.com (top-level), partitioned cookie z iframe 
   NIE jest dostępne (inny kontekst)

Rozwiązanie: Storage Access API lub First-Party Sets (FPS/RWS)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Partitioned bez Secure lub SameSite=None

```http
# BŁĄD: brak wymaganych atrybutów
Set-Cookie: session=TOKEN; Partitioned
# Przeglądarka odrzuca! Partitioned wymaga SameSite=None i Secure

# POPRAWKA:
Set-Cookie: __Host-session=TOKEN; SameSite=None; Secure; Path=/; Partitioned
```

### Błąd 2: Niezrozumienie że unpartitioned cookies są blokowane

```javascript
// BŁĄD: oczekiwanie że stary third-party cookie nadal działa
// W Chrome 2024+: third-party cookies bez Partitioned = BLOKOWANE!

// Widget próbuje ustawić:
Set-Cookie: session=TRACK; SameSite=None; Secure
// W Chrome 2024+ z Phase-out → cookie jest blokowane w cross-site kontekście!

// POPRAWKA: dodaj Partitioned lub zrefaktoruj bez cookies
```

### Błąd 3: Użycie partitioned cookies dla sharing danych między domenami

```javascript
// BŁĄD koncepcyjny: próba udostępnienia danych między domain.com i partner.com
// przez partitioned cookie

// Partitioned celowo uniemożliwia to!
// Widget.com na domain.com != widget.com na partner.com → różne partycje

// Alternatywy dla legitimate cross-site data sharing:
// - Storage Access API (user consent)
// - Related Website Sets / First-Party Sets
// - Federated Identity (FedCM)
```

### Błąd 4: Brak migracji z unpartitioned na partitioned

```javascript
// BŁĄD: stary widget używa SameSite=None bez Partitioned
// Po Chrome phase-out → widget przestaje działać

// Plan migracji:
// 1. Audit: które cookies są cross-site?
// 2. Dodaj Partitioned do cookies które powinny być izolowane
// 3. Dla cookies wymagających cross-site sharing → Storage Access API lub FedCM
// 4. Testuj w Chrome z flagą: chrome://flags/#test-third-party-cookie-phaseout
```

---

## 9. Znaczenie dla bezpieczeństwa

### Eliminacja cross-site tracking

CHIPS fundamentalnie eliminuje możliwość śledzenia użytkownika przez third-party cookies:

```
Bez CHIPS: reklamy na 1000 stronach → jeden cookie → pełny profil użytkownika
Z CHIPS: reklamy na 1000 stronach → 1000 różnych partitioned cookies → brak profilu
```

### Zmniejszenie surface area dla cookie theft

```
Jeśli atakujący kradnie session cookie widget.com:
- Bez Partitioned: kradnie GLOBALNĄ sesję (cross-site tracking danych)
- Z Partitioned: kradnie sesję TYLKO dla jednego top-level site

Skradziony token ma znacznie mniejszy impact.
```

### Information leakage przez unpartitioned

```javascript
// Bez CHIPS — side-channel information:
// Atakujący na evil.com embedduje widget.com (SameSite=None)
// Widget.com widzi cookie z poprzednich wizyt na innych stronach
// → może ujawnić że użytkownik odwiedzał competitor.com (przez user profile)

// Z CHIPS: evil.com widzi tylko evil.com-partitioned cookies
// → brak cross-site information leakage
```

### Privacy Sandbox i CHIPS

CHIPS jest częścią Google Privacy Sandbox — zestawu API zastępujących third-party cookies:
- **CHIPS**: partitioned cookies
- **FLoC → Topics API**: grupowanie zainteresowań bez indywidualnego trackingu
- **FLEDGE (Protected Audience)**: reklamy bez ujawniania danych
- **Attribution Reporting**: konwersje reklam bez cross-site tracking
- **Federated Identity (FedCM)**: SSO bez trzeciej strony trackującej

### Powiązane CWE

- **CWE-614** — Sensitive Cookie in HTTPS Session Without 'Secure' Attribute
- **CWE-1004** — Sensitive Cookie Without 'HttpOnly' Flag
- Nie ma dedykowanego CWE dla CHIPS — to mechanizm prywatności, nie typowa podatność

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź Set-Cookie headers

```bash
# Sprawdź czy third-party cookies są partitioned:
curl -c /dev/null https://widget.example.com/init \
    -H "Referer: https://shop.com" | grep -i "set-cookie"

# Szukaj:
# - SameSite=None bez Partitioned → unpartitioned cross-site cookie (tracking?)
# - Partitioned → CHIPS (izolowane)
# - Brak SameSite=None na cross-site cookies → może nie działać poprawnie
```

### DevTools → Application → Cookies

```
1. Na stronie z embeddowanym third-party:
2. DevTools → Application → Cookies
3. Kliknij na domenę third-party
4. Szukaj kolumny "Partition Key" — jeśli jest wartość → partitioned cookie
5. Brak Partition Key przy third-party cookie → unpartitioned (problematyczne)
```

### Test tracking

```javascript
// Sprawdź czy widget może śledzić między domenami:
// 1. Wejdź na site-a.com z embeddem widget.com
// 2. Sprawdź cookie widget.com → zapisz wartość
// 3. Wejdź na site-b.com z embeddem widget.com  
// 4. Sprawdź cookie widget.com → czy ta sama wartość?

// Jeśli CHIPS: wartości RÓŻNE ✓ (poprawne)
// Jeśli nie CHIPS: wartości IDENTYCZNE → cross-site tracking!
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy third-party cookies mają atrybut Partitioned?
□ Czy Partitioned cookies mają SameSite=None i Secure?
□ Czy unpartitioned SameSite=None cookies nadal istnieją (legacy)?
□ Czy widget działa po Chrome third-party cookie phase-out?
□ Czy cookie rozmiar nie przekracza 10KB (Chrome limit dla partitioned)?
□ Czy nie używasz partitioned cookies do cross-site data sharing (niemożliwe!)?
□ Czy Storage Access API jest używane tam gdzie shared state jest potrzebny?
□ Czy widget nie zakłada dostępu do unpartitioned cookies?
```

---

## 12. Jak się zabezpieczać

```http
# Prawidłowa konfiguracja third-party cookies z CHIPS:
Set-Cookie: __Host-widget_session=TOKEN; 
            SameSite=None; 
            Secure; 
            Path=/; 
            Partitioned;
            Max-Age=3600

# Dla cookie które NIE potrzebują cross-site → same-site cookie:
Set-Cookie: __Host-user_pref=dark; SameSite=Lax; Secure; HttpOnly; Path=/
```

```javascript
// === Node.js: helper do ustawiania partitioned cookies ===

function setPartitionedCookie(res, name, value, options = {}) {
    const {
        maxAge = 3600,
        path = '/',
        httpOnly = true,
    } = options;
    
    // Partitioned cookie wymaga ręcznego header bo większość frameworków
    // nie ma natywnego wsparcia (2024)
    const cookieParts = [
        `${name}=${encodeURIComponent(value)}`,
        `SameSite=None`,
        `Secure`,
        `Path=${path}`,
        `Partitioned`,
        `Max-Age=${maxAge}`,
    ];
    
    if (httpOnly) cookieParts.push('HttpOnly');
    
    res.setHeader('Set-Cookie', cookieParts.join('; '));
}

// Użycie:
app.get('/widget/init', (req, res) => {
    setPartitionedCookie(res, '__Host-widget_session', generateToken(), {
        maxAge: 3600,
        httpOnly: true,
    });
    res.json({ initialized: true });
});
```

```javascript
// === Feature detection i graceful degradation ===

// Na stronie widget.com:
async function initSession() {
    // Sprawdź czy unpartitioned cookies działają (starsze przeglądarki):
    const testResponse = await fetch('/test-cookie', { credentials: 'include' });
    const { cookieReceived } = await testResponse.json();
    
    if (!cookieReceived) {
        // Chrome 2024+ phase-out → użyj Storage Access API:
        if (typeof document.requestStorageAccess !== 'undefined') {
            await document.requestStorageAccess();
            // Teraz partitioned storage jest dostępny
        }
        // lub: użyj własnej session przez postMessage z parent frame
    }
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. CHIPS = `Partitioned` atrybut cookie — izoluje cross-site cookies per top-level site
2. Eliminuje cross-site tracking przez third-party cookies
3. Wymaga `SameSite=None` + `Secure` + `Partitioned` w Set-Cookie
4. Partition key = top-level site (schemat + eTLD+1) gdzie cookie zostało ustawione
5. Chrome Phase-out 2024: unpartitioned third-party cookies → blokowane; CHIPS = legalna alternatywa

**Dla pentestera:**
- Sprawdź `Set-Cookie` w iframe/embedded zasobach: `Partitioned` obecny?
- Brak `Partitioned` na `SameSite=None` cookie → możliwe cross-site tracking
- Testuj czy te same cookie wartości pojawiają się na różnych top-level sites (brak CHIPS → tracking)
- DevTools → Application → Cookies → sprawdź kolumnę "Partition Key"

---

## Powiązania

```
CHIPS (Partitioned Cookies)
    │
    ├──► SameSite Cookies (Rozdział 40)
    │         SameSite=None wymagane dla Partitioned
    │         CHIPS uzupełnia, nie zastępuje SameSite
    │
    ├──► Storage Access API (Rozdział 47)
    │         SAA: alternatywny mechanizm dostępu do unpartitioned storage
    │         CHIPS: izoluje cookies bez potrzeby user gesture
    │
    ├──► Cookies (Rozdział 14)
    │         Podstawy cookie security: HttpOnly, Secure, SameSite
    │
    └──► Fenced Frames (Rozdział 48)
              Kolejny element Privacy Sandbox
              Bezpieczne embedding bez cross-site data sharing
```
