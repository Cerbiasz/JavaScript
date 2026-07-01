# Rozdział 6: Prototype Chain

## 1. Czym jest Prototype Chain

**Prototype Chain** (łańcuch prototypów) to mechanizm dziedziczenia w JavaScript. Każdy obiekt ma wewnętrzną właściwość `[[Prototype]]` wskazującą na inny obiekt (lub `null`). Gdy silnik szuka właściwości na obiekcie i jej nie znajdzie, przechodzi do `[[Prototype]]` — i tak dalej, aż do `null`. To właśnie jest łańcuch prototypów.

JavaScript używa **prototypowego** modelu dziedziczenia, w odróżnieniu od klasowego modelu Javy czy C#. Nie ma "klas" w sensie bytów oddzielonych od obiektów (choć słowo kluczowe `class` z ES6 jest lukrem syntaktycznym na ten sam mechanizm).

Prototype Chain jest częścią specyfikacji ECMAScript. Działa we wszystkich przeglądarkach i środowiskach JS.

### Dostęp do Prototype Chain

- `Object.getPrototypeOf(obj)` — zwraca `[[Prototype]]` obiektu (zalecane)
- `obj.__proto__` — accessor (getter/setter) do `[[Prototype]]` — dostępny przez `Object.prototype`; historycznie używany, ale `getPrototypeOf` jest zalecane
- `Constructor.prototype` — prototyp obiektów tworzonych przez dany konstruktor
- `Object.prototype` — sam szczyt łańcucha (poza `null`)

---

## 2. Dlaczego powstał

### Problem: dziedziczenie bez klas

Brendan Eich projektując JavaScript miał na celu uproszczony model obiektowy. Zamiast klas z hierarchią dziedziczenia, wybrał model prototypowy — obiekty dziedziczą bezpośrednio z innych obiektów, bez pośrednika w postaci klasy.

### Problem: współdzielenie metod

Bez prototypów każda instancja musiałaby mieć własną kopię wszystkich metod:

```javascript
// Bez prototypów — marnowanie pamięci
function User(name) {
    this.name = name;
    this.greet = function() {  // kopia funkcji dla każdego User!
        return "Hello, " + this.name;
    };
}

const u1 = new User("Alice"); // ma własną kopię greet
const u2 = new User("Bob");   // ma własną kopię greet — duplikat!
```

Prototype Chain rozwiązuje to przez współdzielenie metod:

```javascript
// Z prototypami — jedna funkcja greet dla wszystkich instancji
function User(name) {
    this.name = name; // własne dane
}

User.prototype.greet = function() { // współdzielona metoda
    return "Hello, " + this.name;
};

const u1 = new User("Alice"); // nie ma własnej kopii greet
const u2 = new User("Bob");   // korzysta z tej samej User.prototype.greet
```

---

## 3. Jak działa

### Lookup właściwości krok po kroku

```
obj.property → ?

1. Sprawdź obj → znalazł? STOP, użyj
                nie znalazł? →

2. Sprawdź obj.__proto__ (Object.getPrototypeOf(obj)) → znalazł? STOP
                nie znalazł? →

3. Sprawdź obj.__proto__.__proto__ → znalazł? STOP
                nie znalazł? →

4. ... kontynuuj łańcuch ...

5. Sprawdź Object.prototype → znalazł? STOP
                nie znalazł? →

6. null → nie znalazł → zwróć undefined
```

### Diagram łańcucha

```
┌─────────────────────────────────────────────────┐
│                    null                          │
│                     ▲                            │
│                     │ [[Prototype]]              │
│          ┌──────────────────────┐                │
│          │    Object.prototype  │                │
│          │  toString()          │                │
│          │  hasOwnProperty()    │                │
│          │  valueOf()           │                │
│          │  ...                 │                │
│          └──────────────────────┘                │
│                     ▲                            │
│                     │ [[Prototype]]              │
│          ┌──────────────────────┐                │
│          │    User.prototype    │                │
│          │  greet()             │                │
│          │  constructor → User  │                │
│          └──────────────────────┘                │
│                     ▲                            │
│                     │ [[Prototype]]              │
│          ┌──────────────────────┐                │
│          │  { name: "Alice" }   │                │
│          │  (instancja User)    │                │
│          └──────────────────────┘                │
└─────────────────────────────────────────────────┘

alice.greet()
  → szuka 'greet' w alice → nie ma
  → szuka w User.prototype → JEST → wywołuje z this = alice
  → zwraca "Hello, Alice"

alice.toString()
  → szuka w alice → nie ma
  → szuka w User.prototype → nie ma
  → szuka w Object.prototype → JEST → wywołuje
```

### Operator new — jak tworzy obiekt z prototypem

```javascript
function User(name) {
    this.name = name;
}

const alice = new User("Alice");
```

`new User("Alice")` wykonuje cztery operacje:

1. Tworzy nowy pusty obiekt: `{}`
2. Ustawia `[[Prototype]]` nowego obiektu na `User.prototype`
3. Wywołuje `User` z `this` = nowy obiekt
4. Zwraca nowy obiekt (chyba że konstruktor zwraca inny obiekt)

### class jako lukier syntaktyczny

```javascript
// ES6 class
class Animal {
    constructor(name) {
        this.name = name;
    }
    
    speak() {
        return `${this.name} makes a sound.`;
    }
}

class Dog extends Animal {
    speak() {
        return `${this.name} barks.`;
    }
}

// Identyczne z:
function Animal(name) {
    this.name = name;
}
Animal.prototype.speak = function() {
    return this.name + " makes a sound.";
};

function Dog(name) {
    Animal.call(this, name);
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.speak = function() {
    return this.name + " barks.";
};
```

Class syntax to tylko czytelniejszy zapis — pod spodem to ten sam Prototype Chain.

### hasOwnProperty vs in operator

```javascript
const obj = { own: "własna" };

"own" in obj;              // true — szuka w całym łańcuchu
"toString" in obj;         // true — znaleziona w Object.prototype

obj.hasOwnProperty("own");       // true — tylko własna
obj.hasOwnProperty("toString");  // false — nie własna, z prototype
```

---

## 4. Co dzieje się wewnętrznie

### [[Prototype]] — wewnętrzny slot

Każdy obiekt JavaScript ma wewnętrzny slot `[[Prototype]]`. Nie jest to normalna właściwość (nie pojawi się w `for...in` jak normalna właściwość). Dostęp przez:

- `Object.getPrototypeOf(obj)` — czyta `[[Prototype]]`
- `Object.setPrototypeOf(obj, proto)` — ustawia `[[Prototype]]` (wolna operacja!)
- `Object.create(proto)` — tworzy obiekt z podanym `[[Prototype]]`

### Performance implikacje

Modyfikacja `[[Prototype]]` po tworzeniu obiektu jest bardzo kosztowna — silnik V8 musi deoptymalizować wszystkie inline caches (IC) dla tego obiektu i powiązanych. To jest powód, dla którego `Object.setPrototypeOf()` jest odradzane.

### Property Descriptor

Właściwości na prototypie mogą mieć deskryptory:

```javascript
Object.defineProperty(User.prototype, "role", {
    value: "user",
    writable: false,    // tylko do odczytu
    enumerable: false,  // nie pojawia się w for...in
    configurable: false // nie można zmienić deskryptora
});
```

Próba nadpisania własności `writable: false` przez przypisanie (nie przez defineProperty) w strict mode rzuci TypeError.

---

## 5. Analogiczny przykład z życia

Wyobraź sobie firmę z procedurami:

- **Pracownik (instancja)** ma własne dane: imię, stanowisko, wynagrodzenie
- **Dział HR (prototype)** ma procedury: jak złożyć urlop, jak zgłosić chorobę
- **Cała firma (Object.prototype)** ma ogólne zasady: godziny pracy, kodeks etyczny

Gdy pracownik potrzebuje procedury urlopowej:
1. Sprawdza własne notatki → nie ma
2. Idzie do HR (prototype) → ma! Używa procedury HR
3. Procedura HR jest *jedna* — każdy pracownik korzysta z tej samej

Gdyby pracownik potrzebował zmiany nazwy firmy:
1. Sprawdza własne notatki → nie ma
2. Idzie do HR → nie ma
3. Idzie do zarządu (Object.prototype) → nie ma
4. Nie istnieje → undefined

**Prototype Pollution** = atakujący podmienia procedury w HR lub zarządzie — nagle WSZYSCY pracownicy korzystają z podmienionej procedury.

---

## 6. Przykład kodu

```javascript
// Konstruktor — definiuje własne właściwości
function Vehicle(make, model, year) {
    this.make = make;       // własna właściwość instancji
    this.model = model;     // własna właściwość instancji
    this.year = year;       // własna właściwość instancji
}

// Metody na prototype — współdzielone przez wszystkie instancje
Vehicle.prototype.getAge = function() {
    return new Date().getFullYear() - this.year; // this = bieżąca instancja
};

Vehicle.prototype.describe = function() {
    return `${this.year} ${this.make} ${this.model}`;
};

// Tworzenie instancji
const car = new Vehicle("Toyota", "Camry", 2020);

// Dostęp do własnych właściwości
console.log(car.make);    // "Toyota" — własna
console.log(car.model);   // "Camry" — własna

// Dostęp przez Prototype Chain
console.log(car.getAge()); // 4 (2024-2020) — z Vehicle.prototype

// Sprawdzenie własności
console.log(car.hasOwnProperty("make"));   // true — własna
console.log(car.hasOwnProperty("getAge")); // false — z prototype

// Łańcuch prototypów
console.log(Object.getPrototypeOf(car) === Vehicle.prototype); // true
console.log(Object.getPrototypeOf(Vehicle.prototype) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype)); // null

// instanceof
console.log(car instanceof Vehicle); // true — sprawdza łańcuch
console.log(car instanceof Object);  // true — Object.prototype jest w łańcuchu
```

---

## 7. Przykład z prawdziwej aplikacji

### Express.js — middleware chain jako analogia

```javascript
// Express nie używa bezpośrednio prototype chain dla middleware,
// ale Route objects korzystają z prototypu
const express = require('express');
const app = express();

// Każdy Request object ma prototyp z metodami jak req.param(), req.is()
app.use((req, res, next) => {
    // req jest instancją http.IncomingMessage z właściwościami z prototype
    console.log(Object.getPrototypeOf(req).constructor.name); // IncomingMessage
});
```

### Lodash deep merge — podatność Prototype Pollution

```javascript
// Wewnętrzna implementacja merge w podatnych wersjach lodash:
function merge(destination, source) {
    for (let key in source) {
        if (source[key] && typeof source[key] === 'object') {
            destination[key] = destination[key] || {};
            merge(destination[key], source[key]); // rekurencja
        } else {
            destination[key] = source[key]; // przypisanie
        }
    }
    return destination;
}

// Payload atakującego (JSON z serwera lub user input):
const maliciousPayload = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, maliciousPayload);

// Skutek: Object.prototype.isAdmin = true
// Teraz KAŻDY obiekt ma .isAdmin = true!
const obj = {};
console.log(obj.isAdmin); // true — przez prototype chain!
```

### Vue/React — klasy komponentów

```javascript
// React.Component używa prototype chain
class MyComponent extends React.Component {
    // Dziedziczy: setState, forceUpdate, render, componentDidMount...
    // z React.Component.prototype przez łańcuch:
    // MyComponent.prototype → React.Component.prototype → Object.prototype
    
    render() {
        return <div>{this.props.name}</div>;
    }
}

// Pentester szuka: czy prototype.render jest nadpisywalne?
// Czy prototype.setState może być podmienione przez Prototype Pollution?
```

---

## 8. Typowe błędy programistów

### Błąd 1: Modyfikowanie wbudowanych prototypów

```javascript
// Bardzo złe — modyfikacja Array.prototype wpływa na CAŁY kod
Array.prototype.last = function() {
    return this[this.length - 1];
};

// Konflikty z innymi bibliotekami
// Podatność Prototype Pollution jeśli dane użytkownika trafiają do prototypu
```

### Błąd 2: for...in bez hasOwnProperty

```javascript
const obj = { a: 1, b: 2 };

// Błąd: for...in iteruje przez CAŁY łańcuch prototypów
for (let key in obj) {
    console.log(key); // może wypisać klucze z Object.prototype!
}

// Poprawka:
for (let key in obj) {
    if (obj.hasOwnProperty(key)) { // tylko własne właściwości
        console.log(key);
    }
}

// Lub lepiej:
Object.keys(obj).forEach(key => console.log(key)); // tylko własne
```

### Błąd 3: instanceof po cross-frame

```javascript
// W iframes lub VM contexts:
// Array z innego frame ma inny Array.prototype
const iframe = document.createElement("iframe");
document.body.appendChild(iframe);
const IframeArray = iframe.contentWindow.Array;

const arr = new IframeArray(1, 2, 3);
console.log(arr instanceof Array); // false! — inne Array.prototype
console.log(Array.isArray(arr));   // true — bezpieczniejszy sposób sprawdzania
```

### Błąd 4: setPrototypeOf w performance-critical code

```javascript
// Bardzo wolne! — deoptymalizuje V8 inline caches
const obj = {};
Object.setPrototypeOf(obj, customProto); // unikaj!

// Szybciej: twórz obiekty z właściwym prototypem od razu
const obj2 = Object.create(customProto); // wydajniejsze
```

---

## 9. Znaczenie dla bezpieczeństwa

### Prototype Pollution (szczegóły w rozdziale 7)

Podstawowe zagrożenie wynikające z Prototype Chain. Jeśli atakujący może modyfikować `Object.prototype` lub `Array.prototype`, wpływa na każdy obiekt w aplikacji.

### Security by Inheritance

Wiele frameworków używa Prototype Chain do enkapsulacji zabezpieczeń:

```javascript
// Framework może ukryć metodę sprawdzania uprawnień:
Object.defineProperty(User.prototype, '_checkPerm', {
    value: function(perm) { ... },
    enumerable: false,  // niewidoczna w for...in
    writable: false     // nie można nadpisać przez przypisanie
});
```

Ale: `Object.defineProperty` z `configurable: true` nadal można nadpisać przez `defineProperty` ponownie.

### Prototype Chain jako wektor XSS

```javascript
// Jeśli atakujący osiągnie XSS, może modyfikować prototypy:
Object.prototype.toString = function() {
    // Podmieniono wbudowaną metodę toString!
    exfiltrate(this); // wysyła każdy obiekt który wywołuje toString
    return "[object Object]"; // normalna odpowiedź
};
// Teraz każde wywołanie `String(obj)`, `${obj}`, `console.log(obj)` eksfiltruje dane
```

### Property Lookup Security Issues

```javascript
// Problem: propertiesy dziedziczone z prototypu mogą kolidować z logiką
const userInput = "__proto__"; // lub "constructor", "toString", etc.

const config = {};
config[userInput] = "modified"; // config.__proto__ = "modified"
// Nie ustawiło właściwości config! Zmodyfikowało Object.prototype!

// Bezpieczna alternatywa:
const safeConfig = Object.create(null); // null prototype — brak wbudowanych
safeConfig[userInput] = "modified"; // teraz bezpieczne
```

### hasOwnProperty jako security check

```javascript
// Sprawdzanie czy klucz naprawdę należy do obiektu
function allowlisted(config, key) {
    return config.hasOwnProperty(key); // nie sprawdza prototypu
}
// Ale! Co jeśli config ma __proto__ z hasOwnProperty = false?
// Bezpieczniej:
function allowlisted(config, key) {
    return Object.prototype.hasOwnProperty.call(config, key);
}
```

### Powiązane CWE

- **CWE-1321** — Improperly Controlled Modification of Object Prototype Attributes
- **CWE-915** — Improperly Controlled Modification of Dynamically-Determined Object Attributes

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Console — eksploracja Prototype Chain

```javascript
// Sprawdź łańcuch prototypów dowolnego obiektu
let obj = new SomeClass();
let proto = obj;
while (proto = Object.getPrototypeOf(proto)) {
    console.log(proto.constructor?.name, Object.getOwnPropertyNames(proto));
}

// Szukaj wrażliwych metod
Object.getOwnPropertyNames(Object.prototype); // wbudowane — benchmark
// Jeśli widać metody których nie powinno być → Prototype Pollution!
```

### Sprawdzanie Object.prototype

```javascript
// Sprawdź czy Object.prototype jest "czysty"
const clean = ["hasOwnProperty", "isPrototypeOf", "propertyIsEnumerable",
               "toString", "toLocaleString", "valueOf", "__defineGetter__",
               "__defineSetter__", "__lookupGetter__", "__lookupSetter__"];

const current = Object.getOwnPropertyNames(Object.prototype);
const extra = current.filter(k => !clean.includes(k));
console.log("Dodatkowe właściwości na Object.prototype:", extra);
// Jeśli niepusta → możliwa Prototype Pollution
```

### Szukanie podatnych merge/assign w kodzie

W DevTools → Sources → Ctrl+Shift+F:
```
Object.assign(
merge(
extend(
deepCopy(
JSON.parse
__proto__
constructor[
["constructor"]
```

### Network → Payload Analysis

W Burp Suite → szukaj w requestach JSON:
```json
{"__proto__": {"isAdmin": true}}
{"constructor": {"prototype": {"isAdmin": true}}}
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja wykonuje deep merge/assign obiektów z danymi użytkownika?
□ Czy Object.prototype ma dodatkowe właściwości (wskazuje na pollution)?
□ Czy for...in pętle są chronione hasOwnProperty?
□ Czy używany jest Object.create(null) dla słowników/map (bez prototype)?
□ Czy wbudowane prototypy (Array.prototype, String.prototype) są modyfikowane?
□ Czy biblioteki (lodash, jQuery, merge) są aktualne (podatności Prototype Pollution)?
□ Czy dane JSON z użytkownika trafiają do funkcji merge bez sanityzacji?
□ Czy instanceof jest używane cross-frame (może dać false negatives)?
□ Czy Object.prototype.hasOwnProperty może być nadpisane (payload: {hasOwnProperty: ...})?
□ Czy wrażliwe metody na prototypie są niemodyfikowalne (writable: false)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Prototype Pollution przez JSON merge

**Wymagania:** Aplikacja akceptuje JSON i merguje go z obiektem bez sanityzacji kluczy.

**Payload:**
```json
POST /api/user/settings
Content-Type: application/json

{
    "theme": "dark",
    "__proto__": {
        "isAdmin": true
    }
}
```

**Przebieg:**
1. Serwer odbiera JSON, parsuje przez `JSON.parse()` — `__proto__` staje się normalnym kluczem obiektu
2. Funkcja merge iteruje klucze i przypisuje: `target.__proto__.isAdmin = true`
3. Przez Prototype Chain `Object.prototype.isAdmin = true`
4. Każdy sprawdzający `obj.isAdmin` otrzymuje `true`

**Weryfikacja:**
```javascript
// Sprawdź po wysłaniu payloadu:
const test = {};
console.log(test.isAdmin); // true → podatność potwierdzona
```

**Wpływ:** Eskalacja uprawnień, ominięcie autoryzacji, RCE (w Node.js z template engines).

### Scenariusz 2: Podmiana constructor.prototype

**Payload alternatywny:**
```json
{
    "constructor": {
        "prototype": {
            "isAdmin": true
        }
    }
}
```

Niektóre implementacje merge śledzą `constructor.prototype` zamiast `__proto__` — ten payload obchodzi filtry sprawdzające tylko `__proto__`.

---

## 13. Jak się zabezpieczać

### Filtrowanie niebezpiecznych kluczy

```javascript
function safeMerge(target, source) {
    const FORBIDDEN = new Set(["__proto__", "constructor", "prototype"]);
    
    for (const key of Object.keys(source)) {
        if (FORBIDDEN.has(key)) continue; // pomiń niebezpieczne klucze
        
        if (typeof source[key] === 'object' && source[key] !== null) {
            target[key] = target[key] || {};
            safeMerge(target[key], source[key]);
        } else {
            target[key] = source[key];
        }
    }
    return target;
}
```

### Object.create(null) dla słowników

```javascript
// Słownik bez prototype — nie można zanieczyszczać przez __proto__
const userPermissions = Object.create(null);
userPermissions["read"] = true;
userPermissions["write"] = false;

// Nie ma Object.prototype → nie ma __proto__, toString, constructor etc.
// Wstrzyknięcie "__proto__" to teraz po prostu normalny klucz, nie atak
```

### Zamrożenie Object.prototype

```javascript
// Zabezpieczenie: zamróź Object.prototype
// UWAGA: może zepsuć biblioteki które modyfikują prototypy!
Object.freeze(Object.prototype);

// Teraz próba modyfikacji rzuci TypeError (strict mode) lub po cichu nie zadziała
```

### Walidacja schema JSON (np. Ajv)

```javascript
const Ajv = require("ajv");
const ajv = new Ajv();

const schema = {
    type: "object",
    properties: {
        theme: { type: "string" },
        language: { type: "string" }
    },
    additionalProperties: false // BLOKUJE __proto__, constructor itp.
};

const validate = ajv.compile(schema);
const valid = validate(userInput);
// Payload z __proto__ zostanie odrzucony przez additionalProperties: false
```

### Aktualizuj biblioteki

```bash
# Sprawdź znane podatności Prototype Pollution
npm audit
npm audit fix

# Szukaj konkretnych CVE:
# CVE-2019-10744 — lodash < 4.17.12 (merge)
# CVE-2020-8203 — lodash < 4.17.19 (zipObjectDeep)
# CVE-2019-0393 — jQuery < 3.4.0 (extend)
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Prototype Chain = mechanizm dziedziczenia w JS — każdy obiekt ma `[[Prototype]]` wskazujący na inne obiekty
2. Gdy właściwość nie zostanie znaleziona na obiekcie, silnik szuka w górę łańcucha do `null`
3. `class` w ES6 to lukier syntaktyczny na Prototype Chain — pod spodem ten sam mechanizm
4. Modyfikacja `Object.prototype` wpływa na KAŻDY obiekt w programie
5. `Object.create(null)` tworzy obiekt bez prototype — bezpieczne dla słowników

**Najczęstsze nieporozumienia:**

- "class w JavaScript to klasy jak w Javie" — NIE. To Prototype Chain z ładniejszą składnią.
- "`__proto__` to właściwość obiektu" — NIE. To accessor na `Object.prototype` — korzysta z `getPrototypeOf`/`setPrototypeOf`.
- "Prototype Pollution to problem tylko Node.js" — NIE. Dotyka też front-end JS (XSS, eskalacja uprawnień w kliencie).

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Console** → `Object.getPrototypeOf(dowolnyObiekt)` → eksploruj łańcuch
2. **Console** → `Object.getOwnPropertyNames(Object.prototype)` → sprawdź czy są dodatkowe właściwości (wskaźnik pollution)
3. **Sources** → szukaj: `merge(`, `assign(`, `extend(`, `deepCopy(` → czy przetwarzają dane użytkownika?
4. **Burp Suite** → Repeater → wyślij payload `{"__proto__": {"x": 1}}` → sprawdź w konsoli `({}).x === 1`

---

## Powiązania

```
Prototype Chain
        │
        ├──► Prototype Pollution (Rozdział 7)
        │         Bezpośrednia podatność wynikająca z mechanizmu Prototype Chain
        │
        ├──► Execution Context (Rozdział 2)
        │         this binding w EC determinuje który obiekt ma swój Prototype Chain
        │
        ├──► DOM (Rozdział 9)
        │         DOM elements mają własne Prototype Chains (HTMLElement → Element → Node)
        │
        └──► DOM Clobbering (Rozdział 10)
                  DOM Clobbering może zakłócać dostęp do właściwości przez łańcuch
```
