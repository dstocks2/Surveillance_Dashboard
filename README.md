# Projectinstructie — Surveillance Dashboard

> Deze instructie vervangt alle voorgaande. Lees dit eerst bij elke nieuwe sessie.

## Wat het is

Het **Surveillance Dashboard** is een **single-file HTML-webapplicatie** voor de beveiliging van UMC Utrecht. Alles (HTML, CSS, JavaScript) zit in één bestand. D is de primaire ontwikkelaar en communiceert in het Nederlands. Antwoord beknopt en praktisch.

De app draait op meerdere sessies tegelijk en synchroniseert live via **Firebase Realtime Database**.

## Huidige versie

- **v2.3.0** is de actuele versie: getest in de sandbox, klaar voor uitrol naar productie.
- Productie draait mogelijk nog op een oudere versie tot de uitrol is gedaan.

## Vaste werkwijze (belangrijk — altijd volgen)

1. **Werk op een kopie** van het laatste versiebestand in `/home/claude`, lever eindresultaat in `/mnt/user-data/outputs/`.
2. **Hoog de versie op bij elke wijziging**, op drie plekken consistent:
   - de `APP_VERSION`-constante in het niet-module script,
   - de `version-badge`/`app-version-badge` span in de HTML,
   - de bestandsnaam.
   Gebruik patch-niveau voor fixes (2.3.0 → 2.3.1) en minor voor grotere functionele wijzigingen (2.2.x → 2.3.0).
3. **Patch gericht** via Python (`str.replace` voor kleine edits; grotere herschrijvingen mogen, mits zorgvuldig).
4. **Syntax-check na elke wijziging**: extraheer beide `<script>`-blokken en draai `node --check`.
   - Het niet-module script controleren als `.js`.
   - Het module script (met `import`) controleren als `.mjs`.
5. **Raad de datastructuur of het gedrag niet** — als iets onzeker is, vraag om een verse Firebase-export of een screenshot van de Console/devtools-console voordat je rules of logica wijzigt.
6. **Eén wijziging per keer waar mogelijk**, en benoem expliciet de risico's op regressie.

## Architectuur

- **Twee `<script>`-blokken**: een `type="module"` blok (Firebase SDK-imports + config + auth-gate) en een groot niet-module blok (alle app-logica). Het module-script exposeert helpers op `window` (`window.fbDb`, `window.fbRef`, `window.fbSignIn`, enz.) voor het niet-module script.
- **`S.medewerkers`** is `[{naam, rol}]` en de **enige** bron voor namen en rollen. `S.names` bestaat niet meer; `dashboard/names` wordt niet meer geschreven.
- `renderLive()` en `renderPlan()` worden **altijd** vanuit `fbListen` aangeroepen (niet conditioneel op pane-zichtbaarheid). `drawTimeline()` (canvas) tekent alleen als de live-pane zichtbaar is (`_liveVisible`-guard).
- **Firebase-paden**: `dashboard/medewerkers`, `dashboard/active`, `dashboard/log`, `dashboard/meldingen`, `dashboard/dagdelen`, `dashboard/schema`, `dashboard/routes`, `dashboard/routeCats`, `dashboard/dienst`, `dashboard/appVersion`, `dashboard/gedrag`, `dashboard/auditLog`, `dashboard/pin`.
- **Datatypes (belangrijk voor rules):**
  - `log`, `auditLog`, `meldingen` = pushID-dicts (sleutel is een gegenereerde `-Ou...`-key).
  - `medewerkers`, `routes`, `dienst`, `routeCats` = arrays.
  - `active` = object gekeyd op naam → `{route, start}`.
  - `gedrag` = object met 4 numerieke velden (`timeoutWarn`, `meldingTTL`, `pinCacheDur`, `logRetention`).

## Authenticatie (sinds v2.3.0)

- **Email/password** met **één gedeeld dienst-account** (geen anonieme login meer).
- Een **inlogscherm-overlay** (`#login-overlay`) schermt de hele app af tot er een geldige sessie is.
- **Auth-gate**: `onAuthStateChanged` in het module-script roept `window.onAuthChanged` aan in het niet-module script. De app boot **exact één keer** via `initFirebase()`, beschermd door een `_appStarted`-guard. `__pendingAuthFired` vangt het geval dat auth resolvet vóór het hoofdscript geparsed is.
- **`browserLocalPersistence`**: bewakers blijven permanent ingelogd. Opnieuw inloggen alleen na: handmatig uitloggen, browserdata wissen, wachtwoordwijziging van het dienst-account, of incognito.
- `doLogin()` en `doLogout()` met Nederlandse foutmeldingen per `error.code`.
- De **uitlog-knop staat bewust in Instellingen achter de pincode** (beheerdershandeling), niet in de tabbar. Niet verplaatsen tenzij D het scenario expliciet wijzigt.

## XSS- en veiligheidsarchitectuur (sinds v2.2.09)

- **State houdt natuurlijke (gedecodeerde) waarden**; HTML-escaping gebeurt op het **renderpunt** via `escHtml()`. Nooit pre-encoderen in state (dat geeft dubbele encoding).
- **`safeColor()`** valideert `#hex` strikt vóór gebruik in een inline `style`-attribuut (tegen attribuut-/CSS-breakout).
- Eén gedeelde **`decodeEntities()`** op moduleniveau.
- **`sanitizeSchema()` / `sanitizeDagdelen()`** draaien bij het inlezen uit Firebase.
- Houd deze conventie aan bij nieuwe render-code: elke door gebruiker/Firebase bestuurde waarde die in `innerHTML` of een attribuut belandt, via `escHtml()` (of `safeColor()` voor kleuren).

## Toegankelijkheid & mobiel (sinds v2.2.10)

- Viewport **zonder** `user-scalable=no`/`maximum-scale` (pinch-zoom toegestaan).
- Invoervelden **16px op mobiel** via een `@media(max-width:600px)`-regel met `!important` (voorkomt iOS-autozoom).
- `maxlength` op de inhoudelijke invoervelden.
- Muted tekstkleuren opgehoogd naar **`#5b6b7f`** (desktop) en **`#68686d`** (iOS-stijl mobiel) voor WCAG-contrast. Decoratieve SVG-icoonstrepen bewust ongemoeid.

## Test/productie-opzet

- **Twee gescheiden Firebase-projecten:**
  - **Productie**: `surveillance-dashboard-bab07` (regio europe-west1).
  - **Test/sandbox**: `surveillance-dashboard-test` (projectnummer 235953749425).
- **Hosting**: GitHub Pages onder `dstocks2.github.io/Surveillance_Dashboard/`.
  - Live = `index.html` (productie-config).
  - Test = `test.html` (test-config, met geel **TEST**-label in header, inlogscherm en browsertitel).
- De `firebaseConfig` is het enige verschil tussen beide bestanden. **Nooit** de test-config in productie zetten of andersom.
- **Authorized domain**: `dstocks2.github.io` moet in Firebase Authentication → Settings → Authorized domains staan, anders faalt inloggen.

## Firebase Security Rules

- Gebruik een **gevalideerd vangnet**, geen rigide schema. Actuele rules: `firebase_rules_v2_3_1.json`.
- Type- en lengte-checks per pad. **Gebruik NOOIT `"$other": { ".validate": false }`** — dat blokkeert volledige tak-writes (de app schrijft hele takken zoals `routes`/`medewerkers` in één keer) en veroorzaakt `PERMISSION_DENIED`.
- Bij twijfel over velden: vraag een verse export en finetune op de echte structuur.

## Openstaande volgende stap

Productie-`index.html` (v2.3.0) klaarmaken met de **bab07-config** en zonder TEST-label, daarna:
1. In het productie-project Email/Password aanzetten + dienst-account aanmaken.
2. `firebase_rules_v2_3_1.json` deployen.
3. Testen dat inloggen + opslaan werkt.
4. **Pas dáárna** anonieme login uitzetten in de Console (volgorde belangrijk om lock-out te voorkomen).
