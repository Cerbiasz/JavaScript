# Rozdział 37: Trusted Types

## 1. Czym są Trusted Types

**Trusted Types** to API przeglądarki i mechanizm CSP umożliwiający egzekwowanie sanityzacji HTML/URL/Script w punktach wstrzyknięcia do DOM (tzw. **injection sinks**). Zamiast polegać na tym że deweloper pamięta o sanityzacji, Trusted Types wymusza aby do `innerHTML`, `document.write()`, `eval()` i innych niebezpiecznych sinks mogły trafić tylko **zaufane** wartości — instancje specjalnych typów stworzonych przez zarejestrowane polityki.

Trusted Types rozwiązują **fundamentalny problem XSS**: nie blokują wstrzyknięcia string, ale wymagają aby string był przepuszczony przez zarejestrowaną politykę (gdzie sanityzacja może być wymuszona).

Składowe:
- **Trusted Types Policy** — obiekt z metodami `createHTML()`, `createScript()`, `createScriptURL()`
- **TrustedHTML** — opakowany, "zaufany" HTML string
- **TrustedScript** — opakowany, "zaufany" JavaScript string
- **TrustedScriptURL** — opakowany, "zaufany" URL dla skryptów
- **CSP dyrektywa** — `require-trusted-types-for 'script'`

---

## 2. Dlaczego powstał

### Problem: XSS przez DOM sinks

Pomimo CSP, `DOMPurify`, code review — XSS nadal się pojawiają bo:
- Deweloper zapomina sanityzować
- Sanityzacja jest w złym miejscu (za wcześnie lub za późno)
- Biblioteki mają własne sinks które omijają sanityzację

Trusted Types (Google, 2018, Chrome 83+) zmienia model:
- Zamiast "sanityzuj w momencie otrzymania danych" → "wymuś sanityzację w momencie użycia"
- Przeglądarka blokuje typ `string` w niebezpiecznych sinks
- Tylko `TrustedHTML`/`TrustedScript`/`TrustedScriptURL` obiekty są dozwolone
- Wszystkie "drogi" do sinks muszą przechodzić przez rejestrowane polityki

---

## 3. Jak działa

### CSP dyrektywa włączająca Trusted Types

```http
Content-Security-Policy: require-trusted-types-for 'script'
```

Po włączeniu — jakikolwiek `string` przypisany do sink rzuca `TypeError`:

```javascript
// Po włączeniu require-trusted-types-for 'script':
document.body.innerHTML = '<p>Hello</p>';
// TypeError: This document requires 'TrustedHTML' assignment.
```

### Tworzenie polityki

```javascript
// Zarejestruj politykę
const policy = trustedTypes.createPolicy('my-policy', {
    createHTML: (input) => {
        // Sanityzacja — TUTAJ jest miejsce na DOMPurify lub inny sanitizer
        return DOMPurify.sanitize(input, {
            ALLOWED_TAGS: ['b', 'i', 'strong', 'em', 'p', 'br', 'a'],
            ALLOWED_ATTR: ['href', 'class']
        });
    },
    createScript: (input) => {
        // Rzadko używane — zwykle NIE pozwalamy na tworzenie skryptów
        throw new Error('Script creation not allowed');
    },
    createScriptURL: (input) => {
        // Whitelist URL dla skryptów
        const url = new URL(input, location.origin);
        if (url.origin === location.origin) return url.href;
        throw new Error('Only same-origin scripts allowed');
    }
});

// Użycie polityki:
const trustedHtml = policy.createHTML('<p>User <b>content</b></p>');
document.body.innerHTML = trustedHtml; // OK! trustedHtml jest TrustedHTML

// Tworzenie URL dla skryptu:
const trustedUrl = policy.createScriptURL('/scripts/module.js');
const script = document.createElement('script');
script.src = trustedUrl; // OK!
```

### Domyślna polityka (fallback)

```javascript
// Specjalna polityka 'default' przechwytuje wszystkie string-to-sink konwersje:
trustedTypes.createPolicy('default', {
    createHTML: (input) => {
        console.warn('Unchecked HTML assignment:', input);
        return DOMPurify.sanitize(input); // sanityzuj wszystko
    }
});

// Teraz: element.innerHTML = 'string'; → przechodzi przez default policy!
// Dobra strategia migracji starszych aplikacji
```

### Sprawdzanie obsługi

```javascript
if (window.trustedTypes && trustedTypes.createPolicy) {
    // Trusted Types są obsługiwane
    const policy = trustedTypes.createPolicy('my-policy', { createHTML: sanitize });
} else {
    // Fallback — manualnie sanityzuj
    // Zalecaj aktualizację przeglądarki
}
```

---

## 4. Co dzieje się wewnętrznie

### Lista chronionych sinks

Trusted Types chroni następujące sinks (wymagają TrustedHTML):
```
innerHTML, outerHTML, insertAdjacentHTML()
document.write(), document.writeln()
DOMParser.parseFromString()
Range.createContextualFragment()
Element.setHTML()

Sinks wymagające TrustedScript:
eval(), Function()
setTimeout(string), setInterval(string)
new Worker(string)

Sinks wymagające TrustedScriptURL:
<script src>
Worker constructor (URL)
importScripts()
```

### Integracja z DOMPurify

DOMPurify obsługuje Trusted Types natywnie:

```javascript
// DOMPurify zwraca TrustedHTML jeśli Trusted Types są aktywne:
const policy = trustedTypes.createPolicy('dompurify', {
    createHTML: (html) => DOMPurify.sanitize(html, { RETURN_TRUSTED_TYPE: true })
});

// Lub: DOMPurify automatycznie wykrywa i zwraca TrustedHTML:
element.innerHTML = DOMPurify.sanitize(dirtyHtml); // zwraca TrustedHTML jeśli TT aktywne
```

### Raportowanie naruszeń

Podobnie do CSP, Trusted Types wspiera report-only mode:

```http
Content-Security-Policy-Report-Only: require-trusted-types-for 'script'; report-to tt-endpoint
```

---

## 5. Analogiczny przykład z życia

Trusted Types to procedura podpisywania dokumentów:

- Każdy dokument wchodzący do szafy (DOM sink) musi być podpisany przez notariusza (Trusted Types Policy)
- Zwykłe kartki papieru (stringi) są odrzucane
- Notariusz przed podpisaniem sprawdza czy dokument jest zgodny z prawem (sanityzacja)
- Stworzenie nowego notariusza jest możliwe ale wszystkich ich działania są audytowalne

---

## 6. Przykład kodu

```javascript
// === Kompletna integracja Trusted Types w aplikacji React-like ===

// 1. Inicjalizacja polityki
const htmlPolicy = trustedTypes.createPolicy('app-html', {
    createHTML: (input, context) => {
        // Różna sanityzacja zależnie od kontekstu
        if (context === 'user-content') {
            return DOMPurify.sanitize(input, {
                ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
                ALLOWED_ATTR: ['href', 'target', 'rel']
            });
        }
        if (context === 'admin-content') {
            return DOMPurify.sanitize(input); // mniej restrykcyjne dla admina
        }
        // Domyślnie: stripuj wszystko
        return DOMPurify.sanitize(input, { ALLOWED_TAGS: [] });
    }
});

const scriptUrlPolicy = trustedTypes.createPolicy('app-script', {
    createScriptURL: (url) => {
        const ALLOWED_ORIGINS = ['https://cdn.example.com', location.origin];
        const parsed = new URL(url, location.origin);
        if (ALLOWED_ORIGINS.includes(parsed.origin)) return url;
        throw new Error(`Script URL not allowed: ${url}`);
    }
});

// 2. Użycie w kodzie
function renderUserContent(element, userHtml) {
    // Zamiast: element.innerHTML = userHtml;
    element.innerHTML = htmlPolicy.createHTML(userHtml, 'user-content');
}

function loadScript(url) {
    const script = document.createElement('script');
    script.src = scriptUrlPolicy.createScriptURL(url);
    document.head.appendChild(script);
}

// 3. Migracja starszego kodu przez 'default' policy
// Gdy 'require-trusted-types-for' jest włączone a aplikacja ma legacy innerHTML:
trustedTypes.createPolicy('default', {
    createHTML: (input) => {
        console.warn('[TT] Unreviewed innerHTML:', input.substring(0, 100));
        // Loguj stack trace dla debugowania:
        console.trace('[TT] Stack:');
        return DOMPurify.sanitize(input);
    }
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Migracja do Trusted Types krok po kroku

```javascript
// Krok 1: Włącz Report-Only
// Content-Security-Policy-Report-Only: require-trusted-types-for 'script'; report-to tt

// Krok 2: Zbierz raporty — dowiedz się gdzie masz innerHTML itp.

// Krok 3: Stwórz 'default' policy do przechwycenia wszystkiego
trustedTypes.createPolicy('default', {
    createHTML: (html) => {
        if (process.env.NODE_ENV === 'development') {
            console.error('Unchecked innerHTML!', new Error().stack);
        }
        return DOMPurify.sanitize(html);
    }
});

// Krok 4: Stopniowo refaktoruj — zastąp stringi politykami
// Krok 5: Usuń 'default' policy — teraz każde użycie musi być jawne
```

### Bypass Trusted Types przez DOM Clobbering

```javascript
// Jeśli atakujący może wstrzyknąć HTML (np. przez atrybut), może próbować:
// DOM Clobbering na trustedTypes (trudne, ale teoretycznie możliwe w edge cases)

// Dlaczego to jest trudne: trustedTypes jest własnością window
// DOM Clobbering może nadpisać window properties przez named access
// ALE: Trusted Types obiekt jest non-configurable/non-writable (implementacja-zależna)

// Bezpieczniejsze: dostęp przez lokalną zmienną, nie globalną:
const { createPolicy } = trustedTypes; // zapisz przed potencjalnym clobbering
```

---

## 8. Typowe błędy programistów

### Błąd 1: Polityka bez sanityzacji

```javascript
// BŁĄD: polityka nie sanityzuje — po prostu "przepuszcza"
const policy = trustedTypes.createPolicy('my-policy', {
    createHTML: (html) => html, // BEZ sanityzacji!
});
// Trusted Types jest bezużyteczne — każdy string zostaje "zaufany"
// Daje fałszywe poczucie bezpieczeństwa!
```

### Błąd 2: Zbyt wiele polityk "for-old-code"

```javascript
// ANTYPATTERN: tworzysz politykę "bypass" dla każdego problematycznego miejsca
const legacyPolicy = trustedTypes.createPolicy('legacy-bypass', {
    createHTML: (html) => html // NIEBEZPIECZNE
});
// Używasz przy każdym innerHTML bez sanityzacji
// → Trusted Types nie chroni wcale
```

### Błąd 3: Nieobsłużony TypeError

```javascript
// BŁĄD: Trusted Types aktywne, brak fallback na starszych przeglądarkach
element.innerHTML = policy.createHTML(html); // TypeError na Chrome < 83, Firefox < 110

// POPRAWKA: sprawdzenie dostępności
const safeHtml = (typeof trustedTypes !== 'undefined') 
    ? policy.createHTML(html)
    : DOMPurify.sanitize(html); // fallback
element.innerHTML = safeHtml;
```

---

## 9. Znaczenie dla bezpieczeństwa

### Trusted Types vs DOMPurify alone

| | DOMPurify | Trusted Types + DOMPurify |
|-|-----------|--------------------------|
| Wymuszenie | Manualne (deweloper pamięta) | Automatyczne (przeglądarka wymusza) |
| Zapomniany sink | XSS! | TypeError (blokowanie) |
| Audytowalność | Trudna (rozrzucona po kodzie) | Łatwa (centralne polityki) |
| Legacy code | Wymagane zmiany wszędzie | 'default' policy jako safety net |

### Trusted Types jako "defense in depth"

CSP blokuje wykonanie niechcianych skryptów z zewnątrz. Trusted Types blokuje **wstrzyknięcie** do DOM. Razem:

```
Atakujący wstrzykuje string → Trusted Types blokuje przed DOM
                           → Jeśli ominie → CSP blokuje wykonanie
```

### Obsługa przeglądarek

- Chrome 83+ — pełne wsparcie
- Firefox 110+ — wsparcie za flagą, planowane domyślnie
- Safari — brak natywnego wsparcia (2024), polyfill możliwy

Dla Firefox i Safari — 'default' policy jako safety net lub polyfill.

### Powiązane CWE

- **CWE-79** — XSS (Trusted Types jako mitygacja)
- **CWE-116** — Improper Encoding or Escaping of Output

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź czy Trusted Types są aktywne

```javascript
// W konsoli przeglądarki:
typeof trustedTypes; // 'object' jeśli wspierane
trustedTypes.getPolicyNames(); // lista zarejestrowanych polityk
```

```bash
curl -I https://target.com | grep -i "trusted-types\|require-trusted-types"
```

### Testuj sink bezpośrednio

```javascript
// Jeśli Trusted Types aktywne:
document.body.innerHTML = 'test'; 
// → TypeError: This document requires 'TrustedHTML' assignment.

// Jeśli brak błędu → Trusted Types nieaktywne lub jest 'default' policy
```

### Analiza polityk

```javascript
// Lista zarejestrowanych polityk:
trustedTypes.getPolicyNames();
// ['app-html', 'default', 'dompurify']

// Jeśli 'default' policy istnieje i nie sanityzuje — słaba ochrona
// Jeśli polityki mają zbyt liberalne createHTML → słaba ochrona
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy CSP zawiera require-trusted-types-for 'script'?
□ Czy zarejestrowane polityki faktycznie sanityzują (nie przepuszczają)?
□ Czy 'default' policy jest używana jako bezpieczna sieć (sanityzująca)?
□ Czy obsługa przeglądarek bez Trusted Types jest bezpieczna (fallback)?
□ Czy polityki mają zbyt broad createHTML (przepuszczają wszystko)?
□ Czy raportowanie naruszeń Trusted Types jest skonfigurowane?
□ Czy biblioteki (React, Angular, Vue) mają integrację TT?
```

---

## 12. Jak się zabezpieczać

```http
# CSP z Trusted Types:
Content-Security-Policy:
    default-src 'none';
    script-src 'nonce-{random}' 'strict-dynamic';
    require-trusted-types-for 'script';
    trusted-types app-html dompurify;
    report-to tt-endpoint
```

```javascript
// Inicjalizacja na początku aplikacji (przed innymi skryptami):
if (window.trustedTypes) {
    trustedTypes.createPolicy('dompurify', {
        createHTML: (html) => DOMPurify.sanitize(html, {
            RETURN_TRUSTED_TYPE: true,
            ALLOWED_TAGS: ['b', 'i', 'strong', 'em', 'p', 'br', 'a', 'ul', 'li', 'span'],
            ALLOWED_ATTR: ['class', 'href', 'target', 'rel', 'aria-label']
        })
    });
    
    // Bezpieczna fallback dla legacy
    trustedTypes.createPolicy('default', {
        createHTML: (html) => DOMPurify.sanitize(html)
    });
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Trusted Types = przeglądarkowe wymuszenie sanityzacji w DOM sink points
2. CSP dyrektywa: `require-trusted-types-for 'script'` aktywuje ochronę
3. `trustedTypes.createPolicy()` definiuje gdzie i jak sanityzować
4. 'default' policy przechwytuje wszystkie niezabezpieczone sinks (dobry dla migracji)
5. Chrome 83+ wspiera natywnie; Firefox i Safari mają ograniczone wsparcie

**Dla pentestera:**
- `typeof trustedTypes !== 'undefined'` → sprawdź obsługę
- `trustedTypes.getPolicyNames()` → lista polityk
- `document.body.innerHTML = 'test'` → czy rzuca TypeError?
- Sprawdź czy polityki faktycznie sanityzują w `createHTML`

---

## Powiązania

```
Trusted Types
    │
    ├──► CSP (Rozdział 36)
    │         require-trusted-types-for 'script' w CSP
    │
    ├──► DOM/XSS (Rozdział 9)
    │         Trusted Types chroni sinks: innerHTML, eval, itp.
    │
    └──► DOMPurify
              Standardowa integracja — DOMPurify jako implementacja polityki
```
