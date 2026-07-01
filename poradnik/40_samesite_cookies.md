# Rozdział 40: SameSite Cookies

## 1. Czym są SameSite Cookies

**SameSite** to atrybut ciasteczka HTTP kontrolujący w jakich sytuacjach przeglądarka wysyła cookie w cross-site requestach. Został zaprojektowany jako mechanizm ochrony przed **CSRF (Cross-Site Request Forgery)** — atakach polegających na nakłonieniu przeglądarki ofiary do wykonania unwanted requestów na innym serwisie gdzie ofiara jest zalogowana.

Trzy wartości atrybutu `SameSite`:

| Wartość | Zachowanie |
|---------|-----------|
| `Strict` | Cookie wysyłane TYLKO gdy site strony requestującej = site serwera |
| `Lax` | Cookie wysyłane dla nawigacji top-level + GET; blokuje cross-site POST/fetch/img |
| `None` | Cookie zawsze wysyłane (wymaga `Secure`) |

```http
Set-Cookie: session=abc123; SameSite=Strict; HttpOnly; Secure
Set-Cookie: pref=dark; SameSite=Lax
Set-Cookie: tracking=xyz; SameSite=None; Secure
```

---

## 2. Dlaczego powstał

### Problem: Cookie wysyłane automatycznie w każdym cross-site request

Przed SameSite przeglądarka wysyłała cookie do domeny X przy każdym request do X — niezależnie od tego skąd ten request pochodzi:

```html
<!-- Na evil.com: -->
<form action="https://bank.com/transfer" method="POST">
    <input name="to" value="attacker">
    <input name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
```

Przeglądarka ofiary:
1. Wykonuje POST do bank.com
2. **Automatycznie dołącza cookie sesji** bank.com
3. Bank.com widzi zalogowanego użytkownika → wykonuje przelew

SameSite (Google, 2016, RFC 6265bis) rozwiązuje to przez selektywne wysyłanie cookies.

---

## 3. Jak działa

### SameSite=Strict

```
Cookie wysyłane TYLKO gdy top-level navigation (lub request) pochodzi z tego samego site.

Scenariusz 1: Użytkownik wpisuje https://bank.com bezpośrednio
    → Sec-Fetch-Site: none → cookie wysyłane ✓

Scenariusz 2: Klik na link z https://bank.com/page1 do https://bank.com/page2
    → Sec-Fetch-Site: same-origin → cookie wysyłane ✓

Scenariusz 3: Klik na link z https://evil.com do https://bank.com
    → Sec-Fetch-Site: cross-site → cookie NIE wysyłane ✗

Scenariusz 4: Form POST z https://evil.com do https://bank.com
    → cross-site → cookie NIE wysyłane ✗ (CSRF ochrona!)

Problem Strict:
    Email z linkiem do https://bank.com/dashboard → kliknięcie → brak sesji!
    Użytkownik wygląda na niezalogowanego mimo że sesja jest ważna.
    Dlatego Strict jest zbyt agresywne dla UX.
```

### SameSite=Lax (domyślna od Chrome 80)

```
Cookie wysyłane dla:
1. Top-level navigation GET (kliknięcie linku, redirect) → wysyłane ✓
2. Preloading resources (prefetch, prerender) z GET → wysyłane ✓

Cookie NIE wysyłane dla:
3. Cross-site POST / PUT / DELETE → nie wysyłane ✗ (CSRF ochrona!)
4. Cross-site XHR/fetch() → nie wysyłane ✗
5. Cross-site resources (<img>, <iframe>, <script src>) → nie wysyłane ✗

Kompromis:
+ Email link do https://bank.com/dashboard → cookie wysyłane (zalogowany!) ✓
- Cross-site form submit → cookie nie wysyłane (CSRF blokowany!) ✓
```

### SameSite=None

```
Cookie zawsze wysyłane, niezależnie od źródła requestu.
WYMAGA atrybutu Secure (tylko HTTPS).

Używane dla:
- Third-party cookies (reklamy, tracking)
- Embedded widgets (płatności, mapy) wymagające sesji
- Federated login (OAuth, SSO cross-domain)

Set-Cookie: tracking=xyz; SameSite=None; Secure
```

### Definicja "same-site"

```
"Same-site" jest luźniejsze niż "same-origin":

same-origin: protokół + host + port muszą być identyczne
same-site: wystarczy Registrable Domain (eTLD+1)

Przykłady:
https://app.example.com i https://api.example.com
    → same-site (eTLD+1: example.com)
    → different origin

https://example.com i https://example.net
    → cross-site (różne eTLD+1)

https://evil.com.example.com i https://example.com
    → Uwaga: to NIE jest same-site! (eTLD+1 evil.com.example.com vs example.com)
    → Sprawdzaj przez Public Suffix List
```

### Schemeful SameSite (od Chrome 89)

```
Wcześniej: http://example.com i https://example.com → same-site
Teraz (Schemeful): http i https traktowane jak różne sites

http://example.com → https://example.com = cross-site (downgrade attack!)
https://example.com → https://sub.example.com = same-site (OK)
```

---

## 4. Co dzieje się wewnętrznie

### Cookie Jar i filtrowanie

Gdy przeglądarka przygotowuje request, filtruje cookies z "cookie jar":

```
Dla każdego cookie w jar docelowej domeny:
    
    if cookie.SameSite == 'None':
        → dołącz cookie (jeśli Secure i HTTPS)
    
    if cookie.SameSite == 'Strict':
        → dołącz tylko jeśli request.site == cookie.site (top-level)
    
    if cookie.SameSite == 'Lax':
        → dołącz jeśli:
            - request jest same-site LUB
            - request jest top-level GET navigation
    
    if brak SameSite (legacy):
        → Chrome 80+: traktuj jak Lax
        → Starsze: traktuj jak None (bez Secure)
```

### "Lax + POST" okno czasowe

Chrome implementował tymczasowe złagodzenie: cookie bez SameSite (traktowane jak Lax) były wysyłane przez 2 minuty po ustawieniu — dla kompatybilności z starymi formularzami POST. To okno zostało usunięte w Chrome 84.

### Cookie Prefixes a SameSite

Prefiksy `__Host-` i `__Secure-` współdziałają z SameSite:

```http
# __Host- prefix: wymusza Secure + Path=/ + brak Domain
Set-Cookie: __Host-session=abc; SameSite=Strict; Secure; Path=/

# __Secure- prefix: wymusza Secure
Set-Cookie: __Secure-session=abc; SameSite=Lax; Secure
```

---

## 5. Analogiczny przykład z życia

SameSite to **zasada "goście tylko przy właścicielu"** w restauracji:

- Właściciel (bank.com) ma klucz do sejfu (cookie)
- `Strict`: klucz działa TYLKO gdy właściciel wchodzi sam z siebie (same-site)
- `Lax`: klucz działa gdy właściciel przychodzi przez główne wejście (top-level GET), ale NIE gdy ktoś próbuje go wepchnąć tylnym wejściem (cross-site POST)
- `None`: klucz zawsze działa (ale wtedy obowiązkowo musi być zamknięty na Secure=zamek)
- Bez SameSite (legacy): klucz działa dla każdego kto go posiada bez względu na wejście

---

## 6. Przykład kodu

```javascript
// === Serwer: ustawianie cookies z SameSite ===

// Express.js:
res.cookie('session', sessionToken, {
    httpOnly: true,
    secure: true,
    sameSite: 'strict',   // lub 'lax' lub 'none'
    maxAge: 3600000,      // 1 godzina w ms
    path: '/',
});

// Dla third-party cookie (embedded widget):
res.cookie('widget_session', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'none',    // MUSI mieć secure: true
    domain: '.example.com',
});
```

```python
# Flask:
from flask import make_response

response = make_response(jsonify({'status': 'logged in'}))
response.set_cookie(
    'session',
    value=session_token,
    httponly=True,
    secure=True,
    samesite='Strict',
    max_age=3600,
    path='/',
)
return response
```

```http
# Raw HTTP response headers:
HTTP/2 200 OK
Set-Cookie: __Host-session=TOKEN; SameSite=Strict; Secure; HttpOnly; Path=/
Set-Cookie: preferences=dark; SameSite=Lax; Path=/
Set-Cookie: analytics_id=XYZ; SameSite=None; Secure; Path=/
```

```javascript
// === JavaScript: odczyt cookies (nie można odczytać HttpOnly!) ===

// SameSite nie wpływa na odczyt cookies w JS — tylko na wysyłanie w requestach
// document.cookie zwraca tylko non-HttpOnly cookies

// Sprawdź które cookies istnieją:
console.log(document.cookie); // 'preferences=dark' (ale nie HttpOnly session!)

// Fetch API z cookies:
fetch('/api/data', {
    credentials: 'include',  // wysyła cookies
    method: 'POST',
    body: JSON.stringify(data),
});
// Cookie wysłane jeśli SameSite to pozwala (same-origin → Lax/Strict OK)
```

---

## 7. Przykład z prawdziwej aplikacji

### Bypass SameSite=Lax przez GET request

```html
<!-- SameSite=Lax blokuje POST ale nie GET navigation! -->
<!-- Jeśli endpoint zmienia stan przez GET (antypattern!): -->

<!-- Na evil.com: -->
<a href="https://bank.com/transfer?to=attacker&amount=1000">Kliknij po nagrodę!</a>
<!-- Lub: automatyczne przekierowanie -->
<script>window.location = 'https://bank.com/transfer?to=attacker&amount=1000'</script>

<!-- Sec-Fetch-Mode: navigate, Sec-Fetch-Dest: document → Lax cookie wysyłane! -->
<!-- Jeśli bank.com wykonuje transfer przez GET → CSRF mimo SameSite=Lax! -->
```

Obrona: endpointy modyfikujące stan ZAWSZE POST + CSRF token.

### Bypass SameSite=Strict przez same-site redirect

```javascript
// Jeśli atakujący może wywołać Open Redirect na same-site:
// https://example.com/redirect?url=https://example.com/delete?id=123

// Redirect pochodzi z example.com → same-site → Strict cookie wysyłane!
// Mimo że atakujący inicjował z external site

// Łańcuch: evil.com → example.com/redirect → example.com/delete?id=123
// SameSite=Strict widzę same-site redirect → wysyła cookie!
```

### Atak na OAuth przez SameSite cookie

```
OAuth flow cross-site:
1. Użytkownik na evil.com → klika "Login with Google"
2. Redirect do google.com → logowanie
3. Google redirect do app.com/callback?code=AUTH_CODE

W Strict mode: step 3 redirect z google.com do app.com → cross-site!
→ Pliki cookie app.com NIE są wysyłane z /callback request
→ State nie jest zweryfikowany → OAuth flow failure

Rozwiązanie: callback nie potrzebuje sesji, albo użyj Lax dla state cookie
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak SameSite (legacy cookies)

```http
# BŁĄD: brak SameSite
Set-Cookie: session=TOKEN; HttpOnly; Secure
# Chrome 80+ traktuje to jako Lax (domyślnie)
# Ale starsze przeglądarki jako None → CSRF ryzyko

# POPRAWKA: jawnie ustaw SameSite
Set-Cookie: session=TOKEN; HttpOnly; Secure; SameSite=Lax
```

### Błąd 2: SameSite=None bez Secure

```http
# BŁĄD: None bez Secure → przeglądarka odrzuci cookie
Set-Cookie: tracker=XYZ; SameSite=None
# Chrome odrzuca — skutek: cookie w ogóle nie jest ustawione!

# POPRAWKA:
Set-Cookie: tracker=XYZ; SameSite=None; Secure
```

### Błąd 3: Poleganie tylko na SameSite (bez CSRF token)

```javascript
// PROBLEM: SameSite nie chroni przed:
// 1. Same-site requestami (np. inny subdomain z XSS)
// 2. Lax + GET endpointy modyfikujące stan
// 3. Przeglądarkach bez wsparcia SameSite

// POPRAWKA: SameSite + CSRF token = defense in depth
```

### Błąd 4: Błędne rozumienie same-site vs same-origin

```javascript
// MYLENIE POJĘĆ:
// same-origin: https://app.example.com = https://app.example.com (identyczne)
// same-site: https://app.example.com i https://api.example.com (to samo eTLD+1)

// SameSite=Strict pozwala: https://app.example.com → https://api.example.com
// Bo to same-site (example.com)
// Ale blokuje: https://evil.com → https://example.com
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przed CSRF

```
SameSite=Strict: blokuje wszystkie cross-site requests z cookie
SameSite=Lax: blokuje POST/fetch/img cross-site, przepuszcza GET top-level
SameSite=None: brak ochrony CSRF (ale wymagane dla third-party cookies)

Zalecenie: SameSite=Strict lub Lax dla session cookies
```

### Same-site vs same-origin a podatności

```
Subdomena z XSS może obejść SameSite:
- sub.example.com ma XSS
- Cookie example.com jest SameSite=Lax
- Request z sub.example.com → example.com = same-site → cookie wysyłane!
- XSS na subdomenie może atakować główną domenę

Dlatego: SameSite nie eliminuje potrzeby ochrony subdomen
```

### Chrome 80 domyślne Lax — wielka zmiana 2020

Od Chrome 80 (luty 2020) cookies bez SameSite są traktowane jako `Lax`. Spowodowało to:
- Wiele aplikacji przestało działać (cross-site third-party cookies bez `None; Secure`)
- Wzrost bezpieczeństwa (CSRF bardziej utrudniony dla legacy app)
- Problemy z federowanym logowaniem (OAuth, SSO)

### CHIPS i Phase-out third-party cookies

Third-party cookies (SameSite=None) są wycofywane przez Chrome (2024+). Zastępuje je CHIPS (Partitioned Cookies — rozdział 41). To eliminuje cross-site tracking ale też łamie niektóre legitimate use cases.

### Powiązane CWE

- **CWE-352** — Cross-Site Request Forgery (SameSite jako mitygacja)
- **CWE-614** — Sensitive Cookie without 'Secure' Attribute
- **CWE-1004** — Sensitive Cookie without 'HttpOnly' Flag

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź atrybuty cookies

```bash
# Sprawdź nagłówki Set-Cookie:
curl -c /dev/null -I https://target.com/login \
    -d "user=test&pass=test" \
    --request POST | grep -i "set-cookie"

# Szukaj:
# - Brak SameSite → legacy cookie, może być podatny na CSRF
# - SameSite=None bez Secure → błąd konfiguracji (przeglądarka ignoruje)
# - SameSite=Lax + GET endpointy modyfikujące stan → CSRF via GET
```

### DevTools → Application → Cookies

```
1. DevTools → Application → Storage → Cookies
2. Sprawdź kolumnę "SameSite" dla każdego cookie
3. Brak wartości = Lax (Chrome 80+) lub None (stare)
4. Szukaj session cookies bez HttpOnly lub Secure
```

### CSRF test z SameSite=Lax

```html
<!-- Test: czy SameSite=Lax pozwala na CSRF via GET? -->
<!-- Sprawdź które endpointy przyjmują GET z side effects -->

<!-- Jeśli /api/delete?id=X to GET: -->
<img src="https://target.com/api/delete?id=123">
<!-- Jeśli request jest wykonany (Sec-Fetch-Dest: image, GET) → 
     SameSite=Lax Cookie jest wysyłane → CSRF via image load! -->
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy session cookies mają SameSite=Strict lub Lax?
□ Czy SameSite=None cookies mają atrybut Secure?
□ Czy endpointy modyfikujące stan używają POST (nie GET)?
□ Czy SameSite jest uzupełniony przez CSRF token (defense in depth)?
□ Czy subdomenowe XSS mogą obejść SameSite?
□ Czy OAuth/SSO flow działa poprawnie z SameSite=Lax?
□ Czy third-party cookies mają uzasadnienie (SameSite=None)?
□ Czy cookies mają też HttpOnly i Secure atrybuty?
□ Czy legacy cookies (bez SameSite) są zaktualizowane?
```

---

## 12. Jak się zabezpieczać

```http
# Zalecana konfiguracja session cookie:
Set-Cookie: __Host-session=TOKEN; SameSite=Strict; Secure; HttpOnly; Path=/

# Jeśli potrzebujesz cross-site nawigacji z sesją (np. SSO):
Set-Cookie: session=TOKEN; SameSite=Lax; Secure; HttpOnly; Path=/; Domain=.example.com

# Third-party cookie (embed, widget, analytics):
Set-Cookie: tracker=XYZ; SameSite=None; Secure; Path=/
```

```javascript
// Express.js — kompletna konfiguracja:
const sessionConfig = {
    name: '__Host-session',  // __Host- prefix dla extra bezpieczeństwa
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
        secure: true,        // tylko HTTPS
        httpOnly: true,       // brak dostępu przez JS
        sameSite: 'strict',  // CSRF ochrona
        maxAge: 3600000,     // 1 godzina
        path: '/',           // wymagane przez __Host- prefix
        // domain: undefined  // __Host- prefix wymaga braku Domain
    }
};

app.use(session(sessionConfig));
```

```python
# Django settings.py:
SESSION_COOKIE_SAMESITE = 'Strict'  # lub 'Lax'
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_AGE = 3600  # sekundy
CSRF_COOKIE_SAMESITE = 'Strict'
CSRF_COOKIE_SECURE = True
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. `SameSite=Strict` — cookie wysyłane tylko same-site (najsilniejsza ochrona, ograniczenia UX)
2. `SameSite=Lax` — domyślne od Chrome 80; blokuje cross-site POST, przepuszcza GET navigation
3. `SameSite=None` — third-party cookies; WYMAGA `Secure`; podatne na CSRF
4. "Same-site" ≠ "same-origin" — same-site jest luźniejsze (eTLD+1, np. subdomenowe)
5. Nie zastępuje CSRF tokenów — to **defense in depth**

**Dla pentestera:**
- Sprawdź `Set-Cookie` header: brak SameSite = potencjalny CSRF
- Testuj GET endpointy z side-effects: `SameSite=Lax` nie chroni przed CSRF via GET navigation
- Subdomena z XSS może obejść SameSite (same-site includes subdomains)
- `SameSite=None` bez `Secure` = cookie jest ignorowane przez Chrome → błąd konfiguracji

---

## Powiązania

```
SameSite Cookies
    │
    ├──► CSRF (Rozdział 14)
    │         SameSite to główna obrona przed CSRF
    │         Uzupełnienie: CSRF token
    │
    ├──► Fetch Metadata (Rozdział 39)
    │         Oba chronią przed cross-site requestami
    │         FM: po stronie serwera; SameSite: po stronie cookies
    │
    ├──► CHIPS (Rozdział 41)
    │         CHIPS to rozszerzenie dla third-party cookies
    │         (partitioned SameSite=None)
    │
    └──► Cookies (Rozdział 14)
              HttpOnly, Secure, SameSite = trinity cookie security
```
