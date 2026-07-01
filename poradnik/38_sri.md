# Rozdział 38: Subresource Integrity (SRI)

## 1. Czym jest Subresource Integrity

**Subresource Integrity (SRI)** to mechanizm bezpieczeństwa umożliwiający przeglądarce weryfikację czy zasób pobrany z zewnętrznego źródła (CDN, third-party) nie został zmodyfikowany. Polega na dołączeniu kryptograficznego hasha oczekiwanej zawartości do tagu `<script>` lub `<link>` — przeglądarka odrzuci zasób jeśli hash nie pasuje.

```html
<script 
    src="https://cdn.example.com/jquery.min.js"
    integrity="sha384-abc123xyz..."
    crossorigin="anonymous">
</script>
```

SRI chroni przed:
- **CDN Compromise** — atakujący włamuje się do CDN i podmienia pliki
- **Supply Chain Attack** — malicious npm package/script
- **MITM** — pośrednik podmieniający zasoby
- **Accidental modification** — niezamierzone zmiany na serwerze zewnętrznym

---

## 2. Dlaczego powstał

### Problem: zaufanie do zewnętrznych zasobów

Ładowanie bibliotek z CDN jest powszechne:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.7.0/jquery.min.js"></script>
```

Ale co jeśli:
1. CDN jest skompromitowane (cloudflare atak 2017?)
2. `cdnjs.cloudflare.com` jest podmienione przez DNS hijacking
3. Deweloper CDN wciśnie malicious update
4. MITM na połączeniu HTTP (bez HTTPS)

Bez SRI — przeglądarka bezkrytycznie ładuje i wykonuje plik. Jeden skompromitowany CDN → **XSS na wszystkich stronach** które go używają.

SRI (W3C, implementacja 2015) rozwiązuje to przez cryptographic verification.

---

## 3. Jak działa

### Generowanie hasha SRI

```bash
# SHA-384 (zalecany):
curl -s https://cdn.example.com/library.js | openssl dgst -sha384 -binary | openssl base64 -A
# → Wynik: ABC123xyz...

# Format integrity attribute:
# <algorytm>-<base64-hash>
# sha256-..., sha384-... (zalecany), sha512-...

# Możliwe wiele hashów (przeglądarka sprawdza dowolny):
integrity="sha384-hash1 sha512-hash2"
```

### HTML implementacja

```html
<!-- Skrypt z SRI: -->
<script 
    src="https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.min.js"
    integrity="sha384-ACTUAL_HASH_HERE"
    crossorigin="anonymous">
</script>

<!-- Style z SRI: -->
<link 
    rel="stylesheet" 
    href="https://cdn.example.com/bootstrap.min.css"
    integrity="sha384-ACTUAL_HASH_HERE"
    crossorigin="anonymous">

<!-- UWAGA: crossorigin="anonymous" jest wymagane dla zasobów cross-origin! -->
<!-- Bez tego: przeglądarka nie może porównać hasha dla resources bez CORS -->
```

### Zachowanie przy niezgodnym hashu

```
Przeglądarka pobiera zasób → oblicza hash → porównuje z integrity

Jeśli PASUJE:
    → Zasób załadowany i wykonany

Jeśli NIE PASUJE:
    → Zasób odrzucony
    → Błąd w konsoli: "Failed to find a valid digest in the 'integrity' attribute for resource"
    → Skrypt NIE jest wykonywany
    → Błąd jest raportowany (jeśli CSP report-uri skonfigurowane)
```

### SRI w import maps (nowoczesne)

```html
<!-- Import Maps z SRI (eksperymentalne, Chrome 111+): -->
<script type="importmap">
{
    "imports": {
        "vue": "https://cdn.example.com/vue.esm-browser.js"
    },
    "integrity": {
        "https://cdn.example.com/vue.esm-browser.js": "sha384-hash"
    }
}
</script>
```

---

## 4. Co dzieje się wewnętrznie

### Algorytm weryfikacji

1. Przeglądarka parsuje `integrity` attribute → listę `<algorithm>-<hash>` par
2. Pobiera zasób
3. Oblicza hash używając każdego algorytmu z listy
4. Jeśli choć jeden hash się zgadza → zasób jest akceptowany
5. Jeśli żaden nie pasuje → zasób odrzucony (TypeError network error)

### crossorigin="anonymous" requirement

SRI dla cross-origin zasobów wymaga `crossorigin="anonymous"` (lub `"use-credentials"`). Bez tego przeglądarka korzysta z "tainted" mode — nie ma dostępu do danych zasobu aby obliczyć hash (CORS restriction).

Dla same-origin zasobów `crossorigin` nie jest wymagane.

### Dynamiczne zasoby i SRI

SRI nie działa dla dynamicznie ładowanych zasobów przez JavaScript (bez specjalnego podejścia):

```javascript
// SRI NIE działa automatycznie:
const script = document.createElement('script');
script.src = 'https://cdn.example.com/lib.js';
document.head.appendChild(script); // brak weryfikacji SRI!

// Możliwe przez Fetch API z integrity option:
const response = await fetch('https://cdn.example.com/lib.js', {
    integrity: 'sha384-expected-hash'
});
// fetch zweryfikuje hash przed zwróceniem response
```

---

## 5. Analogiczny przykład z życia

SRI to pieczęć lakowa na liście dyplomatycznym:

- Wysyłasz list przez posłańca (CDN) z pieczęcią (hash)
- Odbiorca przed otwarciem sprawdza czy pieczęć jest nienaruszona (weryfikacja hasha)
- Jeśli pieczęć jest złamana lub podmieniona → list odrzucony (nie załadowany)
- Poseł nie może zmienić treści listu bez zerwania pieczęci

---

## 6. Przykład kodu

```bash
#!/bin/bash
# === Skrypt do generowania SRI hashów ===

generate_sri() {
    local url="$1"
    local algorithm="${2:-sha384}"
    
    hash=$(curl -sL "$url" | openssl dgst -"$algorithm" -binary | openssl base64 -A)
    echo "${algorithm}-${hash}"
}

# Użycie:
SRI=$(generate_sri "https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js")
echo "integrity=\"$SRI\""
# integrity="sha384-abc123..."
```

```javascript
// === Node.js: generowanie SRI w build pipeline ===
const crypto = require('crypto');
const fs = require('fs');

function generateSRI(filePath, algorithm = 'sha384') {
    const content = fs.readFileSync(filePath);
    const hash = crypto.createHash(algorithm).update(content).digest('base64');
    return `${algorithm}-${hash}`;
}

// W build script (webpack plugin):
class SRIPlugin {
    apply(compiler) {
        compiler.hooks.emit.tapAsync('SRIPlugin', (compilation, callback) => {
            const assets = Object.keys(compilation.assets);
            const manifest = {};
            
            assets.forEach(asset => {
                const content = compilation.assets[asset].source();
                const hash = crypto.createHash('sha384')
                    .update(typeof content === 'string' ? content : Buffer.from(content))
                    .digest('base64');
                manifest[asset] = `sha384-${hash}`;
            });
            
            // Zapisz manifest do pliku
            fs.writeFileSync('public/sri-manifest.json', JSON.stringify(manifest, null, 2));
            callback();
        });
    }
}
```

```html
<!-- Przykładowy HTML z SRI dla popularnych bibliotek: -->
<!DOCTYPE html>
<html>
<head>
    <!-- Bootstrap CSS -->
    <link rel="stylesheet" 
          href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css"
          integrity="sha384-ACTUAL_HASH" 
          crossorigin="anonymous">
</head>
<body>
    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"
            integrity="sha384-ACTUAL_HASH"
            crossorigin="anonymous">
    </script>
    
    <!-- Font Awesome -->
    <link rel="stylesheet"
          href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css"
          integrity="sha512-ACTUAL_HASH"
          crossorigin="anonymous">
</body>
</html>
```

---

## 7. Przykład z prawdziwej aplikacji

### Supply Chain Attack — event-stream (2018)

Jeden z najgłośniejszych supply chain ataków:
1. Popularny npm pakiet `event-stream` (2M pobierań/tydzień) był utrzymywany przez jedną osobę
2. Twórca przekazał ownership nieznanemu maintainerowi
3. Nowy maintainer dodał malicious dependency
4. Pakiet przez łańcuch zależności trafił do `copay` (Bitcoin wallet)
5. Malicious kod kradł Bitcoin z portfeli

SRI mogłoby to ograniczyć — gdyby hash bundla był zweryfikowany, podmieniona wersja zostałaby odrzucona.

### Cloudflare CDNJS Vulnerability (2021)

Badacze odkryli że przez CDNJS możliwe było uruchomienie dowolnego kodu który załadują wszystkie strony używające CDNJS. SRI broniłoby strony korzystające z SRI — plik CDN jest inny → hash nie pasuje → odrzucony.

---

## 8. Typowe błędy programistów

### Błąd 1: Brak SRI dla zewnętrznych zasobów

```html
<!-- BŁĄD: brak SRI dla CDN -->
<script src="https://cdn.example.com/library.js"></script>
<!-- Nie ma weryfikacji → CDN może służyć malicious code -->

<!-- POPRAWKA: -->
<script src="https://cdn.example.com/library.js"
        integrity="sha384-..."
        crossorigin="anonymous"></script>
```

### Błąd 2: Używanie SHA-1 (deprecated)

```html
<!-- BŁĄD: SHA-1 jest kryptograficznie złamany -->
<script src="..." integrity="sha1-weakHash"></script>

<!-- POPRAWKA: SHA-384 lub SHA-512 -->
<script src="..." integrity="sha384-strongHash"></script>
```

### Błąd 3: Hash statyczny dla dynamicznie aktualizowanego pliku

```html
<!-- BŁĄD: wskazujesz na "latest" ale hash jest dla konkretnej wersji -->
<script src="https://cdn.example.com/library@latest/dist/lib.js"
        integrity="sha384-hash-of-old-version">
<!-- Przy aktualizacji CDN → nowa wersja ma inny hash → ładowanie FAIL -->

<!-- POPRAWKA: pinuj konkretną wersję -->
<script src="https://cdn.example.com/library@1.2.3/dist/lib.js"
        integrity="sha384-hash-of-1.2.3">
```

### Błąd 4: Brak crossorigin dla cross-origin zasobów

```html
<!-- BŁĄD: bez crossorigin SRI nie działa dla CDN zasobów -->
<script src="https://external.cdn.com/lib.js" integrity="sha384-...">
<!-- Może fail bez crossorigin -->

<!-- POPRAWKA: -->
<script src="https://external.cdn.com/lib.js" 
        integrity="sha384-..." 
        crossorigin="anonymous">
```

---

## 9. Znaczenie dla bezpieczeństwa

### SRI vs HTTPS

HTTPS zapewnia poufność i integralność w transporcie. SRI zapewnia integralność **zawartości** niezależnie od transportu:

| Atak | HTTPS | SRI |
|------|-------|-----|
| MITM podmiana | Ochrona | Ochrona (fallback) |
| CDN compromise | Brak ochrony | Ochrona |
| DNS hijacking + fake CDN | Częściowa | Ochrona |
| Malicious update od CDN | Brak ochrony | Ochrona |

### Kompatybilność z CSP

CSP `require-sri-for` (eksperymentalne) wymaga SRI dla wszystkich skryptów/styli:
```http
Content-Security-Policy: require-sri-for script style
```

### SRI a Service Workers

Service Worker może cache'ować zasoby. Jeśli SW cache'uje zasób **po** weryfikacji SRI → cache jest bezpieczny. Ale jeśli SW serwuje zasób **bez** veryfikacji SRI → możliwe cached poisoning.

### Powiązane CWE

- **CWE-494** — Download of Code Without Integrity Check
- **CWE-829** — Inclusion of Functionality from Untrusted Control Sphere

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź czy SRI jest używane

```bash
# Szukaj tagów script/link bez integrity:
curl -s https://target.com | grep -E '<(script|link)[^>]+src=' | grep -v 'integrity'
```

### DevTools — Network Tab

```
Network → filtruj po "script" / "stylesheet"
Sprawdź requestów do zewnętrznych źródeł (CDN)
Brak integrity na zewnętrznych zasobach = brak SRI
```

### Console — sprawdzenie załadowanych zasobów

```javascript
// Lista zewnętrznych skryptów bez SRI:
Array.from(document.querySelectorAll('script[src]'))
    .filter(s => !s.integrity && !s.src.startsWith(location.origin))
    .map(s => ({ src: s.src, hasSRI: false }));
```

### Co szukać

- Biblioteki JavaScript z CDN bez `integrity`
- Style z CDN bez `integrity`
- Wersja "latest" lub "current" (zmienna) bez SRI
- Font Awesome, Bootstrap, jQuery z CDN bez SRI

---

## 11. Jak testować bezpieczeństwo

```
□ Czy wszystkie zewnętrzne skrypty mają integrity attribute?
□ Czy wszystkie zewnętrzne style mają integrity attribute?
□ Czy używany jest SHA-384 lub SHA-512 (nie SHA-1)?
□ Czy crossorigin="anonymous" jest ustawione na cross-origin zasobach?
□ Czy wersja zasobu jest pinowana (nie @latest)?
□ Czy hasha są weryfikowane w build pipeline?
□ Czy SRI hash jest aktualny po aktualizacji biblioteki?
□ Czy fetch() z integrity weryfikuje dynamicznie ładowane zasoby?
□ Czy Service Worker weryfikuje SRI przy cache'owaniu?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Automatyczne generowanie SRI w webpack/vite ===
// @mdn/browser-compat-data
// vite-plugin-sri

// vite.config.js:
import { defineConfig } from 'vite';
import { VitePluginSri } from 'vite-plugin-sri';

export default defineConfig({
    plugins: [VitePluginSri()],
    build: {
        // Automatycznie generuje i dodaje integrity do HTML
    }
});
```

```html
<!-- Aktualne hasha dla popularnych bibliotek (przykładowe, sprawdź aktualne!): -->
<!-- jQuery 3.7.1: -->
<script src="https://code.jquery.com/jquery-3.7.1.min.js"
        integrity="sha256-/JqT3SQfawRcv/BIHPThkBvs0OEvtFFmqPF/lYI/Cxo="
        crossorigin="anonymous"></script>

<!-- React 18: -->
<script src="https://unpkg.com/react@18/umd/react.production.min.js"
        integrity="sha384-CURRENT_HASH"
        crossorigin="anonymous"></script>
```

```bash
# Narzędzia do generowania SRI:
# Online: https://www.srihash.org/
# CLI: openssl dgst -sha384 -binary file.js | openssl base64 -A
# npm: sri-toolbox, webpack-subresource-integrity
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. SRI = kryptograficzna weryfikacja zasobów z CDN przed wykonaniem
2. Atrybut `integrity="sha384-..."` na `<script>` i `<link>` elementach
3. `crossorigin="anonymous"` wymagane dla cross-origin zasobów
4. Chroni przed CDN compromise, supply chain attacks, MITM
5. SHA-384 zalecany algorytm (SHA-1 deprecated, SHA-256 minimalne)

**Dla pentestera:**
- Szukaj `<script src="https://...">` bez `integrity` attribute
- Zewnętrzne zasoby bez SRI = potencjalny supply chain risk
- Sprawdź czy używane CDN mają historię incydentów bezpieczeństwa

---

## Powiązania

```
Subresource Integrity
    │
    ├──► CSP (Rozdział 36)
    │         require-sri-for w CSP
    │
    ├──► Web Components (Rozdział 33)
    │         Zewnętrzne Web Component biblioteki → SRI
    │
    ├──► Service Workers (Rozdział 19)
    │         Interakcja SW caching a SRI verification
    │
    └──► Fetch API (Rozdział 11)
              fetch() z integrity option dla dynamicznych zasobów
```
