---
read_when:
    - Anpassen der Mac-Menüoberfläche oder Statuslogik
summary: Statuslogik der Menüleiste und für Benutzer sichtbare Informationen
title: Menüleiste
x-i18n:
    generated_at: "2026-07-26T18:27:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## Was angezeigt wird

- Der aktuelle Arbeitsstatus des Agenten wird im Menüleistensymbol und in der ersten Statuszeile des Menüs angezeigt.
- Der Systemzustand wird ausgeblendet, solange Arbeit aktiv ist; er wird wieder angezeigt, sobald alle Sitzungen inaktiv sind.
- Ein übergeordneter Eintrag „Kontext“ öffnet ein Untermenü mit den letzten Sitzungen, statt sie im Hauptmenü aufzuklappen.
- Ein Block „Nodes“ im Hauptmenü führt nur gekoppelte **Geräte** auf (aus `node.list`), keine Client-/Präsenzeinträge.
- Wenn Momentaufnahmen der Provider-Nutzung verfügbar sind, wird unter „Kontext“ ein übergeordneter Abschnitt „Nutzung“ angezeigt, gefolgt von Kostendetails, sofern verfügbar.
- **Schnellchat** öffnet den schwebenden Editor der Hauptsitzung; das aktuelle globale Tastaturkürzel wird neben dem Eintrag angezeigt.

## Zustandsmodell

- Quelle: `WorkActivityStore` (`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`).
- Ereignisse gehen als `ControlAgentEvent` mit einer `runId` ein; der Handler (`ControlChannel.routeWorkActivity`) liest `sessionKey` aus den Ereignisdaten und verwendet standardmäßig `"main"`, wenn der Wert fehlt.
- Priorität: Die Hauptsitzung (standardmäßig `sessionKey == "main"`) hat immer Vorrang. Wenn die Hauptsitzung aktiv ist, wird ihr Zustand sofort angezeigt. Wenn die Hauptsitzung inaktiv ist, wird stattdessen die zuletzt aktive Neben­sitzung angezeigt. Der Speicher wechselt nicht während einer laufenden Aktivität; er wechselt nur, wenn die aktuelle Sitzung inaktiv wird oder die Hauptsitzung aktiv wird.
- Aktivitätsarten:
  - `job`: Ausführung eines übergeordneten Befehls (`state: started|streaming|done|error|...`).
  - `tool`: `phase: start|result` mit `name`, optional `meta`/`args`.

## IconState-Enumeration (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (Debug-Überschreibung)

### ActivityKind -> Abzeichensymbol

`ActivityKind` umschließt entweder ein `ToolKind` (`bash`, `read`, `write`, `edit`, `attach`, `other`) oder ein einfaches `job`. Jedem Typ ist ein SF-Symbol als Abzeichen zugeordnet, das über dem Tierchensymbol gezeichnet wird (`IconState.badgeSymbolName`):

| Art             | Symbol                             |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### Visuelle Zuordnung

- `idle`: normales Tierchen, kein Abzeichen.
- `workingMain`: Abzeichen mit Symbol, volle Einfärbung (`.primary`-Hervorhebung), „Arbeits“-Animation der Beine.
- `workingOther`: Abzeichen mit Symbol, gedämpfte Einfärbung (`.secondary`-Hervorhebung), kein Herumhuschen.
- `overridden`: verwendet unabhängig von der tatsächlichen Aktivität das ausgewählte Symbol und die ausgewählte Einfärbung.

## Kontext-Untermenü

- Das Hauptmenü zeigt eine Zeile „Kontext“ mit Sitzungsanzahl und -status; sie öffnet ein Untermenü (`MenuSessionsInjector`).
- Die Kopfzeile des Untermenüs zeigt die Anzahl der aktiven Sitzungen innerhalb der letzten 24 Stunden.
- Jede Sitzungszeile behält ihre Token-Leiste, ihr Alter, ihre Vorschau, den Umschalter für Nachdenken/ausführliche Ausgabe sowie die Aktionen zum Zurücksetzen, Komprimieren und Löschen.
- Meldungen zum Laden, zu einer getrennten Verbindung und zu Fehlern beim Laden von Sitzungen werden im Kontext-Untermenü angezeigt.
- Die Abschnitte zu Nutzung und Kosten verbleiben unter „Kontext“ auf der Hauptebene, damit sie ohne Öffnen des Untermenüs auf einen Blick sichtbar sind.

## Text der Statuszeile (Menü)

- Während Arbeit aktiv ist: `<Session role> · <activity label>` (`"\(roleLabel) · \(activity.label)"` in `MenuContentView`), wobei die Rollenbezeichnung `Main` oder `Other` lautet.
- Bei Inaktivität: greift auf die Zusammenfassung des Systemzustands zurück.

## Ereignisaufnahme

- Quelle: `agent`-Ereignisse des Steuerungskanals, weitergeleitet durch `ControlChannel.routeWorkActivity(from:)`.
- Analysierte Felder:
  - `stream: "job"` mit `data.state` für Start/Stopp.
  - `stream: "tool"` mit `data.phase`, `data.name`, optional `data.meta`/`data.args`.
- Werkzeugbezeichnungen stammen aus `ToolDisplayRegistry.resolve(name:args:meta:)`; nicht aufgelöste Namen greifen auf den unverarbeiteten Werkzeugnamen zurück.

## Debug-Überschreibung

- Settings > Debug > „Icon override“-Auswahl:
  - `System (auto)` (Standard)
  - `Working: main` / `Working: other` (je Werkzeugart: bash, read, write, edit, other)
  - `Idle`
- Gespeichert unter dem Schlüssel `openclaw.iconOverride` in `UserDefaults`; zugeordnet zu `IconState.overridden`.

## Testcheckliste

- Auftrag der Hauptsitzung auslösen: Das Symbol wechselt sofort, und die Statuszeile zeigt die Bezeichnung der Hauptsitzung.
- Auftrag einer Neben­sitzung auslösen, während die Hauptsitzung inaktiv ist: Symbol und Status zeigen die Neben­sitzung an und bleiben stabil, bis sie abgeschlossen ist.
- Hauptsitzung starten, während eine andere Sitzung aktiv ist: Das Symbol wechselt sofort zur Hauptsitzung.
- Schnelle Werkzeugfolgen: Das Abzeichen flackert nicht (Kulanzzeit von 2s vor dem Entfernen eines abgeschlossenen Werkzeugs, `WorkActivityStore.toolResultGrace`).
- Die Zeile zum Systemzustand wird wieder angezeigt, sobald alle Sitzungen inaktiv sind.

## Verwandte Themen

- [macOS-App](/de/platforms/macos)
- [Menüleistensymbol](/de/platforms/mac/icon)
