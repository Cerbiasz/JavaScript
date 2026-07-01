# Rozdział 26: Server-Sent Events

## 1. Czym są Server-Sent Events

**Server-Sent Events (SSE)** to technologia umożliwiająca serwerowi **jednostronne wysyłanie** strumienia zdarzeń do klienta przez standardowe połączenie HTTP. W odróżnieniu od WebSocket (pełny duplex), SSE jest **single-directional** — tylko serwer → klient. Klient może wysyłać dane do serwera tylko przez oddzielne requesty HTTP.

SSE jest implementowane przez interfejs `EventSource` w JavaScript i używa prostego tekstowego formatu nad HTTP/1.1 lub HTTP/2:

```
Content-Type: text/event-stream

data: Hello World

event: priceUpdate
data: {"symbol":"BTC","price":42000}
id: 1234
retry: 3000
```

Kluczowe cechy:
- Oparte na standardowym HTTP (nie nowy protokół jak WS)
- Automatyczne reconnect (wbudowane w przeglądarkę)
- Obsługa named events (`event:` field)
- Last-Event-ID dla resumable streams
- Objęte Same-Origin Policy (inaczej niż WebSocket!)
- Tylko tekst (UTF-8) — brak natywnej obsługi binary

---

## 2. Dlaczego powstał

### Problem: real-time updates jednostronne

WebSocket jest overengineerowany dla wielu przypadków użycia gdzie potrzebne są tylko **powiadomienia z serwera**:
- Aktualizacje dashboardu (giełda, monitoring)
- Notyfikacje (nowe wiadomości, alerty)
- Live feed (newsy, social media stream)

SSE dostarcza prosty mechanizm dla tych przypadków:
- Brak nowego protokołu — działa jak długo trwający HTTP response
- Prostsze niż WebSocket (brak handshake, framing, close codes)
- Native reconnect — przeglądarka automatycznie wznawia połączenie
- Działa przez HTTP/2 multiplexing (wiele SSE streamów przez jedno połączenie)

---

## 3. Jak działa

### Format strumienia

Serwer wysyła response z `Content-Type: text/event-stream`. Każde zdarzenie to blok linii zakończony pustą linią:

```
data: prosta wiadomość\n
\n

event: customType\n
data: {"key": "value"}\n
id: 42\n
\n

: komentarz (ignorowany przez klienta)\n
\n

retry: 5000\n
\n
```

Pola zdarzenia:
- `data:` — payload zdarzenia (może być wieloliniowy, każda linia z `data:`)
- `event:` — typ zdarzenia (domyślnie: `message`)
- `id:` — identyfikator zdarzenia (wysyłany jako `Last-Event-ID` przy reconnect)
- `retry:` — czas oczekiwania przed reconnect (ms)
- `: ` — komentarz (keepalive, ignorowany)

### JavaScript API — EventSource

```javascript
// Tworzenie połączenia SSE
const source = new EventSource('/api/events');
// Domyślnie: same-origin, GET request, bez credentials

// Z credentials (cookies):
const source = new EventSource('/api/events', { withCredentials: true });

// Odbieranie domyślnych zdarzeń (type: 'message')
source.onmessage = (event) => {
    console.log('Data:', event.data);     // string
    console.log('ID:', event.lastEventId); // ostatni ID
    console.log('Type:', event.type);     // 'message'
};

// Odbieranie named events
source.addEventListener('priceUpdate', (event) => {
    const price = JSON.parse(event.data);
    updateDisplay(price);
});

source.addEventListener('error', (event) => {
    console.log('Połączenie:', source.readyState);
    // 0=CONNECTING, 1=OPEN, 2=CLOSED
    if (source.readyState === EventSource.CLOSED) {
        console.log('Trwale zamknięte');
    }
});

// Zamknięcie
source.close(); // zatrzymuje połączenie i reconnect
```

### Automatic reconnect

EventSource automatycznie wznawia połączenie gdy zostanie przerwane:
1. Połączenie zerwane → czekaj `retry` ms (domyślnie 3000)
2. Wyślij GET z nagłówkiem `Last-Event-ID: <ostatni_id>` (jeśli był)
3. Serwer może wznowić strumień od tego ID

```
// Klient po reconnect wysyła:
GET /api/events HTTP/1.1
Last-Event-ID: 42
Cache-Control: no-cache
Accept: text/event-stream
```

---

## 4. Co dzieje się wewnętrznie

### HTTP/1.1 chunked transfer

SSE over HTTP/1.1 używa **chunked transfer encoding** — serwer wysyła dane w "kawałkach" gdy ma co wysłać, połączenie HTTP pozostaje otwarte. Browser buforuje strumień.

### HTTP/2 multiplexing

Over HTTP/2 SSE jest multiplexowany — wiele SSE streamów (i normalnych requestów) może działać przez jedno TCP połączenie. To eliminuje problem "max 6 connections per origin" z HTTP/1.1 (gdzie każde SSE zajmowało jedno połączenie).

### CORS i SSE

SSE jest objęte CORS. `EventSource` z cross-origin URL wymaga CORS headers na serwerze:
```
Access-Control-Allow-Origin: https://app.com
```

Ale: nie jest wysyłane `Origin` w CORS preflight dla SSE (bo EventSource używa GET i nie ma custom headers w domyślnej konfiguracji → simple request CORS).

---

## 5. Analogiczny przykład z życia

SSE to radio:

- Stacja radiowa (serwer) nadaje ciągły strumień
- Słuchacze (klienci) odbierają
- Komunikacja jest jednostronna (słuchacze nie mówią do stacji przez radio)
- Jeśli stracisz sygnał → tuner automatycznie próbuje się znowu dostroić (auto-reconnect)
- Named events = różne programy (news, muzyka) na tej samej stacji

---

## 6. Przykład kodu

```javascript
// === Klient ===
class EventStreamClient {
    constructor(url, options = {}) {
        this.url = url;
        this.handlers = {};
        this.source = null;
        this.connect(options);
    }
    
    connect(options) {
        this.source = new EventSource(this.url, options);
        
        this.source.onopen = () => this.emit('connected');
        
        this.source.onmessage = ({ data, lastEventId }) => {
            try {
                this.emit('message', JSON.parse(data));
            } catch {
                this.emit('message', data);
            }
        };
        
        this.source.onerror = (event) => {
            if (this.source.readyState === EventSource.CLOSED) {
                this.emit('disconnected');
            }
        };
        
        // Named events
        ['priceUpdate', 'notification', 'alert'].forEach(type => {
            this.source.addEventListener(type, ({ data }) => {
                this.emit(type, JSON.parse(data));
            });
        });
    }
    
    on(event, handler) {
        this.handlers[event] = handler;
        return this; // chaining
    }
    
    emit(event, data) {
        this.handlers[event]?.(data);
    }
    
    close() {
        this.source.close();
    }
}

// === Serwer (Node.js/Express) ===
app.get('/api/events', (req, res) => {
    // Sprawdź autentykację
    if (!req.session?.userId) {
        res.status(401).send('Unauthorized');
        return;
    }
    
    // Nagłówki SSE
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('X-Accel-Buffering', 'no'); // wyłącz buforowanie nginx
    
    // Wyślij initial event
    res.write('data: {"type":"connected"}\n\n');
    
    // Keepalive comment co 15 sekund (proxy nie zamkną połączenia)
    const keepalive = setInterval(() => {
        res.write(': keepalive\n\n');
    }, 15000);
    
    // Subskrypcja na zdarzenia
    const userId = req.session.userId;
    const sendEvent = (eventType, data, id) => {
        if (id) res.write(`id: ${id}\n`);
        res.write(`event: ${eventType}\n`);
        res.write(`data: ${JSON.stringify(data)}\n\n`);
    };
    
    eventBus.subscribe(userId, sendEvent);
    
    // Cleanup gdy klient się rozłączy
    req.on('close', () => {
        clearInterval(keepalive);
        eventBus.unsubscribe(userId, sendEvent);
    });
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Podatność: SSE bez autentykacji per-event

```javascript
// PROBLEM: SSE endpoint nie weryfikuje tokenu w każdym reconnect
// Jeśli token wygaśnie → stare połączenie nadal aktywne przez połączenie HTTP

// BEZPIECZNIEJSZY WZORZEC: token w URL lub per-message validation
app.get('/api/events', async (req, res) => {
    const token = req.query.token; // lub req.headers.authorization
    const user = await validateToken(token);
    
    if (!user) {
        res.status(401).json({ error: 'Unauthorized' });
        return;
    }
    
    res.setHeader('Content-Type', 'text/event-stream');
    // ...
    
    // Sprawdzaj token periodycznie
    const tokenCheck = setInterval(async () => {
        const valid = await validateToken(token);
        if (!valid) {
            res.write('event: unauthorized\ndata: {}\n\n');
            res.end();
            clearInterval(tokenCheck);
        }
    }, 60000); // co minutę
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak `Cache-Control: no-cache`

```javascript
// BŁĄD: serwer nie wysyła Cache-Control
// Pośrednie proxy mogą cache'ować strumień → klienci dostają stare dane
res.setHeader('Content-Type', 'text/event-stream');
// Brak: res.setHeader('Cache-Control', 'no-cache');

// POPRAWKA:
res.setHeader('Cache-Control', 'no-cache');
res.setHeader('X-Accel-Buffering', 'no'); // dla nginx
```

### Błąd 2: Niepoprawny format zdarzeń

```javascript
// BŁĄD: brak podwójnego \n na końcu zdarzenia
res.write('data: hello\n'); // ← tylko jedno \n → nie jest to zdarzenie!

// POPRAWKA:
res.write('data: hello\n\n'); // ← dwa \n = koniec zdarzenia
```

### Błąd 3: Brak cleanup przy disconnect

```javascript
// BŁĄD: nie usuwasz subscriptions/listenersów gdy klient odejdzie
app.get('/events', (req, res) => {
    // ... 
    setInterval(() => res.write('data: ping\n\n'), 1000); // LEAK!
    // Brak req.on('close', cleanup)
});
```

---

## 9. Znaczenie dla bezpieczeństwa

### SSE objęte CORS (inaczej niż WebSocket)

EventSource jest objęty CORS — cross-origin SSE wymaga odpowiednich nagłówków CORS. To jest zaletą bezpieczeństwa nad WebSocket.

### Wrażliwe dane w strumieniu

SSE może być podatne na te same problemy co inne endpointy:
- Brak autentykacji → dowolny klient dostaje stream
- Information disclosure przez zdarzenia
- IDOR — `/api/events?userId=X` bez walidacji że X to zalogowany użytkownik

### XSS przez SSE data

```javascript
// Podatny handler który renderuje dane z SSE bez sanityzacji:
source.onmessage = ({ data }) => {
    document.getElementById('news').innerHTML = data; // XSS!
    // Jeśli serwer wysyła user-generated content przez SSE
};
```

### Powiązane CWE

- **CWE-79** — XSS (przez podatne renderowanie SSE data)
- **CWE-862** — Missing Authorization
- **CWE-200** — Exposure of Sensitive Information

---

## 10. Jak identyfikować podczas pentestu

### DevTools — Network Tab

1. **Network** → filtruj po `EventStream`
2. Kliknij SSE endpoint → zakładka **EventStream** — wszystkie zdarzenia
3. Sprawdź nagłówki (Authorization, cookies)

### Burp Suite

1. SSE to standardowy HTTP request → widoczny w Proxy History
2. Można przechwycić initial request i sprawdzić autentykację
3. Response body: strumień text/event-stream

### Console

```javascript
// Testuj SSE endpoint bezpośrednio:
const es = new EventSource('/api/events');
es.onmessage = e => console.log(e.data);
es.addEventListener('customEvent', e => console.log('Custom:', e.data));

// Cross-origin test (jeśli masz CORS):
const es2 = new EventSource('https://target.com/api/events', { withCredentials: true });
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy SSE endpoint wymaga autentykacji?
□ Czy jest IDOR — czy klient dostaje zdarzenia innych użytkowników?
□ Czy dane w strumieniu są sanityzowane zanim trafią do DOM?
□ Czy SSE strumień jest dostępny przez CORS bez autoryzacji?
□ Czy token/sesja jest weryfikowana podczas reconnect?
□ Czy Last-Event-ID jest walidowane po stronie serwera?
□ Czy rate limiting jest implementowane?
□ Czy SSE endpoint wymaga wss/https?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Autentykuj każde połączenie
app.get('/api/events', authenticateMiddleware, (req, res) => {
    // req.user ustawiony przez middleware
});

// 2. Izoluj strumienie — użytkownik widzi tylko swoje zdarzenia
eventBus.subscribe(req.user.id, sendEvent); // tylko po userId

// 3. Sanityzuj dane przez SSE jeśli trafią do DOM
source.onmessage = ({ data }) => {
    const div = document.createElement('div');
    div.textContent = data; // textContent zamiast innerHTML
    container.appendChild(div);
};

// 4. Używaj HTTPS/WSS
// Tylko https:// → tylko przez wss:// (SSE przez http: = plaintext)
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. SSE = jednostronny strumień serwer → klient, przez standardowy HTTP
2. `EventSource` API z automatycznym reconnect i Last-Event-ID
3. Objęte CORS (inaczej niż WebSocket) — cross-origin wymaga CORS headers
4. Tylko tekst UTF-8 — JSON jako najpopularniejszy format
5. Główne zagrożenia: brak autentykacji, IDOR, XSS przez podatne renderowanie

**Dla pentestera:**
- Network → EventStream w DevTools — podgląd strumienia
- Sprawdź autentykację SSE endpointu
- IDOR — czy możesz dostać zdarzenia innych użytkowników?

---

## Powiązania

```
Server-Sent Events
    │
    ├──► WebSocket (Rozdział 25)
    │         WS = pełny duplex; SSE = jednostronny (prostszy)
    │
    ├──► Fetch API (Rozdział 11)
    │         SSE przez HTTP GET — Fetch może też czytać streaming response
    │
    └──► CORS (Rozdział 13)
              SSE cross-origin wymaga CORS (inaczej niż WebSocket)
```
