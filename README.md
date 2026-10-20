# Controlebeheer

Lokale desktop-applicatie voor het **plannen, uitvoeren en opvolgen van controles** — zonder accounts, zonder cloud, zonder API-sleutels. Alle gegevens staan in één SQLite-database op de eigen computer. Ontwikkeld met Electron, React, TypeScript en Vite; volledig in het Nederlands.

---

## Functies

| Scherm | Wat het doet |
| --- | --- |
| **Dashboard** | KPI-kaarten (controles, vandaag op de planning, verlopen, open actiepunten), uitvoeringen per maand, statusoverzicht, komende 14 dagen en open actiepunten. |
| **Controles** | Aanmaken/bewerken/kopiëren/verwijderen van controles met filters (status, categorie, locatie, verantwoordelijke, prioriteit, periode), sorteerbare tabel en detailvenster met drie tabs: *Overzicht*, *Historie* en *Acties*. |
| **Planning** | Kalender met maand-, week- en dagweergave; geplande controles én al uitgevoerde ronden, met vandaag-markering en snelnavigatie. |
| **Uitvoeren** | Werkvoorraad (achterstallig / vandaag / ≤ 14 dagen / alle) en een 3-staps-wizard per uitvoering: voorbereiding → controlepunten (voldoet / afgekeurd / n.v.t. + foto's/bijlagen) → samenvatting. Elk uitgevoerd controlepunt wordt als **kopie** bewaard; latere wijzigingen in het sjabloon veranderen oude uitvoeringen niet. |
| **Sjablonen** | Herbruikbare controlelijsten met het herordenen van punten, kopiëren en veilig aanpassen: punten van een sjabloon worden bij elke uitvoering als kopie bewaard, dus het sjabloon mag zonder risico worden gewijzigd. Bij eerste start worden enkele voorbeeldsjablonen aangeleverd. |
| **Actiepunten** | Automatisch gegenereerd bij **afgekeurde controlepunten**, plus losse acties. Statusbeheer (open / bezig / opgelost) waarbij „opgelost" een expliciete bevestiging (met vinkje) vergt. Volledige historie en opmerkingen per actie. |
| **Rapportages** | Periodevoorinstellingen en aangepaste periodes, KPI's, donut- en staafdiagrammen, sectierapport; export naar **CSV** (puntkomma + BOM, opent direct in Excel), **PDF** (via Electron printToPDF) en afdrukken. |
| **Instellingen** | Thema (donker/licht), datumnotatie, 24-uurs klok, standaardpagina, herinneringen (actief, dagen vooraf, achterstallig, interval ― met testknop), back-up **exporteren/importeren** met vooraf-zien en bevestiging, database-onderhoud (vrijmaken opslag, opruimen zwevende bestanden, VACUUM) en app-info. |

Daarnaast: globaal zoeken via de topbalk (Ctrl+K) over controles, uitvoeringen, sjablonen en actiepunten; Windows-meldingen voor aanstaande en achterstallige controles (zonder dubbele meldingen); in- en uitklapbare zijbalk; donker-/lichtthema.

## Technologie

- **Electron 38** met main-, preload- en renderer-processen, `contextIsolation` en een strikte CSP (geen externe verbindingen).
- **React 18 + TypeScript + Vite 6** voor de renderer (HashRouter met 8 routes).
- **SQLite via `node:sqlite`** (`DatabaseSync` in de meegeleverde Node 22-runtime van Electron) — géén native modules die gecompileerd moeten worden.
- Strikte **IPC-whitelist** (~45 kanalen) in de preload-brug; de renderer kan niets anders aanroepen.
- Handgemaakt CSS-ontwerpsysteem (design-tokens, donker navy-standaard, licht via `[data-theme]`).
- Installer via **electron-builder + NSIS** (Windows x64).

## Systeemvereisten

- **Windows 10/11 (x64)** voor de geïnstalleerde app.
- **Node.js ≥ 20** alleen voor de ontwikkelingstaken; de app zelf draait op de gebundelde Electron-runtime (geen runtime-dependencies).

## Installeren (eindgebruiker)

1. Download `Controlebeheer Setup 1.0.0.exe` (uit de `release/`-map).
2. Voer het bestand uit. Omdat de installer niet is ondertekend, kan Windows SmartScreen een waarschuwing tonen: kies **Meer info → Toch uitvoeren**.
3. Volg de installatiewizard (installatiemap mag gewijzigd worden) en start Controlebeheer.

De app maakt bij eerste start zelf de database en mapstructuur aan; niets wordt buiten de eigen gebruikersmap geschreven (plus optioneel de door u gekozen back-uplocatie).

## Ontwikkelen

```powershell
# 1. Dependencies installeren (aanbevolen Node 20 LTS of nieuwer)
npm install

# Mocht npm 12 vragen om 'install scripts' goed te keuren:
npm install-scripts approve electron esbuild

# 2. Ontwikkelmodus starten (hot reload voor de renderer)
npm run dev
```

- `scripts/dev.mjs` compileert de main- en preloadprocessen, start de Vite-devserver (poort 5173) en opent Electron daartegen.
- Wijzigingen in main/preload vragen een herstart van `npm run dev`; de renderer herlaadt automatisch.
- De productiebuild is niet van de devserver afhankelijk: `dist/main/main.cjs` laadt de gebouwde renderer uit `dist/renderer`.

### Alle scripts

| Script | Wat het doet |
| --- | --- |
| `npm run dev` | Ontwikkelen met Vite-devserver + hot reload. |
| `npm run build` | Bouwt main + preload + renderer naar `dist/`. |
| `npm run typecheck` | TypeScript-controle voor Node- en webprojecten (twee tsconfigs). |
| `npm run smoke` | Bouwt en neemt screenshots van alle pagina's (zie hieronder). |
| `npm run dist` | Bouwt plus NSIS-installer in `release/`. |
| `npm run icons` | Genereert `resources/icon.ico` opnieuw uit `icon.png` (gebeurt ook automatisch bij `npm install`). |

### Smoke-test

De smokecheck start de echte app (productiebuild) per route, navigeert daarheen, schrijft één screenshot per pagina naar de projectmap en logt daarna de werkelijk gerenderde pagina (`hash` + `h1`) als extra controle:

```powershell
node scripts/smoke.mjs                              # alleen het dashboard
node scripts/smoke.mjs /controls /planning /execute # geselecteerde routes
node scripts/smoke.mjs /templates /actions /reports /settings
```

Resultaat: `smoke-dashboard.png`, `smoke-controls.png`, enz. In de log staat per route `dom na navigatie: {"hash":"#/…","h1":"…"}` — de echte DOM-status, onafhankelijk van de compositor. Verouderde screenshots (een bekende `capturePage`-race in Electron) zijn hiermee ondervangen: smoke wacht vóór elk screenshot op een nieuw compositing-frame (dubbele `requestAnimationFrame`).

Optionele omgevingsvariabelen: `CB_SMOKE_WAIT` (standaard 5000, wachttijd vóór navigeren), `CB_SMOKE_NAV_WAIT` (standaard 900, wachttijd ná navigeren), `CB_SMOKE_ROUTE`, `CB_SMOKE_OUT`.

### Installer bouwen

```powershell
npm run dist
```

Resultaat in `release/`:

- `Controlebeheer Setup 1.0.0.exe` — NSIS-installer (met keuze van de installatiemap en snelkoppeling op het bureaublad).
- `win-unpacked/Controlebeheer.exe` — draagbare versie die direct kan draaien.

Handmatige smoketest van de packaging:

```powershell
$env:CB_SMOKE='1'; $env:CB_SMOKE_OUT="$PWD\smoke-packaged.png"
.\release\win-unpacked\Controlebeheer.exe
```

## Wat moet u zeker weten

- **Geen code-ondertekening**: de installer/exe is niet ondertekend; Windows toont bij eerste start een waarschuwing over een onbekende uitgever. Wilt u het bestand ondertekenen? Voeg uw certificaat toe in `electron-builder.json` (velden `win.certificateFile` / `certificatePassword`).
- **Meldingen**: herinneringen draaien in het mainproces op een configureerbaar interval; dezelfde controle wordt niet dubbel gemeld. De testknop in Instellingen stuurt direct een voorbeeldmelding.
- **Tijdzones**: alle opgeslagen datums en tijdstippen zijn lokaal (`YYYY-MM-DD` resp. `YYYY-MM-DDTHH:mm`); er wordt nooit naar UTC omgerekend, dus "vandaag" is altijd de lokale dag.
- **Onveranderlijke historie**: uitvoeringen kopiëren hun controlepunten op uitvoeringsmoment; verwijderen van een uitvoering vraagt altijd eerst bevestiging en de controlestatus wordt daarna herberekend.
- **Opgeloste acties**: de status "opgelost" kan alleen mét expliciet bevestigingsvinkje worden gezet; daarna is bewerken en opmerkingen toevoegen niet meer mogelijk.

## Gegevensopslag & back-ups

| Wat | Waar |
| --- | --- |
| SQLite-database | `%APPDATA%\Controlebeheer\controlebeheer.db` |
| Bijlagen (foto's/documenten) | `%APPDATA%\Controlebeheer\bijlagen\` |
| Back-ups | Standaardmap uit Instellingen of door u gekozen locatie via 'Exporteren' |

**Back-upbeleid**: bij exporteren kiest u een bestandsnaam; het bestand is een JSON-database-dump (alle tabellen). Bij **importeren** wordt de inhoud eerst gepresenteerd ter controle (aantallen per tabel), vóórdat de actieve database wordt vervangen. Vóór vervanging wordt automatisch een veiligheidskopie van de huidige database gemaakt; als de import faalt (de validatie controleert structuur en verplichte velden) blijft de bestaande database onaangetast. Na een geslaagde import worden alle overzichten direct ververst.

**Onderhoud** (Instellingen → Database): "Opslag vrijmaken" toont het bestandsverbruik en voert daarna VACUUM uit; "Opschonen" verwijdert bijlagebestanden waarnaar geen enkele databaseverwijzing meer wijst. Beide vragen eerst bevestiging.

## Globale sneltoetsen

| Sneltoets | Actie |
| --- | --- |
| `Ctrl+K` | Zet de focus in het zoekveld van de topbalk |
| `Escape` | Sluit het zoekpaneel en wist de zoekopdracht (als het zoekveld focus heeft) |
| `Ctrl+Shift+I` | Developer Tools openen/sluiten (menu: Weergave) |

## Projectstructuur

```
├── electron-builder.json       # Installer-config (NSIS, Windows x64)
├── package.json                # Scripts en allowScripts-beleid
├── index.html                  # Renderer-host (CSP wordt in productie ingebouwd)
├── scripts/
│   ├── dev.mjs                 # Vite-devserver + Electron
│   ├── smoke.mjs               # Smoketest: screenshot per route
│   └── make-icon.mjs           # Genereert resources/icon.ico uit icon.png
├── resources/                  # App-iconen (ico/png)
├── src/
│   ├── main/                   # Electron-main: app, window, SQLite-DB, IPC en services
│   │   ├── app.ts              # Bootstrap, DB-migraties, seed, smoke-modus
│   │   ├── db/                 # connectie + migraties + seed (node:sqlite)
│   │   ├── ipc/                # alle IPC-handlers (whitelist)
│   │   ├── window/             # vensterbeheer (titleBarOverlay-systeem)
│   │   └── services/           # controls, runs, templates, actions, statistics,
│   │                           #   search, rapportage, export, back-up, bijlagen,
│   │                           #   herinneringen, instellingen
│   ├── renderer/               # React UI
│   │   ├── App.tsx             # Router (8 routes, hash-router)
│   │   ├── layout/AppLayout.tsx# Zijbalk, topbalk met globaal zoeken, pushkanalen
│   │   ├── pages/              # 8 paginacomponenten
│   │   ├── components/         # Icon-set (~45 SVG-iconen), UI-bibliotheek
│   │   ├── state/              # Instellingen-, bevestigings- en toast-contexten
│   │   ├── lib/                # API-brug, hooks, formattering, bus
│   │   └── styles/             # Design-tokens, base.css, components.css
│   └── shared/                 # Gedeelde types, constanten, date-recurrence
└── release/                    # Uitvoer van npm run dist (niet in versiebeheersing)
```

**Gemaakt voor de praktijk van dagelijkse controlevoering: elke knop doet wat hij belooft, de historie blijft bewaard, en niemand hoeft een account aan te maken om te beginnen.**
