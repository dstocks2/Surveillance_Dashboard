Surveillance Dashboard — Project Instructies
Wat dit is
Een single-file HTML webapp voor ronderegistratie bij beveiligingsdiensten (UMC Utrecht). Hosting via GitHub Pages, data-synchronisatie via Firebase Realtime Database met anonymous auth. Werkt op tablet, telefoon en desktop.
Huidige versie
v2.1.65 — staat in const APP_VERSION = '...' en als badge op het ⚙ tabblad.
Werkwijze per wijziging

Lees eerst het projectbestand via bash of view
Maak de wijziging zo gericht mogelijk (geen brede refactors)
Verhoog versienummer met 0.0.1 in:

const APP_VERSION = '...'
<span id="app-version-badge">v...</span>


Sla op als Surveillance_Dashboard_vX.X.X.html in /home/claude/ én /mnt/user-data/outputs/
Update memory met nieuwe versie
Houd antwoorden kort — geen overbodige uitleg

Architectuur

State: alles in S object (names, routes, schema, active, log, dienst, sns, sr, tick, tickL, tickTL)
Persistence: Firebase = source of truth, localStorage = fallback
Firebase paden: /dashboard/{names, routes, dienst, active, log, pin, appVersion, meldingen, dagdelen, gedrag, eventLog}
Firebase listeners: fbListen() op root /dashboard, fbListenMeldingen(), fbListenConnected(), fbListenEventLog()
Sanitization: alle Firebase-input via sanitizeFromFirebase()
Nacht-tijden: 00:00–06:59 opgeslagen als h+24 (24–30)

Log-architectuur (v2.1.65+)

/dashboard/log is een object van {pushKey: entry} — geen array meer
Schrijven: fbPushLogEntry(entry) per retour — nooit de hele log overschrijven
Lezen: object → array in fbListen (backward compatible met oud array-formaat)
fbSaveActive() schrijft alleen /dashboard/active, nooit de log
pruneLog() verwijdert individuele entries via fbRemoveLogEntry(fbKey)
wisLog() verwijdert het hele /dashboard/log pad via fbRemove
logInWindow(cutoff) filtert alleen op r.end >= cutoff (niet op r.start)
Dubbele retour geblokkeerd in doRetour() op basis van name+route+start
Log-merge in fbListen dedupliceet op name+route+start

PIN

SHA-256 gehasht opgeslagen in Firebase /dashboard/pin
Nooit plaintext opslaan, tonen of in code schrijven
Firebase is leidend: geen pin in Firebase = localStorage wissen + setup-popup
Cache: 30 min na ontgrendelen (localStorage rn_cfg_unlocked)
PIN invoer via numeriek keypad (4 dots, auto-submit bij 4e cijfer, keyboard-support)
Setup en wijzigen delen dezelfde modal (pin-keypad-overlay)
Vergrendelen → direct naar registratie-tab, geen PIN-venster tonen

Header

Links: logo + divider + "Surveillance Dashboard" + datum/dagdeel subtekst
Rechts: sync-status + divider + dienst-badge + klok (HH:MM)
Klok: verborgen op mobiel (@media max-width:600px), desktop-only
Dienst-badge mobiel: compact ● 4/40 formaat, donkere pill-stijl
Klok-timer: synct op minuutwissel via scheduleHeader()
updateHeader() aangeroepen vanuit render() en elke 30s

Tabs (in volgorde)

Registratie — naam + ronde selecteren → start/retour
Overzicht — wie is onderweg + 24u tijdlijn
Voorgestelde ronde — planning op basis van schema
Op dienst — alfabetische lettergroepen
⚙ Instellingen — PIN-vergrendeld, bevat medewerkers/rondes/dagplanning beheer

Dagdelen

S.dagdelen = array {naam, start, eind, kleur} — tekstKleur auto-afgeleid via autoTekstKleur(hex)
Default DAGDELEN_DEFAULT (vroeg/laat/nacht), sync via /dashboard/dagdelen
Overlap niet toegestaan; wrapping shifts (start > eind) ondersteund
Helpers: hhmmToMin, dagdeelAtMinute, dagdeelAtHour, dagdeelBoundariesInWindow

Meldingen systeem

MELDINGEN[] array lokaal, voegMeldingToe(type, naam, sub, actieLabel, actieFn)
Gedeelde meldingen (grijs) via Firebase /dashboard/meldingen met TTL (instelbaar)
schrijfGedeeldeMelding() altijd NA save()+render() — anders race condition
saveGedrag() waarschuwt bij TTL-verlaging als er actieve gedeelde meldingen zijn

Retour flow

Optimistic update: kaart verdwijnt direct
Firebase write (active via update, log via push) → bij fout: rollback + toast + rode melding
Live-kaarten gebruiken event delegation (elActive.onclick)

Event log

Audit-log in Firebase /dashboard/eventLog, TTL 7 dagen
Types: start, retour, dienst, settings, auto, sync, system, melding, pin, error
UI in instellingen-tab (PIN-vergrendeld): filter, zoek, CSV export

Bekende bugfixes (niet opnieuw introduceren)

_slotStartMs in minuten ipv ms — fixed v2.0.55
cfg-lock dubbele display property — fixed v2.0.54
tl-toggle-icon duplicate ID — fixed v2.0.54
dienst toggle race (schrijfGedeeldeMelding vóór save) — fixed v2.0.62
doStart fire-and-forget zonder rollback — fixed v2.0.64
Dubbele retour bij sync-conflict (twee apparaten) — fixed v2.1.64
logInWindow filterde op r.start waardoor nachtrondes verdwenen — fixed v2.1.64

Conventies

Nederlands voor UI-tekst, Engels voor code/comments
showToast() voor korte meldingen, customConfirm() voor bevestigingen, sanitizeInput() bij alle user-invoer
Mobile-first: alles werkt op kleine schermen
Geen breaking changes zonder waarschuwing

Wat NIET doen

Geen automatische refactors van werkende code
Geen nieuwe Firebase paden zonder rules-update te vermelden
Geen kleuren/uiterlijk-aanpassingen
Geen externe libraries toevoegen
Geen wijzigingen aan firebaseConfig
Nooit de PIN in plaintext schrijven, tonen of opslaan
Nooit de log als array overschrijven via fbUpdate — altijd fbPushLogEntry()
