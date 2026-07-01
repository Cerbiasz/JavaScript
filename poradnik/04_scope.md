# Rozdział 4: Scope

## 1. Czym jest Scope

**Scope** (zakres widoczności) to zestaw reguł określający, które zmienne i funkcje są dostępne w danym miejscu kodu. Scope definiuje "widoczność" identyfikatorów — z jakiego miejsca możesz "zobaczyć" i użyć danej zmiennej.

Scope w JavaScript jest **leksykalny** (statyczny) — oznacza to, że zakres zmiennej jest określany przez jej *fizyczne położenie* w kodzie źródłowym, a nie przez sposób wywołania funkcji. Scope jest ustalany przez parser podczas fazy parsowania — zanim kod zostanie wykonany.

Scope nie jest obiektem dostępnym programiście. To abstrakcja opisująca Lexical Environment tworzony przez Execution Context.

### Rodzaje Scope

1. **Global Scope** — zmienne dostępne wszędzie w programie. W przeglądarce to właściwości `window`.
2. **Function Scope** — zmienne `var` są dostępne wewnątrz funkcji, w której zostały zadeklarowane, i we wszystkich zagnieżdżonych funkcjach.
3. **Block Scope** — zmienne `let` i `const` są dostępne tylko wewnątrz bloku `{}`, w którym zostały zadeklarowane.
4. **Module Scope** — zmienne w module ES6 są lokalne dla modułu (nie trafiają do `window`).

### Czy jest częścią JavaScript

Scope jest częścią specyfikacji ECMAScript — opisany w kontekście Lexical Environments.

---

## 2. Dlaczego powstał

### Problem: kolizje nazw i enkapsulacja

Wyobraź sobie duży program bez żadnego zakresu widoczności — wszystkie zmienne byłyby globalne. Dwie biblioteki definiujące `var count = 0` kolidowałyby ze sobą. Każda zmiana jednej zmiennej wpływałaby na cały program.

Scope rozwiązuje ten problem poprzez:
1. **Enkapsulację** — zmienne lokalne są niewidoczne z zewnątrz
2. **Bezpieczeństwo** — wrażliwe dane mogą być ukryte wewnątrz funkcji
3. **Reużywalność** — funkcje mogą używać lokalnych nazw bez martwienia się o kolizje

### Ewolucja: od var do let/const

Oryginalne `var` ma scope funkcyjny, co prowadziło do nieoczekiwanych zachowań:

```javascript
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);
}
// Wynik: 3, 3, 3 — nie 0, 1, 2!
// var i jest FUNKCYJNY, nie blokowy — jedna zmienna i dla całej pętli
```

ES6 (2015) wprowadził `let` i `const` z scope blokowym, rozwiązując ten problem:

```javascript
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 0);
}
// Wynik: 0, 1, 2 — każda iteracja ma własne i
```

---

## 3. Jak działa

### Scope Chain (łańcuch zakresów)

Gdy kod szuka zmiennej, silnik JS przeszukuje Scope Chain — serię Lexical Environments połączonych referencjami "outer":

```
┌──────────────────────────────────────────┐
│          GLOBAL SCOPE                    │
│  globalVar = "global"                    │
│  ↑                                       │
│  │ outer reference                       │
│  ├──────────────────────────────────┐    │
│  │     OUTER FUNCTION SCOPE         │    │
│  │  outerVar = "outer"              │    │
│  │  ↑                               │    │
│  │  │ outer reference               │    │
│  │  ├─────────────────────────┐     │    │
│  │  │  INNER FUNCTION SCOPE   │     │    │
│  │  │  innerVar = "inner"     │     │    │
│  │  │                         │     │    │
│  │  │  console.log(innerVar)  │     │    │
│  │  │    → szuka w Inner ✓    │     │    │
│  │  │  console.log(outerVar)  │     │    │
│  │  │    → nie w Inner        │     │    │
│  │  │    → szuka w Outer ✓    │     │    │
│  │  │  console.log(globalVar) │     │    │
│  │  │    → nie w Inner/Outer  │     │    │
│  │  │    → szuka w Global ✓   │     │    │
│  │  └─────────────────────────┘     │    │
│  └──────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

### Var vs Let/Const — zasadnicza różnica

```javascript
function example() {
    // var — function scope
    if (true) {
        var funcScoped = "widzę cały function scope";
    }
    console.log(funcScoped); // "widzę cały function scope" — działa!
    
    // let — block scope
    if (true) {
        let blockScoped = "widzę tylko blok";
    }
    console.log(blockScoped); // ReferenceError: blockScoped is not defined
}
```

### Shadowing — przysłanianie zmiennych

Wewnętrzny scope może "przysłonić" zewnętrzną zmienną o tej samej nazwie:

```javascript
let x = "global";

function outer() {
    let x = "outer"; // przysłania globalny x
    
    function inner() {
        let x = "inner"; // przysłania x z outer
        console.log(x); // "inner"
    }
    
    inner();
    console.log(x); // "outer"
}

outer();
console.log(x); // "global"
```

Każde `x` to osobna zmienna w osobnym Lexical Environment.

### Hoisting a Scope

`var` jest hoistowany do scope *funkcji* (nie bloku):

```javascript
function test() {
    console.log(a); // undefined — hoistowane, ale bez wartości
    
    if (true) {
        var a = 5; // deklaracja hoistowana do scope funkcji
    }
    
    console.log(a); // 5
}
```

`let`/`const` są hoistowane do scope *bloku*, ale z TDZ:

```javascript
{
    console.log(b); // ReferenceError — TDZ
    let b = 5;
    console.log(b); // 5
}
```

---

## 4. Co dzieje się wewnętrznie

### Lexical Environment jako implementacja Scope

Każdy Scope jest implementowany jako **Lexical Environment** (LE) — para:
- `EnvironmentRecord` — mapa identyfikatorów na wartości
- `outer` — referencja do zewnętrznego LE

Scope Chain to właśnie łańcuch takich referencji.

### Static vs Dynamic Scope

JavaScript używa **static scope** (leksykalny) — zakres jest określony przez strukturę kodu, nie przez sposób wywołania:

```javascript
let x = "global";

function showX() {
    console.log(x); // zawsze odczytuje x z miejsca DEFINICJI (global)
}

function changeContext() {
    let x = "local";
    showX(); // x = "global" — nie "local"!
    // W języku z dynamic scope byłoby "local"
}

changeContext(); // "global"
```

Gdyby JS używał dynamic scope (jak niektóre wersje Lispu czy Bash), `showX()` wypisałoby "local". Static scope jest bardziej przewidywalny i bezpieczniejszy.

### IIFE — Immediately Invoked Function Expression

Przed modułami ES6 IIFE był głównym sposobem na tworzenie prywatnego scope:

```javascript
(function() {
    // Prywatny scope — nic stąd nie wycieka do globalnego
    var privateVar = "nie widzę Cię z zewnątrz";
    
    // Publicznie ujawnione przez window
    window.myLib = {
        doSomething: function() { ... }
    };
})();

// privateVar niedostępna tutaj
```

### with Statement i dynamiczny scope

`with` tymczasowo dodaje obiekt do Scope Chain:

```javascript
const obj = { x: 10, y: 20 };

with (obj) {
    console.log(x + y); // 30 — x i y z obj przez Scope Chain
}
```

`with` jest zakazany w strict mode i deprecated. Psuje optymalizacje silnika (kompilator nie może statycznie określić scope) i tworzy zagrożenia bezpieczeństwa.

---

## 5. Analogiczny przykład z życia

Wyobraź sobie firmę z hierarchią pomieszczeń:

- **Hol wejściowy (Global Scope)** — widoczny dla wszystkich. Tablica z ogłoszeniami (globalne zmienne) dostępna każdemu.
- **Piętro 1 (Function Scope)** — pracownicy na tym piętrze widzą swoje dokumenty (lokalne zmienne) ORAZ ogłoszenia z holu.
- **Gabinet na piętrze 1 (Block Scope)** — tylko pracownicy w gabinecie widzą dokumenty w gabinecie. Dokumenty nie wychodzą poza drzwi.

**Scope Chain**: pracownik w gabinecie szukający dokumentu najpierw sprawdza gabinet, potem piętro, potem hol. Nigdy nie idzie do innego budynku (inny globalny scope).

**Shadowing**: jeśli i w gabinecie, i w holu jest tablica "PLAN DNIA", pracownik w gabinecie używa tablicy gabinetowej — nie widzi tablicy z holu (jest "przysłonięta").

---

## 6. Przykład kodu

```javascript
// === GLOBAL SCOPE ===
const API_URL = "https://api.example.com"; // stała globalna

function createUserSession(userId) {
    // === FUNCTION SCOPE createUserSession ===
    
    const sessionId = generateSessionId(); // lokalna dla tej funkcji
    let attempts = 0;                       // lokalna, mutable
    
    function authenticate(password) {
        // === FUNCTION SCOPE authenticate (zagnieżdżona) ===
        
        attempts++; // dostęp do 'attempts' z zewnętrznego scope — closure!
        
        if (attempts > 3) {
            // === BLOCK SCOPE tego if ===
            const lockMessage = "Konto zablokowane"; // tylko w tym bloku
            logSecurityEvent(lockMessage, userId); // userId z najwyższego FEC
            return false;
            // lockMessage przestaje istnieć po opuszczeniu bloku
        }
        
        // lockMessage niedostępne tutaj (block scope)
        return verifyPassword(password, sessionId); // sessionId z FEC createUserSession
    }
    
    return {
        login: authenticate,  // eksponujemy funkcję (closure)
        getSessionId: () => sessionId // closure na sessionId
    };
}

const session = createUserSession("user123");
// sessionId, attempts — niedostępne bezpośrednio
// session.login() — dostęp przez closure
```

**Wyjaśnienie:**

- `API_URL` — Global Scope → dostępny wszędzie
- `sessionId`, `attempts` — Function Scope `createUserSession` → dostępne w tej funkcji i zagnieżdżonych
- `lockMessage` — Block Scope `if {}` → tylko wewnątrz bloku
- `authenticate` przez closure "widzi" `attempts` i `sessionId` z zewnętrznego scope

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — ukrywanie danych przez Scope

```javascript
// Moduł autoryzacji — zakres modułu (Module Scope)
const AuthModule = (function() {
    // Private — niedostępne poza IIFE
    const SECRET_KEY = process.env.JWT_SECRET;
    let activeTokens = new Map();
    
    function validateToken(token) {
        // SECRET_KEY dostępny przez closure
        return jwt.verify(token, SECRET_KEY);
    }
    
    // Public API
    return {
        login(credentials) {
            const payload = authenticate(credentials);
            const token = jwt.sign(payload, SECRET_KEY);
            activeTokens.set(token, Date.now());
            return token;
        },
        verify: validateToken,
        logout(token) {
            activeTokens.delete(token);
        }
    };
})();

// SECRET_KEY i activeTokens niewidoczne z zewnątrz
// AuthModule.SECRET_KEY → undefined
// AuthModule.login() → działa
```

**Znaczenie dla pentestera:** 
- `SECRET_KEY` nie jest w `window.*` — nie można go odczytać z konsoli
- Ale jeśli aplikacja jest podatna na XSS — atakujący wykonuje kod *wewnątrz* modułu i może użyć closure

### Podatność — var w pętli

```javascript
// Legacy kod w systemie e-commerce
// Autoryzuje przeglądanie produktów tylko dla premium users
function loadProducts(products, isPremium) {
    var handlers = [];
    
    for (var i = 0; i < products.length; i++) {
        if (!isPremium && products[i].premiumOnly) {
            // Zamierzone: blokuj premium
            handlers.push(function() {
                console.log("Dostęp zabroniony");
            });
        } else {
            handlers.push(function() {
                // BUG: var i → ostatnia wartość pętli!
                // Wszystkie handlery pokażą produkty[products.length-1]
                showProduct(products[i]);
            });
        }
    }
    
    return handlers;
}
// Użycie let zamiast var naprawia problem
```

---

## 8. Typowe błędy programistów

### Błąd 1: Var w pętli (klasyczny)

```javascript
// Błąd
const buttons = document.querySelectorAll("button");
for (var i = 0; i < buttons.length; i++) {
    buttons[i].addEventListener("click", () => {
        console.log(i); // zawsze wypisze buttons.length, nie indeks
    });
}

// Poprawka 1: let
for (let i = 0; i < buttons.length; i++) { // każda iteracja ma własne i
    buttons[i].addEventListener("click", () => console.log(i));
}

// Poprawka 2: forEach (własny scope callbacka)
buttons.forEach((btn, i) => {
    btn.addEventListener("click", () => console.log(i));
});
```

### Błąd 2: Przypadkowe zmienne globalne

```javascript
function processData(input) {
    result = transform(input); // brak const/let/var → globalna!
    return result;
}

processData("test");
console.log(window.result); // widoczna globalnie!
```

### Błąd 3: Shadowing mylący logikę

```javascript
const user = { name: "Admin", role: "admin" };

function checkPermission(user) { // parametr przysłania globalnego user
    if (user.role === "admin") { // sprawdza PARAMETR, nie globalny obiekt
        grantAccess(); // może to zamierzone, może bug
    }
}

// Lepiej: jednoznaczne nazwy
function checkPermission(requestingUser) { ... }
```

### Błąd 4: With statement zaciemniający scope

```javascript
// Zakaz: with utrudnia analizę statyczną i jest zabroniony w strict mode
with (document) { // dodaje document do scope
    with (document.body) { // i document.body
        // Co to jest 'style'? document.style? document.body.style? lokalna?
        style.backgroundColor = "red";
    }
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Scope jako mechanizm izolacji

Scope to pierwsza linia obrony przed nieuprawnionym dostępem do danych. Prywatne zmienne w zamkniętym scope nie są dostępne bezpośrednio z zewnątrz.

Ale: XSS łamie tę izolację. Jeśli atakujący wstrzyknie kod wykonujący się w tym samym Execution Context, ma dostęp do closure'ów i zmiennych tego kontekstu.

### Global Scope Pollution

Zmienne w Global Scope (`window.*`) są dostępne z każdego miejsca, w tym:
- Z wstrzykniętego kodu (XSS)
- Z console DevTools przez użytkownika
- Z zewnętrznych skryptów na tej samej stronie

**Zasada minimalizacji Scope:** wrażliwe dane powinny być w możliwie najmniejszym scope.

### Variable Shadowing jako wektor

Rzadko spotykany, ale realny:

```javascript
// Biblioteka bezpieczeństwa definiuje globalnie:
window.isSecure = true;

// Strona atakuje przez DOM Clobbering (rozdział 10)
// lub przez Prototype Pollution (rozdział 7):
Object.prototype.isSecure = false;

// Kod sprawdzający:
function checkSecurity() {
    console.log(isSecure); // false — z prototype, nie z window!
}
```

### Powiązane CWE

- **CWE-1071** — Empty Catch Block (ukrywanie błędów scope)
- **CWE-1100** — Insufficient Isolation of System-Dependent Functions
- **CWE-1188** — Insecure Default Initialization of Resource

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Scope Panel

Podczas debugowania (breakpoint aktywny):

```
Panel Scope → sekcja Local:
    │  parametry funkcji
    │  zmienne let/const/var

Panel Scope → sekcja Closure:
    │  zmienne z zewnętrznych zakresów "złapane" przez closure

Panel Scope → sekcja Script:
    │  zmienne na poziomie pliku JS (module scope)

Panel Scope → sekcja Global:
    │  window.* — wszystkie globalne
```

### Szukanie globalnych wycieków

```javascript
// W konsoli DevTools — znajdź niestandardowe globalne zmienne
const nativeProps = new Set(Object.getOwnPropertyNames(window.__proto__.__proto__));
Object.keys(window).filter(k => !nativeProps.has(k));

// Lub prościej:
Object.keys(window).filter(k => typeof window[k] !== 'function' && 
                                  !k.startsWith('_') &&
                                  window[k] !== null);
```

### Szukanie var w pętlach (Source Review)

W DevTools → Ctrl+Shift+F → szukaj `for (var`:

```javascript
for (var i  // potencjalny closure bug
```

### Szukanie with statement

```javascript
with (  // deprecated, potencjalnie niebezpieczny
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy wrażliwe dane (tokeny, klucze) są w Global Scope (window.*)?
□ Czy zmienne globalne mogą być nadpisane z zewnątrz?
□ Czy with() jest używany (zakazany w strict mode, podatny)?
□ Czy var w pętlach tworzy niezamierzone closures (logika security)?
□ Czy moduły ES6 lub IIFE izolują prywatne dane?
□ Czy strict mode ('use strict') jest włączony — zapobiegając przypadkowym globalom?
□ Czy shadowing zmiennych nie zaciemnia logiki autoryzacji?
□ Czy po XSS atakujący może odczytać dane z Scope przez closure?
□ Czy DevTools ujawnia wrażliwe dane w panelu Scope?
□ Czy source maps ujawniają prywatne zmienne i ich nazwy?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: XSS → dostęp do zamkniętych danych przez Scope

**Wymagania:** XSS w kontekście strony, która przechowuje wrażliwe dane w zamkniętym scope.

**Przebieg:**
```javascript
// Strona ofiara:
(function() {
    const authToken = "secret_jwt_token_here";
    
    document.getElementById("login-btn").addEventListener("click", () => {
        fetch("/api/login", {
            headers: { Authorization: authToken } // używa authToken przez closure
        });
    });
})();

// Payload XSS wstrzyknięty w komentarzu lub innym wektorze:
// Wykonuje się w GLOBAL scope — NIE ma dostępu do authToken!
// Ale może przechwycić przez monkey-patching fetch:
const originalFetch = fetch;
window.fetch = function(url, options) {
    // Przechwytuje wywołania fetch — widzi nagłówki!
    exfiltrate(options?.headers?.Authorization);
    return originalFetch.apply(this, arguments);
};
```

**Ograniczenia:** CSP blokuje external fetch. Wymaga XSS w tym samym origin.

### Scenariusz 2: Nadpisanie globalnej zmiennej konfiguracyjnej

**Wymagania:** Aplikacja przechowuje konfigurację bezpieczeństwa w Global Scope.

**Przebieg:**
1. Pentester identyfikuje: `window.SECURITY_CONFIG = { requireMFA: true }`
2. XSS lub DOM Clobbering nadpisuje: `window.SECURITY_CONFIG = { requireMFA: false }`
3. Logika aplikacji sprawdza `SECURITY_CONFIG.requireMFA` → `false` → pomija MFA

**Wpływ:** Obejście mechanizmów bezpieczeństwa.

---

## 13. Jak się zabezpieczać

### Minimalizuj Global Scope

```javascript
// Zamiast zmiennych globalnych — użyj modułów
// config.mjs
export const API_URL = "https://api.example.com";
export const TIMEOUT = 5000;

// app.mjs
import { API_URL } from './config.mjs';
// API_URL nie jest w window.API_URL
```

### Strict Mode eliminuje przypadkowe globale

```javascript
"use strict";

function processData(input) {
    result = transform(input); // ReferenceError w strict mode — nie cicha globalna!
}
```

### Preferuj const > let > var

```javascript
// const — block scope, nie może być reassigned
const MAX_RETRIES = 3;

// let — block scope, może być reassigned
let retryCount = 0;

// var — UNIKAJ poza legacy code (function scope, hoistowany, podatny na bugi)
```

### Izoluj moduły przez IIFE lub ES modules

```javascript
// ES6 module (zalecane)
// Plik: auth.mjs
const privateKey = "..."; // nie w window

export function verify(token) {
    return jwt.verify(token, privateKey); // closure na privateKey
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Scope = reguły widoczności zmiennych, statycznie określone przez strukturę kodu
2. JavaScript ma 4 rodzaje scope: Global, Function, Block (ES6), Module (ES6)
3. Scope Chain = łańcuch Lexical Environments przeszukiwany od wewnętrznego do globalnego
4. `var` → Function Scope (hoistowany); `let`/`const` → Block Scope (TDZ)
5. Shadowing = wewnętrzna zmienna przysłania zewnętrzną o tej samej nazwie

**Najczęstsze nieporozumienia:**

- "Scope to to samo co Execution Context" — EC to szerszy koncept zawierający m.in. Scope (Lexical Environment)
- "Closure tworzy kopię zmiennej" — NIE. Closure przechowuje *referencję* do Lexical Environment — zmiany zmiennej są widoczne przez closure
- "var jest block-scoped w ES6" — NIE. var zawsze ma function scope. ES6 dodał let/const z block scope

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Sources → breakpoint → panel Scope** — widzisz wszystkie zakresy
2. **Console** → `Object.keys(window)` — szukaj niestandardowych globalnych zmiennych
3. **Sources → Ctrl+Shift+F** → szukaj `var ` w pętlach, `with (`, przypadkowych globali (brak deklaracji)
4. **Application → Local Storage** — czy klucze sesji są w storage zamiast zamkniętego scope?
5. Szukaj w kodzie: zmiennych dostępnych jako `window.x` — czy zawierają wrażliwe dane?

---

## Powiązania

```
Scope
    │
    ├──► Execution Context (Rozdział 2)
    │         Scope implementowany przez Lexical Environment wewnątrz EC
    │
    ├──► Closures (Rozdział 5)
    │         Closure = funkcja zachowująca referencję do zewnętrznego Scope
    │
    ├──► Prototype Chain (Rozdział 6)
    │         Szukanie zmiennych w Scope Chain vs właściwości w Prototype Chain
    │
    └──► DOM Clobbering (Rozdział 10)
                DOM Clobbering może przysłonić globalne zmienne przez window.*
```
