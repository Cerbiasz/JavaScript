# Rozdział 48: Fenced Frames

## 1. Czym są Fenced Frames

**Fenced Frames** to nowy typ osadzania treści webowych (podobny do `<iframe>`) zaprojektowany z zasadą **privacy-by-design** — izoluje embeddowaną stronę od kontekstu strony nadrzędnej tak, że żadna ze stron nie może odczytać danych drugiej. Jest to fundamentalny element Google Privacy Sandbox, zaprojektowany by umożliwić targetowane reklamy i inne use cases wymagające cross-site danych **bez ujawniania tych danych** stronie embedującej.

```html
<!-- Fenced Frame element (eksperymentalne, Chrome 113+): -->
<fencedframe src="https://ads.example.com/ad" mode="opaque-ads"></fencedframe>
```

Kluczowe właściwości Fenced Frame:
- **Komunikacja w górę zablokowana**: fenced frame NIE może komunikować się z parent page (brak `window.parent.postMessage`)
- **Komunikacja w dół zablokowana**: parent NIE może wstrzykiwać JS do fenced frame
- **URL opaque**: parent nie zna URL embedowanej zawartości (jeśli `mode="opaque-ads"`)
- **Storage izolowany**: fenced frame ma własną partitioned storage
- **Niezależna sieć**: żądania z fenced frame nie niosą cookies parent page

---

## 2. Dlaczego powstał

### Problem: reklamy wymagają cross-site danych ale naruszają prywatność

Tradycyjny model reklamowy:
```
1. Advertiser (shop.com): user widział buty → chce retargetować
2. Ad network: udziela cross-site profilu przez third-party cookies
3. News.com: ładuje reklamę butów w iframe → ad iframe dostaje cookie z shop.com
4. Efekt: cross-site tracking → profilowanie → naruszenie prywatności

Z CHIPS/phase-out: reklamy tracą effective targeting → model biznesowy zagrożony
```

Fenced Frames + Privacy Sandbox APIs (Protected Audience, Topics):
```
1. Browser przechowuje interesy lokalnie (Topics) i bids (Protected Audience)
2. Ad auction dzieje się WEWNĄTRZ przeglądarki (Private Aggregation API)
3. Wynik aukcji: Opaque URL do reklamy → przekazany do fenced frame
4. Fenced Frame wyświetla reklamę ALE:
   - Parent (news.com) nie wie którą reklamę widzi user
   - Reklama nie wie na której stronie jest wyświetlana (context isolation!)
   - Brak komunikacji między parent a fenced frame
5. Konwersja raportowana przez Attribution Reporting API (bez cross-site ID)
```

---

## 3. Jak działa

### Tryby Fenced Frame

```javascript
// mode="opaque-ads":
// URL jest opakiem (parent nie może go odczytać)
// src jest zwracane przez selectURL() Protected Audience API
// Używane dla reklamowych use cases

// mode="default":
// URL jest widoczny (jak iframe)
// Ale nadal pełna izolacja storage/komunikacji
```

### Protected Audience API + Fenced Frame

```javascript
// 1. Serwer reklamowy wywołuje ad auction:
const config = await navigator.runAdAuction({
    seller: 'https://ad-exchange.com',
    decisionLogicURL: 'https://ad-exchange.com/decision.js',
    interestGroupBuyers: ['https://shoes-advertiser.com'],
    // ...
});

// config to AuctionAdConfig — zawiera "opaque URL" do reklamy
// config.url jest niedostępne bezpośrednio (opaque!)

// 2. Wyświetl wygraną reklamę:
const frame = document.createElement('fencedframe');
frame.config = config;  // przekazanie opaque config
document.body.appendChild(frame);

// Parent nie wie co wyświetla fenced frame!
// Fenced frame nie wie na jakiej stronie jest!
```

### Ograniczenia komunikacji

```javascript
// Fenced Frame → Parent: ZABLOKOWANE
// (wewnątrz fenced frame):
window.parent.postMessage('data', '*');  // TypeError: cross-fenced-frame communication blocked

// Parent → Fenced Frame: ZABLOKOWANE
const frame = document.querySelector('fencedframe');
frame.contentWindow;  // null lub SecurityError

// Wyjątek: reportEvent() do reporting server (nie do parent!)
// (wewnątrz fenced frame):
window.fence?.reportEvent({
    eventType: 'click',
    eventData: 'buy-button',
    destination: ['buyer', 'seller'],  // do reporting endpoints, nie do parent!
});
```

### Shared Storage i Fenced Frame

```javascript
// Shared Storage API + Fenced Frame (Privacy Sandbox):
// Pozwala na przechowywanie cross-site danych (ograniczone)
// ale dostęp tylko przez Worklet (nie bezpośrednio)

// 1. Ustaw shared storage (np. na shop.com po zakupie):
await window.sharedStorage.set('purchased_shoes', '1');

// 2. Odczyt przez Worklet (nie JS):
class AdsSelectURLOperation {
    async run(urls, data) {
        const hasPurchased = await this.sharedStorage.get('purchased_shoes');
        return hasPurchased === '1' ? 1 : 0;  // indeks URL
    }
}
registerURLSelectionOperation('ads-select-url', AdsSelectURLOperation);

// 3. Wywołanie selectURL() → opaque result → do fenced frame
const url = await window.sharedStorage.selectURL('ads-select-url', [url1, url2]);
// url jest opaque — parent nie zna wartości!
```

---

## 4. Co dzieje się wewnętrznie

### Izolacja modelu bezpieczeństwa

```
Klasyczny iframe:
┌────────────────────────────────────────────┐
│ Parent Page (shop.com)                     │
│  ┌─────────────────────────────────────┐   │
│  │ iframe (ads.com)                    │   │
│  │  ↔ window.parent dostępne          │   │
│  │  ↔ postMessage w obu kierunkach    │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘

Fenced Frame:
┌────────────────────────────────────────────┐
│ Parent Page (news.com)                     │
│  ┌─────────────────────────────────────┐   │
│  │ Fenced Frame (ad.opaque)            │   │
│  │  ✗ window.parent → null/error      │   │
│  │  ✗ postMessage → zablokowane       │   │
│  │  ✗ parent nie zna URL              │   │
│  │  ✗ shared cookies → brak           │   │
│  │  ✓ reportEvent() → reporting only  │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
```

### Network isolation

```
Requests z fenced frame:
- Brak cookies parent page
- Brak cookies z poprzednich sesji (storage partitioning)
- Partitioned storage (tylko dla tego fenced frame kontekstu)
- Credentials: anonymous mode (jak crossorigin="anonymous")
```

### Klasa FencedFrameConfig

```javascript
// FencedFrameConfig to opaque object:
const config = await navigator.runAdAuction({ /* ... */ });

// Możliwe operacje:
config instanceof FencedFrameConfig;  // true
config.url;  // undefined — URL jest opaque!

// Tylko przypisanie do frame.config jest dozwolone:
document.createElement('fencedframe').config = config;  // OK
```

---

## 5. Analogiczny przykład z życia

Fenced Frame to **okno bez szyby** dla zewnętrznego artysty malującego ścianę:

- Właściciel domu (parent page) wynajmuje artystę (fenced frame) przez agenta (Protected Audience)
- Artysta maluje ścianę (wyświetla reklamę) ale właściciel nie widzi projektu
- Artysta nie wie gdzie jest dom (nie zna URL parent)
- Artysta nie może nic zabrać z domu (brak dostępu do cookies parent)
- Artysta może tylko powiedzieć agentowi: "skończyłem" (reportEvent — nie do właściciela!)
- Właściciel nie może ingerować w pracę artysty (brak contentWindow dostępu)

---

## 6. Przykład kodu

```javascript
// === Protected Audience API z Fenced Frame ===

// 1. Advertiser (shoes.com) — dołącz do Interest Group:
await navigator.joinAdInterestGroup({
    owner: 'https://shoes-advertiser.com',
    name: 'shoe-remarketing',
    ads: [{ renderURL: 'https://ads.shoes.com/ad1.html', metadata: { price: 99 } }],
    biddingLogicURL: 'https://shoes-advertiser.com/bidding.js',
    trustedBiddingSignalsURL: 'https://shoes-advertiser.com/signals.json',
    lifetimeSecs: 2592000,  // 30 dni
}, 3600);  // timeout 1 godzina

// 2. Publisher (news.com) — uruchom aukcję:
async function runAdAuction() {
    if (!navigator.runAdAuction) {
        // Fallback: tradycyjne reklamy (z third-party cookies jeśli dostępne)
        return loadTraditionalAd();
    }
    
    const auctionConfig = {
        seller: 'https://ad-exchange.com',
        decisionLogicURL: 'https://ad-exchange.com/decision.js',
        interestGroupBuyers: ['https://shoes-advertiser.com'],
        auctionSignals: { pageCategory: 'news' },
        sellerSignals: { minPrice: 0.01 },
        perBuyerSignals: { 'https://shoes-advertiser.com': { userSegment: 'premium' } },
    };
    
    const config = await navigator.runAdAuction(auctionConfig);
    
    if (config === null) {
        // Brak bids → pokaż default content
        return;
    }
    
    // 3. Wyświetl wygraną reklamę w Fenced Frame:
    const fencedFrame = document.createElement('fencedframe');
    fencedFrame.config = config;
    fencedFrame.setAttribute('width', '300');
    fencedFrame.setAttribute('height', '250');
    document.getElementById('ad-slot').appendChild(fencedFrame);
}
```

```javascript
// === Wewnątrz fenced frame (ad.html) ===

// Zgłoś zdarzenie (click) do reporting endpoints:
document.querySelector('#cta-button').addEventListener('click', () => {
    window.fence?.reportEvent({
        eventType: 'click',
        eventData: JSON.stringify({ adId: 'shoes-123', price: 99 }),
        destination: ['buyer', 'seller'],  // raport do kupca i sprzedawcy, NIE do parent!
    });
    
    // Nawigacja do advertiser strony:
    window.fence?.navigateAdURL({ url: 'https://shoes.com/product/123' });
});
```

---

## 7. Przykład z prawdziwej aplikacji

### Privacy Sandbox Ad Stack

```
Pełny flow:
1. User na shoes.com → joinAdInterestGroup() (zainteresowanie butami)
2. User na news.com → publisher uruchamia aukcję runAdAuction()
3. Przeglądarka lokalnie wykonuje aukcję (bidding.js, decision.js)
   → Dane nie wychodzą na zewnątrz!
4. Wynik: opaque FencedFrameConfig
5. Fenced Frame ładuje wygraną reklamę
6. User klika → reportEvent() → Attribution Reporting API
7. Konwersja raportowana bez cross-site ID → prywatność zachowana!
```

### Current Limitations (2024)

```javascript
// Fenced Frames są nadal eksperymentalne:
// Chrome: flaga #enable-fenced-frames (dostępne w Chrome 113+)
// Firefox: nie wspierane
// Safari: nie wspierane

// Sprawdzenie wsparcia:
if (typeof HTMLFencedFrameElement !== 'undefined') {
    console.log('Fenced Frames obsługiwane');
} else {
    console.log('Fenced Frames niedostępne — użyj iframe z privacy sandbox APIs');
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Próba komunikacji z fenced frame

```javascript
// BŁĄD: próba postMessage do fenced frame
const frame = document.querySelector('fencedframe');
frame.contentWindow?.postMessage('data', '*');
// Niedostępne! contentWindow jest null

// BŁĄD: wewnątrz fenced frame — próba dostępu do parent:
window.parent.postMessage('click', '*');
// SecurityError: cross-fenced-frame message zablokowane

// POPRAWKA: użyj reportEvent() dla raportowania
window.fence?.reportEvent({ eventType: 'click', destination: ['seller'] });
```

### Błąd 2: Używanie src zamiast config

```javascript
// BŁĄD: ustawianie src bezpośrednio (ignoruje ochronę opaque URL)
fencedFrame.src = 'https://ads.example.com/ad';
// Może działać w trybie 'default' ale nie daje ochrony Privacy Sandbox

// POPRAWKA: zawsze używaj config od Protected Audience/selectURL:
const config = await navigator.runAdAuction({ /* ... */ });
fencedFrame.config = config;  // opaque config
```

### Błąd 3: Brak fallback

```javascript
// BŁĄD: brak obsługi gdy fenced frames niedostępne
document.createElement('fencedframe');
// undefined w Firefox i Safari!

// POPRAWKA: feature detection:
function createAdSlot(config) {
    if (typeof HTMLFencedFrameElement !== 'undefined') {
        const ff = document.createElement('fencedframe');
        ff.config = config;
        return ff;
    } else {
        // Fallback: iframe z partitioned storage (mniej prywatny)
        const iframe = document.createElement('iframe');
        iframe.src = config.url || 'https://ads.example.com/default';
        return iframe;
    }
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Izolacja a bezpieczeństwo

```
Fenced Frame model:
+ Brak JavaScript injection z parent → brak XSS przez parent
+ Brak storage sharing → brak session fixation cross-frame
+ Brak URL reading → brak information leakage parent→frame
+ Brak postMessage → brak data exfiltration frame→parent
- Ale: fenced frame MOŻE załadować malicious content przez Protected Audience
  → Jeśli attacker kontroluje interest group → malicious ad URL
  → Fenced frame jest izolowany — ale nadal renderuje HTML/JS
  → Potencjalny XSS WEWNĄTRZ fenced frame (nie wycieka do parent)
```

### Ad fraud i clickjacking w Fenced Frame

```javascript
// Wewnątrz fenced frame: kliki na overlay
// Fenced frame: renderowanie elementów ponad innymi elementami parent?
// Nie! Fenced frame podlega tym samym regułom CSS co iframe
// (z-index, overflow hidden, itp.)

// Ale: fenced frame renderuje swój własny overlay wewnątrz — możliwy clickjacking
// do reportEvent() lub nawigacji wewnątrz frame
```

### Privacy przez design

```
Kluczowy model bezpieczeństwa prywatności:
- Reklamy mogą być targetowane (na podstawie danych przeglądarki)
- Dane targetowania NIE opuszczają przeglądarki
- Publisher nie zna szczegółów targetowania
- Advertiser nie zna publisher URL
- Żadna strona nie może śledzić użytkownika między stronami

To fundamentalna zmiana: dane służą targetowaniu ALE nie ujawniają tożsamości
```

### Powiązane CWE

- **CWE-668** — Exposure of Resource to Wrong Sphere (izolacja Fenced Frame adresuje to)
- **CWE-200** — Information Exposure (cross-site leaks przez iframe)
- **CWE-1021** — Improper Restriction of Rendered UI Layers (clickjacking)

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź Fenced Frame implementację

```javascript
// W konsoli przeglądarki:
typeof HTMLFencedFrameElement;  // 'function' jeśli Chrome z flagą

// Znajdź fenced frames na stronie:
document.querySelectorAll('fencedframe');  // HTMLCollection

// Sprawdź tryb:
const ff = document.querySelector('fencedframe');
ff.mode;  // 'opaque-ads' lub 'default'
```

### Test izolacji

```javascript
// Sprawdź czy parent ma dostęp do fenced frame:
const ff = document.querySelector('fencedframe');
console.log(ff.contentWindow);  // null = właściwa izolacja
console.log(ff.contentDocument); // null

// Próba wstrzyknięcia:
ff.contentWindow?.eval('alert(1)');  // TypeError — nie dostępne
```

### Sprawdź Protected Audience API

```javascript
// Sprawdź czy strona używa Interest Groups:
// (Brak bezpośredniego API do wylistowania, ale devtools pomagają)
// Chrome DevTools: Application → Interest Groups

// Sprawdź czy strona uruchamia aukcje:
// Network tab → szukaj navigation do 'urn:uuid:' URLs (opaque ad URLs)
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy fenced frame jest izolowany (contentWindow === null)?
□ Czy postMessage cross-fenced-frame jest blokowane?
□ Czy URL fenced frame jest opaque (brak src attribute z wartością)?
□ Czy reportEvent() jest używane zamiast postMessage do parent?
□ Czy fenced frame nie może czytać cookies parent?
□ Czy Protected Audience auction jest przeprowadzana lokalnie (brak exfiltracji)?
□ Czy Attribution Reporting nie ujawnia cross-site identifiers?
□ Czy Shared Storage worklet nie ujawnia danych przez side-channels?
□ Czy fenced frame nie może prowadzić clickjacking na parent?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Bezpieczne wdrożenie Fenced Frame ===

// Jako publisher — bezpieczna konfiguracja aukcji:
async function setupAdSlot(slotElement) {
    // Feature detection:
    if (typeof HTMLFencedFrameElement === 'undefined') {
        console.warn('Fenced Frames niedostępne — użyj CHIPS iframe fallback');
        return setupIframeAdSlot(slotElement);
    }
    
    try {
        const config = await navigator.runAdAuction({
            seller: 'https://trusted-exchange.com',
            decisionLogicURL: 'https://trusted-exchange.com/decision.js',
            interestGroupBuyers: ['https://trusted-advertiser.com'],
            
            // Bezpieczna konfiguracja:
            auctionSignals: {
                pageCategory: getPageCategory(),  // kategoria, nie PII!
                // NIE wysyłaj: user ID, email, historia zakupów, itp.
            },
        });
        
        if (!config) return;  // brak bids
        
        const fencedFrame = document.createElement('fencedframe');
        fencedFrame.config = config;
        fencedFrame.setAttribute('width', '300');
        fencedFrame.setAttribute('height', '250');
        
        // Nie próbuj komunikować się z frame:
        // fencedFrame.addEventListener('message', ...) // nie zadziała
        
        slotElement.appendChild(fencedFrame);
        
    } catch (err) {
        console.error('Ad auction failed:', err);
    }
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. Fenced Frames = izolowany embed bez komunikacji bidirectional z parent (brak window.parent, brak postMessage)
2. Zaprojektowany dla Privacy Sandbox: reklamy bez cross-site tracking
3. Opaque URL: parent nie zna URL wyświetlanej reklamy (`mode="opaque-ads"`)
4. reportEvent() jedyna dopuszczona komunikacja — do reporting endpoints, nie do parent
5. Eksperymentalne (2024): Chrome 113+, brak wsparcia Firefox/Safari

**Dla pentestera:**
- `document.querySelector('fencedframe').contentWindow` === null = właściwa izolacja
- Test: czy parent może wstrzykiwać JS do fenced frame? → nie powinien móc
- Sprawdź Protected Audience aukcje w DevTools → Application → Interest Groups
- Brak cross-frame postMessage = poprawna implementacja
- Fenced Frame NIE eliminuje XSS wewnątrz fenced frame (zewnętrzna zawartość nadal renderuje JS!)

---

## Powiązania

```
Fenced Frames
    │
    ├──► CHIPS (Rozdział 41)
    │         Partitioned Cookies: izolacja per top-level site
    │         Fenced Frames: izolacja per fenced frame kontekst
    │
    ├──► Credentialless iframes (Rozdział 49)
    │         Credentialless: iframe bez third-party credentials
    │         Fenced Frames: pełna izolacja + opaque URL
    │
    ├──► postMessage (Rozdział 23)
    │         postMessage: mechanizm komunikacji cross-frame
    │         W Fenced Frames: celowo zablokowany
    │
    └──► Storage Access API (Rozdział 47)
              SAA: dostęp do unpartitioned storage z user gesture
              Fenced Frames: celowo brak storage access
```
