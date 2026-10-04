# Norsk offentlig nettsted — sikkerhets- og API-testing

## Sammendrag

Denne casestudien dokumenterer praktisk sikkerhets- og kodetesting av den **offentlige, ikke-autentiserte delen av et reelt norsk nettsted**. Navnet på organisasjonen og identifiserende produksjonsdetaljer er bevisst fjernet.

Testingen avdekket flere reproduserbare feil og sikkerhetsrelevante observasjoner:

- Feil håndtering av reserverte tegn og URL-kodede verdier
- Oppførsel relatert til HTTP Parameter Pollution (HPP)
- HTTP 500-feil ved ugyldige parametere
- Inkonsistent validering av API-input
- Observasjoner knyttet til Content Security Policy (CSP)

Disse problemene kan føre til at en forespørsel blir forskjellig fra det brukeren faktisk skrev inn, endre strukturen på URL-parametere, skape ulik behandling mellom applikasjonslag eller føre til at frontend forventer JSON, men mottar en HTML-feilside.

Testingen ble utført på offentlig funksjonalitet, HTTP-forespørsler og -svar, offentlige API-er og offentlig tilgjengelig JavaScript, basert på prinsipper fra **OWASP Web Security Testing Guide (WSTG)**.

Det ble ikke utført omgåelse av autentisering, tilgang til private data, destruktive handlinger, brute-force eller tjenestenekt-testing (DoS).

**Ingen Critical/High-sårbarhet, SQL Injection, XSS eller omgåelse av autorisasjon ble bekreftet.**

---

## Funn 1 — Feil ved koding av søkeparametere

**Klassifisering:** Bekreftet funksjonell feil / potensielt sikkerhetsrelevant oppførsel  
**Alvorlighetsgrad:** Low  
**Relatert:** OWASP WSTG-INPV-04, CWE-116 / CWE-20

En offentlig søkefunksjon bevarte ikke alltid reserverte tegn korrekt når de ble sendt inn som brukerinput.

### Test

    Input:
    TEST_A&TEST_B

    Forventet:
    Hele teksten forblir én søkeverdi.

    Observert:
    "&" kunne tolkes som URL-syntaks,
    noe som endret den faktiske forespørselen.

Tilsvarende oppførsel ble observert med `#`.

Analyse av offentlig tilgjengelig JavaScript identifiserte en kodebane der URL-parametere ble satt sammen uten konsekvent bruk av sikker serialisering av parametere.

**Mulige konsekvenser:** Endrede søkeforespørsler, utilsiktede query-parametere og forskjeller mellom frontend- og backend-tilstand.

Ingen omgåelse av autorisasjon eller tilgang til private data ble påvist.

---

## Funn 2 — Kodede verdier endres av API-et

**Klassifisering:** Bekreftet feil i behandling av input  
**Alvorlighetsgrad:** Low

Korrekt URL-kodede reserverte tegn ble ikke alltid bevart under behandlingen.

### Testede verdier

    %23 → #
    %26 → &
    %2B → +

Den observerte oppførselen inkluderte avkorting eller endring av den faktiske verdien.

Dette indikerer at problemet kan omfatte mer enn bare URL-konstruksjon i frontend, og kan oppstå under dekoding eller behandling av parametere mellom ulike applikasjonslag.

Den nøyaktige årsaken i backend ble ikke fastslått.

---

## Funn 3 — Ugyldige parametere utløser HTTP 500

**Klassifisering:** Bekreftet validerings-/feilhåndteringsfeil  
**Alvorlighetsgrad:** Informational / Low

### Kontroll

    page=0
    → HTTP 200
    → JSON-svar

### Ugyldig input

    page=-1
    → HTTP 500

    page=invalid
    → HTTP 500

Ugyldige filterverdier førte også til HTTP 500 under testingen.

**Mulige konsekvenser:**

- Frontend forventer JSON, men mottar HTML
- Behandling av svaret kan feile
- Ugyldig input når serverens interne feilhåndtering
- API-et får inkonsistent oppførsel

Et kontrollert `400 Bad Request`-svar ville normalt vært mer hensiktsmessig.

Ingen tjenestenekt-effekt ble testet eller påvist.

---

## Funn 4 — HTTP Parameter Pollution-kandidat

**Klassifisering:** Potensiell sårbarhet  
**Alvorlighetsgrad:** Low

Dupliserte query-parametere ble behandlet inkonsistent.

    parameter=value1&parameter=value2

Noen dupliserte parametere førte til HTTP 500, mens en annen duplisert parameter ble slått sammen til en kombinert verdi.

Ulik tolkning av dupliserte parametere mellom forskjellige applikasjonslag kan føre til sikkerhetsrelevant oppførsel.

Ingen omgåelse av autentisering, autorisasjon eller tilgang til beskyttede data ble påvist.

---

## Funn 5 — Inkonsistent API-feilhåndtering

Forskjellige offentlige API-er viste ulik valideringsoppførsel.

Noen endepunkter returnerte korrekt:

- HTTP 400 ved ugyldige forespørsler
- HTTP 404 for ukjente ressurser

Et annet testet API returnerte **HTTP 500** for sammenlignbare ugyldige parametere.

Dette bidro til å skille en konkret validerings-/robusthetsfeil fra normal oppførsel på plattformen.

---

## Funn 6 — CORS-gjennomgang

Kontrollerte eksterne `Origin`-verdier ble testet mot utvalgte offentlige endepunkter.

De testede svarene viste ikke permissiv `Access-Control-Allow-Origin` eller deling av credentials med de eksterne origin-verdiene.

**Resultat:** Ingen permissiv CORS-sårbarhet ble påvist.

---

## Funn 7 — Input Reflection / XSS-kontroller

Nøytrale spesialtegn ble testet:

    < > " '

I de testede kontekstene ble tegnene escaped i stedet for å bli gjengitt som kjørbar markup.

**Resultat:** Ingen uescaped refleksjon ble påvist i de testede kontekstene.

Kjørbare XSS-payloads ble ikke brukt. Resultatet betyr derfor ikke at alle mulige XSS-veier i applikasjonen er utelukket.

---

## Funn 8 — CSP / sikkerhetskonfigurasjon

Content Security Policy og relatert HTTP-sikkerhetskonfigurasjon ble gjennomgått.

Defense-in-depth-observasjoner inkluderte direktiver som:

    'unsafe-inline'
    'unsafe-eval'

samt inkonsistent CSP-dekning på utvalgte offentlige sider.

Dette er **observasjoner knyttet til sikkerhetskonfigurasjon**, ikke bevis på en utnyttbar XSS-sårbarhet.

---

## Ytterligere sikkerhetskontroller

### SQL Injection

Nøytral input med anførselstegn førte ikke til SQL-feil, SQLSTATE-meldinger eller database-stack traces.

**Resultat:** SQL Injection ble ikke bekreftet.

### NoSQL Injection

En JSON-lignende markør forble vanlig tekstdata i det observerte svaret.

**Resultat:** NoSQL Injection ble ikke bekreftet.

### Server-Side Template Injection

Nøytrale template-lignende markører viste ingen tegn til server-side evaluering.

**Resultat:** SSTI ble ikke bekreftet.

---

## Samlet resultat

Testingen identifiserte reproduserbare feil knyttet til:

- URL- og parameterhåndtering
- Behandling av kodet input
- HTTP 500-feilhåndtering
- Dupliserte parametere / HPP
- Validering av API-input
- Defensiv sikkerhetskonfigurasjon

Testingen påviste **ikke** en Critical/High exploit, omgåelse av autentisering, eksponering av private data, SQL Injection eller XSS.

Denne casestudien demonstrerer praktisk **Web/API Security Testing, analyse av offentlig JavaScript, reproduserbar testing, OWASP-metodikk, teknisk dokumentasjon og ansvarlig klassifisering av sikkerhetsfunn**.