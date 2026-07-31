---
read_when:
    - Sie möchten, dass ein Agent Bereiche der Control UI teilt, fokussiert, schließt oder zwischen ihnen navigiert
    - Sie möchten, dass ein Agent die Seitenleiste, das Terminal oder die Browser-Bereiche ein- oder ausblendet
    - Sie benötigen die `ui.command`-Capability und den Fan-out-Vertrag
sidebarTitle: Screen
summary: Lassen Sie die verbundene Control UI von einem Agenten einrichten
title: Bildschirm
x-i18n:
    generated_at: "2026-07-26T18:41:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df2215db96af29fa6b0db8abad79a0a2787a194dab6d00f9ef32f45521907ae1
    source_path: tools/screen.md
    workflow: 16
---

Das Tool `screen` ermöglicht einem Agenten, die browserbasierte Control UI anzuordnen. Es ist eine
typisierte Oberfläche für Layout und Navigation, keine Screenshot-Erfassung oder Browser-
Automatisierung.

Das Tool wird nur bereitgestellt, wenn der ursprüngliche Client die
Fähigkeit `ui-commands` angibt. Mindestens eine geeignete Control UI muss weiterhin
verbunden sein, wenn das Tool ausgeführt wird; andernfalls gibt das Gateway `UNAVAILABLE` zurück.

## Aktionen

| Aktion                            | Auswirkung                                     | Optionale Eingaben                                |
| --------------------------------- | ------------------------------------------ | ---------------------------------------------- |
| `split_right`                     | Teilt den Bereich der Zielsitzung nach rechts | `sessionKey` (standardmäßig die aktuelle Sitzung) |
| `split_down`                      | Teilt den Bereich der Zielsitzung nach unten     | `sessionKey` (standardmäßig die aktuelle Sitzung) |
| `close_pane`                      | Schließt den Bereich der Zielsitzung              | `sessionKey` (standardmäßig die aktuelle Sitzung) |
| `focus`                           | Fokussiert den Bereich der Zielsitzung              | `sessionKey` (standardmäßig die aktuelle Sitzung) |
| `navigate`                        | Öffnet die Zielsitzung                    | `sessionKey` (standardmäßig die aktuelle Sitzung) |
| `sidebar_show` / `sidebar_hide`   | Blendet die Hauptseitenleiste ein oder aus              | -                                              |
| `terminal_show` / `terminal_hide` | Blendet das Operator-Terminalpanel ein oder aus   | `dock` (`bottom` oder `right`) beim Einblenden      |
| `browser_show` / `browser_hide`   | Blendet das Browserpanel ein oder aus             | `dock` (`bottom` oder `right`) beim Einblenden      |

Ein erfolgreicher Befehl gibt `{ "ok": true }` zurück, nachdem das Gateway
das typisierte Ereignis `ui.command` übertragen hat.

## Routing und Sicherheit

Protokoll v1 sendet den Befehl absichtlich an jede verbundene Control UI, die
`ui-commands` angibt; es wird nicht gezielt ein einzelner Browser-Tab angesprochen. Dies ist relevant, wenn
derselbe Operator mehrere Dashboards geöffnet hat.

Der Gateway-RPC erfordert `operator.write`. Das Tool kann ausschließlich den Darstellungs-
zustand ändern: Es kann weder Pixel auslesen noch Screenshots erstellen, auf beliebige Seiten-
inhalte klicken oder die Berechtigungen der ausgewählten Sitzungs- und Operator-
panels umgehen.

## Verwandte Themen

- [Control UI](/de/web/control-ui)
- [Gateway-Protokoll](/de/gateway/protocol#method-families)
- [Browser-Tool](/de/tools/browser)
