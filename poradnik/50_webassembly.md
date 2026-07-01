# Rozdział 50: WebAssembly

## 1. Czym jest WebAssembly

**WebAssembly (WASM)** to niskopoziomowy format binarny przeznaczony do uruchamiania w przeglądarkach i środowiskach serwerowych (Node.js, Deno, Wasmtime). Pozwala na uruchamianie kodu napisanego w językach kompilowanych (C, C++, Rust, Go, AssemblyScript) w przeglądarce — z wydajnością zbliżoną do kodu natywnego, w izolowanej przestrzeni (sandbox).

Kluczowe właściwości:
- **Binarny format** — kompaktowy, szybki do parsowania (nie tekstowe JS)
- **Silny sandbox** — brak bezpośredniego dostępu do DOM, OS, sieci (tylko przez JavaScript "glue")
- **Liniowa pamięć** — dostęp tylko do własnej liniowej przestrzeni adresowej
- **Stack-based VM** — deterministyczne wykonanie

```javascript
// Ładowanie i wykonanie modułu WASM:
const response = await fetch('/module.wasm');
const buffer = await response.arrayBuffer();
const module = await WebAssembly.compile(buffer);
const instance = await WebAssembly.instantiate(module, importObject);

// Wywołanie eksportowanej funkcji WASM:
const result = instance.exports.add(10, 20);
console.log(result); // 30
```

---

## 2. Dlaczego powstał

### Problem: JavaScript nie jest wystarczająco szybki dla CPU-intensive tasks

Przed WebAssembly (W3C MVP 2017):
- Gry 3D, video codecs, kryptografia, kompresja, machine learning → zbyt wolne w JS
- asm.js (Mozilla 2013): podzbiór JS z deklaracjami typów → AOT compilation → 50% prędkości native
- Potrzeba prawdziwego formatu binarnego z kontrolą typów

WebAssembly rozwiązuje:
1. **Wydajność**: zbliżona do C/C++ (bez JIT overhead)
2. **Przenośność**: jeden moduł działa w Chrome, Firefox, Safari, Node.js
3. **Bezpieczeństwo**: sandbox — nie ma bezpośredniego dostępu do zasobów OS
4. **Wielojęzyczność**: Rust, C++, Go, C# i inne → WASM

---

## 3. Jak działa

### Format modułu

```
WebAssembly moduł binarny składa się z sekcji:
- Type section: definicje typów funkcji
- Import section: importy z JavaScript (funkcje, pamięć, tabele)
- Function section: deklaracje funkcji WASM
- Memory section: konfiguracja liniowej pamięci
- Global section: zmienne globalne
- Export section: eksporty do JavaScript
- Code section: bytecode funkcji
```

### Liniowa pamięć

```javascript
// Pamięć WASM: liniowy bufor bajtów
const memory = new WebAssembly.Memory({ initial: 1 }); // 1 page = 64KB

// WASM może czytać/pisać TYLKO w tej pamięci
// JavaScript może też czytać/pisać (przez TypedArray):
const buffer = new Uint8Array(memory.buffer);
buffer[0] = 42;

// Wywołanie WASM z pamięcią:
const { instance } = await WebAssembly.instantiateStreaming(
    fetch('/module.wasm'),
    { js: { memory } }  // importObject
);
```

### Import/Export model

```javascript
// WASM może importować funkcje z JavaScript:
const importObject = {
    env: {
        // WASM może wywołać te funkcje:
        console_log: (value) => console.log(value),
        memory: new WebAssembly.Memory({ initial: 10 }),
        table: new WebAssembly.Table({ initial: 0, element: 'anyfunc' }),
    }
};

const { instance } = await WebAssembly.instantiate(wasmBytes, importObject);

// JavaScript wywołuje WASM:
instance.exports.myFunction(42);
```

### Typy WebAssembly

```
Podstawowe typy WASM:
i32, i64  — 32 i 64-bitowe liczby całkowite
f32, f64  — 32 i 64-bitowe liczby zmiennoprzecinkowe
v128      — 128-bitowy wektor (SIMD, wymaga crossOriginIsolated)
funcref   — referencja do funkcji
externref — zewnętrzna wartość (np. JS obiekt)
```

---

## 4. Co dzieje się wewnętrznie

### Sandbox model WASM

```
WebAssembly security model:
1. Memory safety: tylko liniowa pamięć — brak dostępu do adresów spoza niej
   → Buffer overflow ograniczony do liniowej pamięci WASM (nie system memory)

2. Type safety: silnie typowany — brak rzutowania między typami jak w C
   → Bezpieczniejszy niż surowy C/C++ (ale: nie wszystkie błędy wyeliminowane!)

3. Control flow integrity: call stack jest wirtualny, nie native OS stack
   → Stack overflow w WASM → trap, nie crashowanie OS

4. Brak dostępu do:
   - Sieci (tylko przez JS imports: fetch)
   - DOM (tylko przez JS callbacks)
   - OS (tylko przez WASI w środowiskach poza przeglądarką)
   - Kryptografii (tylko przez JS import lub WebCrypto)

5. ALE: kod JavaScript "glue" może być podatny!
   WASM sam jest bezpieczny ale JS wrapper może mieć luki
```

### Wielowątkowość i SharedArrayBuffer

```javascript
// WASM Threads wymaga crossOriginIsolated (COOP + COEP):
if (!crossOriginIsolated) {
    throw new Error('WASM threads wymagają cross-origin isolation');
}

const sharedMemory = new WebAssembly.Memory({
    initial: 10,
    maximum: 100,
    shared: true,  // SharedArrayBuffer pod spodem!
});

// Worker:
const worker = new Worker('worker.js');
worker.postMessage({ memory: sharedMemory }, [sharedMemory.buffer]);
// Teraz WASM może używać Atomics do synchronizacji między threadami
```

### WASM → JavaScript call overhead

```
Każde wywołanie przez granicę WASM↔JS ma overhead (boundary crossing):
- Konwersja typów (i32 ↔ Number)
- Context switching
- Sprawdzenie bezpieczeństwa

Dlatego: batch operacje zamiast wielu małych wywołań
WASM jest najszybszy dla pure computation
```

---

## 5. Analogiczny przykład z życia

WebAssembly to **certyfikowany program w elektronicznym sandbox**:

- Program (WASM moduł) jest zweryfikowany przy ładowaniu (walidacja bytecode)
- Działa w zamkniętej skrzynce (sandbox VM) — nie ma bezpośredniego dostępu do systemu
- Gdy potrzebuje zasobów zewnętrznych (np. wypisać na ekran): prosi strażnika (JS import) który wykonuje operację w jego imieniu
- Strażnik (JS glue code) to jedyna droga do świata zewnętrznego
- Sandbox gwarantuje że program nie może "uciec" poza przydzieloną mu pamięć (liniową)

---

## 6. Przykład kodu

```javascript
// === Bezpieczne ładowanie i weryfikacja WASM ===

async function loadWasmModule(url, importObject = {}) {
    // Sprawdź wsparcie:
    if (!WebAssembly) {
        throw new Error('WebAssembly niedostępne');
    }
    
    // UWAGA: jeśli moduł pochodzi z CDN — użyj SRI!
    // <script type="module"> z integrity dla skryptów ładujących WASM
    
    try {
        // instantiateStreaming jest preferowane (szybsze):
        const { instance, module } = await WebAssembly.instantiateStreaming(
            fetch(url, {
                // Zawsze z credentials domyślnymi lub jawnie:
                credentials: 'same-origin',
            }),
            importObject
        );
        
        return instance;
        
    } catch (err) {
        // Fallback dla serwerów bez poprawnego Content-Type:
        if (err instanceof TypeError) {
            const response = await fetch(url);
            const bytes = await response.arrayBuffer();
            const { instance } = await WebAssembly.instantiate(bytes, importObject);
            return instance;
        }
        throw err;
    }
}

// Użycie:
const wasmInstance = await loadWasmModule('/crypto.wasm', {
    env: {
        memory: new WebAssembly.Memory({ initial: 10 }),
        // Tylko bezpieczne imports!
    }
});
```

```javascript
// === WASM z izolowaną pamięcią — bezpieczny dostęp ===

class WasmSandbox {
    constructor(instance, memoryExport) {
        this.instance = instance;
        this.memory = memoryExport;
    }
    
    // Bezpieczny odczyt stringa z WASM memory:
    readString(ptr, length) {
        const view = new Uint8Array(this.memory.buffer, ptr, length);
        return new TextDecoder().decode(view);
    }
    
    // Bezpieczny zapis stringa do WASM memory:
    writeString(str, ptr) {
        const encoded = new TextEncoder().encode(str);
        const view = new Uint8Array(this.memory.buffer);
        
        // UWAGA: sprawdź bounds! Zapis poza buforem → błąd
        if (ptr + encoded.length > this.memory.buffer.byteLength) {
            throw new RangeError('WASM memory write out of bounds');
        }
        
        view.set(encoded, ptr);
    }
    
    // Wywołanie WASM funkcji z error handling:
    call(funcName, ...args) {
        const func = this.instance.exports[funcName];
        if (!func) throw new Error(`WASM export not found: ${funcName}`);
        
        try {
            return func(...args);
        } catch (err) {
            // WASM trap (unreachable, out of bounds itp.):
            if (err instanceof WebAssembly.RuntimeError) {
                console.error('WASM trap:', err.message);
                throw new Error('WASM runtime error: ' + err.message);
            }
            throw err;
        }
    }
}
```

```c
// === C kod kompilowany do WASM (dla kontekstu) ===

// example.c:
#include <stdint.h>
#include <string.h>

// Eksportowana funkcja:
__attribute__((used))
int32_t safe_add(int32_t a, int32_t b) {
    // WASM i32 overflow: zachowanie jak modular arithmetic (wrapping)
    return a + b;
}

// Praca na pamięci:
__attribute__((used))
void copy_data(uint8_t* dst, const uint8_t* src, size_t len) {
    // Kompilacja do WASM: wskaźniki są offsetami w liniowej pamięci
    memcpy(dst, src, len);
}

// Kompilacja:
// emcc example.c -o example.wasm -s EXPORTED_FUNCTIONS='["_safe_add","_copy_data"]'
```

---

## 7. Przykład z prawdziwej aplikacji

### Malvertising przez WASM

CVE-2019 i inne przypadki: złośliwe skrypty reklamowe używały WASM do:

```javascript
// Malicious ad script → loader → WASM cryptominer:
const wasmBytes = atob('AGFzbQEAAAABBwFgAn9/AX8DAgEABwgBBGFkZAAAA...');
// (zakodowany skompresowany WASM cryptominer)

const buffer = new Uint8Array(wasmBytes.length);
wasmBytes.split('').forEach((c, i) => buffer[i] = c.charCodeAt(0));

const module = await WebAssembly.compile(buffer.buffer);
const instance = await WebAssembly.instantiate(module, {
    env: { memory: new WebAssembly.Memory({ initial: 256 }) }
});

// Cryptomining w tle:
function mine() {
    instance.exports.mine_block(/* params */);
    requestAnimationFrame(mine);
}
mine();
```

Detekcja i ochrona:
- CSP: `script-src 'wasm-unsafe-eval'` — kontrola WASM evaluation
- Performance monitoring: wysoki CPU = możliwy cryptominer
- WASM moduły mogą być blokowane przez CSP

### WASM dla kryptografii w przeglądarce

```javascript
// Libsodium.js (używa WASM) — bezpieczna kryptografia:
const sodium = await SodiumPlus.auto();

// Szyfrowanie:
const key = await sodium.crypto_secretbox_keygen();
const nonce = await sodium.randombytes_buf(sodium.CRYPTO_SECRETBOX_NONCEBYTES);
const ciphertext = await sodium.crypto_secretbox(plaintext, nonce, key);

// WASM daje przewagę:
// - Szybkość: operacje kryptograficzne 10-100x szybsze niż pure JS
// - Stały czas: WASM bytecode ma bardziej deterministyczne timing (mniej timing attacks)
```

### Deobfuscation: WASM jako wektor ukrywania kodu

```javascript
// Atakujący używa WASM do ukrycia malicious logic:
const wasmInstance = await WebAssembly.instantiate(obfuscatedBytes);
const decodeKey = wasmInstance.exports.decode(0xDEADBEEF);
// Logika dekodera jest w WASM → trudna do analizy przez statyczne narzędzia

// Obrona: 
// - WASM można decompilować (wasm2wat, Ghidra, IDA)
// - CSP 'wasm-unsafe-eval' kontroluje dynamiczne WASM
// - Monitoruj sieć: duże binarne payload → możliwy WASM loader
```

---

## 8. Typowe błędy programistów

### Błąd 1: Trustowanie danych ze WASM jak trustowanemu JS

```javascript
// BŁĄD: WASM zwraca dane traktowane jako safe
const result = wasmInstance.exports.process_user_input(userInputPtr);
const output = readStringFromMemory(result);
document.getElementById('output').innerHTML = output;  // XSS!

// Nawet jeśli WASM jest bezpieczny, dane mogą zawierać HTML:
// Jeśli WASM przetwarza user input i zwraca HTML string → XSS przez innerHTML

// POPRAWKA: sanityzuj output WASM jak każdy inny niezaufany input:
document.getElementById('output').textContent = output;  // bezpieczne
// lub: DOMPurify.sanitize(output) jeśli HTML jest wymagany
```

### Błąd 2: Brak weryfikacji Content-Type dla WASM

```javascript
// BŁĄD: instantiateStreaming bez sprawdzenia Content-Type
await WebAssembly.instantiateStreaming(fetch(url));
// Serwer może zwrócić cokolwiek z niepoprawnym Content-Type → error lub podatność

// Serwer MUSI zwracać: Content-Type: application/wasm
// Przeglądarka blokuje WASM z niepoprawnym Content-Type (modern browsers)
// ALE: starsze przeglądarki mogą go zaakceptować

// POPRAWKA: sprawdź Content-Type jeśli potrzebny fallback:
const response = await fetch(url);
if (response.headers.get('Content-Type') !== 'application/wasm') {
    throw new Error('Invalid Content-Type for WASM');
}
```

### Błąd 3: CSP bez 'wasm-unsafe-eval'

```http
# CSP blokuje WASM compilation domyślnie od Chrome 97!
Content-Security-Policy: script-src 'self' 'nonce-RANDOM'
# → WebAssembly.instantiate() jest blokowane przez unsafe-eval policy!

# POPRAWKA: dodaj 'wasm-unsafe-eval' jeśli używasz WASM:
Content-Security-Policy: script-src 'self' 'nonce-RANDOM' 'wasm-unsafe-eval'
# 'wasm-unsafe-eval' jest bezpieczniejszy niż 'unsafe-eval' (pozwala tylko WASM, nie eval())
```

### Błąd 4: Ładowanie WASM z CDN bez SRI

```javascript
// BŁĄD: ładowanie WASM z external CDN bez integrity check
const { instance } = await WebAssembly.instantiateStreaming(
    fetch('https://cdn.example.com/crypto.wasm')
    // BRAK integrity check!
);

// CDN może być skompromitowany → malicious WASM zamiast oczekiwanego!

// POPRAWKA: sprawdź hash WASM modułu po pobraniu:
async function loadWasmWithVerification(url, expectedHash) {
    const response = await fetch(url);
    const bytes = await response.arrayBuffer();
    
    // Weryfikacja hash:
    const hashBuffer = await crypto.subtle.digest('SHA-384', bytes);
    const hash = btoa(String.fromCharCode(...new Uint8Array(hashBuffer)));
    
    if (hash !== expectedHash) {
        throw new Error('WASM integrity check failed!');
    }
    
    return WebAssembly.instantiate(bytes, importObject);
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Pozytywne aspekty bezpieczeństwa WASM

```
1. Memory Safety (w granicach sandboxa):
   - Brak buffer overflow do zewnętrznej pamięci
   - Dostęp tylko do własnej liniowej pamięci
   - Trap zamiast undefined behavior

2. Type Safety:
   - Silne typowanie (nie C-style casting)
   - Brak type confusion przez implicit coercions

3. Control Flow Integrity:
   - Indirect calls sprawdzane przy runtime
   - Brak arbitrary jump poza zdefiniowane funkcje

4. Brak bezpośredniego IO:
   - Sieć, DOM, OS → tylko przez JS imports
   - Izolacja od systemu
```

### Negatywne aspekty / zagrożenia

```
1. Cryptomining:
   - WASM umożliwia wydajne crypto mining (Monero, itp.)
   - Trudniejszy do detekcji niż JS (bytecode = obfuscated)
   - Ochrona: CSP 'wasm-unsafe-eval', Permissions-Policy, monitoring CPU

2. Deobfuscation barrier:
   - Malware w WASM jest trudniejszy do analizy statycznej
   - Wymaga decompilacji (wasm2wat, Ghidra)
   - Obfuscated JS często ładuje WASM jako drugi etap

3. Timing attacks przez WASM:
   - WASM może mierzyć timing precyzyjnie (szczególnie z SAB)
   - Spectre: WASM + SAB + SharedArrayBuffer = wysokiej rozdzielczości timer
   - Ochrona: crossOriginIsolated dla SAB

4. JS Glue code podatności:
   - WASM sam jest bezpieczny ale JS wrapper może mieć XSS, SQL injection itp.
   - Dane z WASM nie są automatycznie "bezpieczne" do wstrzyknięcia w DOM

5. Supply chain WASM:
   - Biblioteki kompilowane do WASM z podatnym kodem C/C++
   - CVE w libpng, zlib itp. → WASM build też podatny
   - Aktualizuj biblioteki i rekompiluj!
```

### CSP i WASM

```http
# Kontrola WASM przez CSP:

# Blokuj całkowicie (jeśli nie używasz WASM):
Content-Security-Policy: default-src 'self'
# WebAssembly.compile/instantiate jest zablokowane przez default!

# Zezwól na WASM:
Content-Security-Policy: script-src 'self' 'wasm-unsafe-eval'
# 'wasm-unsafe-eval': tylko WASM compilation — NIE eval() ani Function()

# Najbardziej restrykcyjne z WASM:
Content-Security-Policy: script-src 'self' 'nonce-{RANDOM}' 'strict-dynamic' 'wasm-unsafe-eval'
```

### Powiązane CWE

- **CWE-703** — Improper Check or Handling of Exceptional Conditions (WASM traps)
- **CWE-119** — Improper Restriction of Operations within Memory Buffer (WASM memory access)
- **CWE-400** — Uncontrolled Resource Consumption (cryptomining przez WASM)
- **CWE-79** — XSS (output z WASM wstrzyknięty do DOM)

---

## 10. Jak identyfikować podczas pentestu

### Wykryj WASM na stronie

```javascript
// DevTools → Network → filtruj "wasm"
// Szukaj requestów do plików .wasm

// Programatycznie:
const resources = performance.getEntriesByType('resource')
    .filter(r => r.initiatorType === 'fetch' || r.name.endsWith('.wasm'));
console.table(resources.map(r => ({ url: r.name, size: r.encodedBodySize })));
```

```bash
# Burp Suite: wyszukaj .wasm w historii
# Lub intercept: sprawdź Content-Type: application/wasm

# Decompiluj WASM do WAT (WebAssembly Text Format):
wasm2wat module.wasm -o module.wat
# Czytaj code section, import/export sections
```

### Sprawdź CSP dla WASM

```bash
curl -I https://target.com | grep -i "content-security-policy"
# Szukaj 'wasm-unsafe-eval' lub 'unsafe-eval'
# Brak → WASM może być zablokowane lub nie używane
```

### Test cryptomining detection

```javascript
// Monitoring CPU przez WASM:
// Sprawdź czy istnieje pętla uruchamiana przez rAF lub setInterval:

// Podejrzane patterns w Network:
// - Duże (.wasm) pliki ~100KB-500KB bez widocznej funkcjonalności
// - fetch() do .wasm z dynamicznie generowanych URL
// - atob() na dużych stringach → możliwy encrypted WASM loader

// Sprawdź: wysoki CPU przy otwartej stronie = możliwy cryptominer
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy CSP zawiera 'wasm-unsafe-eval' jeśli WASM jest używane?
□ Czy WASM moduły z CDN mają SRI (hash weryfikację)?
□ Czy serwer zwraca Content-Type: application/wasm dla .wasm plików?
□ Czy output z WASM jest sanityzowany przed wstrzyknięciem do DOM?
□ Czy WASM wielowątkowy (SharedArrayBuffer) wymaga crossOriginIsolated?
□ Czy .wasm pliki są sprawdzane pod kątem złośliwego kodu (reverse engineering)?
□ Czy biblioteki C/C++ kompilowane do WASM są aktualne (brak CVE)?
□ Czy WASM może wykonywać cryptomining (monitoring CPU)?
□ Czy dynamiczny WASM (z fetch + atob) nie jest używany do ukrycia malware?
□ Czy błędy WASM (traps, RuntimeError) są poprawnie obsługiwane?
```

---

## 12. Jak się zabezpieczać

```http
# Kompletna konfiguracja bezpieczeństwa dla aplikacji z WASM:

# CSP:
Content-Security-Policy: 
    default-src 'none';
    script-src 'self' 'nonce-{RANDOM}' 'strict-dynamic' 'wasm-unsafe-eval';
    connect-src 'self' https://api.example.com;
    img-src 'self' data: https://cdn.example.com;

# Cross-origin isolation (dla WASM threads):
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp

# SRI na script który ładuje WASM:
# <script src="/wasm-loader.js" integrity="sha384-..." crossorigin="anonymous"></script>
```

```javascript
// === Bezpieczna fabryka WASM modułów z walidacją ===

const WASM_HASHES = {
    'crypto.wasm': 'sha384-EXPECTED_HASH_HERE',
    'image.wasm': 'sha384-EXPECTED_HASH_HERE_2',
};

async function loadVerifiedWasm(name, importObject) {
    const expectedHash = WASM_HASHES[name];
    if (!expectedHash) {
        throw new Error(`Nieznany moduł WASM: ${name}`);
    }
    
    const response = await fetch(`/wasm/${name}`);
    if (!response.ok) {
        throw new Error(`Nie można załadować ${name}: ${response.status}`);
    }
    
    const bytes = await response.arrayBuffer();
    
    // Weryfikacja integralności:
    const [algorithm, expected] = expectedHash.split('-');
    const hashBuffer = await crypto.subtle.digest(
        algorithm.toUpperCase(),
        bytes
    );
    const actual = btoa(String.fromCharCode(...new Uint8Array(hashBuffer)));
    
    if (actual !== expected) {
        throw new Error(`WASM integrity check FAILED dla ${name}!`);
    }
    
    const { instance } = await WebAssembly.instantiate(bytes, importObject);
    return instance;
}

// Wrapper sanityzujący output:
function sanitizeWasmOutput(wasmResult, context) {
    if (typeof wasmResult === 'string') {
        // Zawsze sanityzuj stringi z WASM jeśli pójdą do DOM:
        return DOMPurify.sanitize(wasmResult);
    }
    return wasmResult;
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. WebAssembly = niskopoziomowy binarny format uruchamiany w sandboxie przeglądarki
2. Dostęp do DOM/sieci/OS tylko przez JavaScript imports — pełna izolacja
3. Zagrożenia: cryptomining (wydajny WASM), ukrywanie malware (trudniejszy do analizy), XSS przez niesanityzowany output
4. CSP: `'wasm-unsafe-eval'` wymagane dla WASM; lepsze niż `'unsafe-eval'`
5. WASM Threads wymaga `crossOriginIsolated = true` (COOP + COEP)

**Dla pentestera:**
- Szukaj `.wasm` plików w Network tab — co ładują? Skąd?
- Sprawdź CSP: czy `'wasm-unsafe-eval'` jest obecne?
- Wykryj cryptominer: monitoring CPU przy otwartej stronie
- Decompiluj WASM: `wasm2wat module.wasm` → analiza importów i logiki
- Sprawdź czy output WASM jest sanityzowany przed wstrzyknięciem do DOM
- WASM z CDN bez SRI = supply chain risk

---

## Powiązania

```
WebAssembly
    │
    ├──► Web Workers (Rozdział 20)
    │         WASM często uruchamiany w Workers (nie blokuje main thread)
    │         SharedArrayBuffer dla WASM threads wymaga crossOriginIsolated
    │
    ├──► COOP + COEP (Rozdziały 42-43)
    │         Wymagane dla crossOriginIsolated → WASM threads + SAB
    │
    ├──► CSP (Rozdział 36)
    │         'wasm-unsafe-eval': kontrola WASM compilation
    │         Bez CSP: WASM może być dynamicznie ładowane (cryptominer risk)
    │
    └──► SRI (Rozdział 38)
              Weryfikacja integralności .wasm plików z CDN
              Krytyczne: supply chain attacks przez podmieniony WASM
```
