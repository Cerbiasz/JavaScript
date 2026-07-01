# Rozdział 47: Storage Access API

## 1. Czym jest Storage Access API

**Storage Access API** to interfejs przeglądarki umożliwiający stronom embeddowanym w iframach lub cross-site kontekstach żądanie dostępu do swojego **unpartitioned storage** — czyli cookies, localStorage i sessionStorage z własnego originu, w sytuacji gdy normalnie byłyby blokowane przez mechanizmy ochrony prywatności (ITP, Enhanced Tracking Protection, CHIPS).

Dwie kluczowe metody:

```javascript
// Sprawdź czy masz dostęp do unpartitioned storage:
const hasAccess = await document.hasStorageAccess();

// Zażądaj dostępu (wymaga gestu użytkownika):
await document.requestStorageAccess();
```

Od 2023 rozszerzony o SAA for unpartitioned cookies i Related Website Sets (obecnie Permissioned Web):

```javascript
// Rozszerzone API (Chrome 119+, Firefox 117+):
await document.requestStorageAccess({ cookies: true });
await document.requestStorageAccessFor('https://provider.example.com');  // top-level
```

Storage Access API jest używany przez embedded iframes (SSO, payment widgets, chat) które potrzebują dostępu do swojego własnego storage — żeby utrzymać sesję użytkownika który jest zalogowany u danego dostawcy.

---

## 2. Dlaczego powstał

### Problem: blokada third-party storage = broken SSO

Po wdrożeniu przez przeglądarki mechanizmów ochrony prywatności (ITP w Safari 2017+, ETP w Firefox 2019+, partycjonowanie w Chrome 2024+):

```
Scenariusz SSO bez Storage Access API:
1. Użytkownik jest zalogowany na sso.provider.com (pełna sesja w first-party kontekście)
2. Użytkownik odwiedza shop.com które embedduje sso.provider.com w iframe
3. Iframe sso.provider.com: document.cookie → PUSTE! (third-party storage blokowane)
4. Iframe nie widzi sesji → myśli że user jest niezalogowany → broken SSO!
```

Storage Access API (WebKit 2019, W3C Level 2 2022) rozwiązuje to przez:
- Danie embedowanym stronom możliwości jawnego żądania dostępu
- Wymuszenie **gestu użytkownika** i jego **zgody**
- Zachowanie granularnej kontroli prywatności (user musi zatwierdzić)

---

## 3. Jak działa

### hasStorageAccess()

```javascript
// W iframe sso.provider.com embeddowanym na shop.com:
const hasAccess = await document.hasStorageAccess();

if (hasAccess) {
    // Mamy dostęp do unpartitioned storage
    const sessionToken = getCookie('sso_session');
    // Sprawdź sesję...
} else {
    // Brak dostępu → poproś użytkownika
    await promptUserForAccess();
}
```

### requestStorageAccess()

```javascript
// Musi być wywołane w odpowiedzi na gest użytkownika (klik, tap):
button.addEventListener('click', async () => {
    try {
        await document.requestStorageAccess();
        // Dostęp przyznany!
        console.log('Storage access granted');
        // Teraz document.cookie, localStorage, IndexedDB działają
        // z unpartitioned storage sso.provider.com
        
    } catch (err) {
        // Odmowa lub brak gestu
        console.error('Storage access denied:', err);
    }
});
```

### Warunki przyznania dostępu

Przeglądarki różnie implementują politykę, ale ogólnie:

```
Safari (ITP):
1. User musiał poprzednio odwiedzić provider.com (first-party)
2. Opóźnienie: pierwsza prośba → dialog zgody
3. Przeniesialne między powiązanymi domenami (Related Website Sets)

Firefox (ETP):
1. Dialog pozwolenia wyświetlany użytkownikowi
2. "Allow [sso.provider.com] to use its cookies?"
3. Automatyczne przyznanie jeśli user poprzednio przyznał

Chrome (Storage Partitioning):
1. Wymaga gestu użytkownika
2. Pokazuje Permission prompt przy pierwszym żądaniu
3. Related Website Sets (RWS) = automatyczne przyznanie wewnątrz zestawu
```

### Related Website Sets (RWS / First-Party Sets)

```json
// /.well-known/related-website-set.json na primary domain:
{
    "primary": "https://primary.example",
    "associatedSites": ["https://associated1.example", "https://associated2.example"],
    "serviceSites": ["https://services.example"],
    "rationaleBySite": {
        "https://associated1.example": "Powiązany produkt tej samej firmy"
    }
}

// Chrome: requestStorageAccess() w ramach RWS → automatyczna zgoda (bez promptu!)
// Poza RWS: dialog zgody
```

---

## 4. Co dzieje się wewnętrznie

### Storage Access Grant

```
requestStorageAccess() → grant'owany?

Zależy od:
1. Czy jest gest użytkownika? (bez gestu → odrzucone automatycznie)
2. Czy user poprzednio odwiedzał provider.com? (heurystyki ITP)
3. Czy jest Previous Grant? (user już przyznał dostęp)
4. Czy jest RWS/Related Website Sets? (automatyczne wewnątrz zestawu)
5. Browser-specific heurystyki

Jeśli grant: dostęp aktywny dla tej sesji iframe (nie persystentny!)
Przy refresh: trzeba prosić znowu (lub przeglądarka może cachować grant)
```

### Storage Access vs Storage Partitioning

```
Storage Partitioning (CHIPS, Rozdział 41):
- Cookie wyizolowane per top-level site
- Automatyczne, bez user gesture
- "Partitioned storage" — nowa, ograniczona sesja per site

Storage Access API:
- Dostęp do UNPARTITIONED storage (globalnej sesji)
- Wymaga user gesture + grant
- "Otwórz bramkę" do pełnego storage
```

### requestStorageAccessFor() — top-level API

```javascript
// Na top-level stronie (shop.com), request dostępu dla embedded resource:
// (Chrome 119+)
await document.requestStorageAccessFor('https://sso.provider.com');
// Wymaga: RWS membership lub explicit user permission
// Pozwala serwisowi shop.com żądać dostępu w imieniu sso.provider.com
```

---

## 5. Analogiczny przykład z życia

Storage Access API to **przepustka do archiwum w budynku**:

- Gość (embedded iframe sso.provider.com) chce dostać się do archiwum (unpartitioned cookies)
- Normalna procedura: archiwum zamknięte dla gości (third-party storage blokowane)
- Właściciel biura (user) może wystawić przepustkę jednorazową (grant po geście)
- "Miałeś tu wcześniej biuro?" (heurystyki — user musiał odwiedzić provider.com)
- RWS: związane firmy w tym samym budynku → przepustka automatyczna dla pracowników grupy
- Przepustka wygasa po sesji — przy następnej wizycie trzeba ponownie prosić

---

## 6. Przykład kodu

```javascript
// === Kompletna implementacja Storage Access w SSO widget ===

class SSOWidget {
    constructor() {
        this.sessionToken = null;
    }
    
    async init() {
        // 1. Sprawdź czy mamy storage access:
        const hasAccess = await document.hasStorageAccess();
        
        if (hasAccess) {
            return this.loadSession();
        }
        
        // 2. Spróbuj cichego requestu (bez gestu):
        // Nie jest możliwe! requestStorageAccess WYMAGA gestu
        
        // 3. Pokaż UI dla użytkownika:
        this.showLoginButton();
    }
    
    showLoginButton() {
        const button = document.createElement('button');
        button.textContent = 'Zaloguj przez SSO';
        button.addEventListener('click', () => this.requestAccess());
        document.body.appendChild(button);
    }
    
    async requestAccess() {
        // To jest gest użytkownika → requestStorageAccess może być wywołane
        try {
            await document.requestStorageAccess();
            await this.loadSession();
        } catch (err) {
            console.error('Odmowa dostępu:', err);
            this.showError('Dostęp do sesji odmówiony. Sprawdź ustawienia przeglądarki.');
        }
    }
    
    async loadSession() {
        // Po uzyskaniu dostępu — cookies z sso.provider.com są dostępne:
        const token = document.cookie.match(/sso_token=([^;]+)/)?.[1];
        
        if (token) {
            this.sessionToken = token;
            this.showLoggedIn();
        } else {
            this.showLoginForm();
        }
    }
    
    showLoggedIn() {
        // UI zalogowanego użytkownika
    }
    
    showLoginForm() {
        // Formularz logowania — user nie jest zalogowany w SSO
    }
    
    showError(message) {
        console.error(message);
    }
}

// Inicjalizacja:
const sso = new SSOWidget();
sso.init();
```

```javascript
// === Feature Detection i Fallback ===

async function checkStorageAccess() {
    // Sprawdź czy API jest dostępne:
    if (typeof document.hasStorageAccess === 'undefined') {
        console.log('Storage Access API niedostępne (stara przeglądarka)');
        // Fallback: użyj redirect-based OAuth (popup lub redirect)
        return useRedirectAuth();
    }
    
    // Sprawdź aktualny stan:
    const hasAccess = await document.hasStorageAccess();
    console.log('Storage access:', hasAccess);
    
    // Sprawdź czy RequestStorageAccessFor jest dostępne (Chrome 119+):
    if (document.requestStorageAccessFor) {
        console.log('Top-level SAA dostępne');
    }
    
    return hasAccess;
}

// Feature detection dla RWS:
async function checkRWS() {
    // Nie ma bezpośredniego API do sprawdzenia RWS membership
    // Ale requestStorageAccess() bez promptu = w RWS
    try {
        await document.requestStorageAccess();
        // Sukces bez promptu = automatyczny grant (RWS lub previous grant)
        return { granted: true, fromRWS: true };
    } catch {
        return { granted: false, fromRWS: false };
    }
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Embedded Payment Widget

```
Scenariusz: stripe.com embeddowany na shop.com
1. User jest zalogowany na stripe.com (ma tam konto)
2. shop.com embedduje Stripe payment form w iframe
3. Iframe stripe.com: hasStorageAccess() → false (Safari ITP)
4. Iframe pyta: "Zapamiętaj kartę?" → ale nie może czytać zapisanych kart!
5. requestStorageAccess() → dialog dla usera → zgoda → dostęp do cookies stripe.com
6. Teraz iframe widzi zapisane karty użytkownika
```

### Exploit: clickjacking + Storage Access

```html
<!-- Potencjalny atak: ukryty iframe + clickjacking na przycisk SAA: -->
<iframe 
    src="https://sso.provider.com/widget"
    style="opacity: 0; position: absolute; top: 0; left: 0; z-index: 9999;"
></iframe>

<!-- Widoczny przycisk dekoy: -->
<button style="position: absolute; top: 0; left: 0;">
    Wygraj nagrodę! Kliknij tutaj!
</button>

<!-- User myśli że klika przycisk dekoy ale tak naprawdę klika przycisk SAA w iframe! -->
<!-- requestStorageAccess() dostaje "gest użytkownika" → grant! -->
<!-- Atakujący teraz ma dostęp do cookies sso.provider.com przez swój iframe! -->
```

**Ochrona przeglądarek:** Przeglądarki implementują dodatkowe zabezpieczenia:
- Safari: wymaga **wcześniejszej interakcji** użytkownika z provider.com
- Chrome: sprawdza czy request jest w "trusted" iframe (nie ukryty)
- Firefox: dialog musi być widoczny dla użytkownika

Dodatkowo: X-Frame-Options i CSP frame-ancestors na sso.provider.com chronią przed embeddowaniem na złośliwych stronach.

### Phishing przez SAA

```javascript
// Phishing strona embedduje legitimate SSO i prosi o SAA:
// "Zaloguj się przez Google" (wyglądający oryginalnie)
// requestStorageAccess() → dialog przeglądarki
// User potwierdza → phishing.com teraz ma dostęp do SWOICH cookies
// (nie do Google cookies! SAA daje dostęp do storage phishing.com, nie Google!)

// Wniosek: SAA nie umożliwia kradzieży storage innego originu
// Daje tylko dostęp do własnego storage (sso.provider.com → sso.provider.com cookies)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Wywołanie requestStorageAccess bez gestu użytkownika

```javascript
// BŁĄD: wywoływanie poza event handlerem
window.addEventListener('load', async () => {
    await document.requestStorageAccess(); // Odrzucone! Brak gestu
});

// POPRAWKA: tylko w event handlerze gestu:
button.addEventListener('click', async () => {
    await document.requestStorageAccess(); // OK!
});
```

### Błąd 2: Brak obsługi odrzucenia

```javascript
// BŁĄD: brak try/catch → unhandled rejection
await document.requestStorageAccess();
// Jeśli odrzucone → nieobsłużony promise rejection

// POPRAWKA:
try {
    await document.requestStorageAccess();
} catch (err) {
    // User odmówił lub brak wymaganych warunków
    showFallbackLoginFlow();
}
```

### Błąd 3: Zakładanie że grant jest persystentny

```javascript
// BŁĄD: brak ponownego sprawdzenia przy refresh
// Grant może wygasnąć lub być ograniczony do sesji

async function init() {
    // BŁĄD: zakładamy że poprzedni grant nadal działa
    const data = getCookieData();  // może być puste po refresh!
    
    // POPRAWKA: zawsze sprawdzaj hasStorageAccess():
    const hasAccess = await document.hasStorageAccess();
    if (!hasAccess) {
        await requestAccess();
    }
    const data = getCookieData();
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przez wymaganie gestu użytkownika

```
requestStorageAccess() BEZ gestu → automatyczne odrzucenie
→ Skrypt działający w tle nie może "po cichu" uzyskać dostępu
→ Wymaga świadomej interakcji użytkownika

Atakujący przez XSS:
→ Może wywołać requestStorageAccess() przez XSS
→ Ale: dialog przeglądarki jest wymagany (UI prompt)
→ User musi zatwierdzić → trudny do ukrycia atak
```

### Clickjacking i SAA

```
Ryzyko: clickjacking aby przymusić user do kliknięcia przycisku SAA
→ Przeglądarki implementują zabezpieczenia: wykrywanie hidden iframes
→ X-Frame-Options: DENY / SAMEORIGIN na stronach SSO → uniemożliwia embedding
→ CSP frame-ancestors: kontroluje kto może embeddować
```

### Privacy Sandbox i SAA

```
SAA jest częścią Privacy Sandbox:
- Nie całkowita blokada third-party cookies (jak Google chciało)
- Kompromis: user-gated dostęp do unpartitioned storage
- Related Website Sets: automatyczne zgody wewnątrz grupy firm

Kontrowersje:
- RWS może być nadużywane: firmy tworzą "sets" tylko dla ominięcia blokady
- Regulatorzy (UK CMA): nadzór nad RWS Chrome'a
```

### Powiązane CWE

- **CWE-352** — CSRF (SAA może ułatwiać cross-site requests)
- **CWE-1021** — Clickjacking (ryzyko przy SAA UI)
- **CWE-200** — Information Exposure (storage access = potencjalny tracking)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź czy Storage Access jest używane

```javascript
// W konsoli iframe na target.com:
typeof document.hasStorageAccess;    // 'function' jeśli obsługiwane
typeof document.requestStorageAccess; // 'function'

// Sprawdź stan:
document.hasStorageAccess().then(v => console.log('Has access:', v));
```

### Test clickjacking na SAA

```html
<!-- Test: czy można osadzić stronę z SAA w ukrytym iframe? -->
<iframe src="https://target.com/widget" style="opacity: 0.01; width: 200px; height: 50px;">
</iframe>
<!-- Sprawdź: czy X-Frame-Options lub CSP frame-ancestors blokuje? -->
<!-- Jeśli nie → potencjalny clickjacking risk na SAA przyciski -->
```

### Sprawdź Related Website Sets

```bash
# Sprawdź czy target.com deklaruje RWS:
curl https://target.com/.well-known/related-website-set.json
# lub: https://target.com/.well-known/first-party-set.json (stara nazwa)

# Sprawdź czy powiązane domeny mają odpowiednie nagłówki
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy requestStorageAccess() jest wywoływane tylko po geście użytkownika?
□ Czy hasStorageAccess() jest sprawdzane przed requestem?
□ Czy grant expiresowania jest obsługiwane (sprawdzanie przy powrocie)?
□ Czy strona embeddująca ma X-Frame-Options lub CSP frame-ancestors?
□ Czy błędy requestStorageAccess (odmowa) są obsługiwane gracefully?
□ Czy SAA nie jest używane do celów trackingowych (naruszenie prywatności)?
□ Czy RWS membership jest uzasadnione?
□ Czy fallback działia gdy SAA jest odmówiony/niedostępny?
□ Czy UI dla requestu storage access nie jest misleading (dark patterns)?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Bezpieczna implementacja Storage Access ===

const StorageAccessManager = {
    async initialize() {
        if (!document.hasStorageAccess) {
            return { supported: false, hasAccess: false };
        }
        
        const hasAccess = await document.hasStorageAccess();
        return { supported: true, hasAccess };
    },
    
    async requestWithUserGesture() {
        // MUSI być wywołane w event handler:
        try {
            await document.requestStorageAccess();
            return true;
        } catch (err) {
            // Odmowa lub inne error
            return false;
        }
    },
};

// Ochrona przed embeddowaniem (po stronie SSO provider):
// HTTP Headers:
// X-Frame-Options: SAMEORIGIN
// Content-Security-Policy: frame-ancestors 'self' https://trusted-shop.com
```

```http
# Nagłówki ochrony dla SSO provider:
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: frame-ancestors 'self' https://trusted-partner.com

# Lub pozwól embeddowanie tylko przez RWS:
# (konfiguracja przez Chrome's Related Website Sets mechanism)
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Storage Access API = `document.hasStorageAccess()` + `document.requestStorageAccess()` dla embedded iframes
2. Pozwala na dostęp do **unpartitioned** (globalnego) storage cross-site, z wymaganiem gestu użytkownika
3. Wymagany do zachowania SSO/login state gdy third-party cookies są blokowane
4. Related Website Sets (RWS): automatyczne granty wewnątrz zadeklarowanej grupy domen
5. Ryzyko: clickjacking na przyciski SAA — ochrona przez X-Frame-Options i CSP frame-ancestors

**Dla pentestera:**
- Sprawdź czy SSO iframe wymaga SAA i czy implementacja jest poprawna
- Testuj clickjacking: czy stronę SSO można osadzić w ukrytym iframe?
- Sprawdź brak gestu użytkownika → czy requestStorageAccess() jest wywoływane automatycznie?
- Brak X-Frame-Options/CSP frame-ancestors na SSO = clickjacking risk

---

## Powiązania

```
Storage Access API
    │
    ├──► CHIPS (Rozdział 41)
    │         CHIPS: partitioned cookies bez user gesture
    │         SAA: dostęp do unpartitioned cookies z user gesture
    │         Komplementarne podejścia
    │
    ├──► Cookies (Rozdział 14)
    │         SAA daje dostęp do unpartitioned cookies
    │         HttpOnly, SameSite, Secure nadal obowiązują
    │
    ├──► Credential Management API (Rozdział 45)
    │         FedCM: alternatywa dla SAA w kontekście logowania
    │         Oba rozwiązują problem third-party cookie block
    │
    └──► Fenced Frames (Rozdział 48)
              Fenced Frames: bardziej restrykcyjny embed bez storage access
              Alternatywa dla embeddowania z SAA
```
