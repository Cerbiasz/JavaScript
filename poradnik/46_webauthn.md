# Rozdział 46: WebAuthn i Passkeys

## 1. Czym jest WebAuthn i Passkeys

**WebAuthn** (Web Authentication API) to standard W3C/FIDO2 umożliwiający silne uwierzytelnianie bez haseł — przez kryptografię klucza publicznego. Zamiast hasła użytkownik używa **klucza prywatnego** przechowywanego bezpiecznie w urządzeniu (klucz nie opuszcza urządzenia), a strona weryfikuje podpis kryptograficzny za pomocą **klucza publicznego**.

**Passkeys** to przyjazna dla użytkownika implementacja WebAuthn — klucze synchronizowane między urządzeniami przez platformy (Apple iCloud Keychain, Google Password Manager, Windows Hello). Passkeys zastępują hasła w pełni.

Komponenty WebAuthn:
- **Authenticator** — urządzenie/oprogramowanie przechowujące klucze (Touch ID, Face ID, YubiKey, Windows Hello, Google Password Manager)
- **Relying Party (RP)** — aplikacja webowa weryfikująca uwierzytelnianie
- **User** — osoba logująca się
- **Credential** — para kluczy (prywatny na authenticatorze, publiczny na serwerze)

```javascript
// Rejestracja (tworzenie passkey):
const credential = await navigator.credentials.create({ publicKey: options });

// Uwierzytelnianie (logowanie passkey):
const assertion = await navigator.credentials.get({ publicKey: options });
```

---

## 2. Dlaczego powstał

### Problem: hasła jako fundamentalnie wadliwy mechanizm

```
Problemy z hasłami:
1. Phishing → użytkownik wpisuje hasło na fałszywej stronie
2. Password reuse → jeden breach → dostęp do wielu kont
3. Credential stuffing → listy haseł z breachów → atak na inne serwisy
4. Brute force → słabe hasła → łatwe złamanie
5. Data breaches → przechowywane hashe → rainbow tables, GPU cracking
6. Social engineering → wyłudzanie haseł

FIDO Alliance (2012) i WebAuthn (W3C 2019) rozwiązują to przez:
- Klucz prywatny nigdy nie opuszcza urządzenia
- Każda para kluczy jest unikalna dla kombinacji (RP, user, authenticator)
- Nie ma "hasła do wyłudzenia" — tylko podpis kryptograficzny
- Phishing niemożliwy — klucz powiązany z konkretnym RP ID (domeną)
```

---

## 3. Jak działa

### Rejestracja (Registration / Attestation)

```javascript
// Krok 1: Serwer generuje challenge i wysyła options do klienta
// Krok 2: Klient wywołuje create():

const registrationOptions = {
    challenge: new Uint8Array(32),  // random challenge z serwera
    rp: {
        name: 'Moja Aplikacja',
        id: 'app.example.com',      // RP ID = domena
    },
    user: {
        id: Uint8Array.from('user-id-123', c => c.charCodeAt(0)),
        name: 'user@example.com',
        displayName: 'Jan Kowalski',
    },
    pubKeyCredParams: [
        { alg: -7, type: 'public-key' },   // ES256 (ECDSA P-256)
        { alg: -257, type: 'public-key' }, // RS256 (RSA)
    ],
    authenticatorSelection: {
        authenticatorAttachment: 'platform',  // 'platform' (Touch ID) lub 'cross-platform' (YubiKey)
        requireResidentKey: true,              // passkey — resident key
        userVerification: 'required',          // biometria lub PIN
    },
    timeout: 60000,
    attestation: 'none',  // 'none' | 'indirect' | 'direct'
};

const credential = await navigator.credentials.create({ publicKey: registrationOptions });

// Krok 3: Wyślij response do serwera
const response = {
    id: credential.id,
    rawId: Array.from(new Uint8Array(credential.rawId)),
    response: {
        attestationObject: Array.from(new Uint8Array(credential.response.attestationObject)),
        clientDataJSON: Array.from(new Uint8Array(credential.response.clientDataJSON)),
    },
    type: credential.type,
};
await fetch('/api/webauthn/register', { method: 'POST', body: JSON.stringify(response) });
```

### Uwierzytelnianie (Authentication / Assertion)

```javascript
// Krok 1: Serwer generuje nowy challenge i listę dozwolonych credentials
// Krok 2: Klient wywołuje get():

const authOptions = {
    challenge: new Uint8Array(32),  // nowy random challenge z serwera!
    rpId: 'app.example.com',
    allowCredentials: [{            // opcjonalne: które credentials są dozwolone
        id: credentialId,
        type: 'public-key',
    }],
    userVerification: 'required',   // weryfikuj użytkownika (biometria/PIN)
    timeout: 60000,
};

const assertion = await navigator.credentials.get({ publicKey: authOptions });

// Krok 3: Wyślij assertion do serwera do weryfikacji
const assertionResponse = {
    id: assertion.id,
    response: {
        authenticatorData: Array.from(new Uint8Array(assertion.response.authenticatorData)),
        clientDataJSON: Array.from(new Uint8Array(assertion.response.clientDataJSON)),
        signature: Array.from(new Uint8Array(assertion.response.signature)),
        userHandle: assertion.response.userHandle 
            ? Array.from(new Uint8Array(assertion.response.userHandle))
            : null,
    },
};
await fetch('/api/webauthn/authenticate', { method: 'POST', body: JSON.stringify(assertionResponse) });
```

### Weryfikacja po stronie serwera

```javascript
// Serwer Node.js (używając biblioteki @simplewebauthn/server):
const { verifyAuthenticationResponse } = require('@simplewebauthn/server');

const verification = await verifyAuthenticationResponse({
    response: assertionResponse,
    expectedChallenge: storedChallenge,
    expectedOrigin: 'https://app.example.com',
    expectedRPID: 'app.example.com',
    authenticator: {
        credentialID: storedCredential.id,
        credentialPublicKey: storedCredential.publicKey,
        counter: storedCredential.counter,
    },
});

if (verification.verified) {
    // Aktualizuj counter w bazie:
    updateCredential(credentialId, verification.authenticationInfo.newCounter);
    // Zaloguj użytkownika
}
```

---

## 4. Co dzieje się wewnętrznie

### Kryptografia WebAuthn

```
Rejestracja:
1. Authenticator generuje parę kluczy (ES256: ECDSA P-256):
   - Klucz prywatny → bezpieczne przechowywanie (TPM, SE, Keychain)
   - Klucz publiczny → wysyłany do RP (serwer)

2. Authenticator tworzy "credential ID" (losowy identyfikator)

3. Authenticator podpisuje odpowiedź (attestationObject):
   - Zawiera: klucz publiczny, RP ID hash, counter, user handle
   - Podpisane przez authenticator (atestacja)

Uwierzytelnianie:
1. Serwer → challenge (random nonce)
2. Authenticator podpisuje: challenge + RP ID hash + counter + user verification flag
3. Serwer weryfikuje podpis używając stored public key
4. Counter: rośnie przy każdym użyciu → wykrywanie klonowania!
```

### RP ID (Relying Party ID)

```javascript
// RP ID jest kluczowy dla bezpieczeństwa:
rp: { id: 'app.example.com' }

// Passkey dla app.example.com działa NA:
// - https://app.example.com ✓
// - https://app.example.com:443 ✓

// Passkey NIE działa NA:
// - https://evil.com ✗ (inne RP ID)
// - https://phishing-app.example.com ✗ (RP ID jest konkretną domeną!)

// Dlaczego phishing jest niemożliwy:
// Przeglądarka weryfikuje że current origin pasuje do RP ID
// Nawet jeśli user wchodzi na phishing.com → przeglądarka nie ma passkey dla phishing.com
```

### Resident Keys (Passkeys)

```javascript
// Resident key (Discoverable Credential):
authenticatorSelection: {
    requireResidentKey: true,  // lub residentKey: 'required'
}
// Credential ID jest przechowywane na authenticatorze
// Serwer może wysłać pustą listę allowCredentials → usernameless login!

// Non-resident key (Server-side credential):
authenticatorSelection: {
    requireResidentKey: false,
}
// Credential ID jest podawane przez serwer przy każdym logowaniu
```

### Counter i clone detection

```
Przy każdym uwierzytelnieniu: counter rośnie o 1
Serwer przechowuje ostatnią wartość countera

Jeśli authenticator jest sklonowany:
- Oryginalny counter: 5
- Klon też ma counter: 5
- Klon loguje się: counter = 6 → serwer zapisuje 6
- Oryginał loguje się: serwer widzi counter ≤ 6 → ALARM! → klonowanie wykryte!

(Dla platform passkeys syncowanych przez chmurę: counter może być 0 zawsze — chmura ma inny mechanizm)
```

---

## 5. Analogiczny przykład z życia

WebAuthn/Passkeys to **paszport kryptograficzny**:

- Każdy kraj (RP) wydaje paszport (credential) powiązany z konkretną osobą
- Paszport ma chip z kluczem prywatnym (biometria PIN/odcisk palca go odblokuje)
- Na granicy (logowanie): strażnik (przeglądarka) sprawdza paszport przez zeskanowanie chipu i weryfikację podpisu
- Fałszywy paszport (phishing): strażnik widzi że chip NIE jest podpisany przez właściwy kraj (RP ID) → odmowa
- Klucz prywatny nigdy nie wychodzi z paszportu — tylko podpis do weryfikacji

---

## 6. Przykład kodu

```javascript
// === Kompletny WebAuthn flow (frontend) ===

class WebAuthnManager {
    constructor(apiBase = '/api/webauthn') {
        this.apiBase = apiBase;
    }
    
    // Pomocnik: base64url encode/decode
    bufferToBase64url(buffer) {
        return btoa(String.fromCharCode(...new Uint8Array(buffer)))
            .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
    }
    
    base64urlToBuffer(base64url) {
        const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
        const binary = atob(base64.padEnd(base64.length + (4 - base64.length % 4) % 4, '='));
        return Uint8Array.from(binary, c => c.charCodeAt(0)).buffer;
    }
    
    // Rejestracja passkey:
    async register(username) {
        // 1. Pobierz options z serwera:
        const optionsResponse = await fetch(`${this.apiBase}/register/options`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ username }),
            credentials: 'include',
        });
        const options = await optionsResponse.json();
        
        // Dekoduj challenge:
        options.challenge = this.base64urlToBuffer(options.challenge);
        options.user.id = this.base64urlToBuffer(options.user.id);
        
        // 2. Wywołaj authenticator:
        const credential = await navigator.credentials.create({ publicKey: options });
        
        // 3. Wyślij do serwera:
        const response = await fetch(`${this.apiBase}/register/verify`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                id: credential.id,
                rawId: this.bufferToBase64url(credential.rawId),
                response: {
                    attestationObject: this.bufferToBase64url(credential.response.attestationObject),
                    clientDataJSON: this.bufferToBase64url(credential.response.clientDataJSON),
                },
                type: credential.type,
            }),
            credentials: 'include',
        });
        
        return await response.json();
    }
    
    // Logowanie passkey:
    async authenticate(username = null) {
        // 1. Pobierz challenge:
        const optionsResponse = await fetch(`${this.apiBase}/authenticate/options`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ username }),  // null = usernameless (resident key)
            credentials: 'include',
        });
        const options = await optionsResponse.json();
        
        options.challenge = this.base64urlToBuffer(options.challenge);
        if (options.allowCredentials) {
            options.allowCredentials = options.allowCredentials.map(cred => ({
                ...cred,
                id: this.base64urlToBuffer(cred.id),
            }));
        }
        
        // 2. Uwierzytelnianie:
        const assertion = await navigator.credentials.get({ publicKey: options });
        
        // 3. Weryfikacja na serwerze:
        const response = await fetch(`${this.apiBase}/authenticate/verify`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                id: assertion.id,
                response: {
                    authenticatorData: this.bufferToBase64url(assertion.response.authenticatorData),
                    clientDataJSON: this.bufferToBase64url(assertion.response.clientDataJSON),
                    signature: this.bufferToBase64url(assertion.response.signature),
                    userHandle: assertion.response.userHandle
                        ? this.bufferToBase64url(assertion.response.userHandle)
                        : null,
                },
            }),
            credentials: 'include',
        });
        
        return await response.json();
    }
}
```

---

## 7. Przykład z prawdziwej aplikacji

### Phishing próba na passkeys

```
Scenariusz ataku:
1. Atakujący klonuje stronę bank.com → tworzy phishing-bank.com
2. User dostaje email z linkiem do phishing-bank.com
3. User klika "Zaloguj z Passkey"
4. Przeglądarka: navigator.credentials.get({ publicKey: { rpId: 'bank.com' } })
   ALE: current origin to phishing-bank.com, nie bank.com!
5. Przeglądarka ODMAWIA: RP ID 'bank.com' nie pasuje do current origin
6. PHISHING NIEMOŻLIWY! ✓

Nawet jeśli atakujący wpisze rpId: 'phishing-bank.com':
→ User nie ma passkey dla phishing-bank.com
→ Przeglądarka nie znajdzie matching credential
```

### Passkey MFA i step-up authentication

```javascript
// Passkey jako 2FA (dodatkowe do hasła):
async function stepUpAuth() {
    const options = await fetch('/api/webauthn/step-up/options', {
        credentials: 'include',
    }).then(r => r.json());
    
    options.challenge = base64urlToBuffer(options.challenge);
    options.allowCredentials = options.allowCredentials.map(c => ({
        ...c,
        id: base64urlToBuffer(c.id),
    }));
    
    const assertion = await navigator.credentials.get({
        publicKey: { ...options, userVerification: 'required' }
    });
    
    // Wyślij do serwera — weryfikuj jako step-up auth
    await fetch('/api/webauthn/step-up/verify', {
        method: 'POST',
        credentials: 'include',
        body: JSON.stringify(formatAssertion(assertion)),
    });
    
    // Teraz użytkownik ma elevated privileges
}
```

---

## 8. Typowe błędy programistów

### Błąd 1: Nieweryfikowanie RP ID po stronie serwera

```javascript
// BŁĄD: serwer nie sprawdza RP ID w clientDataJSON
// Atakujący może podmienić challenge lub origin

// POPRAWKA: zawsze weryfikuj przez bibliotekę:
const { verifyRegistrationResponse } = require('@simplewebauthn/server');
const verification = await verifyRegistrationResponse({
    response: body,
    expectedChallenge: challenge,
    expectedOrigin: 'https://app.example.com',  // ← weryfikuje origin!
    expectedRPID: 'app.example.com',            // ← weryfikuje RP ID!
});
```

### Błąd 2: Niewalidowanie countera

```javascript
// BŁĄD: ignorowanie countera → klonowanie authenticatora niezauważone
const verification = await verifyAuthentication(assertion, { skipCounter: true });

// POPRAWKA: zawsze sprawdzaj counter
if (verification.newCounter <= storedCounter && storedCounter > 0) {
    throw new Error('Counter mismatch — possible authenticator clone!');
}
await updateCounter(credentialId, verification.newCounter);
```

### Błąd 3: Brak fallback dla przeglądarek bez WebAuthn

```javascript
// BŁĄD: brak obsługi przeglądarek bez WebAuthn
const credential = await navigator.credentials.create({ publicKey: options });
// TypeError na starszych przeglądarkach!

// POPRAWKA: feature detection + fallback
async function register() {
    if (!window.PublicKeyCredential) {
        // Fallback: tradycyjne hasło + TOTP 2FA
        return registerWithPassword();
    }
    
    // Sprawdź czy authenticator jest dostępny:
    const available = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
    if (!available) {
        // Zaproponuj roaming authenticator (YubiKey) lub fallback
    }
    
    // Normalna rejestracja WebAuthn
}
```

### Błąd 4: Przechowywanie challenge bez expiry

```javascript
// BŁĄD: challenge bez TTL → replay attack możliwy
session.webauthnChallenge = challenge;
// challenge może być użyty godziny/dni później!

// POPRAWKA: challenge z TTL:
session.webauthnChallenge = {
    value: challenge,
    expiresAt: Date.now() + 5 * 60 * 1000, // 5 minut
};

// Przy weryfikacji:
if (Date.now() > session.webauthnChallenge.expiresAt) {
    throw new Error('Challenge expired');
}
```

---

## 9. Znaczenie dla bezpieczeństwa

### Ochrona przed phishingiem

```
WebAuthn jest PHISHING-RESISTANT:
- Klucz powiązany z RP ID (domeną)
- Przeglądarka weryfikuje origin przed użyciem
- Nawet jeśli user wejdzie na phishing.com → passkey dla bank.com nie zostanie użyty
- BRAK możliwości wyłudzenia hasła (bo nie ma hasła!)
```

### Ochrona przed credential stuffing

```
Bez hasła → brak hash do zbrzucia
Brak reużywania → każdy RP ma unikatową parę kluczy
Brak danych do exfiltracji → serwer przechowuje TYLKO klucz publiczny
```

### Ochrona przed man-in-the-middle

```
MITM nie może:
- Przechwycić klucza prywatnego (nigdy nie opuszcza authenticatora)
- Replay assertion (challenge jest jednorazowy)
- Zmienić origin (przeglądarka weryfikuje HTTPS)
```

### Słabości WebAuthn

```
Możliwe ataki:
1. Phishing authenticator setup: "Dodaj passkey przez naszą stronę" → phishing site pyta o passkey dla RP ID
   → Nie możliwe! Passkey tworzony dla bieżącego origin, nie dowolnego

2. Physical access: jeśli atakujący ma fizyczny dostęp do urządzenia
   → Authenticator wymaga biometrii/PIN → ochrona

3. Platform passkey backup risk: syncowane przez iCloud/Google
   → Kto ma dostęp do konta Apple/Google może uzyskać passkeys
   → Enterprise: hardware security keys (YubiKey) bez synchronizacji

4. FIDO2 downgrade: serwer oferuje też password → atakujący może wymusić hasło
   → Nie oferuj fallback na tradycyjne hasło jeśli to możliwe
```

### Powiązane CWE

- **CWE-287** — Improper Authentication (WebAuthn jako mitygacja)
- **CWE-308** — Use of Single-factor Authentication
- **CWE-836** — Use of Password Hash with Insufficient Computational Effort
- **CWE-640** — Weak Password Recovery Mechanism

---

## 10. Jak identyfikować podczas pentestu

### Sprawdź WebAuthn implementację

```javascript
// W konsoli przeglądarki:
typeof navigator.credentials.create; // 'function' → WebAuthn dostępne
PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable().then(console.log);
```

### Sprawdź serwer-side weryfikację

```bash
# Sprawdź czy serwer weryfikuje:
# 1. challenge (czy jest single-use?)
# 2. origin (czy sprawdza expected origin?)
# 3. RP ID (czy sprawdza?)
# 4. counter (czy sprawdza? klonowanie)

# Spróbuj replay attack:
# Przechwytaj assertion → wyślij ponownie → czy serwer akceptuje?
# Jeśli TAK → brak replay protection!

# Test: zmień origin w clientDataJSON → serwer powinien odrzucić
```

### Test RP ID binding

```javascript
// Sprawdź czy RP ID jest hardcoded vs dynamiczne:
// Jeśli RP ID pochodzi z request → możliwa manipulacja!

// Bezpieczna implementacja:
const EXPECTED_RP_ID = 'app.example.com';  // hardcoded po stronie serwera
// NIE: req.body.rpId
```

---

## 11. Jak testować bezpieczeństwo

```
□ Czy serwer weryfikuje challenge przy każdym request?
□ Czy challenge jest jednorazowy (single-use, z TTL)?
□ Czy serwer weryfikuje expected origin?
□ Czy serwer weryfikuje RP ID?
□ Czy counter jest walidowany i aktualizowany?
□ Czy replay attacks są blokowane?
□ Czy fallback do hasła jest zabezpieczony (może być słabszy link)?
□ Czy attestation jest sprawdzana jeśli wymagana (enterprise)?
□ Czy credential ID jest powiązany z konkretnym userem?
□ Czy użycie bez userVerification jest blokowane dla wrażliwych operacji?
```

---

## 12. Jak się zabezpieczać

```javascript
// === Serwer Node.js: kompletna weryfikacja (SimpleWebAuthn) ===
const { verifyAuthenticationResponse, verifyRegistrationResponse } = require('@simplewebauthn/server');

// Weryfikacja rejestracji:
async function verifyRegistration(body, expectedChallenge) {
    return await verifyRegistrationResponse({
        response: body,
        expectedChallenge,
        expectedOrigin: process.env.WEBAUTHN_ORIGIN,  // 'https://app.example.com'
        expectedRPID: process.env.WEBAUTHN_RP_ID,     // 'app.example.com'
        requireUserVerification: true,  // wymagaj biometrii/PIN
    });
}

// Weryfikacja uwierzytelniania:
async function verifyAuthentication(body, storedCredential, expectedChallenge) {
    const verification = await verifyAuthenticationResponse({
        response: body,
        expectedChallenge,
        expectedOrigin: process.env.WEBAUTHN_ORIGIN,
        expectedRPID: process.env.WEBAUTHN_RP_ID,
        authenticator: {
            credentialID: storedCredential.id,
            credentialPublicKey: storedCredential.publicKey,
            counter: storedCredential.counter,
        },
        requireUserVerification: true,
    });
    
    if (verification.verified) {
        // Aktualizuj counter:
        await db.credentials.update({
            id: storedCredential.id,
            counter: verification.authenticationInfo.newCounter,
        });
    }
    
    return verification;
}
```

---

## 13. Podsumowanie

**Kluczowe fakty:**

1. WebAuthn = standard FIDO2 — kryptografia klucza publicznego zamiast haseł
2. Passkeys = WebAuthn z synchronizacją między urządzeniami (iCloud, Google, Windows Hello)
3. Phishing-resistant: klucz prywatny powiązany z RP ID (domeną) — nie działa na innej domenie
4. Klucz prywatny **nigdy** nie opuszcza authenticatora
5. Counter chroni przed klonowaniem hardware authenticatora

**Dla pentestera:**
- Test replay attack: przechwytaj assertion → wyślij ponownie → czy serwer akceptuje?
- Sprawdź czy serwer weryfikuje expected origin i RP ID
- Sprawdź czy challenge ma TTL i jest single-use
- Fallback na hasło = słabszy link w łańcuchu bezpieczeństwa
- Brak counter validation = brak wykrywania klonowania authenticatora

---

## Powiązania

```
WebAuthn / Passkeys
    │
    ├──► Credential Management API (Rozdział 45)
    │         PublicKeyCredential przez navigator.credentials
    │         Credential Management łączy WebAuthn z przeglądarką
    │
    ├──► HTTPS / Secure Context
    │         WebAuthn wymaga HTTPS (Secure Context)
    │         RP ID musi pasować do HTTPS origin
    │
    ├──► Cookies (Rozdział 14)
    │         Po WebAuthn → session cookie jako token sesji
    │         HttpOnly + SameSite=Strict dla session cookie
    │
    └──► CSP (Rozdział 36)
              CSP chroni przed XSS który mógłby wywołać WebAuthn API
              Silna CSP + WebAuthn = silna ochrona
```
