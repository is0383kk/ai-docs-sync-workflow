---
read_when:
    - De gebruikersinterface van het Mac-menu of de statuslogica aanpassen
summary: Statuslogica van de menubalk en wat aan gebruikers wordt getoond
title: Menubalk
x-i18n:
    generated_at: "2026-07-27T05:38:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## Wat wordt weergegeven

- De huidige werkstatus van de agent wordt weergegeven in het menubalkpictogram en in de eerste statusrij van het menu.
- De gezondheidsstatus is verborgen zolang er werk actief is; deze keert terug zodra alle sessies inactief zijn.
- Een hoofditem "Context" opent een submenu met recente sessies in plaats van ze in het hoofdmenu uit te vouwen.
- Een blok "Nodes" in het hoofdmenu vermeldt alleen gekoppelde **apparaten** (uit `node.list`), geen client-/aanwezigheidsvermeldingen.
- Een hoofdsectie "Gebruik" verschijnt onder Context wanneer momentopnamen van providergebruik beschikbaar zijn, gevolgd door kostendetails indien beschikbaar.
- **Snelle chat** opent het zwevende invoerveld voor de hoofdsessie; de huidige algemene sneltoets staat naast het item.

## Statusmodel

- Bron: `WorkActivityStore` (`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`).
- Gebeurtenissen komen binnen als `ControlAgentEvent` met een `runId`; de handler (`ControlChannel.routeWorkActivity`) leest `sessionKey` uit de gebeurtenispayload en gebruikt standaard `"main"` als dit ontbreekt.
- Prioriteit: de hoofdsessie (standaard `sessionKey == "main"`) heeft altijd voorrang. Als de hoofdsessie actief is, wordt de status ervan onmiddellijk weergegeven. Als de hoofdsessie inactief is, wordt in plaats daarvan de laatst actieve niet-hoofdsessie weergegeven. De store wisselt niet tijdens activiteit; deze wisselt alleen wanneer de huidige sessie inactief wordt of de hoofdsessie actief wordt.
- Activiteitstypen:
  - `job`: uitvoering van opdrachten op hoog niveau (`state: started|streaming|done|error|...`).
  - `tool`: `phase: start|result` met `name`, optioneel `meta`/`args`.

## IconState-enum (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (debug-overschrijving)

### ActivityKind -> badgesymbool

`ActivityKind` omvat een `ToolKind` (`bash`, `read`, `write`, `edit`, `attach`, `other`) of een losse `job`. Elk type wordt toegewezen aan een SF Symbols-badge die over het beestjespictogram wordt getekend (`IconState.badgeSymbolName`):

| Type            | Symbool                            |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### Visuele toewijzing

- `idle`: normaal beestje, geen badge.
- `workingMain`: badge met symbool, volledige tint (prominentie `.primary`), animatie waarbij de pootjes "werken".
- `workingOther`: badge met symbool, gedempte tint (prominentie `.secondary`), geen scharrelbeweging.
- `overridden`: gebruikt het gekozen symbool/de gekozen tint ongeacht de werkelijke activiteit.

## Context-submenu

- Het hoofdmenu toont één rij "Context" met het aantal sessies/de status; deze opent een submenu (`MenuSessionsInjector`).
- De kop van het submenu toont het aantal actieve sessies in de afgelopen 24 uur.
- Elke sessierij behoudt de tokenbalk, ouderdom, voorvertoning, schakelaar voor denken/uitgebreide uitvoer en acties voor opnieuw instellen, comprimeren en verwijderen.
- Berichten over laden, verbroken verbindingen en fouten bij het laden van sessies worden binnen het Context-submenu weergegeven.
- De secties voor gebruik en kosten blijven op hoofdniveau onder Context staan, zodat ze in één oogopslag zichtbaar blijven zonder het submenu te openen.

## Tekst van de statusrij (menu)

- Zolang er werk actief is: `<Session role> · <activity label>` (`"\(roleLabel) · \(activity.label)"` in `MenuContentView`), waarbij het rollabel `Main` of `Other` is.
- Bij inactiviteit: valt terug op het gezondheidsoverzicht.

## Verwerking van gebeurtenissen

- Bron: gebeurtenissen van het besturingskanaal `agent`, gerouteerd door `ControlChannel.routeWorkActivity(from:)`.
- Geparseerde velden:
  - `stream: "job"` met `data.state` voor starten/stoppen.
  - `stream: "tool"` met `data.phase`, `data.name`, optioneel `data.meta`/`data.args`.
- Toollabels zijn afkomstig van `ToolDisplayRegistry.resolve(name:args:meta:)`; niet-herkende namen vallen terug op de onbewerkte toolnaam.

## Debug-overschrijving

- Settings > Debug > keuzelijst "Icon override":
  - `System (auto)` (standaard)
  - `Working: main` / `Working: other` (per tooltype: bash, lezen, schrijven, bewerken, overig)
  - `Idle`
- Opgeslagen onder sleutel `openclaw.iconOverride` van `UserDefaults`; toegewezen aan `IconState.overridden`.

## Testchecklist

- Start een taak in de hoofdsessie: het pictogram wisselt onmiddellijk en de statusrij toont het hoofdlabel.
- Start een taak in een niet-hoofdsessie terwijl de hoofdsessie inactief is: het pictogram/de status toont de niet-hoofdsessie en blijft stabiel totdat deze is voltooid.
- Start de hoofdsessie terwijl een andere sessie actief is: het pictogram wisselt onmiddellijk naar de hoofdsessie.
- Snelle opeenvolgingen van tools: de badge flikkert niet (respijtperiode van 2s voordat een voltooide tool wordt gewist, `WorkActivityStore.toolResultGrace`).
- De gezondheidsrij verschijnt opnieuw zodra alle sessies inactief zijn.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [Menubalkpictogram](/nl/platforms/mac/icon)
