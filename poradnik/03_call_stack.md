# Rozdział 3: Call Stack

## 1. Czym jest Call Stack

**Call Stack** (stos wywołań) to struktura danych LIFO (Last In, First Out — ostatni wchodzi, pierwszy wychodzi) używana przez silnik JavaScript do śledzenia, które funkcje są aktualnie wykonywane. Każde wywołanie funkcji dodaje nowy element na wierzchołek stosu (**push**). Gdy funkcja zwraca wartość, element jest zdejmowany ze stosu (**pop**).

Call Stack jest częścią **silnika JavaScript** (ECMAScript) — nie Web API. Każdy silnik implementuje go niezależnie, ale mechanizm jest taki sam. W V8 jest on zaimplementowany w C++ jako stos ramek (stack frames).

Call Stack ma ograniczony rozmiar. Każdy silnik/przeglądarka definiuje maksymalną głębokość stosu — zazwyczaj kilka tysięcy ramek (V8: ~10 000 przy domyślnym stosie). Przekroczenie limitu powoduje **Stack Overflow** — `RangeError: Maximum call stack size exceeded`.

### API i narzędzia związane z Call Stack

Call Stack nie jest bezpośrednio dostępny w JavaScript. Pośrednio widoczny jest przez:
- `Error().stack` — zwraca stack trace jako string
- `console.trace()` — drukuje aktualny stack trace do konsoli
- DevTools → zakładka **Sources** podczas debugowania — panel **Call Stack**
- Błędy `RangeError: Maximum call stack size exceeded`

---

## 2. Dlaczego powstał

### Problem: śledzenie powrotu z funkcji

Gdy program wywołuje funkcję, musi wiedzieć *gdzie wrócić* po jej zakończeniu. W asemblerze adres powrotu jest odkładany na stos procesora. JavaScript — jako język wyższego poziomu — potrzebuje analogicznego mechanizmu.

Dodatkowo, każde wywołanie funkcji tworzy własny Execution Context z własnymi zmiennymi lokalnymi. Call Stack przechowuje te konteksty w odpowiedniej kolejności — kontekst bieżącej funkcji jest zawsze na wierzchołku.

### Dlaczego LIFO (stos, nie kolejka)?

Funkcje zagnieżdżają się: A wywołuje B, B wywołuje C. C musi wrócić do B *zanim* B wróci do A. To naturalna kolejność stosu — ostatnio wywołana funkcja wraca pierwsza.

```
A() wywołuje B() wywołuje C()

Stos (od dołu do góry):
┌─────┐
│  C  │ ← aktualnie wykonywana
├─────┤
│  B  │ ← czeka na C
├─────┤
│  A  │ ← czeka na B
├─────┤
│MAIN │ ← globalny EC
└─────┘

C() zwraca → pop C → wykonuje B
B() zwraca → pop B → wykonuje A
A() zwraca → pop A → wykonuje MAIN
```

---

## 3. Jak działa

### Ramka stosu (Stack Frame)

Każdy element Call Stack to **ramka stosu** (stack frame lub activation record). Zawiera:

- Referencję do Execution Context (zmienne lokalne, this)
- Adres powrotu (gdzie wrócić po zakończeniu)
- Argumenty przekazane do funkcji
- Wskaźnik na poprzednią ramkę

### Krok po kroku

**Krok 1: Start programu**

Silnik tworzy Global Execution Context i umieszcza go na Call Stack:

```
┌─────────────────┐
│  global main()  │  ← na stosie
└─────────────────┘
```

**Krok 2: Wywołanie funkcji**

```javascript
function greet(name) {
    return "Hello, " + name;
}

let msg = greet("Alice");
```

Silnik napotyka `greet("Alice")`:
1. Tworzy nowy FEC (Function Execution Context) dla `greet`
2. Umieszcza ramkę `greet` na wierzchołku stosu

```
┌─────────────────┐
│  greet("Alice") │  ← aktualnie wykonywana
├─────────────────┤
│  global main()  │
└─────────────────┘
```

**Krok 3: Zwrot z funkcji**

`greet` zwraca wartość `"Hello, Alice"`. Silnik:
1. Zdejmuje ramkę `greet` ze stosu (pop)
2. Kontynuuje w `global main()` — przypisuje wynik do `msg`

```
┌─────────────────┐
│  global main()  │  ← kontynuuje
└─────────────────┘
```

### Zagnieżdżone wywołania

```javascript
function a() {
    console.log("a start");
    b();
    console.log("a end");
}

function b() {
    console.log("b start");
    c();
    console.log("b end");
}

function c() {
    console.log("c");
}

a();
```

Timeline Call Stack:

```
Czas →

[main]
[main][a]
[main][a][console.log]  → "a start"
[main][a]
[main][a][b]
[main][a][b][console.log]  → "b start"
[main][a][b]
[main][a][b][c]
[main][a][b][c][console.log]  → "c"
[main][a][b][c]
[main][a][b]  → "b end"
[main][a]  → "a end"
[main]
[]  (stos pusty)
```

### Rekurencja i Stack Overflow

Rekurencja bez warunku stopu:

```javascript
function infinite() {
    infinite(); // zawsze wywołuje siebie
}

infinite(); // RangeError: Maximum call stack size exceeded
```

Stos rośnie bez ograniczeń:

```
[main][infinite][infinite][infinite][infinite]...[limit osiągnięty]
```

### Asynchroniczność a Call Stack

Funkcje asynchroniczne (setTimeout, fetch, Promise) *nie blokują* Call Stack:

```javascript
console.log("1");

setTimeout(() => {
    console.log("3"); // callback wykona się gdy Call Stack będzie PUSTY
}, 0);

console.log("2");

// Wynik: 1, 2, 3
```

Gdy Call Stack jest zajęty (np. długa pętla synchroniczna), callbacki czekają w Callback Queue. Event Loop przenosi je na Call Stack dopiero gdy jest pusty.

---

## 4. Co dzieje się wewnętrznie

### Stos systemowy vs Call Stack JavaScript

V8 używa systemowego stosu procesu operacyjnego (OS stack) do implementacji Call Stack. Każda ramka JS to ramka na stosie C++. Dlatego limit stosu JS jest powiązany z limitem stosu systemu operacyjnego.

### Stack Trace — jak jest budowany

Gdy rzucany jest błąd (`throw` lub wyjątek):

```javascript
function c() {
    throw new Error("Coś poszło nie tak");
}
function b() { c(); }
function a() { b(); }

try {
    a();
} catch (e) {
    console.log(e.stack);
}
```

Wynik:

```
Error: Coś poszło nie tak
    at c (script.js:2:11)
    at b (script.js:4:17)
    at a (script.js:5:17)
    at script.js:8:5
```

Stack trace jest czytany od dołu do góry: `main → a → b → c`. Top of stack trace = miejsce błędu.

### Tail Call Optimization (TCO)

ES2015 wprowadził Tail Call Optimization (TCO) — jeśli wywołanie rekurencyjne jest ostatnią operacją funkcji, silnik *nie* dodaje nowej ramki, lecz zastępuje bieżącą:

```javascript
// Bez TCO: O(n) ramek na stosie
function factorial(n, acc = 1) {
    if (n <= 1) return acc;
    return factorial(n - 1, n * acc); // tail call — tylko Safari to optymalizuje w JS
}
```

W praktyce TCO jest zaimplementowane tylko w JavaScriptCore (Safari). V8 i SpiderMonkey go nie implementują.

### V8 Frame Pointer Omission

V8 może pomijać frame pointers dla optymalizacji. Utrudnia to debugowanie, ale poprawia wydajność. DevTools v8 kompensuje to przechowując shadow stack dla symbolizacji.

---

## 5. Analogiczny przykład z życia

Wyobraź sobie bibliotekarza obsługującego złożone zapytanie:

1. Czytelnik pyta o "historię Napoleona" (wywołanie `a()`)
2. Bibliotekarz podnosi notes zadań (stos) i zapisuje: "obsługuję: historię Napoleona"
3. Musi sprawdzić katalog — odkłada notes, bierze katalog (wywołanie `b()`)
4. W katalogu musi sprawdzić datę (wywołanie `c()`)
5. Sprawdza datę → wraca do katalogu → wraca do pytania o historię → odpowiada czytelnikowi

Każde "schowanie" wcześniejszego zadania i zajęcie się nowym to push na Call Stack. Każde "wróciłem" to pop ze stosu. Bibliotekarz zawsze wraca dokładnie tam, skąd wyszedł.

**Stack Overflow** = czytelnik pyta pytanie, które generuje nieskończony łańcuch podpytań. Bibliotekarz w końcu zapełnia wszystkie notes i krzyczy "brak miejsca!".

---

## 6. Przykład kodu

```javascript
function validateUser(userId) {
    // Ramka: validateUser
    const user = fetchUserFromDB(userId); // push fetchUserFromDB
    return user !== null;                  // pop fetchUserFromDB, powrót tutaj
}

function fetchUserFromDB(id) {
    // Ramka: fetchUserFromDB (nad validateUser)
    const query = buildQuery(id); // push buildQuery
    return executeQuery(query);    // pop buildQuery, push executeQuery, pop executeQuery
}

function buildQuery(id) {
    // Ramka: buildQuery (nad fetchUserFromDB nad validateUser)
    return `SELECT * FROM users WHERE id = ${id}`;
    // UWAGA: interpolacja bez sanityzacji — SQL Injection jeśli id pochodzi od użytkownika
}

function executeQuery(query) {
    // Symulacja
    return { id: 1, name: "Alice" };
}

// Wywołanie — zaczyna proces
const result = validateUser("1 OR 1=1"); // SQL Injection przez Call Stack!
```

**Wyjaśnienie Call Stack:**

```
Moment 1: [global]
Moment 2: [global][validateUser("1 OR 1=1")]
Moment 3: [global][validateUser][fetchUserFromDB("1 OR 1=1")]
Moment 4: [global][validateUser][fetchUserFromDB][buildQuery("1 OR 1=1")]
  → zwraca: "SELECT * FROM users WHERE id = 1 OR 1=1"  ← SQL Injection!
Moment 5: [global][validateUser][fetchUserFromDB][executeQuery("SELECT...")]
  → zwraca: wynik zapytania
Moment 6: [global][validateUser]
  → zwraca: true
Moment 7: [global]
```

Debugger zatrzymany w `buildQuery` pokazałby pełny stack trace — pentester widzi *ścieżkę wywołania* i może śledzić jak dane płyną przez aplikację.

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja e-commerce — analiza stack trace

```javascript
// Minifikowany kod produkcyjny (bez source map):
// Błąd w konsoli: "Cannot read property 'price' of undefined at e.js:1:2847"

// Z source map:
function calculateOrderTotal(order) {      // <- widać w stack trace
    return order.items.reduce((sum, item) => {
        return sum + getItemPrice(item);   // <- linia błędu
    }, 0);
}

function getItemPrice(item) {
    return item.product.price * item.quantity; // <- TypeError jeśli product = null
}
```

Stack trace ujawniony przez `error.stack`:
```
TypeError: Cannot read property 'price' of undefined
    at getItemPrice (checkout.js:145:25)
    at calculateOrderTotal (checkout.js:138:20)
    at processCheckout (checkout.js:89:15)
    at handleFormSubmit (app.js:234:10)
```

**Znaczenie dla pentestera:** Stack trace może ujawniać:
- Ścieżki plików serwera (jeśli błąd serwerowy)
- Wewnętrzną architekturę aplikacji
- Nazwy funkcji i zmiennych (bez minifikacji)
- Wersje bibliotek (przez nazwy i linie)

### Stack Overflow jako wektor DoS

```javascript
// Rekurencja wywoływana danymi użytkownika
function parseNestedJSON(data) {
    if (typeof data === 'object') {
        return Object.keys(data).reduce((acc, key) => {
            acc[key] = parseNestedJSON(data[key]); // rekurencja
            return acc;
        }, {});
    }
    return data;
}

// Payload atakującego: głęboko zagnieżdżony JSON
// {"a":{"a":{"a":{"a":{"a":...}}}}}}  — 10000 poziomów
// → Stack Overflow → crash aplikacji → DoS
const maliciousInput = JSON.parse('{"a":'.repeat(10000) + '1' + '}'.repeat(10000));
parseNestedJSON(maliciousInput); // RangeError!
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brakujący warunek zakończenia rekurencji

```javascript
// Błąd: brak warunku stopu
function countDown(n) {
    console.log(n);
    countDown(n - 1); // RangeError gdy stos się przepełni
}

// Dobrze:
function countDown(n) {
    if (n <= 0) return; // warunek stopu
    console.log(n);
    countDown(n - 1);
}
```

### Błąd 2: Wzajemna rekurencja

```javascript
// Mniej oczywisty stack overflow
function isEven(n) {
    if (n === 0) return true;
    return isOdd(n - 1);   // wywołuje isOdd
}

function isOdd(n) {
    if (n === 0) return false;
    return isEven(n - 1);  // wywołuje isEven → stack overflow dla dużych n
}

isEven(100000); // RangeError
```

### Błąd 3: Ujawnianie stack trace w produkcji

```javascript
// Błąd: wysyłanie stack trace do klienta
app.use((err, req, res, next) => {
    res.status(500).json({
        error: err.message,
        stack: err.stack // NIGDY w produkcji!
    });
});

// Dobrze:
app.use((err, req, res, next) => {
    logger.error(err); // loguj wewnętrznie
    res.status(500).json({ error: "Internal Server Error" }); // ogólny błąd dla klienta
});
```

### Błąd 4: Synchroniczna blokada Call Stack

```javascript
// Błąd: długa pętla blokuje UI i callbacki
function findInLargeArray(arr, target) {
    for (let i = 0; i < arr.length; i++) { // może trwać sekundy
        if (arr[i] === target) return i;
    }
    return -1;
}

// Konsekwencja: żaden callback (timeout sesji, animacje, eventy) nie wykona się
// podczas działania tej funkcji

// Rozwiązanie: chunking z setTimeout/requestIdleCallback
async function findInLargeArrayAsync(arr, target) {
    const CHUNK = 1000;
    for (let i = 0; i < arr.length; i += CHUNK) {
        const chunk = arr.slice(i, i + CHUNK);
        const found = chunk.indexOf(target);
        if (found !== -1) return i + found;
        await new Promise(r => setTimeout(r, 0)); // oddaj kontrolę Event Loop
    }
    return -1;
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Stack Trace Information Disclosure

Ujawniony stack trace to informacja wywiadowcza dla atakującego:
- Wewnętrzna struktura katalogów serwera: `/home/app/src/controllers/UserController.js:145`
- Nazwy i wersje frameworków
- Wewnętrzna logika (nazwy funkcji, przepływ kontroli)
- Potencjalne wektory ataku (co wywołuje co)

**CWE-209** — Generation of Error Message Containing Sensitive Information

### ReDoS — Regular Expression Denial of Service

Złożone wyrażenia regularne mogą generować eksponencjalne cofanie (backtracking) — każdy krok cofania to wywołanie funkcji, które może przepełnić Call Stack lub zamrozić Event Loop:

```javascript
// Podatne regex (catastrophic backtracking)
const vulnerableRegex = /^(a+)+$/;
// Input: "aaaaaaaaaaaaaaaaaab"
// Silnik cofa setki tysięcy razy zanim stwierdzi brak dopasowania
vulnerableRegex.test("aaaaaaaaaaaaaaaaaab"); // zamraża Event Loop na sekundy/minuty
```

**CWE-1333** — Inefficient Regular Expression Complexity (ReDoS)

### Stack Overflow jako DoS

Aplikacje przyjmujące zagnieżdżone dane użytkownika (JSON, XML, obiekty) bez ograniczenia głębokości są podatne na celowe przepełnienie stosu.

### Prototype Pollution przez Call Stack

Gdy aplikacja rekurencyjnie mergeuje obiekty bez sprawdzania kluczy:

```javascript
function merge(target, source) {
    for (let key in source) {
        if (typeof source[key] === 'object') {
            target[key] = target[key] || {};
            merge(target[key], source[key]); // rekurencja
        } else {
            target[key] = source[key];
        }
    }
}

// Payload: {"__proto__": {"admin": true}}
// Call Stack: merge → merge (dla __proto__) → przypisuje admin do Object.prototype
```

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Call Stack Panel

1. Otwórz **Sources** → ustaw breakpoint w dowolnej funkcji
2. Wywołaj akcję w aplikacji
3. Gdy wykonanie się zatrzyma, po prawej stronie zobaczysz panel **Call Stack**
4. Lista pokazuje aktualny stos wywołań od góry (bieżąca) do dołu (global)

```
Panel Call Stack (przykład):
▶ processPayment        payment.js:45
  handleFormSubmit      app.js:234
  EventListener.submit  (anonymous)
  (anonymous)           index.html:12
```

Kliknij dowolną ramkę → skoczysz do tej linii kodu → zobaczysz jej lokalny Scope.

### Szukanie ujawnionych stack traces

W Burp Suite → szukaj w odpowiedziach:
```
at .* \(.*\.js:\d+:\d+\)
Error:.*\n.*at
Traceback \(most recent call last\)
```

Regularne wyrażenie w Burp → Proxy → HTTP History → Response Body.

### Testowanie ReDoS

```javascript
// W konsoli DevTools — przetestuj czy regex jest podatny
const start = Date.now();
/^(a+)+$/.test("a".repeat(20) + "b");
console.log(Date.now() - start + "ms"); // jeśli >1000ms → podatny
```

### Testowanie głębokości rekurencji

```python
# Burp Intruder / ręcznie — payload zagnieżdżonego JSON
import json

def nested_json(depth):
    data = "1"
    for _ in range(depth):
        data = f'{{"a":{data}}}'
    return data

# Wyślij z depth=100, 500, 1000, 5000, 10000
# Sprawdź czy serwer zwraca 500 lub się zawiesza
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy stack traces są ujawniane w odpowiedziach HTTP (status 500)?
□ Czy stack traces pojawiają się w konsoli JS w produkcji?
□ Czy aplikacja przyjmuje głęboko zagnieżdżone dane bez limitu głębokości?
□ Czy wyrażenia regularne są podatne na ReDoS (catastrophic backtracking)?
□ Czy rekurencyjne funkcje mają ograniczenia głębokości?
□ Czy nagłówek X-Powered-By lub inne ujawniają wersję frameworka?
□ Czy synchroniczne operacje blokują Event Loop (DoS przez obciążenie CPU)?
□ Czy source maps są dostępne w produkcji (ujawniają oryginalny kod)?
□ Czy Error handling ujawnia wewnętrzne ścieżki plików?
□ Czy użyte biblioteki mają znane podatności CVE (weryfikacja przez npm audit)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Information Disclosure przez Stack Trace

**Wymagania:** Aplikacja nie obsługuje błędów i ujawnia wyjątki klientowi.

**Przebieg:**
1. Pentester wysyła celowo nieprawidłowe dane wejściowe (null byte, specjalne znaki, zbyt długi ciąg)
2. Aplikacja rzuca wyjątek bez obsługi błędów
3. Stack trace jest zwracany w odpowiedzi HTTP (status 500)
4. Pentester analizuje ścieżki plików, nazwy klas, wersje

**Wpływ:** MEDIUM — informacje wywiadowcze ułatwiające kolejne ataki.

### Scenariusz 2: ReDoS — Zamrożenie Event Loop

**Wymagania:** Aplikacja waliduje dane użytkownika podatnym wyrażeniem regularnym.

**Przebieg:**
1. Pentester identyfikuje pola walidowane po stronie serwera (Node.js)
2. Testuje czas odpowiedzi z normalnym inputem: 50ms
3. Tworzy payload wywołujący catastrophic backtracking
4. Wysyła payload — serwer nie odpowiada przez kilka sekund
5. Serwer Node.js (jednowątkowy jak przeglądarka) nie może obsługiwać innych requestów

**Wpływ:** HIGH — Denial of Service bez uwierzytelnienia.

### Scenariusz 3: Stack Overflow przez zagnieżdżony JSON

**Wymagania:** API przyjmuje dowolne JSON body i przetwarza je rekurencyjnie.

**Przebieg:**
```python
import requests, json

def nested(depth):
    d = "1"
    for _ in range(depth):
        d = f'{{"x":{d}}}'
    return d

payload = nested(10000)
requests.post("/api/process", data=payload, 
              headers={"Content-Type": "application/json"})
# → Stack Overflow w Node.js → crash procesu → DoS
```

**Wpływ:** CRITICAL — crash serwera prowadzący do niedostępności usługi.

---

## 13. Jak się zabezpieczać

### Ukryj stack traces w produkcji

```javascript
// Express.js — middleware obsługi błędów
app.use((err, req, res, next) => {
    // Loguj szczegóły wewnętrznie
    logger.error({
        message: err.message,
        stack: err.stack,
        url: req.url,
        method: req.method
    });
    
    // Odpowiedź dla klienta — minimum informacji
    const isProduction = process.env.NODE_ENV === 'production';
    res.status(500).json({
        error: isProduction ? 'Internal Server Error' : err.message
        // stack NIGDY nie trafia do klienta
    });
});
```

### Ogranicz głębokość rekurencji

```javascript
function safeParseNested(data, maxDepth = 10, currentDepth = 0) {
    if (currentDepth > maxDepth) {
        throw new Error("Maximum nesting depth exceeded");
    }
    
    if (typeof data !== 'object' || data === null) return data;
    
    return Object.keys(data).reduce((acc, key) => {
        acc[key] = safeParseNested(data[key], maxDepth, currentDepth + 1);
        return acc;
    }, {});
}
```

### Waliduj rozmiar inputu JSON

```javascript
// Express.js — ogranicz rozmiar body
const express = require('express');
app.use(express.json({ limit: '100kb' })); // domyślnie 100kb — dostosuj do potrzeb

// Flatted/fast-json-stringify z limitem głębokości
const flatted = require('flatted');
```

### Bezpieczne regex

```javascript
// Unikaj nested quantifiers: (a+)+ (a*)* (a+)*
// Używaj atomic groups lub possessive quantifiers (gdy dostępne)
// Testuj z biblioteką safe-regex lub vuln-regex-detector

const safeRegex = require('safe-regex');
const userRegex = new RegExp(userInput);

if (!safeRegex(userRegex)) {
    throw new Error("Potentially vulnerable regular expression");
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Call Stack śledzi wykonywanie funkcji — każde wywołanie to push, każdy return to pop
2. Stack jest synchroniczny i jednowątkowy — tylko jedna ramka jest aktywna naraz
3. Przepełnienie stosu (Stack Overflow) = za głęboka rekurencja → `RangeError`
4. Stack Trace ujawnia wewnętrzną strukturę aplikacji — ukrywaj w produkcji
5. Blokowanie Call Stack blokuje cały UI i wszystkie callbacki

**Najczęstsze nieporozumienia:**

- "Asynchroniczne funkcje są na Call Stack" — NIE. `setTimeout`, `fetch` etc. są obsługiwane poza Call Stack przez Web API. Callback wraca na Call Stack dopiero przez Event Loop.
- "Stack Overflow to błąd przeglądarki" — to błąd logiki programu (brakujący warunek zakończenia rekurencji).
- "Stack trace jest widoczny tylko w devtools" — może być wysyłany w HTTP response jeśli aplikacja go nie ukryje.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Sources** → ustaw breakpoint → wywołaj akcję → panel **Call Stack** po prawej
2. **Console** → `console.trace()` wewnątrz kodu → wydrukuje aktualny stack
3. **Network** → odpowiedzi 500 → sprawdź body pod kątem stack trace
4. **Console** → szukaj logów `Error:` z `at funkcja (plik:linia:kolumna)`
5. Burp Suite → Response body → regex `at .* \(.*:\d+:\d+\)` → stack trace

---

## Powiązania

```
Call Stack
        │
        ├──► Execution Context (Rozdział 2)
        │         każda ramka Call Stack = jeden EC
        │
        ├──► Event Loop (Rozdział 8)
        │         Event Loop przenosi callbacki na Call Stack gdy jest pusty
        │
        ├──► Prototype Pollution (Rozdział 7)
        │         rekurencyjny merge obiektów → Call Stack → pollutia
        │
        └──► Service Workers (Rozdział 19)
                  Service Worker ma własny, niezależny Call Stack
```
