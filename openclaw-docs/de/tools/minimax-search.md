---
read_when:
    - Sie möchten MiniMax für `web_search` verwenden
    - Sie benötigen einen MiniMax-Token-Plan-Schlüssel oder ein OAuth-Token
    - Sie benötigen Hinweise zum MiniMax-Suchhost für China bzw. weltweit.
summary: MiniMax-Suche über die Such-API des Token-Plans
title: MiniMax-Suche
x-i18n:
    generated_at: "2026-07-26T18:10:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb851614bbe43f011e07fe3e80d5390f1ba515f3e00ba749c91999617ad2d1e2
    source_path: tools/minimax-search.md
    workflow: 16
---

OpenClaw unterstützt MiniMax über die MiniMax Token Plan Search API als `web_search`-Provider. Sie gibt strukturierte Suchergebnisse mit Titeln, URLs, Textauszügen und verwandten Suchanfragen zurück.

## Zugangsdaten für einen Token Plan abrufen

<Steps>
  <Step title="Schlüssel erstellen">
    Erstellen oder kopieren Sie einen MiniMax-Token-Plan-Schlüssel von der
    [MiniMax-Plattform](https://platform.minimax.io/user-center/basic-information/interface-key).
    OAuth-Konfigurationen können stattdessen `MINIMAX_OAUTH_TOKEN` wiederverwenden.
  </Step>
  <Step title="Schlüssel speichern">
    Legen Sie `MINIMAX_CODE_PLAN_KEY` in der Gateway-Umgebung fest oder konfigurieren Sie ihn über:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw akzeptiert außerdem `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` und
`MINIMAX_API_KEY` als Umgebungsvariablen-Aliasse, die nach
`MINIMAX_CODE_PLAN_KEY` in dieser Reihenfolge geprüft werden. `MINIMAX_API_KEY` sollte auf Zugangsdaten
für einen Token Plan mit aktivierter Suche verweisen; gewöhnliche API-Schlüssel für MiniMax-Modelle werden
vom Token-Plan-Suchendpunkt möglicherweise nicht akzeptiert.

## Konfiguration

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // optional, wenn eine MiniMax-Token-Plan-Umgebungsvariable festgelegt ist
            region: "global", // oder "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**Alternative über die Umgebung:** Legen Sie `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`,
`MINIMAX_OAUTH_TOKEN` oder `MINIMAX_API_KEY` in der Gateway-Umgebung fest.
Bei einer Gateway-Installation tragen Sie die Variable in `~/.openclaw/.env` ein.

## Regionsauswahl

MiniMax Search verwendet diese Endpunkte:

- Global: `https://api.minimax.io/v1/coding_plan/search`
- CN: `https://api.minimaxi.com/v1/coding_plan/search`

Wenn `plugins.entries.minimax.config.webSearch.region` nicht festgelegt ist, bestimmt OpenClaw
die Region in dieser Reihenfolge:

1. Plugin-eigenes `webSearch.region`
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

Das bedeutet, dass ein CN-Onboarding oder `MINIMAX_API_HOST=https://api.minimaxi.com/...`
MiniMax Search automatisch ebenfalls auf dem CN-Host belässt.

Auch wenn Sie MiniMax über den OAuth-Pfad `minimax-portal` authentifiziert haben,
wird die Websuche weiterhin mit der Provider-ID `minimax` registriert; die Basis-URL des OAuth-Providers
dient als Regionshinweis für die Auswahl des CN-/Global-Hosts, und `MINIMAX_OAUTH_TOKEN`
kann als Bearer-Zugangsdaten für MiniMax Search dienen.

## Unterstützte Parameter

| Parameter | Typ     | Einschränkungen  | Beschreibung                                                                       |
| --------- | ------- | ---------------- | ---------------------------------------------------------------------------------- |
| `query`   | Zeichenfolge | erforderlich     | Zeichenfolge der Suchanfrage.                                                      |
| `count`   | Ganzzahl | 1-10, Standard 5 | Anzahl der zurückzugebenden Ergebnisse. OpenClaw kürzt die zurückgegebene Liste auf diese Größe. |

Provider-spezifische Filter werden derzeit nicht unterstützt.

## Verwandte Themen

- [Übersicht zur Websuche](/de/tools/web) -- alle Provider und automatische Erkennung
- [MiniMax](/de/providers/minimax) -- Einrichtung von Modellen, Bildern, Sprache und Authentifizierung
