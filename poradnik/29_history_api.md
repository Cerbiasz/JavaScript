# Rozdział 29: History API

## 1. Czym jest History API

**History API** to interfejs JavaScript umożliwiający programistyczną manipulację historią przeglądania przeglądarki. Pozwala na:
- Zmianę URL w pasku adresu bez przeładowania strony
- Nawigację po historii (przód/tył)
- Przechowywanie stanu aplikacji powiązanego z konkretnym URL

History API jest fundamentem **Single Page Applications (SPA)** — umożliwia nawigację wewnątrz aplikacji zbudowanej jako jeden dokument HTML, z URL reflektującym aktualny "widok".

Kluczowe metody:
- `history.pushState(state, title, url)` — dodaje nowy wpis w historii
- `history.replaceState(state, title, url)` — zastępuje bieżący wpis
- `history.back()` / `history.forward()` / `history.go(n)` — nawigacja
- `window.onpopstate` — event przy nawigacji przez historię
- `history.length` — liczba wpisów w historii

---

## 2. Dlaczego powstał

### Problem: SPA i nawigacja

Przed History API SPA miały problem z URL:
- Cała aplikacja pod jednym URL (`/`) → brak możliwości bookmarku, share, back/forward
- Hash-based routing: `/#/page1` → obejście ale nieeleganckie
- Problem z SEO (boty nie wykonywały JS)

History API (HTML5, 2009) rozwiązało to przez możliwość zmiany URL bez przeładowania:
- `pushState('/dashboard')` → URL = `/dashboard`, strona nie ładuje się od nowa
- Back button → `popstate` event → aplikacja obsługuje nawigację przez JS

---

## 3. Jak działa

### pushState i replaceState

```javascript
// history.pushState(state, title, url)
// state: dowolny serializowalny obiekt (Structured Clone), 640KB limit
// title: ignorowane przez większość przeglądarek (historycznie), używaj ''
// url: nowy URL (musi być same-origin!)

// Dodaj nowy wpis
history.pushState(
    { page: 'dashboard', userId: 42 }, // state object
    '',                                  // title (ignorowane)
    '/dashboard'                         // nowy URL
);

// Zastąp bieżący wpis (bez nowego wpisu w historii)
history.replaceState(
    { page: 'dashboard', filter: 'active' },
    '',
    '/dashboard?filter=active'
);
```

### Nawigacja przez historię

```javascript
history.back();    // jak przycisk "Wstecz"
history.forward(); // jak przycisk "Naprzód"
history.go(-2);    // dwa kroki wstecz
history.go(0);     // reload strony (lub po prostu location.reload())
```

### popstate event

```javascript
// Wywoływany gdy użytkownik nawiguje przez historię (back/forward)
// NIE jest wywoływany przez pushState/replaceState
window.addEventListener('popstate', (event) => {
    console.log('State:', event.state); // obiekt ze pushState/replaceState
    console.log('URL:', location.pathname);
    
    // Obsłuż nawigację w SPA
    router.navigate(location.pathname, event.state);
});
```

### URL constraints

**Ważne ograniczenie bezpieczeństwa:** URL w `pushState`/`replaceState` musi być **same-origin**:

```javascript
// OK — same-origin
history.pushState({}, '', '/other-page');
history.pushState({}, '', 'https://example.com/page'); // OK jeśli jesteśmy na example.com
history.pushState({}, '', '?query=value'); // OK

// BŁĄD — SecurityError
history.pushState({}, '', 'https://evil.com/phishing'); // DOMException!
history.pushState({}, '', '//evil.com'); // DOMException!
```

### location.pathname vs history.state

```javascript
// Aktualny URL zawsze przez location:
location.href;       // "https://app.com/dashboard?filter=active"
location.pathname;   // "/dashboard"
location.search;     // "?filter=active"
location.hash;       // ""

// Stan powiązany z bieżącym URL:
history.state; // { page: 'dashboard', userId: 42 }
```

---

## 4. Co dzieje się wewnętrznie

### Stack historii

Przeglądarka utrzymuje stos (stack) wpisów historii dla każdego okna/karty. Każdy wpis składa się z:
- URL
- State object (serializowany przez Structured Clone, limit 640KB w Chrome)
- Scroll position (niektóre przeglądarki)

`pushState` dodaje na wierzch stosu; `popstate` jest wywoływany gdy indeks zmienia się przez nawigację użytkownika.

### Same-origin requirement

Ograniczenie same-origin w History API jest kluczowe dla bezpieczeństwa. Bez niego strona mogłaby zmienić URL na `https://bank.com/transfer` bez nawigacji, myląc użytkownika.

Ale: **origin musi być identyczny** — nawet `https://app.com` na `https://app.com:8080` to inny origin.

---

## 5. Analogiczny przykład z życia

History API to GPS z pamięcią tras:

- Każda nawigacja (`pushState`) dodaje punkt na mapie (wpis historii)
- Wstecz/naprzód (`back()`/`forward()`) przemieszcza cię po zapisanej trasie
- `state` object to notatki przy każdym punkcie (jakie były filtry, co przeglądałeś)
- URL zmienia się jak adres na GPS — pokazuje gdzie jesteś, ale nie musisz tam fizycznie jechać (przeładować strony)

---

## 6. Przykład kodu

```javascript
// === Prosty router SPA ===
class Router {
    constructor(routes) {
        this.routes = routes;
        
        // Nasłuchuj nawigacji historii
        window.addEventListener('popstate', (e) => {
            this.navigate(location.pathname, false); // false = nie push do historii
        });
    }
    
    navigate(path, addToHistory = true) {
        const route = this.routes[path] ?? this.routes['/404'];
        
        if (addToHistory) {
            history.pushState({ path }, '', path);
        }
        
        // Renderuj nowy widok
        route.render(document.getElementById('app'));
    }
    
    link(path) {
        return (event) => {
            event.preventDefault(); // nie ładuj strony
            this.navigate(path);
        };
    }
}

// Użycie:
const router = new Router({
    '/': { render: (el) => (el.innerHTML = '<h1>Home</h1>') },
    '/dashboard': { render: (el) => (el.innerHTML = '<h1>Dashboard</h1>') },
    '/404': { render: (el) => (el.innerHTML = '<h1>Not Found</h1>') }
});

// Link w HTML
document.querySelectorAll('[data-link]').forEach(el => {
    el.addEventListener('click', router.link(el.dataset.link));
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Open Redirect przez manipulację historią

```javascript
// Aplikacja używa history.state do redirect po logowaniu:
const returnUrl = history.state?.returnUrl ?? '/dashboard';
// Po logowaniu:
window.location.href = returnUrl; // OPEN REDIRECT!

// Atakujący może:
history.pushState({ returnUrl: 'https://evil.com' }, '', '/login');
// Jeśli użytkownik zaloguje się → zostanie przekierowany na evil.com

// POPRAWKA: waliduj returnUrl
function safeRedirect(url) {
    try {
        const parsed = new URL(url, window.location.origin);
        if (parsed.origin !== window.location.origin) {
            return '/dashboard'; // tylko same-origin redirect
        }
        return parsed.pathname + parsed.search;
    } catch {
        return '/dashboard';
    }
}
```

### History state jako storage (i jego ryzyko)

```javascript
// Używanie history.state jako tymczasowego storage
// (dane przeżywają nawigację ale giną po refresh)
history.replaceState({
    formData: collectFormData(),
    token: generateCsrfToken()
}, '', location.href);

// Przy powrocie:
const { formData, token } = history.state ?? {};
// RYZYKO: dane w history.state są dostępne przez JavaScript
// XSS może odczytać history.state!
```

---

## 8. Typowe błędy programistów

### Błąd 1: URL Spoofing przez pushState

```javascript
// BŁĄD: pushState zmienia URL ale nie zawartość (potencjalny phishing vector)
// Na evil.com:
history.pushState({}, '', 'https://bank.com/login'); // DOMException! (inny origin)
// Ten atak JEST blokowany przez przeglądarkę

// ALE: jeśli atakujący jest na bank.com (przez XSS):
history.pushState({}, '', '/admin/transfer'); // wyświetla URL /admin/transfer
// i renderuje fake login form
// Użytkownik widzi poprawny URL ale fałszywy content!
```

### Błąd 2: Brak obsługi popstate

```javascript
// BŁĄD: pushState dodaje wpisy do historii ale brak obsługi back button
history.pushState({ step: 2 }, '', '/checkout/step2');
// Użytkownik klika "Wstecz" → URL wraca do /checkout/step1
// ale strona NIE aktualizuje się (brak popstate handler)!

// POPRAWKA: zawsze dodaj popstate listener
window.addEventListener('popstate', (e) => {
    if (e.state?.step) {
        showStep(e.state.step);
    }
});
```

### Błąd 3: Wrażliwe dane w state

```javascript
// PROBLEMATYCZNE: tokens w history.state
history.pushState({ csrfToken: 'secret123' }, '', '/dashboard');
// history.state jest dostępne dla CAŁEJ strony
// XSS = dostęp do csrfToken
```

---

## 9. Znaczenie dla bezpieczeństwa

### URL Confusion w SPA

W SPA może wystąpić rozbieżność między URL wyświetlanym a faktycznie renderowanym content:

```javascript
// SPA router z XSS:
function render(path) {
    document.getElementById('app').innerHTML = fetchTemplate(path); // XSS!
}
window.addEventListener('popstate', e => render(location.pathname));

// Atak: użytkownik jest przekierowany do /search?q=<script>alert(1)</script>
history.pushState({}, '', '/search?q=<script>alert(1)</script>');
render(location.pathname); // XSS wykonany przez parametr URL!
```

### Open Redirect przez history.state

Dane z `history.state` mogą być kontrolowane przez skrypt:
- `history.pushState({ returnUrl: 'https://evil.com' }, '', '/login')`
- Jeśli aplikacja używa `history.state.returnUrl` bez walidacji → open redirect

### Phishing przez URL Spoofing

W ramach same-origin, `pushState` może zmienić URL na `/admin` wyświetlając fałszywy formularz. To jest możliwy wektor phishingu **po** XSS — atakujący zmienia URL na coś "oficjalnie wyglądającego".

### CSRF i history.state

History state może być używany do przechowywania CSRF tokenów. To jest OK jeśli token jest również zweryfikowany przez serwer — ale XSS może odczytać `history.state`.

### Powiązane CWE

- **CWE-601** — URL Redirection to Untrusted Site (open redirect)
- **CWE-79** — XSS (przez parametry URL w SPA)

---

## 10. Jak identyfikować podczas pentestu

### Szukaj w source

```javascript
// Grep za:
history.pushState
history.replaceState
history.state
popstate
location.href = history.state
location.replace(history.state
```

### Testuj parametry URL w SPA

```
1. Wejdź na /page?param=value
2. Sprawdź jak parametr jest używany:
   - Wyświetlany w DOM (XSS)?
   - Używany do redirect (open redirect)?
   - Przechowywany w state i potem używany (state injection)?
```

### DevTools

```javascript
// Sprawdź aktualny stan historii:
history.state; // co jest przechowane?

// Sprawdź długość historii:
history.length;

// Monkey-patch pushState:
const orig = history.pushState.bind(history);
history.pushState = function(state, title, url) {
    console.log('[pushState]', { state, url });
    return orig(state, title, url);
};
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy URL parameters są renderowane do DOM bez sanityzacji (XSS)?
□ Czy history.state zawiera wrażliwe dane (tokeny)?
□ Czy returnUrl/redirect z history.state jest walidowany?
□ Czy open redirect jest możliwy przez state manipulation?
□ Czy phishing przez URL spoofing (pushState + fake form) jest możliwy przez XSS?
□ Czy back button powoduje niespójności w state?
□ Czy server-side routing i client-side routing są spójne?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Waliduj URL params przed renderowaniem
function renderPage(query) {
    const params = new URLSearchParams(query);
    const search = params.get('q') ?? '';
    
    // Nie renderuj bezpośrednio parametru
    const el = document.createElement('span');
    el.textContent = search; // textContent, nie innerHTML
    results.appendChild(el);
}

// 2. Waliduj returnUrl
function getReturnUrl() {
    const url = history.state?.returnUrl ?? new URLSearchParams(location.search).get('return');
    if (!url) return '/dashboard';
    
    try {
        const parsed = new URL(url, location.origin);
        if (parsed.origin !== location.origin) return '/dashboard';
        return parsed.pathname + parsed.search;
    } catch {
        return '/dashboard';
    }
}

// 3. Nie przechowuj sekretów w history.state
// history.state jest dostępne przez XSS
// Użyj sessionStorage per-tab lub server-side session

// 4. CSP chroni przed XSS który mógłby czytać state
// Content-Security-Policy: script-src 'self'
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. History API = manipulacja URL i historią bez przeładowania strony
2. `pushState` ograniczone do same-origin (zapobiega URL spoofing cross-origin)
3. `popstate` event przy nawigacji przez historię (nie przy pushState!)
4. `history.state` dostępne przez JavaScript (XSS może to odczytać)
5. Główne zagrożenia: open redirect przez state, XSS przez URL params w SPA

**Dla pentestera:**
- Testuj parametry URL w SPA na XSS i open redirect
- Sprawdź `history.state` w DevTools Console
- Szukaj returnUrl/redirect opartego na state bez walidacji

---

## Powiązania

```
History API
    │
    ├──► URL API (Rozdział 30)
    │         URL parsing i manipulacja — często używane razem z History API
    │
    ├──► DOM/XSS (Rozdział 9)
    │         URL params w SPA renderowane do DOM → XSS
    │
    └──► sessionStorage (Rozdział 16)
              Alternatywny storage dla nawigacyjnego stanu (per-tab)
```
