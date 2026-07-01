# Rozdział 33: Web Components (HTML Templates)

## 1. Czym są Web Components i HTML Templates

**Web Components** to zestaw standardów webowych umożliwiający tworzenie reużywalnych, enkapsulowanych komponentów UI. Składają się z trzech technologii:
1. **Custom Elements** (Rozdział 32) — definicja nowych tagów HTML
2. **Shadow DOM** (Rozdział 31) — enkapsulacja DOM i CSS
3. **HTML Templates** — szablony HTML nieco poza głównym renderowaniem

**HTML `<template>` element** to specjalny element HTML przechowujący "martwy" HTML — parser parsuje zawartość ale **nie renderuje jej** i **nie wykonuje skryptów/ładowania zasobów** dopóki nie zostanie sklonowany i dodany do dokumentu.

**`<slot>` element** to punkt wstrzyknięcia (używany wewnątrz Shadow DOM), gdzie Light DOM content jest "wyświetlany".

Ten rozdział koncentruje się na HTML Templates jako brakującym elemencie Web Components, z uwzględnieniem implikacji bezpieczeństwa całego ekosystemu.

---

## 2. Dlaczego powstał

### Problem: dynamiczne szablony HTML w JS

Przed `<template>`:
- Szablony w JavaScript jako stringi → podatne na XSS, trudne do utrzymania
- Ukryte `<div style="display:none">` → parser ładował zasoby (img src), parsował skrypty
- Template literals bez sanityzacji

`<template>` (HTML5) rozwiązuje:
- Inert content — brak ładowania zasobów, brak wykonywania skryptów
- Klonowanie i instancjonowanie na żądanie
- Integracja z Shadow DOM przez sloty

---

## 3. Jak działa

### HTML Template element

```html
<!-- W HTML: -->
<template id="user-card-template">
    <style>
        :host { display: block; border: 1px solid #ccc; padding: 8px; }
        .name { font-weight: bold; }
    </style>
    <div class="card">
        <h3 class="name"><!-- wypełniane przez JS --></h3>
        <p class="email"></p>
        <slot name="actions"></slot>
    </div>
    
    <!-- Ten img NIE zostanie pobrany dopóki template nie zostanie sklonowany! -->
    <img src="https://api.example.com/placeholder">
    
    <!-- Ten skrypt NIE zostanie wykonany w template! -->
    <script>console.log('nie wykona się!');</script>
</template>
```

```javascript
// JavaScript — klonowanie i użycie template
const template = document.getElementById('user-card-template');
const content = template.content; // → DocumentFragment

// Klonuj template (deep clone)
const clone = content.cloneNode(true);

// Wypełnij danymi
clone.querySelector('.name').textContent = user.name;   // textContent bezpieczne
clone.querySelector('.email').textContent = user.email;

// Dodaj do dokumentu → teraz się renderuje
document.body.appendChild(clone);
// W tym momencie: img zostaje pobrany, skrypt wykonany (gdyby był)
```

### Template z Custom Element i Shadow DOM

```javascript
class UserCard extends HTMLElement {
    connectedCallback() {
        const shadow = this.attachShadow({ mode: 'open' });
        
        // Użyj template z HTML
        const template = document.getElementById('user-card-template');
        const clone = template.content.cloneNode(true);
        
        // Bezpieczne wypełnienie danych
        clone.querySelector('.name').textContent = this.getAttribute('name');
        clone.querySelector('.email').textContent = this.getAttribute('email');
        
        shadow.appendChild(clone);
    }
}
customElements.define('user-card', UserCard);
```

### `<slot>` element

```html
<!-- Shadow DOM template z named slot: -->
<template id="dialog-template">
    <div class="dialog">
        <header><slot name="title">Domyślny tytuł</slot></header>
        <main><slot></slot></main>  <!-- default slot -->
        <footer><slot name="actions"></slot></footer>
    </div>
</template>

<!-- Użycie (Light DOM): -->
<my-dialog>
    <h2 slot="title">Tytuł okna</h2>        <!-- wchodzi do name="title" -->
    <p>Treść dialogu</p>                     <!-- wchodzi do default slot -->
    <button slot="actions">OK</button>       <!-- wchodzi do name="actions" -->
</my-dialog>
```

---

## 4. Co dzieje się wewnętrznie

### DocumentFragment

`template.content` to `DocumentFragment` — lekki kontener DOM który może być sklonowany i dodany do dokumentu. Fragment nie jest częścią głównego DOM dopóki nie zostanie dodany.

### Inert parsing

Przeglądarka parsuje zawartość `<template>` do DocumentFragment, ale:
- Nie ładuje zasobów (`<img src>`, `<link>`, `<script src>`)
- Nie wykonuje `<script>`
- Nie renderuje CSS

Dopiero po `appendChild(clone)` — te efekty się aktywują.

### Slotting mechanism

Sloty działają przez **projection**, nie przez kopiowanie. Light DOM elementy z `slot=""` atrybut:
- Pozostają w Light DOM
- Są renderowane w miejscu odpowiedniego `<slot>` w Shadow DOM
- Można je stylizować zarówno z Light jak i Shadow CSS (`::slotted()`)

---

## 5. Analogiczny przykład z życia

HTML Template to foremka do ciasteczek:

- Foremka (template) ma kształt ale nie jest ciasteczkiem (nie renderuje się)
- Klonujesz foremkę gdy potrzebujesz ciasteczka (cloneNode)
- Wypełniasz foremkę ciastem (danymi) — `textContent`
- Dopiero gdy wkładasz do pieca (appendChild do DOM) → ciasteczko istnieje

---

## 6. Przykład kodu

```javascript
// === Bezpieczna fabryka kart z template ===
const template = document.createElement('template');
template.innerHTML = `
    <style>
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 16px; margin: 8px; }
        .card-title { font-size: 18px; font-weight: bold; }
        .card-body { margin-top: 8px; color: #666; }
    </style>
    <div class="card">
        <div class="card-title"></div>
        <div class="card-body"></div>
        <slot name="actions"></slot>
    </div>
`;

class InfoCard extends HTMLElement {
    static get observedAttributes() { return ['title', 'body']; }
    
    connectedCallback() {
        const shadow = this.attachShadow({ mode: 'open' });
        const clone = template.content.cloneNode(true);
        
        this._titleEl = clone.querySelector('.card-title');
        this._bodyEl = clone.querySelector('.card-body');
        
        this._update(clone);
        shadow.appendChild(clone);
    }
    
    attributeChangedCallback() {
        if (this.shadowRoot) {
            this._titleEl.textContent = this.getAttribute('title') ?? '';
            this._bodyEl.textContent = this.getAttribute('body') ?? '';
        }
    }
    
    _update(clone) {
        (this._titleEl ?? clone.querySelector('.card-title')).textContent 
            = this.getAttribute('title') ?? '';
        (this._bodyEl ?? clone.querySelector('.card-body')).textContent 
            = this.getAttribute('body') ?? '';
    }
}

customElements.define('info-card', InfoCard);

// Dynamiczne tworzenie:
function createCard(title, body) {
    const card = document.createElement('info-card');
    card.setAttribute('title', title);   // bezpieczne — atrybuty enkodowane przez przeglądarkę
    card.setAttribute('body', body);
    return card;
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Template Injection a HTML Templates

`<template>` **nie chroni** przed template injection w serwerowych szablonach (Jinja2, Handlebars, itp.):

```html
<!-- Plik HTML generowany przez serwer (Jinja2): -->
<template id="user-template">
    <div class="user">{{ user.bio }}</div>
    <!--           ^^^^ jeśli bio zawiera }} lub </template> → injection! -->
</template>

<!-- Atak: bio = "</template><script>alert(1)</script><template>" -->
<!-- Generuje: -->
<template id="user-template">
    <div class="user"></template><script>alert(1)</script><template></div>
</template>
<!-- Parser HTML: <script> jest POZA template → wykonuje się! -->
```

---

## 8. Typowe błędy programistów

### Błąd 1: innerHTML w template zamiast textContent

```javascript
// BŁĄD: dynamicznie tworzysz template z user input
const template = document.createElement('template');
template.innerHTML = `<div>${userInput}</div>`; // XSS!
// Mimo że template jest "inert" — innerHTML parsuje HTML
// Po klonowaniu i appendChild → XSS wykonuje się

// POPRAWKA:
const template = document.createElement('template');
template.innerHTML = `<div class="user-content"></div>`;
const clone = template.content.cloneNode(true);
clone.querySelector('.user-content').textContent = userInput; // bezpieczne
```

### Błąd 2: Zapomnienie o cloneNode(true)

```javascript
// BŁĄD: appendChile(template.content) bez klonowania
shadow.appendChild(template.content);
// Przesuwa content z template do shadow — template staje się pusty!
// Kolejne użycie → pusty template

// POPRAWKA:
shadow.appendChild(template.content.cloneNode(true)); // deep clone
```

---

## 9. Znaczenie dla bezpieczeństwa całego ekosystemu Web Components

### XSS wektory w Web Components

1. **Custom Elements + innerHTML** — `attributeChangedCallback` wstawia niesanityzowany atrybut
2. **Shadow DOM + innerHTML** — dynamiczny content w shadow tree
3. **Template + innerHTML** — tworzenie template ze stringów
4. **Slots + XSS** — treść Light DOM w slocie może zawierać XSS jeśli parent jej nie sanityzuje

### Supply Chain Risk

Web Components często ładowane z CDN lub npm:
```html
<script type="module" src="https://cdn.example.com/components.js"></script>
```

Jeśli CDN jest skompromitowany lub brak **Subresource Integrity (SRI)** → malicious komponent zostaje załadowany.

```html
<!-- BEZPIECZNE z SRI (Rozdział 38): -->
<script type="module" 
    src="https://cdn.example.com/components.js"
    integrity="sha384-abc123..."
    crossorigin="anonymous">
</script>
```

### CSP a Custom Elements

Custom Elements wymagają `script-src` pozwalającego na ich ładowanie. Inline Custom Elements w `<script>` tagach wymagają `'nonce-xxx'` lub hash.

### Powiązane CWE

- **CWE-79** — XSS (przez template + innerHTML, attributeChangedCallback)
- **CWE-494** — Download of Code Without Integrity Check (brak SRI dla Web Components)

---

## 10. Jak identyfikować podczas pentestu

### DevTools

```javascript
// Lista wszystkich Custom Elements na stronie:
const customTags = Array.from(document.querySelectorAll('*'))
    .filter(el => el.tagName.includes('-'))
    .map(el => el.tagName.toLowerCase())
    .filter((v, i, a) => a.indexOf(v) === i);
console.log('Custom Elements:', customTags);

// Sprawdź klasy:
customTags.forEach(tag => {
    const cls = customElements.get(tag);
    console.log(tag, '→', cls?.toString().substring(0, 200));
});

// Znajdź templates:
document.querySelectorAll('template').forEach(t => {
    console.log('Template:', t.id, t.innerHTML.substring(0, 200));
});
```

### Co szukać

- `innerHTML` w `attributeChangedCallback` lub `connectedCallback`
- Dynamiczne tworzenie template z user input
- Brak SRI dla zewnętrznych Web Component bibliotek
- Template injection w serwerowych szablonach generujących HTML templates

---

## 11. Jak testować bezpieczeństwo

```
□ Czy innerHTML w Custom Element callbacks jest sanityzowany (DOMPurify)?
□ Czy template.innerHTML jest budowane z user input (XSS)?
□ Czy zewnętrzne Web Component biblioteki mają SRI?
□ Czy serwerowe szablony generujące <template> HTML są odporne na injection?
□ Czy CSP jest skonfigurowane dla Web Components?
□ Czy możliwa jest podmiana Custom Element definicji (component hijacking)?
□ Czy dane wrażliwe nie są przechowywane w template content (widoczne w DevTools)?
```

---

## 12. Jak się zabezpieczać

```javascript
// 1. Zawsze sanityzuj dynamiczny content
class SafeCard extends HTMLElement {
    connectedCallback() {
        const shadow = this.attachShadow({ mode: 'open' });
        const clone = this.constructor.template.content.cloneNode(true);
        
        // textContent zamiast innerHTML dla user content
        clone.querySelector('.title').textContent = this.getAttribute('title');
        
        // DOMPurify dla rich content
        const body = this.getAttribute('body');
        if (body) {
            clone.querySelector('.body').innerHTML = DOMPurify.sanitize(body, {
                ALLOWED_TAGS: ['b', 'i', 'strong', 'em', 'a'],
                ALLOWED_ATTR: ['href']
            });
        }
        
        shadow.appendChild(clone);
    }
    
    static template = (() => {
        const t = document.createElement('template');
        t.innerHTML = `
            <style>:host { display: block; }</style>
            <div class="title"></div>
            <div class="body"></div>
        `;
        return t;
    })();
}

// 2. SRI dla zewnętrznych Web Components
// <script src="..." integrity="sha384-..." crossorigin>

// 3. Trusted Types (Rozdział 37)
// Automatycznie wymusi sanityzację przed każdym innerHTML
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. HTML Template = "inert" HTML który nie renderuje zasobów ani nie wykonuje skryptów dopóki nie sklonowany
2. Web Components = Custom Elements + Shadow DOM + HTML Templates
3. `template.innerHTML` z user input → XSS po klonowaniu i dodaniu do DOM
4. Supply chain risk przez zewnętrzne Web Component biblioteki (SRI wymagane)
5. Server-side template injection może przerywać `<template>` tagi

**Dla pentestera:**
- Szukaj `innerHTML` w Web Component callbacks i template construction
- Sprawdź SRI dla zewnętrznych Web Component bibliotek
- Testuj serwerowe szablony generujące `<template>` na injection
- DevTools → Elements → `#document-fragment` wewnątrz shadow roots

---

## Powiązania

```
Web Components / HTML Templates
    │
    ├──► Custom Elements (Rozdział 32)
    │         Lifecycle + observedAttributes
    │
    ├──► Shadow DOM (Rozdział 31)
    │         Enkapsulacja CSS i DOM
    │
    ├──► Trusted Types (Rozdział 37)
    │         Ochrona innerHTML w Web Components
    │
    ├──► Subresource Integrity (Rozdział 38)
    │         SRI dla zewnętrznych Web Component bibliotek
    │
    └──► DOM/XSS (Rozdział 9)
              XSS w Web Components — te same mechanizmy
```
