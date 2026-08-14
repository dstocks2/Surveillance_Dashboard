# Surveillance Dashboard

Single-file HTML-webapplicatie voor de beveiliging van **UMC Utrecht**: diensten beheren, beveiligingsrondes loggen en live samenwerken over meerdere apparaten (iPads, iPhones, desktops). Draait op 6–7 gelijktijdige sessies en synchroniseert live via Firebase Realtime Database.

**Actuele versie:** v3.60.8

## Functionaliteit

- **Live** — actieve rondes, dienststatus en posten realtime over alle apparaten.
- **Registratie** — beveiligingsrondes starten, loggen en retourneren (solo + groep).
- **Planning** — dagplanning en werkzaamheden per dagdeel.
- **Op dienst** — twee-panel roster met tijdelijke rol-override per medewerker.
- **Posten** — per-dienst capaciteit, bezettingsringen en statusbanners.
- **Diensttelefoons** — koppeling van servicetelefoons per apparaat/dienst.
- **Statistieken** — overzichten voor accounts met statistieken-rol.
- **Instellingen** — beheer van medewerkers, routes, schema, dagdelen, posten en toestellen (beheerder-only).

## Techniek

- **Frontend:** Vanilla HTML/CSS/JS in één bestand, geen externe libraries behalve de Firebase SDK (10.12.0).
- **Backend:** Firebase Realtime Database (`europe-west1`).
- **Auth:** Email/password met gedeeld dienst-account. Drie accounttypes: operationeel (bewakers), statistieken, beheerder.
- **Hosting:** GitHub Pages — `dstocks2.github.io/Surveillance_Dashboard/`. Productie = `index.html`.

## Architectuur

Twee `<script>`-blokken: een `type="module"` blok (Firebase SDK + config + auth-gate) en een niet-module blok (app-logica). Het module-script vertaalt de app-API naar het v3 id-schema via een RTDB-shim met per-item diff-writes.

**Firebase-paden:**
- `/meta` — appVersion, schemaVersion, beheerders, statistieken
- `/config` — med, routes, routeCats, rollen, dagdelen, schema, gedrag, posten, toestellen (id-objects, beheerder-write)
- `/ops` — dienst, active, log, notities, meldingen, toestelKoppeling (push-id, guard-write)
- `/audit/log` — append-only auditlog

## Omgevingen

- **Productie:** `surveillance-dashboard-bab07` (europe-west1)
- **Test:** `surveillance-dashboard-test`

Configs nooit wisselen tussen omgevingen. `dstocks2.github.io` moet in de Authorized Domains staan.

## Ontwikkeling

Single-file: patch gericht, hoog de versie op drie plekken op (`APP_VERSION`, version-badge, bestandsnaam), draai `node --check` op beide scriptblokken en `csp_hash.py` als laatste stap. Rules worden apart geleverd en altijd vóór de HTML gedeployed.

## Beveiliging

- XSS-hardening: state houdt gedecodeerde waarden, escaping op renderpunt via `escHtml()`; `safeColor()` voor hex-kleuren.
- CSP actief met per-scriptblok SHA-256 hashes (`csp_hash.py`).
- Gevalideerde Firebase Security Rules per pad (type- en lengtechecks).

---

Enige ontwikkelaar: **Danny Stockschen** ([@dstocks2](https://github.com/dstocks2))
