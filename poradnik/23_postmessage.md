# Rozdział 23: postMessage

## 1. Czym jest postMessage

**`window.postMessage()`** to API umożliwiające **bezpieczną komunikację cross-origin** między kontekstami przeglądania — oknami, kartami, iframes, Workers. Jest to jedyny zatwierdiony mechanizm komunikacji między różnymi origins w przeglądarce, omijający Same-Origin Policy w kontrolowany sposób.

`postMessage` umożliwia:
- Komunikację `window` → iframe (różny origin)
- Komunikację opener → opened window (window.open)
- Komunikację strona → Worker (Dedicated, Shared, Service)
- Komunikację między Workers

Kluczowe aspekty bezpieczeństwa:
- Nadawca specyfikuje docelowy origin (`targetOrigin`)
- Odbiorca musi weryfikować `event.origin`
- Brak weryfikacji po którejkolwiek stronie → poważne luki bezpieczeństwa

---

## 2. Dlaczego powstał

### Problem: Same-Origin Policy blokuje cross-frame komunikację

Przed `postMessage` (HTML5, 2008):
- Iframes z innym origin nie mogły komunikować się z rodzicem
- Jedynym obejściem było `document.domain` hack (niebezpieczne, ograniczone do subdomains)
- Lub fragmenty URL (`#hash`) — bardzo ograniczone i widoczne w historii

`postMessage` dostarcza:
- Pełną komunikację cross-origin z kontrolą bezpieczeństwa
- Obsługę złożonych typów danych (Structured Clone)
- Weryfikację origin po obu stronach

---

## 3. Jak działa

### Syntax

```javascript
// Nadawca:
targetWindow.postMessage(message, targetOrigin [, transfer]);

// targetWindow: iframe.contentWindow, window.opener, window.parent, worker
// message: dowolna Structured Cloneable wartość
// targetOrigin: URL origin lub '*' (NIEBEZPIECZNE) lub '/'
// transfer: opcjonalnie, tablica Transferable objects
```

### Podstawowy przepływ

```
Strona A (parent, https://parent.com)
        │
        │  iframe.contentWindow.postMessage(data, 'https://child.com')
        │
        ▼
Iframe B (https://child.com)
        │
        │  window.addEventListener('message', handler)
        │  handler: sprawdź event.origin === 'https://parent.com'
        │           przetwórz event.data
        │
        │  event.source.postMessage(response, 'https://parent.com')
        │
        ▼
Strona A odbiera odpowiedź (onmessage)
```

### Message Event

```javascript
// Obiekt MessageEvent w handlerze:
window.addEventListener('message', (event) => {
    event.data   // przesłana wiadomość (Structured Clone)
    event.origin // origin nadawcy: 'https://sender.com'
    event.source // referencja do okna nadawcy (może być null dla Workers)
    event.ports  // MessagePort[] dla MessageChannel
});
```

### targetOrigin

```javascript
// BEZPIECZNE: konkretny origin
iframe.contentWindow.postMessage(data, 'https://trusted.com');

// NIEBEZPIECZNE: '*' = wysyła do dowolnego origin
iframe.contentWindow.postMessage(sensitiveData, '*');
// Jeśli iframe nawigował do evil.com → evil.com dostanie wiadomość!

// Specjalny przypadek: '/' = same-origin
window.postMessage(data, '/');
```

---

## 4. Co dzieje się wewnętrznie

### Structured Clone dla przesyłania danych

Dane są serializowane algorytmem Structured Clone — ten sam co IndexedDB i BroadcastChannel. Obsługuje Date, Map, Set, ArrayBuffer, Blob, ale nie funkcje ani DOM nodes.

### Asynchroniczność

`postMessage` jest asynchroniczne. Wiadomość jest dodawana do kolejki Macrotask, nie jest dostarczana synchronicznie:

```javascript
window.postMessage('hello', '*');
console.log('po postMessage'); // To wykona się PRZED dostarczeniem wiadomości

// Wiadomość dostarczana jest w kolejnym Macrotask cycle
```

### Transfer Ownership

Podobnie jak w Workers, można transferować ArrayBuffer bez kopiowania:

```javascript
const buffer = new ArrayBuffer(1024 * 1024); // 1 MB
iframe.contentWindow.postMessage(buffer, 'https://child.com', [buffer]);
// buffer.byteLength === 0 po transferze w parent!
```

---

## 5. Analogiczny przykład z życia

postMessage to list dyplomatyczny między państwami:

- Wysyłający (parent) adresuje list konkretnie do Francji (`targetOrigin: 'https://france.com'`)
- Jeśli adresat nie ma pieczęci Francji (`event.origin !== 'https://france.com'`), list jest odrzucany
- Treść listu może być dowolna (Structured Clone)
- Wysłanie do `*` to list bez adresata — może go przejąć każdy
- Odbiorca może odpowiedzieć (`event.source.postMessage`)

---

## 6. Przykład kodu

```javascript
// === Bezpieczny wzorzec postMessage ===

// === Parent (https://parent.com) ===
const iframe = document.querySelector('iframe');

// Wyślij po załadowaniu iframe
iframe.addEventListener('load', () => {
    iframe.contentWindow.postMessage(
        { type: 'INIT', userId: 123 },
        'https://child.com' // targetOrigin — konkretny!
    );
});

// Odbierz odpowiedź
window.addEventListener('message', (event) => {
    // KROK 1: Zweryfikuj origin
    if (event.origin !== 'https://child.com') {
        console.warn('Odrzucono wiadomość od:', event.origin);
        return;
    }
    
    // KROK 2: Zweryfikuj source (opcjonalnie ale zalecane)
    if (event.source !== iframe.contentWindow) return;
    
    // KROK 3: Zweryfikuj strukturę danych
    if (!event.data || typeof event.data.type !== 'string') return;
    
    // KROK 4: Obsługuj
    switch (event.data.type) {
        case 'READY':
            console.log('Iframe gotowy');
            break;
        case 'USER_ACTION':
            handleAction(event.data.action);
            break;
    }
});

// === Child (https://child.com) ===
window.addEventListener('message', (event) => {
    // Zawsze weryfikuj origin!
    if (event.origin !== 'https://parent.com') return;
    
    if (event.data.type === 'INIT') {
        initWithUser(event.data.userId);
        // Odpowiedz do parenta
        event.source.postMessage({ type: 'READY' }, 'https://parent.com');
    }
});
```

---

## 7. Przykład z prawdziwej aplikacji

### OAuth popup → parent komunikacja

```javascript
// === main app (parent) ===
let authWindow = null;

function startOAuth() {
    authWindow = window.open(
        'https://auth.example.com/oauth?callback=https://app.com/callback',
        'auth',
        'width=600,height=700'
    );
}

window.addEventListener('message', (event) => {
    // KRYTYCZNE: weryfikuj origin
    if (event.origin !== 'https://auth.example.com') return;
    if (event.data.type !== 'OAUTH_COMPLETE') return;
    
    const { code, state } = event.data;
    
    // Wymień code na token przez server (PKCE flow)
    exchangeCodeForToken(code, state);
    
    authWindow?.close();
});

// === OAuth callback page (child) ===
// Po powrocie z OAuth provider:
const params = new URLSearchParams(window.location.search);
window.opener.postMessage(
    {
        type: 'OAUTH_COMPLETE',
        code: params.get('code'),
        state: params.get('state')
    },
    'https://app.com' // tylko do parent app
);
window.close();
```

### Sandboxed iframe z postMessage API

```javascript
// Aplikacja "plugin" w sandboxed iframe — ograniczone uprawnienia
// HTML: <iframe src="plugin.html" sandbox="allow-scripts" id="plugin">

// main.js:
const plugin = document.getElementById('plugin');
const API = {
    getData: () => ({ users: [{ id: 1, name: 'Jan' }] }),
    setTheme: (theme) => document.body.className = theme
};

window.addEventListener('message', ({ origin, data, source }) => {
    if (origin !== new URL(plugin.src).origin) return;
    
    if (data.method in API) {
        const result = API[data.method](data.args);
        source.postMessage({ id: data.id, result }, origin);
    }
});

// plugin.js (w sandboxed iframe):
function callAPI(method, args) {
    return new Promise(resolve => {
        const id = Math.random().toString(36);
        
        window.addEventListener('message', function handler({ data }) {
            if (data.id === id) {
                window.removeEventListener('message', handler);
                resolve(data.result);
            }
        });
        
        parent.postMessage({ id, method, args }, '*');
        // '*' bo plugin może nie znać origin parenta
        // Ryzyko: parent powinien używać konkretnego origin przy postMessage do pluginu
    });
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak weryfikacji `event.origin` — KRYTYCZNE

```javascript
// BŁĄD: akceptuje wiadomości od DOWOLNEGO origin
window.addEventListener('message', (event) => {
    // Brak: if (event.origin !== 'https://trusted.com') return;
    
    if (event.data.type === 'SET_CONFIG') {
        applyConfig(event.data.config); // XSS z dowolnej strony!
    }
});

// Atak:
// evil.com otwiera popup lub iframe do atakowanej strony
// evil.com.postMessage({ type: 'SET_CONFIG', config: { adminMode: true } }, '*')
// → applyConfig wykonuje się z malicious config!
```

### Błąd 2: Użycie `*` jako targetOrigin z danymi wrażliwymi

```javascript
// BŁĄD: wysłanie tokenu do '*'
window.postMessage({ token: 'sensitive-token' }, '*');
// Każde okno/iframe na stronie dostaje token!

// POPRAWKA:
window.postMessage({ token: 'sensitive-token' }, 'https://specific-target.com');
```

### Błąd 3: eval() na danych z postMessage

```javascript
// KRYTYCZNE: eval na danych z postMessage = RCE dla dowolnego origin
window.addEventListener('message', (event) => {
    eval(event.data.code); // Nie weryfikuje origin — XSS z dowolnego miejsca!
});

// Nawet z weryfikacją origin — eval jest zły:
window.addEventListener('message', (event) => {
    if (event.origin !== 'https://trusted.com') return;
    eval(event.data.code); // Jeśli trusted.com ma XSS → też podatne
});
```

### Błąd 4: Pominięcie weryfikacji `event.source`

```javascript
// Jeśli mamy wiele iframes, weryfikuj też source:
window.addEventListener('message', (event) => {
    if (event.origin !== 'https://child.com') return;
    // Ale który iframe? Jeśli mamy dwa iframes z child.com...
    if (event.source !== trustedIframe.contentWindow) return; // Dodatkowa weryfikacja
});
```

---

## 9. Znaczenie dla bezpieczeństwa

### Brak weryfikacji origin → Universal XSS

Najpopularniejsza podatność postMessage: brak weryfikacji `event.origin`. Efektywnie pozwala dowolnej stronie (evil.com) wysyłać wiadomości i wykonywać akcje w kontekście podatnej strony.

**Warunki ataku:**
1. Podatna strona ma `window.addEventListener('message', handler)` bez origin check
2. Atakujący otwiera iframe lub popup z podatną stroną
3. Atakujący wywołuje `victim.postMessage(malicious, '*')`
4. Handler wykonuje akcję z malicious danymi

### postMessage jako CSRF bypass

```javascript
// Aplikacja sprawdza CSRF token przez AJAX, ale postMessage handler nie sprawdza:
window.addEventListener('message', (event) => {
    // Brak CSRF check!
    if (event.data.action === 'transfer') {
        api.transferFunds(event.data.to, event.data.amount);
    }
});

// Atak z evil.com:
// <script>
//   const victim = window.open('https://bank.com/dashboard');
//   setTimeout(() => {
//     victim.postMessage({ action: 'transfer', to: 'evil-account', amount: 1000 }, '*');
//   }, 3000);
// </script>
```

### Otwarte redirecty i postMessage

```javascript
// Aplikacja ma open redirect: /redirect?url=https://evil.com
// Połączone z postMessage:

// Parent otwiera OAuth popup do auth.example.com
// auth.example.com ma open redirect
// Atakujący nakłania użytkownika do odwiedzenia:
// https://auth.example.com/redirect?url=https://evil.com/fake-oauth

// evil.com/fake-oauth po otrzymaniu code:
window.opener.postMessage({
    type: 'OAUTH_COMPLETE',
    code: 'stolen-code' // Fałszywy lub skradziony kod
}, 'https://app.com');
// Jeśli app.com nie weryfikuje że code pochodzi z prawdziwego auth.example.com
// → atakujący może podmienić code
```

### Powiązane CWE i podatności

- **CWE-79** — XSS (brak origin check → wykona się w kontekście strony)
- **CWE-346** — Origin Validation Error
- **CWE-345** — Insufficient Verification of Data Authenticity
- **OWASP** — postMessage Security Issues

---

## 10. Jak identyfikować podczas pentestu

### Statyczna analiza — szukaj w kodzie

```javascript
// Grep za podatnymi wzorcami:
addEventListener('message',    // nasłuch bez origin check
postMessage(                   // wysyłanie — sprawdź targetOrigin
window.postMessage             // globalny postMessage
self.postMessage               // Worker postMessage

// Sprawdź czy handler weryfikuje origin:
window.addEventListener('message', function(e) {
    // Jeśli tu nie ma: if (e.origin !== '...') return;
    // → PODATNOŚĆ
});
```

### DevTools — monitorowanie

```javascript
// Monkey-patch addEventListener w konsoli:
const originalAddEventListener = window.addEventListener.bind(window);
window.addEventListener = function(type, handler, ...rest) {
    if (type === 'message') {
        const wrappedHandler = function(event) {
            console.log('[postMessage intercepted]', {
                origin: event.origin,
                data: event.data,
                source: event.source
            });
            return handler.call(this, event);
        };
        return originalAddEventListener(type, wrappedHandler, ...rest);
    }
    return originalAddEventListener(type, handler, ...rest);
};
```

```javascript
// Wyślij testową wiadomość do iframe i sprawdź czy akceptuje:
const iframe = document.querySelector('iframe');
iframe.contentWindow.postMessage({ test: 'evil-payload' }, '*');
// Sprawdź czy coś się stało w aplikacji

// Lub wyślij z evil kontrolowanego popup:
const victim = window.open('https://target.com');
setTimeout(() => {
    victim.postMessage({ action: 'deleteAccount' }, 'https://target.com');
}, 2000);
```

### Burp Suite — Postmessage Logger

1. Burp → Proxy → Options → Match and Replace → dodaj JS injection
2. Lub użyj Burp extension: "postmessage-tracker" (KNOXSS)
3. DOM Invader (Burp) automatycznie wykrywa podatne postMessage handlers

---

## 11. Jak testować bezpieczeństwo

```
□ Czy window.addEventListener('message') weryfikuje event.origin?
□ Czy postMessage używa konkretnego targetOrigin (nie '*')?
□ Czy handler wykonuje niebezpieczne operacje (eval, innerHTML, fetch)?
□ Czy handler nie weryfikuje też event.source?
□ Czy wrażliwe dane są wysyłane przez postMessage z targetOrigin: '*'?
□ Czy OAuth/auth flow używa postMessage — czy origin jest weryfikowany?
□ Czy sandbox iframe komunikuje się przez postMessage — czy jest walidacja?
□ Czy postMessage może bypassować CSRF protection?
□ Czy jest open redirect + postMessage attack vector?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz 1: Cross-origin postMessage bez origin check

```
1. Cel: https://app.com ma handler:
   window.addEventListener('message', e => setAdmin(e.data.admin))
   (brak e.origin check)

2. Atakujący: evil.com
   <script>
     const target = window.open('https://app.com');
     target.postMessage({ admin: true }, 'https://app.com');
   </script>

3. Efekt: app.com wykonuje setAdmin(true) z evil.com!
```

### Scenariusz 2: postMessage DOM XSS

```
1. Handler:
   window.addEventListener('message', e => {
     document.getElementById('content').innerHTML = e.data; // brak sanityzacji!
   });

2. Atak:
   victim.postMessage('<img src=x onerror=alert(document.cookie)>', 'https://victim.com');

3. Efekt: XSS wykonany w kontekście victim.com!
```

### Scenariusz 3: Malicious iframe sender

```html
<!-- Podatna strona embedduje user-controlled iframe -->
<iframe src="https://user-content.example.com/widget"></iframe>

<!-- user-content może być kontrolowane przez atakującego -->
<!-- i wysyłać postMessage do parenta bez weryfikacji origin w parentcie -->
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. ZAWSZE weryfikuj event.origin
window.addEventListener('message', (event) => {
    const TRUSTED_ORIGINS = new Set([
        'https://partner.com',
        'https://auth.example.com'
    ]);
    
    if (!TRUSTED_ORIGINS.has(event.origin)) {
        console.warn('Untrusted origin:', event.origin);
        return;
    }
    
    // Teraz bezpiecznie obsłuż
    handleMessage(event.data);
});

// 2. ZAWSZE używaj konkretnego targetOrigin
iframe.contentWindow.postMessage(sensitiveData, 'https://specific-origin.com');
// NIGDY: postMessage(sensitiveData, '*')

// 3. Waliduj strukturę danych
window.addEventListener('message', ({ origin, data }) => {
    if (origin !== 'https://trusted.com') return;
    
    // Whitelist typów wiadomości
    const ALLOWED_TYPES = ['READY', 'USER_ACTION', 'CLOSE'];
    if (!ALLOWED_TYPES.includes(data?.type)) return;
    
    // Waliduj każde pole
    if (data.type === 'USER_ACTION' && typeof data.action !== 'string') return;
    
    handleMessage(data);
});

// 4. Nigdy nie eval() danych z postMessage
// Zamiast: eval(event.data)
// Używaj: switch/dispatch na whitelist akcji

// 5. Używaj MessageChannel dla point-to-point (zamiast postMessage z window ref)
// Rozdział 24 — MessageChannel daje bezpieczniejsze kanały
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. postMessage = jedyny zatwierdzony cross-origin komunikacji mechanizm
2. Nadawca używa targetOrigin — '' pozwala na atak
3. Odbiorca MUSI weryfikować event.origin — brak = Universal XSS
4. Używa Structured Clone — bezpieczna serializacja (brak eval)
5. Główne zagrożenia: brak origin check, targetOrigin: '*', eval na danych

**Dla pentestera:**
- Szukaj `addEventListener('message'` bez `event.origin` check
- Sprawdź wszystkie użycia `postMessage` — czy targetOrigin jest konkretny?
- DOM Invader w Burp automatycznie wykrywa podatne handlery
- Testuj: wyślij postMessage z kontrolowanego origin i obserwuj efekty

---

## Powiązania

```
postMessage
    │
    ├──► BroadcastChannel (Rozdział 22)
    │         BC = broadcast (1 → wiele, same-origin only)
    │         postMessage = point-to-point (może cross-origin)
    │
    ├──► MessageChannel (Rozdział 24)
    │         Bezpieczniejszy kanał point-to-point przez MessagePort
    │
    ├──► Web Workers (Rozdział 20)
    │         Workers komunikują się przez postMessage/onmessage
    │
    ├──► CORS (Rozdział 13)
    │         CORS dla HTTP, postMessage dla window-to-window komunikacji
    │
    └──► iframe Security
              postMessage głównym mechanizmem parent↔iframe komunikacji
```
