# Rozdział 31: Shadow DOM

## 1. Czym jest Shadow DOM

**Shadow DOM** to mechanizm enkapsulacji wbudowany w DOM, umożliwiający tworzenie izolowanego poddrzewa DOM (shadow tree) dołączonego do elementu. Shadow DOM pozwala komponentom webowym posiadać własny, enkapsulowany DOM, CSS i logikę JavaScript — niewidoczną i niezakłócającą resztę strony.

Kluczowe koncepty:
- **Shadow Host** — element DOM do którego dołączony jest Shadow DOM
- **Shadow Root** — korzeń shadow tree (utworzony przez `attachShadow()`)
- **Shadow Tree** — poddrzewo DOM wewnątrz Shadow Root
- **Light DOM** — normalny DOM (poza shadow tree)
- **Slots** — punkty wstrzyknięcia treści z light DOM do shadow tree

Shadow DOM ma dwa tryby:
- **`open`** — Shadow Root dostępny przez `element.shadowRoot`
- **`closed`** — Shadow Root ukryty (brak dostępu przez `element.shadowRoot`)

---

## 2. Dlaczego powstał

### Problem: brak enkapsulacji w CSS i DOM

Przed Shadow DOM:
- CSS globalny — style z zewnątrz wpływały na wnętrze komponentów (i vice versa)
- ID musiały być unikalne globalnie
- Biblioteki CSS (Bootstrap, Material) zaśmiecały globalny namespace
- Tworzenie reużywalnych komponentów wymagało skomplikowanych obejść (BEM, CSS Modules)

Shadow DOM (część Web Components spec, 2011) dostarcza natywną enkapsulację:
- Shadow tree jest izolowane od globalnego CSS
- Selektory zewnętrzne nie wnikają do Shadow DOM (domyślnie)
- ID wewnątrz Shadow DOM nie kolidują z globalnym

---

## 3. Jak działa

### Tworzenie Shadow DOM

```javascript
// Utwórz Shadow Root
const host = document.getElementById('my-component');
const shadow = host.attachShadow({ mode: 'open' });

// mode: 'open' — shadow.shadowRoot dostępny przez: host.shadowRoot
// mode: 'closed' — host.shadowRoot === null (ukryty)

// Dodaj treść do Shadow DOM
shadow.innerHTML = `
    <style>
        /* Ten CSS dotyczy TYLKO shadow tree */
        :host { display: block; border: 1px solid #ccc; }
        p { color: blue; }
    </style>
    <p>Jestem w Shadow DOM!</p>
    <slot></slot>  <!-- treść z Light DOM wchodzi tutaj -->
`;
```

### Izolacja CSS

```html
<!-- Light DOM (główny dokument) -->
<style>
    p { color: red; }  <!-- To NIE wpłynie na p wewnątrz Shadow DOM -->
</style>

<my-component>
    <span>Treść z Light DOM (przejdzie przez <slot>)</span>
</my-component>

<p>Ten paragraph jest czerwony (globalny CSS)</p>
<!-- p wewnątrz shadow — niebieski (shadow CSS) -->
```

### Slots — projekcja Light DOM

```javascript
// Shadow DOM z named slots
shadow.innerHTML = `
    <header><slot name="header">Domyślny nagłówek</slot></header>
    <main><slot></slot></main>
    <footer><slot name="footer"></slot></footer>
`;

// Light DOM:
// <my-component>
//   <h1 slot="header">Mój nagłówek</h1>    <!-- wypełnia named slot 'header' -->
//   <p>Treść główna</p>                     <!-- wypełnia domyślny slot -->
//   <span slot="footer">Stopka</span>       <!-- wypełnia named slot 'footer' -->
// </my-component>
```

### Mode: open vs closed

```javascript
// Open mode:
const shadow = host.attachShadow({ mode: 'open' });
host.shadowRoot; // → shadow root (dostępny!)
shadow === host.shadowRoot; // true

// Closed mode:
const shadow = host.attachShadow({ mode: 'closed' });
host.shadowRoot; // → null
// Możliwy tylko jeśli zachowasz referencję:
// const shadowRef = host.attachShadow({ mode: 'closed' });
// shadowRef dostępny tylko przez lokalną zmienną
```

---

## 4. Co dzieje się wewnętrznie

### Rendering pipeline

Przeglądarka podczas renderowania:
1. Buduje Light DOM tree
2. Dla elementów z Shadow Root — renderuje shadow tree zamiast normalnych children
3. Sloty są "wirtualne" — Light DOM elementy są wizualnie renderowane w miejscu slotu, ale pozostają w Light DOM

### Shadow DOM boundaries

Kilka mechanizmów które NIE są izolowane przez Shadow DOM:
- **JavaScript** — `document.querySelectorAll` nie wnika w zamknięte Shadow DOM, ale `element.shadowRoot.querySelectorAll` (open) tak
- **`event.composedPath()`** — pokazuje pełną ścieżkę zdarzenia przez Shadow DOM
- **focus** — focus może przejść przez Shadow DOM boundary
- **Fonts** — fonty z light DOM są dziedziczone do shadow DOM
- **CSS Custom Properties (variables)** — przechodzą przez Shadow DOM boundary!

### Event Retargeting

Zdarzenia z Shadow DOM są "retargetowane" — listener na Shadow Host widzi zdarzenie jako pochodzące z hosta, nie z wewnętrznego elementu:

```javascript
host.addEventListener('click', (e) => {
    console.log(e.target); // host (nie wewnętrzny element)!
    console.log(e.composedPath()); // pełna ścieżka przez Shadow DOM
});
```

---

## 5. Analogiczny przykład z życia

Shadow DOM to szklana witryna sklepowa:

- Sklep (Shadow DOM) ma własną dekorację wnętrza (CSS)
- Przechodnie (globalne CSS/JS) widzą sklep z zewnątrz ale nie zmieniają wystroju wnętrza
- Niektóre rzeczy (fonty, CSS variables) "przenikają przez szybę"
- Witryna ma otwory (sloty) przez które można umieszczać produkty (Light DOM content)
- W trybie open — klucz do sklepu (shadowRoot reference) jest publiczny
- W trybie closed — tylko właściciel ma klucz

---

## 6. Przykład kodu

```javascript
// === Custom Element z Shadow DOM ===
class SecurePasswordInput extends HTMLElement {
    constructor() {
        super();
        this._shadow = this.attachShadow({ mode: 'closed' }); // tryb closed!
        
        this._shadow.innerHTML = `
            <style>
                :host {
                    display: inline-block;
                    position: relative;
                }
                input {
                    padding: 8px;
                    border: 2px solid #ddd;
                    border-radius: 4px;
                    font-size: 16px;
                    width: 100%;
                    box-sizing: border-box;
                }
                input:focus {
                    border-color: #007bff;
                    outline: none;
                }
                button {
                    position: absolute;
                    right: 8px;
                    top: 50%;
                    transform: translateY(-50%);
                    background: none;
                    border: none;
                    cursor: pointer;
                    font-size: 16px;
                }
            </style>
            <input type="password" placeholder="Hasło">
            <button type="button">👁</button>
        `;
        
        this._input = this._shadow.querySelector('input');
        this._toggle = this._shadow.querySelector('button');
        
        this._toggle.addEventListener('click', () => {
            this._input.type = this._input.type === 'password' ? 'text' : 'password';
        });
    }
    
    get value() {
        return this._input.value;
    }
    
    // WAŻNE: nie ekspozuj shadow poprzez publiczne API!
}

customElements.define('secure-password', SecurePasswordInput);
```

---

## 7. Przykład z prawdziwej aplikacji

### Omijanie enkapsulacji Shadow DOM w closed mode

`mode: 'closed'` NIE jest pełnym zabezpieczeniem:

```javascript
// ATAK: monkey-patch attachShadow przed jego wywołaniem
const originalAttachShadow = Element.prototype.attachShadow;
Element.prototype.attachShadow = function(options) {
    const shadow = originalAttachShadow.call(this, { ...options, mode: 'open' });
    // Teraz KAŻDY Shadow DOM jest open, niezależnie od żądanego mode!
    this.openShadow = shadow; // zachowaj referencję
    return shadow;
};

// Po tym monkey-patchu:
host.attachShadow({ mode: 'closed' }); // ale mode zmieniony na 'open'!
host.openShadow; // dostęp do shadow tree!
host.shadowRoot; // też działa!
```

Ten atak wymaga **wykonania kodu przed załadowaniem komponentu** — możliwe przez XSS, zmodyfikowany CDN, lub jeśli atakujący kontroluje JS ładowany przed komponentem.

### DOM Clobbering a Shadow DOM

Shadow DOM może być używany do ominięcia niektórych filtrów XSS:

```javascript
// Niektóre sanityzery nie analizują shadow DOM prawidłowo
// lub ignorują shadow tree

// Przeglądarki wykonują JS w shadow DOM:
const shadow = host.attachShadow({ mode: 'open' });
shadow.innerHTML = '<img src=x onerror=alert(1)>';
// XSS wykonuje się! (onerror działa w shadow DOM)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Myślenie że closed = bezpieczne

```javascript
// BŁĄD: założenie że closed Shadow DOM jest "bezpiecznym sejfem"
const shadow = element.attachShadow({ mode: 'closed' });
shadow.innerHTML = `<input type="hidden" value="${secretToken}">`;
// XSS może monkey-patch attachShadow PRZED tym kodem → dostęp do shadow!
// Lub: DevTools → shadow DOM jest widoczny zawsze
```

### Błąd 2: innerHTML w Shadow DOM bez sanityzacji

```javascript
// BŁĄD: dynamiczny content w shadow DOM — XSS nadal możliwy!
shadow.innerHTML = `<p>${userContent}</p>`;
// XSS działa tak samo jak w light DOM

// POPRAWKA: DOMPurify lub textContent
const p = document.createElement('p');
p.textContent = userContent;
shadow.appendChild(p);
```

### Błąd 3: Event.target confusion

```javascript
// Nieoczekiwane: target to shadow host, nie wewnętrzny element
host.addEventListener('click', (e) => {
    if (e.target.matches('.button')) { // to NIE będzie działać!
        // e.target to shadow host (retargetowane)
    }
    // Użyj e.composedPath()[0] aby uzyskać prawdziwy element
    const realTarget = e.composedPath()[0];
    if (realTarget.matches('.button')) { } // OK
});
```

---

## 9. Znaczenie dla bezpieczeństwa

### Shadow DOM nie chroni przed XSS

XSS wykonany wewnątrz shadow DOM ma dostęp do:
- `document` (globalny)
- Wszystkich cookies, localStorage
- `window` i wszystkich globalnych API
- Zewnętrznych żądań sieciowych

Shadow DOM izoluje **CSS i DOM strukturę**, nie JavaScript execution context.

### DevTools penetracja Shadow DOM

Każdy Shadow DOM (open i closed) jest **widoczny w DevTools**:
- Elements → rozwiń element z shadow root → widoczna struktura
- W `$0.shadowRoot` lub przez DevTools inspektora

### CSS Variables jako side channel

CSS Custom Properties przechodzą przez Shadow DOM boundary. Atakujący z kontrolą nad zewnętrznym CSS może użyć tego do stylizowania shadow tree (co samo w sobie nie jest atakiem bezpieczeństwa, ale jest warte uwagi).

### Sanitizer API i Shadow DOM

DOMPurify i nowy Sanitizer API (eksperymentalny) sanityzują zawartość **zanim** trafi do Shadow DOM. Shadow DOM sam w sobie nie sanityzuje.

### Powiązane CWE

- **CWE-79** — XSS (w Shadow DOM tak samo jak w Light DOM)
- **CWE-276** — Incorrect Default Permissions (closed mode nie zapewnia rzeczywistej izolacji)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Elements

1. W Elements tab — elementy z Shadow DOM mają wskaźnik `#shadow-root`
2. Rozwiń → widać shadow tree (open I closed)
3. Możesz zaznaczać i modyfikować elementy w shadow tree

### Console

```javascript
// Open shadow:
document.querySelector('my-element').shadowRoot;

// Closed shadow (jeśli nie monkey-patchowany):
document.querySelector('my-element').shadowRoot; // null

// Użyj composedPath by śledzić zdarzenia przez granice Shadow DOM
document.addEventListener('click', e => {
    console.log('composedPath:', e.composedPath());
});

// Sprawdź czy attachShadow jest monkey-patchowany (by atakujący):
Element.prototype.attachShadow.toString(); // powinno zawierać [native code]
if (!Element.prototype.attachShadow.toString().includes('[native code]')) {
    console.warn('attachShadow może być monkey-patchowany!');
}
```

### Szukaj XSS w Shadow DOM

```javascript
// Grep za dynamicznym contentem w shadow:
shadow.innerHTML = 
shadowRoot.innerHTML =
this._shadow.innerHTML =

// Sprawdź czy input jest sanityzowany przed wstawieniem
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy innerHTML w Shadow DOM jest chroniony przed XSS (DOMPurify)?
□ Czy closed mode jest traktowany jako fałszywe poczucie bezpieczeństwa?
□ Czy attachShadow jest monkey-patchowany (XSS przed inicjalizacją)?
□ Czy XSS wewnątrz shadow tree ma dostęp do globalnego document/window?
□ Czy event retargeting może prowadzić do confusion w aplikacji?
□ Czy CSS isolation jest skuteczna (CSS variables mogą wnikać)?
□ Czy sloty nie wprowadzają niewyrenderowanego XSS payload?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Zawsze sanityzuj przed wstawieniem do shadow DOM
const shadow = host.attachShadow({ mode: 'open' });

function setShadowContent(html) {
    shadow.innerHTML = DOMPurify.sanitize(html, {
        RETURN_DOM_FRAGMENT: true,
        FORCE_BODY: false
    });
}

// 2. Używaj textContent dla user-generated text
const p = shadow.querySelector('p');
p.textContent = userText; // zamiast innerHTML

// 3. Trusted Types (Rozdział 37) działają też w Shadow DOM
const policy = trustedTypes.createPolicy('shadow-policy', {
    createHTML: (html) => DOMPurify.sanitize(html)
});
shadow.innerHTML = policy.createHTML(html);

// 4. Nie przechowuj sekretów w shadow DOM (widoczne w DevTools)
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Shadow DOM = izolowane poddrzewo DOM z własnym CSS scope
2. `mode: 'closed'` NIE jest bezpieczną izolacją — można monkey-patchować `attachShadow`
3. DevTools widzi każdy Shadow DOM (open i closed)
4. XSS wewnątrz Shadow DOM ma pełny dostęp do globalnego window/document
5. CSS izolacja działa, ale CSS Custom Properties przechodzą przez granicę

**Dla pentestera:**
- Elements → `#shadow-root` w DevTools
- Sprawdź czy `attachShadow` jest monkey-patchowany
- Testuj XSS w shadow innerHTML (tak samo jak w light DOM)
- `closed` Shadow DOM nie jest rzeczywistą barierą bezpieczeństwa

---

## Powiązania

```
Shadow DOM
    │
    ├──► Custom Elements (Rozdział 32)
    │         Shadow DOM + Custom Elements = Web Components
    │
    ├──► DOM/XSS (Rozdział 9)
    │         XSS dotyczy shadow DOM tak samo jak light DOM
    │
    ├──► Trusted Types (Rozdział 37)
    │         Trusted Types chronią innerHTML w shadow DOM
    │
    └──► DOM Clobbering (Rozdział 10)
              Interakcje Shadow DOM i named access
```
