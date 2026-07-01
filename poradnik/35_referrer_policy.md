# Rozdział 35: Referrer Policy

## 1. Czym jest Referrer Policy

**Referrer Policy** to mechanizm kontrolujący ile informacji o aktualnym URL strony jest przekazywane w nagłówku `Referer` (celowe literówka w specyfikacji HTTP) przy nawigacji do innych stron lub przy wysyłaniu requestów.

Nagłówek `Referer` jest automatycznie wysyłany przez przeglądarkę i zawiera URL strony z której pochodzi request — może to ujawnić:
- Prywatne URL z wrażliwymi parametrami (`/reset-password?token=abc`)
- Wewnętrzne URL systemów
- User ID, session ID w URL
- Treść wyszukiwania (`?q=confidential-project`)

Referrer Policy pozwala na kontrolę nad tymi informacjami.

---

## 2. Dlaczego powstał

### Problem: Referer header ujawnia wrażliwe URL

Klasyczny scenariusz: aplikacja bankowa z URL `/account?token=temporarySecret`. Gdy użytkownik kliknie link do zewnętrznej strony (np. reklama) lub strona ładuje zewnętrzny zasób (obraz, skrypt):

```
GET /external-image.png HTTP/1.1
Host: ads.example.com
Referer: https://bank.com/reset-password?token=abc123xyz
```

Token resetowania hasła trafił do logów reklamodawcy!

Referrer Policy (W3C, 2014) daje twórcom stron kontrolę nad tym zachowaniem.

---

## 3. Jak działa

### Wartości Referrer Policy

```
no-referrer
    → Referer NIE jest wysyłany wcale (nigdy)

no-referrer-when-downgrade  (domyślna wartość przeglądarek)
    → Referer wysyłany dla HTTPS→HTTPS i HTTP→HTTP
    → NIE wysyłany dla HTTPS→HTTP (downgrade)

origin
    → Wysyła tylko ORIGIN (protokół+host+port): https://example.com
    → Nie wysyła ścieżki ani query string

origin-when-cross-origin
    → Same-origin requests: pełny URL
    → Cross-origin requests: tylko origin

same-origin
    → Tylko same-origin requests: pełny URL
    → Cross-origin: brak Referer

strict-origin
    → HTTPS→HTTPS: tylko origin
    → HTTPS→HTTP: brak Referer
    → HTTP→HTTP: tylko origin

strict-origin-when-cross-origin (zalecana!)
    → Same-origin: pełny URL
    → Cross-origin HTTPS→HTTPS: tylko origin
    → Cross-origin HTTPS→HTTP: brak Referer

unsafe-url
    → Zawsze wysyła pełny URL (najbardziej niebezpieczne)
```

### Ustawianie Referrer Policy

**HTTP Header:**
```http
Referrer-Policy: strict-origin-when-cross-origin
```

**Meta tag:**
```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

**Na konkretnym linku lub elemencie:**
```html
<a href="https://external.com" referrerpolicy="no-referrer">Link</a>
<img src="https://cdn.com/image.jpg" referrerpolicy="origin">
```

**W JavaScript (fetch):**
```javascript
fetch('https://api.example.com/data', {
    referrerPolicy: 'no-referrer'        // lub
    referrerPolicy: 'same-origin'        // lub
    referrerPolicy: 'strict-origin'      // itd.
});
```

---

## 4. Co dzieje się wewnętrznie

### Hierarchia policy

Jeśli wiele wartości konkuruje (nagłówek, meta, atrybut), wyższy priorytet ma bardziej restrykcyjna:

```
Na elemencie (referrerpolicy="no-referrer") > meta > HTTP header > browser default
```

### Browser defaults

Chrome, Firefox, Safari — domyślna wartość to `strict-origin-when-cross-origin` (od 2020-2021). Starsze wersje miały `no-referrer-when-downgrade`.

### `document.referrer`

W JavaScript możesz odczytać Referer strony:
```javascript
document.referrer; // URL strony która prowadziła do tej strony
// Może być pusty jeśli user wpisał URL bezpośrednio lub polityka blokuje
```

---

## 5. Analogiczny przykład z życia

Referrer Policy to instrukcja dla kuriera:

- Klient zamawiający dostawę (przeglądarka) może zaznaczyć na paczce (request):
  - "Nie ujawniaj skąd jestem" (`no-referrer`)
  - "Podaj tylko moje miasto" (`origin`)
  - "Podaj pełny adres jeśli w tej samej okolicy, tylko miasto dla innych" (`strict-origin-when-cross-origin`)
- Bez instrukcji → kurier decyduje co ujawnić (browser default)

---

## 6. Przykład kodu

```javascript
// === Fetch z kontrolowanym Referrer ===

// Brak Referer dla zewnętrznych API (np. analytics, logging)
const trackingData = await fetch('https://analytics.third-party.com/track', {
    method: 'POST',
    body: JSON.stringify({ event: 'page_view' }),
    referrerPolicy: 'no-referrer' // nie ujawniaj URL strony
});

// Pełny URL dla same-origin, origin dla cross-origin
const apiData = await fetch('https://api.example.com/data', {
    referrerPolicy: 'strict-origin-when-cross-origin' // zalecane
});

// Sprawdź skąd przyszedł użytkownik (na stronie docelowej):
console.log('Użytkownik przyszedł z:', document.referrer);
// Może być używane do analytics ale też do CSRF validation (nie zalecane!)
```

---

## 7. Przykład z prawdziwej aplikacji

### Token w URL wyciek przez Referer

```
Scenariusz:
1. Aplikacja generuje link resetowania hasła:
   https://app.com/reset?token=SECRET_TOKEN_123

2. Użytkownik odwiedza stronę reset
3. Strona ładuje zewnętrzne zasoby (Google Analytics, CDN fonts, itp.):
   GET /analytics.js HTTP/1.1
   Host: www.google-analytics.com
   Referer: https://app.com/reset?token=SECRET_TOKEN_123  ← TOKEN!

4. Token trafił do Google Analytics logs
5. Ewentualnie do atakującego (jeśli kontroluje any external resource)

OBRONA:
1. Referrer-Policy: no-referrer lub strict-origin
2. Lub: nie umieszczaj sekretów w URL (używaj POST body lub fragmentu #)
3. Lub: ograniczony czas życia tokenu
```

### Referer jako CSRF ochrona (antypattern)

```javascript
// BŁĘDNA praktyka: weryfikacja CSRF przez Referer
app.post('/transfer', (req, res) => {
    const referer = req.headers.referer;
    
    // PROBLEMATYCZNE:
    if (!referer || !referer.startsWith('https://bank.com')) {
        return res.status(403).send('CSRF detected');
    }
    
    // Problemy:
    // 1. Referer może być brak (no-referrer policy) → legalne requesty blokowane
    // 2. Referer może być sfałszowany (proxy, narzędzia) → ochrona ominięta
    // 3. Nie jest to niezawodna metoda anty-CSRF
});

// POPRAWNA ochrona CSRF: dedykowany token (Rozdział 14)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Używanie `unsafe-url`

```http
# BŁĄD: wysyła pełny URL wszędzie, łącznie z cross-origin HTTPS→HTTP
Referrer-Policy: unsafe-url

# POPRAWKA: strict-origin-when-cross-origin
Referrer-Policy: strict-origin-when-cross-origin
```

### Błąd 2: Poleganie na Referer dla bezpieczeństwa

```javascript
// BŁĄD: Referer może być manipulowany lub nieobecny
if (req.headers.referer === expectedReferer) {
    allowAction();
}
// Referer może być:
// - brak (no-referrer policy)
// - zmodyfikowany przez proxy
// - sfałszowany przez narzędzia (curl, Burp)

// Nie używaj Referer jako security check!
```

### Błąd 3: Brak policy → domyślne zachowanie przeglądarki

```html
<!-- BRAK Referrer-Policy header/meta → domyślne przeglądarki (strict-origin-when-cross-origin) -->
<!-- Może być gorsze na starszych przeglądarkach (no-referrer-when-downgrade) -->
<!-- ZAWSZE jawnie ustawiaj politykę! -->
```

---

## 9. Znaczenie dla bezpieczeństwa

### Information Disclosure przez Referer

```
Wektory wycieków przez Referer:
1. Tokeny w URL (reset password, magic links, invite tokens)
2. User ID lub session ID w URL
3. Prywatne dane w query string (?patient-id=123&diagnosis=hiv)
4. Wewnętrzne URL (/admin/users?view=all)
5. Wyszukiwane frazy (?q=competitor+analysis)
```

### Analytics i third-party trackers

Każdy zewnętrzny zasób (Google Analytics, Facebook Pixel, CDN, itp.) dostaje Referer jeśli polityka jest permissive. To znaczy że dostawcy analytics mogą zbierać URL wszystkich podstron aplikacji.

### Referrer Spoofing

Atakujący z Burp Suite może ustawić dowolną wartość `Referer`:
```
GET /transfer HTTP/1.1
Host: bank.com
Referer: https://bank.com/dashboard
```

Referer jest kontrolowany przez klienta — nigdy nie używaj go jako security check.

### Powiązane CWE

- **CWE-598** — Use of GET Request Method with Sensitive Query Strings
- **CWE-200** — Exposure of Sensitive Information
- **CWE-352** — CSRF (Referer jako fałszywa ochrona)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź nagłówki

```bash
# HTTP response headers:
curl -I https://target.com | grep -i "referrer-policy"

# Jeśli brak → browser default (sprawdź jakie)
# Jeśli "no-referrer-when-downgrade" lub "unsafe-url" → zbyt permissive
```

### Sprawdź czy tokeny są w URL

```
1. Kliknij "Zapomnij hasło" → sprawdź email link
2. Sprawdź czy reset link zawiera token w query string (https://app.com/reset?token=...)
3. Otwórz link → sprawdź Network → zewnętrzne requesty mają Referer?
4. Jeśli strona ma Google Analytics → GA dostaje token w Referer!
```

### DevTools → Network

```
1. Otwórz URL z tokenem w query string
2. Network tab → filtruj external requests
3. Sprawdź nagłówek Referer w requestach do zewnętrznych hostów
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Referrer-Policy header jest ustawiony?
□ Czy wrażliwe tokeny są przekazywane w URL (wyciek przez Referer)?
□ Czy zewnętrzne zasoby (analytics, CDN) dostają pełny URL przez Referer?
□ Czy aplikacja sprawdza Referer jako CSRF ochronę (antypattern)?
□ Czy meta referrer jest ustawiony (fallback)?
□ Czy wewnętrzne URL nie są ujawniane przez Referer?
□ Czy aplikacja loguje Referer i mogą być w nim wrażliwe dane?
```

---

## 12. Jak się zabezpieczać

```http
# Zalecana wartość dla większości aplikacji:
Referrer-Policy: strict-origin-when-cross-origin

# Dla aplikacji z wrażliwymi URL (tokeny, PII):
Referrer-Policy: no-referrer
# lub:
Referrer-Policy: same-origin
```

```javascript
// W kodzie: tokeny NIE powinny być w URL
// ŹLE:
window.location.href = `/reset?token=${token}`; // token w URL → wyciek przez Referer

// DOBRZE:
// Używaj POST form lub przechowaj token w sessionStorage/cookie
// Link: /reset, po kliknięciu → AJAX z tokenem z email jako parametr POST
// lub: fragment URL (#token=...) — fragment nie jest wysyłany w Referer!

// Fragment (#) trick:
window.location.href = `/reset#${token}`;
// Fragment nie jest wysyłany przez przeglądarkę w Referer header
// ALE: może być widoczny w URL i historii przeglądarki → nie idealne
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Referrer Policy kontroluje ile URL strony jest ujawniane w nagłówku `Referer`
2. Domyślna (nowoczesne przeglądarki): `strict-origin-when-cross-origin`
3. Tokeny w URL → ryzyko wycieku przez Referer do zewnętrznych zasobów
4. Referer jest kontrolowany przez klienta — nigdy nie używaj jako security check
5. Fragment URL (`#`) nie jest wysyłany w Referer (ale widoczny w historii)

**Dla pentestera:**
- Szukaj tokenów w URL query string
- Sprawdź czy strona ładuje zewnętrzne zasoby gdy tokenów URL są widoczne
- Brak Referrer-Policy header = brak kontroli (sprawdź browser defaults)
- Sprawdź czy aplikacja używa Referer jako CSRF ochrony (antypattern)

---

## Powiązania

```
Referrer Policy
    │
    ├──► Permissions Policy (Rozdział 34)
    │         Oba kontrolują zachowanie przeglądarki przez HTTP headers
    │
    ├──► CORS (Rozdział 13)
    │         Cross-origin requests — oba mają znaczenie
    │
    ├──► CSP (Rozdział 36)
    │         Kompletny security header — uzupełniają się
    │
    └──► Cookies (Rozdział 14)
              Tokeny w cookies (HttpOnly) bezpieczniejsze niż w URL
```
