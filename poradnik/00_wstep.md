# JavaScript Security Pentest Guide
## Profesjonalny poradnik bezpieczeństwa aplikacji webowych

**Autor:** Poradnik przeznaczony dla pentesterów i security engineerów  
**Język:** Polski  
**Poziom:** Średniozaawansowany → Zaawansowany

---

## Wstęp

Ten poradnik powstał z jednej prostej obserwacji: większość pentesterów aplikacji webowych zna narzędzia — Burp Suite, DevTools, payloady XSS — ale nie rozumie głęboko mechanizmów, które te narzędzia eksploitują. Skutkiem jest testowanie z listy checkboxów zamiast rzeczywistego rozumienia powierzchni ataku.

Niniejszy materiał ma wypełnić tę lukę. Każdy rozdział tłumaczy nie tylko *co* robi dany mechanizm, ale *dlaczego istnieje*, *jak działa wewnątrz przeglądarki* oraz *gdzie kryją się podatności*.

### Dla kogo jest ten poradnik

- Pentesterzy aplikacji webowych chcący pogłębić wiedzę techniczną
- Security engineerowie piszący polityki bezpieczeństwa
- Programiści uczący się myśleć jak atakujący
- Osoby przygotowujące się do certyfikatów (OSWE, BSCP, eWPTX)

### Jak czytać ten poradnik

Każdy z 50 rozdziałów jest niezależny, jednak mechanizmy są ze sobą powiązane — na końcu każdego rozdziału znajdziesz diagram zależności. Możesz czytać linearnie lub skakać do tematów, które są Ci aktualnie potrzebne.

### Konwencje

- `kod inline` — fragmenty kodu, wartości nagłówków, nazwy API
- Bloki kodu — kompletne przykłady z wyjaśnieniem linijka po linijce
- Diagramy ASCII — przepływ danych i architektura
- **Pogrubienie** — kluczowe pojęcia przy pierwszym wystąpieniu
- > Uwaga bezpieczeństwa — sekcje wymagające szczególnej uwagi

---

## Spis treści

1. JavaScript Runtime
2. Execution Context
3. Call Stack
4. Scope
5. Closures
6. Prototype Chain
7. Prototype Pollution
8. Event Loop
9. DOM
10. DOM Clobbering
11. Fetch API
12. XMLHttpRequest
13. CORS
14. Cookies
15. localStorage
16. sessionStorage
17. IndexedDB
18. Cache API
19. Service Workers
20. Web Workers
21. Shared Workers
22. BroadcastChannel
23. postMessage
24. MessageChannel
25. WebSocket
26. Server-Sent Events
27. File API
28. Clipboard API
29. History API
30. URL API
31. Shadow DOM
32. Custom Elements
33. Web Components
34. Permissions Policy
35. Referrer Policy
36. Content Security Policy (CSP)
37. Trusted Types
38. Subresource Integrity (SRI)
39. Fetch Metadata Headers
40. SameSite Cookies
41. CHIPS (Partitioned Cookies)
42. COOP
43. COEP
44. CORP
45. Credential Management API
46. WebAuthn i Passkeys
47. Storage Access API
48. Fenced Frames
49. Credentialless iframes
50. WebAssembly
