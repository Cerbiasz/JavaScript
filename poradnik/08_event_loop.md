# Rozdział 8: Event Loop

## 1. Czym jest Event Loop

**Event Loop** (pętla zdarzeń) to mechanizm zarządzający asynchronicznym wykonaniem kodu w JavaScript. Odpowiada na pytanie: "skoro JavaScript jest jednowątkowy, jak może wykonywać operacje asynchroniczne (sieciowe, timery, I/O) bez blokowania interfejsu użytkownika?"

Event Loop to nie silnik JavaScript (V8, SpiderMonkey) — to mechanizm zarządzający interakcją między silnikiem a Web API. W przeglądarkach jest zaimplementowany przez sam silnik przeglądarki. W Node.js jest realizowany przez bibliotekę **libuv**.

Event Loop jest opisany w specyfikacji HTML (Living Standard), nie w ECMAScript. To część środowiska przeglądarki, nie języka JavaScript.

### Składowe systemu

```
┌───────────────────────────────────────────────────────────┐
│                      PRZEGLĄDARKA                         │
│                                                           │
│  ┌─────────────┐   ┌──────────────────────────────────┐  │
│  │  Call Stack │   │         Web API                  │  │
│  │  (silnik JS)│   │  setTimeout, fetch, DOM events   │  │
│  └──────┬──────┘   └────────────────┬─────────────────┘  │
│         │ (synchroniczny kod)        │ (async operacje)   │
│         │                           │                     │
│         │      ┌────────────────────▼──────────────────┐  │
│         │      │         Kolejki zadań                  │  │
│         │      │                                        │  │
│         │      │  ┌─────────────────────────────────┐  │  │
│         │      │  │  Microtask Queue (Promise, MO)   │  │  │
│         │      │  └─────────────────────────────────┘  │  │
│         │      │                                        │  │
│         │      │  ┌─────────────────────────────────┐  │  │
│         │      │  │  Macrotask Queue (setTimeout)    │  │  │
│         │      │  └─────────────────────────────────┘  │  │
│         │      │                                        │  │
│         │      │  ┌─────────────────────────────────┐  │  │
│         │      │  │  Animation Frame Queue (rAF)     │  │  │
│         │      │  └─────────────────────────────────┘  │  │
│         │      └────────────────────────────────────────┘  │
│         │                           │                     │
│         └───────────── Event Loop ──┘                     │
│                    (monitor i przekazywanie)               │
└───────────────────────────────────────────────────────────┘
```

---

## 2. Dlaczego powstał

### Problem: blokujące I/O

W tradycyjnych wielowątkowych modelach (Java, C++) wątek może "czekać" na dane sieciowe lub plik — blokując się. JavaScript jest jednowątkowy i nie może blokować — to zatrzymałoby cały interfejs.

Rozwiązanie: operacje I/O są zlecane środowisku (przeglądarce/OS), a JavaScript dostaje powiadomienie (callback) gdy operacja się zakończy. Między zleceniem a powiadomieniem JavaScript może robić inne rzeczy.

### Problem: race conditions w UI

Wielowątkowe podejście do DOM prowadziłoby do wyścigów — dwa wątki modyfikujące DOM jednocześnie dałyby nieprzewidywalne wyniki. Jednowątkowy model z Event Loop eliminuje ten problem — DOM jest zawsze modyfikowany przez jeden wątek, w jednym momencie.

### Historyczne podejście: callback hell

Zanim Promises i async/await, asynchroniczność była obsługiwana przez callbacki zagnieżdżone w callbackach:

```javascript
// "Callback hell" — głęboko zagnieżdżone callbacki
getUserData(userId, function(user) {
    getOrders(user.id, function(orders) {
        getOrderDetails(orders[0].id, function(details) {
            getProductInfo(details.productId, function(product) {
                // 4 poziomy zagnieżdżenia...
                updateUI(product);
            });
        });
    });
});
```

Event Loop jest fundamentem dla wszystkich tych wzorców.

---

## 3. Jak działa

### Pełny cykl Event Loop

Event Loop działa w nieskończonej pętli:

```
┌─────────────────────────────────────────────────────────┐
│                    EVENT LOOP TICK                       │
│                                                          │
│  1. Wykonaj jeden Macrotask (jeśli Call Stack pusty)    │
│     (np. callback z setTimeout)                          │
│                    │                                     │
│                    ▼                                     │
│  2. Opróżnij Microtask Queue (do końca)                  │
│     (wszystkie Promises, queueMicrotask, MutationObserver)│
│                    │                                     │
│                    ▼                                     │
│  3. Renderowanie (jeśli potrzebne):                     │
│     - requestAnimationFrame callbacks                    │
│     - Layout, Paint, Composite                          │
│                    │                                     │
│                    ▼                                     │
│  4. Wróć do punktu 1                                    │
└─────────────────────────────────────────────────────────┘
```

### Macrotask vs Microtask

**Macrotask (Task Queue):**
- `setTimeout`
- `setInterval`
- `setImmediate` (Node.js)
- I/O callbacks
- UI rendering
- Zdarzenia DOM (click, keydown etc.)
- `postMessage`
- `MessageChannel`

**Microtask (Microtask Queue):**
- `.then()`, `.catch()`, `.finally()` (Promises)
- `async/await` (konwertuje na Promises)
- `queueMicrotask()`
- `MutationObserver` callbacks

**Kluczowa różnica**: Po każdym Macrotask, Event Loop opróżnia *całą* Microtask Queue przed przejściem do następnego Macrotask.

### Przykład z pełnym wyjaśnieniem

```javascript
console.log("1"); // synchroniczny

setTimeout(() => console.log("2"), 0); // macrotask

Promise.resolve()
    .then(() => console.log("3"))  // microtask 1
    .then(() => console.log("4")); // microtask 2 (dodany po rozwiązaniu microtask 1)

queueMicrotask(() => console.log("5")); // microtask 3

console.log("6"); // synchroniczny

// Wynik: 1, 6, 3, 5, 4, 2
```

**Krok po kroku:**

1. Call Stack: `main()` → `console.log("1")` → drukuje "1" → pop
2. Call Stack: `main()` → `setTimeout(fn, 0)` → rejestruje u Web API → pop. Web API uruchamia timer 0ms → natychmiast dodaje callback do Macrotask Queue
3. Call Stack: `main()` → `Promise.resolve().then(fn1).then(fn2)` → `fn1` trafia do Microtask Queue; `fn2` zostanie dodane gdy `fn1` się rozwiąże
4. Call Stack: `main()` → `queueMicrotask(fn5)` → `fn5` trafia do Microtask Queue
5. Call Stack: `main()` → `console.log("6")` → drukuje "6" → pop
6. Call Stack: `main()` → pop. Call Stack PUSTY.
7. Event Loop: sprawdza Microtask Queue → `fn1` → drukuje "3" → pop. `fn2` teraz dodana do Microtask Queue.
8. Event Loop: Microtask Queue → `fn5` → drukuje "5" → pop
9. Event Loop: Microtask Queue → `fn2` → drukuje "4" → pop
10. Event Loop: Microtask Queue pusta → bierz z Macrotask Queue → `fn_setTimeout` → drukuje "2"

### requestAnimationFrame

`requestAnimationFrame(cb)` to specjalna kolejka zsynchronizowana z odświeżaniem ekranu (zazwyczaj 60 Hz = co 16.67ms). Callback jest wywoływany przed renderowaniem kadru.

```
Tick Event Loop:
1. Macrotask
2. Microtasks
3. rAF callbacks ← tutaj, przed renderem
4. Render (layout + paint)
5. Następny tick
```

### Starvation Microtask Queue

Nieskończona Microtask Queue blokuje renderowanie i Macrotasks:

```javascript
function flood() {
    Promise.resolve().then(flood); // tworzy nieskończoną pętlę microtasków
}

flood(); // Event Loop utknął w Microtask Queue — UI zamrożony!
```

---

## 4. Co dzieje się wewnętrznie

### HTML Event Loop Spec

Specyfikacja HTML definiuje Event Loop jako algorytm "processing model". Każda karta przeglądarki ma własny Event Loop (zazwyczaj jeden per proces — Process Per Site lub Process Per Tab).

### Task Sources

Macrotask Queue to w rzeczywistości *wiele* kolejek (Task Sources). Każde źródło ma swoje priorytety:

- **User Input** (kliknięcia, klawiatura) — wysoki priorytet
- **Networking** (fetch callbacks)
- **Timers** (setTimeout, setInterval)
- **DOM** manipulation callbacks

Implementacja może różnicować priorytety zadań z różnych źródeł.

### Specjalne zachowania

**setInterval vs setTimeout**:

```javascript
// setTimeout — jednorazowe opóźnienie
// Callback dodany do Macrotask Queue po X ms

// setInterval — powtarzające się
// Nowy callback co X ms — ale jeśli poprzedni callback jest długi,
// następny nie zostanie dodany przed zakończeniem poprzedniego
```

**Promise chaining i microtask queue growth**:

```javascript
Promise.resolve()
    .then(() => "a")
    .then(() => "b")  // dodane do Microtask Queue po rozwiązaniu poprzedniego
    .then(() => "c"); // tak samo

// Wszystkie .then() wykonają się w tym samym "tick" (przed następnym Macrotask)
```

---

## 5. Analogiczny przykład z życia

Wyobraź sobie restaurację z jednym kucharzem (jednowątkowy JS):

- **Kucharzu = Call Stack** — może gotować tylko jedno danie naraz
- **Obsługa sali = Web API** — kelnery mogą równolegle przyjmować zamówienia, czekać na dostawę składników (sieć), nastawiać timery (piekarnik)
- **Notatnik pilnych zadań = Microtask Queue** — "te spodki gotowe, natychmiast udekoruj" — kucharz musi je wykonać ZANIM przyjmie nowe zamówienie ze stolika
- **Kolejka zamówień ze stolików = Macrotask Queue** — normalne zamówienia, obsługiwane jedno po drugim
- **Menedżer = Event Loop** — pyta kucharza "skończyłeś?" → sprawdza pilne notatki → sprawdza zamówienia → przekazuje kucharzowi następne

---

## 6. Przykład kodu

```javascript
// Demonstracja pełnego cyklu Event Loop z async/await
async function fetchUserData(userId) {
    // async function automatycznie zwraca Promise
    // "await" suspenduje wykonanie i dodaje resztę jako microtask

    console.log("1. Zaczyna fetchUserData"); // synchroniczny

    const response = await fetch(`/api/users/${userId}`);
    // Wszystko PO await to microtask.then() pod spodem:
    // Wartość response jest dostępna gdy Promise z fetch się rozwiąże

    console.log("3. Otrzymano odpowiedź"); // wykona się PO synchronicznym kodzie poniżej

    const user = await response.json();
    console.log("4. Sparsowano JSON");

    return user;
}

console.log("Start programu"); // synchroniczny — 1.

fetchUserData(1); // wywołanie — 2.

console.log("Po wywołaniu async function"); // synchroniczny — wciąż synchroniczny!

// Output:
// "Start programu"
// "1. Zaczyna fetchUserData"
// "Po wywołaniu async function"  ← synchroniczny kod PRZED await callback
// (... sieć ...)
// "3. Otrzymano odpowiedź"
// "4. Sparsowano JSON"
```

**Kluczowa obserwacja**: `fetchUserData` zaczyna się synchronicznie, ale przy `await` "zawiesza się" i wraca do main. Dopiero gdy Event Loop odwiedzi Microtask Queue po skończeniu synchronicznego kodu, kontynuuje po `await`.

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — race condition w autoryzacji

```javascript
// Niebezpieczny wzorzec — Race Condition
let isAuthenticated = false;
let authToken = null;

async function login(credentials) {
    const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify(credentials)
    });
    
    const data = await response.json();
    
    // BUG: te dwie operacje nie są atomowe
    // Między nimi może wykonać się inny microtask!
    isAuthenticated = true;
    authToken = data.token;
    
    // Jeśli inna część aplikacji sprawdzi isAuthenticated=true
    // ale authToken=null (jeszcze nie ustawiony) → błąd bezpieczeństwa
}

// Poprawka: aktualizuj stan atomowo
async function login(credentials) {
    const data = await fetchAndParse('/api/login', credentials);
    
    // Atomowa aktualizacja — oba pola ustawiane razem
    Object.assign(authState, {
        isAuthenticated: true,
        authToken: data.token
    });
}
```

### Panel admina — UI flicker przez Event Loop

```javascript
// Bug: UI miga przez rozdzielone aktualizacje DOM
async function loadDashboard() {
    const users = await fetchUsers();
    renderUserList(users); // renderuje częściowy stan

    const stats = await fetchStats();
    renderStats(stats); // drugie renderowanie

    // Przeglądarka renderuje po każdym Macrotask — użytkownik widzi częściowe UI
}

// Poprawka: zbierz wszystkie dane ZANIM zaktualizujesz DOM
async function loadDashboard() {
    const [users, stats] = await Promise.all([fetchUsers(), fetchStats()]);
    
    // Jedno renderowanie z pełnymi danymi
    renderDashboard(users, stats);
    // Lub użyj requestAnimationFrame dla płynności
}
```

### Service Worker — kontrola kolejności przez Event Loop

```javascript
// Service Worker używa Event Loop do zarządzania cache
// (szczegóły w rozdziale 19)
self.addEventListener('fetch', event => {
    event.respondWith(
        // respondWith musi otrzymać Promise
        // Microtask Queue jest tu kluczowa dla kolejności operacji
        caches.match(event.request)
            .then(cached => cached || fetch(event.request))
    );
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Blokowanie Event Loop przez synchroniczny kod

```javascript
// BŁĄD: Długa pętla blokuje UI
function sortLargeArray(arr) {
    return arr.sort((a, b) => a - b); // może trwać sekundy dla dużych tablic
}

// Podczas sortowania: żadne kliknięcia, animacje, timeouty nie działają
// Aplikacja "zamarza"

// POPRAWKA: Web Worker dla ciężkich obliczeń
const worker = new Worker('sort-worker.js');
worker.postMessage(hugeArray);
worker.onmessage = (e) => displaySorted(e.data);
```

### Błąd 2: Zakładanie kolejności asynchronicznych operacji

```javascript
// BŁĄD: zakładanie że response2 jest zawsze po response1
let result1, result2;

fetch('/api/data1').then(r => r.json()).then(data => result1 = data);
fetch('/api/data2').then(r => r.json()).then(data => result2 = data);

// Który fetch odpowie pierwszy? Nie wiadomo!
// result1 może być undefined gdy result2 jest już ustawione

// POPRAWKA: Promise.all dla zależnych danych
const [result1, result2] = await Promise.all([
    fetch('/api/data1').then(r => r.json()),
    fetch('/api/data2').then(r => r.json())
]);
```

### Błąd 3: setTimeout(fn, 0) ≠ natychmiastowe wykonanie

```javascript
// BŁĄD: programista zakłada że timeout 0ms = natychmiastowy
function showModal() {
    const modal = document.getElementById("modal");
    modal.style.display = "block";
    
    setTimeout(() => {
        modal.style.opacity = "1"; // animacja
    }, 0); // próba opóźnienia - może nie zadziałać jak oczekiwano
    
    // Problem: oba efekty mogą być "zbatchowane" przez przeglądarkę
}

// POPRAWKA: requestAnimationFrame dla animacji
function showModal() {
    const modal = document.getElementById("modal");
    modal.style.display = "block";
    
    requestAnimationFrame(() => {
        requestAnimationFrame(() => {
            modal.style.opacity = "1"; // gwarantuje dwie klatki różnicy
        });
    });
}
```

### Błąd 4: Niezatrzymany setInterval

```javascript
// BŁĄD: interval żyje po zniszczeniu komponentu → wyciek pamięci
class Dashboard extends React.Component {
    componentDidMount() {
        this.interval = setInterval(() => this.refresh(), 5000);
    }
    
    // Brak componentWillUnmount!
    // Nawet po unmount, interval próbuje setState → error
}

// POPRAWKA:
componentWillUnmount() {
    clearInterval(this.interval); // zawsze czyść!
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Timing Attacks przez Event Loop

Czas wykonania operacji asynchronicznych może wyciekać informacje:

```javascript
// Timing Oracle — różny czas odpowiedzi ujawnia informacje
async function checkPassword(username, password) {
    const user = await db.findUser(username);
    
    if (!user) {
        return false; // szybka odpowiedź — ujawnia brak użytkownika!
    }
    
    return bcrypt.compare(password, user.hash); // wolna odpowiedź
}

// Atakujący może zmierzyć czas i odróżnić "brak użytkownika" od "zły hasło"
// Poprawka: stały czas odpowiedzi
async function checkPasswordSafe(username, password) {
    const user = await db.findUser(username);
    const dummyHash = "$2b$10$..."; // zawsze uruchom bcrypt
    
    const passwordOk = await bcrypt.compare(
        password, 
        user ? user.hash : dummyHash // zawsze bcrypt, niezależnie od user
    );
    
    return user && passwordOk;
}
```

### Race Conditions w autoryzacji

Asynchroniczna autoryzacja może prowadzić do race conditions:

```javascript
// Podatny wzorzec — TOCTOU (Time-of-Check vs Time-of-Use)
async function transferFunds(userId, amount) {
    const balance = await getBalance(userId); // TIME OF CHECK
    
    // Między check a use — może wykonać się inny request!
    
    if (balance >= amount) { // sprawdzenie
        await deductFunds(userId, amount); // TIME OF USE
        await addFunds(targetId, amount);
    }
}

// Atakujący wysyła dwa identyczne requesty jednocześnie
// Oba pobierają saldo $100
// Oba weryfikują $100 >= $50 → OK
// Oba odejmują $50 → $100 → -$50 (race condition!)
```

### ReDoS i Event Loop Starvation

Złożone wyrażenia regularne blokują Call Stack → blokują Event Loop:

```javascript
// Podatne regex na wejściu użytkownika
const userPattern = req.body.pattern; // atakujący dostarcza (a+)+$

// Testowanie ciągu aaaaaaaaaaaaaaaaab z tym regex
// może zająć minuty → Event Loop zamrożony → DoS
const regex = new RegExp(userPattern);
regex.test(attackerInput); // DoS!
```

### postMessage Security przez Event Loop

`postMessage` dodaje zdarzenia do Macrotask Queue. Kolejność odbierania wiadomości może być podatna jeśli zależy od czasu:

```javascript
// Niezaufana iframe wysyłając wiadomości w odpowiedniej kolejności
// może próbować manipulować stanem aplikacji przez TOCTOU
window.addEventListener("message", (event) => {
    // Zawsze sprawdzaj origin!
    if (event.origin !== "https://trusted.com") return;
    
    // Przetwarzaj wiadomość...
});
```

### Powiązane CWE

- **CWE-362** — Concurrent Execution using Shared Resource with Improper Synchronization (Race Condition)
- **CWE-1333** — Inefficient Regular Expression Complexity (ReDoS → Event Loop starvation)
- **CWE-400** — Uncontrolled Resource Consumption (Event Loop DoS)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Performance Tab

1. Otwórz **DevTools → Performance**
2. Kliknij **Record**
3. Wykonaj podejrzaną akcję
4. **Stop** → analizuj timeline

Szukaj:
- **Long Tasks** (czerwone paski) > 50ms → możliwy ReDoS lub ciężkie obliczenia
- **JavaScript** frames zajmujące cały wątek → blokada Event Loop
- Luki między taskami → skrzywione timing

### DevTools → Console — testowanie timing

```javascript
// Zmierz czy Event Loop jest zablokowany
let start = Date.now();
let ticks = 0;

const interval = setInterval(() => {
    ticks++;
    const elapsed = Date.now() - start;
    const expectedTicks = Math.floor(elapsed / 100);
    
    if (ticks < expectedTicks - 2) {
        console.warn("Event Loop jest opóźniony!");
    }
}, 100);

// Uruchom podejrzaną operację i obserwuj
```

### Identyfikacja ReDoS

```javascript
// Znajdź podatne wzorce regex w kodzie
// W Sources → Ctrl+Shift+F
const vulnerablePatterns = [
    /\(.*\+\)\+/,   // (a+)+
    /\(.*\*\)\*/,   // (a*)*
    /\(.*\|\).*\1/, // alternacje z backtracking
];

// Zmierz czas konkretnego regex
const testInput = "a".repeat(30) + "b";
const start = performance.now();
/^(a+)+$/.test(testInput);
console.log(`Czas: ${performance.now() - start}ms`); // >1000ms → podatny
```

### Race Condition Testing — Burp Suite

1. W Burp Suite → Proxy → przechwyć żądanie (np. transfer)
2. Send to Repeater
3. Stwórz wiele zakładek Repeater z tym samym żądaniem
4. Wysyłaj jednocześnie (Ctrl+click "Send" w wielu zakładkach)
5. Lub użyj Burp Suite Turbo Intruder z `race_single_packet_attack`

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja jest podatna na Race Condition (jednoczesne requesty modyfikujące stan)?
□ Czy walidacja regex jest podatna na ReDoS (test z catastrophic backtracking)?
□ Czy Long Tasks (>50ms) w Performance tab wskazują na blokowanie Event Loop?
□ Czy setInterval/setTimeout są właściwie czyszczone (clearInterval/clearTimeout)?
□ Czy operacje autoryzacji są atomowe (brak TOCTOU między check a use)?
□ Czy timing odpowiedzi ujawnia informacje (username enumeration, valid vs invalid)?
□ Czy aplikacja obsługuje jednoczesne requesty bezpiecznie (np. podwójne wydatki)?
□ Czy infinite microtask loops mogą być wywołane przez dane użytkownika?
□ Czy postMessage handlers sprawdzają origin (nie tylko zakładają kolejność)?
□ Czy animacje/timery są czyszczone przy wylogowaniu (brak wycieków)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Race Condition w przelewie

**Wymagania:** Endpoint przelewu nie ma transakcyjnej blokady.

**Przebieg (Burp Suite Turbo Intruder):**
```python
# Burp Turbo Intruder script
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=20,
                          requestsPerConnection=1)
    for _ in range(20):
        engine.queue(target.req)  # 20 identycznych requestów jednocześnie

def handleResponse(req, interesting, res, t):
    if "success" in res.content.lower():
        table.add(t, res.status, len(res.content), req.params['amount'])
```

**Cel:** Wysłać $100 więcej razy niż pozwala saldo.

**Identyfikacja:** Sprawdź czy balance_after < 0 lub transakcja wykonana N razy.

### Scenariusz 2: Event Loop DoS przez ReDoS

**Wymagania:** Endpoint walidujący dane regex (Node.js).

**Przebieg:**
```bash
# Znalezienie podatnego endpointu
curl -X POST /api/validate \
     -H "Content-Type: application/json" \
     -d '{"email": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab@"}'

# Zmierz czas odpowiedzi
time curl -s -X POST /api/validate -d '{"email": "'"$(python3 -c "print('a'*50 + '@')")"'"}'
# Jeśli > 10s → EventLoop zablokowany → DoS
```

**Wpływ:** Serwer Node.js nie odpowiada przez czas ataku.

---

## 13. Jak się zabezpieczać

### Atomowe operacje przez transakcje bazodanowe

```javascript
// Poprawka Race Condition — użyj transakcji DB zamiast logiki JS
async function transferFunds(fromId, toId, amount) {
    // Transakcja DB gwarantuje atomowość
    await db.transaction(async (trx) => {
        const [from] = await trx('accounts')
            .where({ id: fromId })
            .forUpdate() // SELECT FOR UPDATE — blokuje wiersz
            .select();
        
        if (from.balance < amount) {
            throw new Error("Insufficient funds");
        }
        
        await trx('accounts').where({ id: fromId }).decrement('balance', amount);
        await trx('accounts').where({ id: toId }).increment('balance', amount);
    });
    // Transakcja gwarantuje że oba UPDATE wykonują się atomowo
}
```

### Ochrona przed ReDoS

```javascript
// Używaj biblioteki weryfikującej bezpieczeństwo regex
const safeRegex = require('safe-regex');
const recheck = require('recheck');

function validateUserRegex(pattern) {
    if (!safeRegex(pattern)) {
        throw new Error("Potentially vulnerable regular expression");
    }
    return new RegExp(pattern);
}

// Timeout dla walidacji regex
function testWithTimeout(regex, input, timeoutMs = 100) {
    return new Promise((resolve, reject) => {
        const timeout = setTimeout(() => reject(new Error("Regex timeout")), timeoutMs);
        try {
            const result = regex.test(input);
            clearTimeout(timeout);
            resolve(result);
        } catch (e) {
            clearTimeout(timeout);
            reject(e);
        }
    });
}
```

### Stały czas odpowiedzi dla autoryzacji

```javascript
const crypto = require('crypto');

// Stałe-czasowe porównanie stringów (timing-safe)
function timingSafeEqual(a, b) {
    const bufA = Buffer.from(a);
    const bufB = Buffer.from(b);
    
    if (bufA.length !== bufB.length) {
        // Nie zwracaj false — zawsze porównuj pełne bufory
        return crypto.timingSafeEqual(
            bufA, 
            Buffer.alloc(bufA.length) // dummy dla zachowania stałego czasu
        ) && false; // zawsze false, ale stały czas
    }
    
    return crypto.timingSafeEqual(bufA, bufB);
}
```

### Idempotentne operacje

```javascript
// Zapobieganie double-spending przez idempotency keys
app.post('/api/transfer', async (req, res) => {
    const idempotencyKey = req.headers['idempotency-key'];
    
    if (!idempotencyKey) {
        return res.status(400).json({ error: "Idempotency-Key required" });
    }
    
    // Sprawdź czy ten klucz był już użyty
    const existing = await redis.get(`transfer:${idempotencyKey}`);
    if (existing) {
        return res.json(JSON.parse(existing)); // zwróć poprzedni wynik
    }
    
    const result = await processTransfer(req.body);
    
    // Zapisz z TTL = 24h
    await redis.setex(`transfer:${idempotencyKey}`, 86400, JSON.stringify(result));
    
    res.json(result);
});
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Event Loop zarządza asynchronicznością w jednowątkowym JS — to mechanizm przeglądarki/Node.js, nie ECMAScript
2. Microtask Queue (Promise, queueMicrotask) ma wyższy priorytet niż Macrotask Queue (setTimeout, DOM events)
3. Blokada Call Stack = blokada Event Loop = zamrożenie UI i wszystkich callbacks
4. Race Conditions są możliwe nawet w jednowątkowym JS przez asynchroniczne operacje
5. ReDoS może blokować Event Loop przez złożone wyrażenia regularne

**Najczęstsze nieporozumienia:**

- "setTimeout(fn, 0) wykonuje się natychmiast" — nie, callback trafia do Macrotask Queue, wykona się po synchronicznym kodzie i wszystkich microtaskach
- "JavaScript jest wielowątkowy bo ma async" — NIE. Async = delegowanie do Web API, nie wielowątkowość w JS
- "Promise.then() i setTimeout(fn, 0) to to samo" — NIE. Promise.then() to Microtask (wyższy priorytet), setTimeout to Macrotask

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Performance** → Record → wykonaj akcję → szukaj Long Tasks (>50ms) — wskaźnik blokady
2. **DevTools → Console** → sprawdź kolejność logów z `setTimeout(fn, 0)` i `Promise.resolve().then(fn)` — czy aplikacja zakłada konkretną kolejność?
3. **Burp Suite** → wyślij identyczne requesty jednocześnie (Repeater × 2) → sprawdź czy race condition modyfikuje stan niepoprawnie
4. **Network tab** → sprawdź `waterfall` — czy requesty są wysyłane sekwencyjnie gdy powinny być równoległe (marnowanie czasu)?

---

## Powiązania

```
Event Loop
        │
        ├──► Call Stack (Rozdział 3)
        │         Event Loop monitoruje Call Stack i przenosi zadania gdy jest pusty
        │
        ├──► Closures (Rozdział 5)
        │         Callbacki asynchroniczne to closures na swój kontekst
        │
        ├──► postMessage (Rozdział 23)
        │         postMessage dodaje zdarzenia do Macrotask Queue
        │
        ├──► Web Workers (Rozdział 20)
        │         Workers mają własny Event Loop — nie blokują głównego
        │
        ├──► Service Workers (Rozdział 19)
        │         SW ma własny Event Loop i może interceptować requesty
        │
        └──► WebSocket (Rozdział 25)
                  WebSocket events trafiają do Macrotask Queue przez Event Loop
```
