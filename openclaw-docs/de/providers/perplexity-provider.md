---
read_when:
    - Sie möchten Perplexity als Provider für die Websuche konfigurieren
    - Sie benötigen den Perplexity-API-Schlüssel oder die OpenRouter-Proxy-Einrichtung.
summary: Einrichtung des Perplexity-Websuch-Providers (API-Schlüssel, Suchmodi, Filterung)
title: Perplexity
x-i18n:
    generated_at: "2026-07-26T18:02:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

Das Perplexity-Plugin registriert einen `web_search`-Provider mit zwei Transportwegen: der
nativen Perplexity Search API (strukturierte Ergebnisse mit Filtern) und Perplexity
Sonar Chat Completions, direkt oder über OpenRouter (KI-generierte Antworten mit
Quellenangaben).

<Note>
Diese Seite behandelt die Einrichtung des Perplexity-**Providers**. Informationen zum Perplexity-**Tool** (wie der Agent es verwendet) finden Sie unter [Perplexity-Suche](/de/tools/perplexity-search).
</Note>

| Eigenschaft | Wert                                                                   |
| ----------- | ---------------------------------------------------------------------- |
| Typ         | Websuch-Provider (kein Modell-Provider)                                |
| Authentifizierung | `PERPLEXITY_API_KEY` (nativ) oder `OPENROUTER_API_KEY` (über OpenRouter) |
| Konfigurationspfad | `plugins.entries.perplexity.config.webSearch.apiKey`                   |
| Überschreibungen | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`       |
| Schlüssel abrufen | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## Plugin installieren

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Erste Schritte

<Steps>
  <Step title="API-Schlüssel festlegen">
    ```bash
    openclaw configure --section web
    ```

    Alternativ können Sie den Schlüssel direkt festlegen:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    Ein als `PERPLEXITY_API_KEY` oder `OPENROUTER_API_KEY` in der Gateway-
    Umgebung exportierter Schlüssel funktioniert ebenfalls.

  </Step>
  <Step title="Suche starten">
    `web_search` erkennt Perplexity automatisch, sobald dessen Schlüssel als
    Suchzugangsdaten verfügbar ist; keine weitere Einrichtung ist erforderlich. So legen Sie den Provider ausdrücklich fest:

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## Suchmodi

Das Plugin bestimmt den Transportweg in dieser Reihenfolge:

1. `webSearch.baseUrl` oder `webSearch.model` festgelegt: Die Weiterleitung erfolgt unabhängig vom Schlüsseltyp immer über Sonar Chat Completions an diesen Endpunkt.
2. Andernfalls bestimmt die Quelle des Schlüssels den Endpunkt: Das Präfix eines konfigurierten Schlüssels legt den Transportweg fest (die Konfiguration hat Vorrang vor Umgebungsvariablen); ein Umgebungsschlüssel verwendet direkt den zugehörigen Endpunkt.

| Schlüsselpräfix | Transportweg                                               | Funktionen                                        |
| --------------- | ---------------------------------------------------------- | ------------------------------------------------- |
| `pplx-`    | Native Perplexity Search API (`https://api.perplexity.ai`) | Strukturierte Ergebnisse, Domain-/Sprach-/Datumsfilter |
| `sk-or-`   | OpenRouter (`https://openrouter.ai/api/v1`), Sonar-Modell  | KI-generierte Antworten mit Quellenangaben        |

Ein konfigurierter Schlüssel mit einem anderen Präfix verwendet ebenfalls die native Search API. Der
Chat-Completions-Pfad verwendet standardmäßig das Modell `perplexity/sonar-pro`; Sie können es
mit `plugins.entries.perplexity.config.webSearch.model` überschreiben.

## Filterung der nativen API

| Filter                               | Beschreibung                                                    | Transportweg |
| ------------------------------------ | --------------------------------------------------------------- | ------------ |
| `count`                              | Ergebnisse pro Suche, 1-10 (Standardwert: 5)                    | Nur nativ    |
| `freshness`                          | Aktualitätszeitraum: `day`, `week`, `month`, `year`                  | Beide        |
| `country`                            | Zweistelliger Ländercode (`us`, `de`, `jp`)                  | Nur nativ    |
| `language`                           | ISO-639-1-Sprachcode (`en`, `fr`, `zh`)                    | Nur nativ    |
| `date_after` / `date_before`         | Veröffentlichungszeitraum im Format `YYYY-MM-DD`               | Nur nativ    |
| `domain_filter`                      | Maximal 20 Domains; Zulassungsliste oder mit `-` präfixierte Sperrliste, niemals gemischt | Nur nativ    |
| `max_tokens` / `max_tokens_per_page` | Inhaltsbudget für alle Ergebnisse / pro Seite                   | Nur nativ    |

Filter, die nur nativ unterstützt werden, geben im Chat-Completions-Pfad einen aussagekräftigen Fehler zurück.
`freshness` kann nicht mit `date_after`/`date_before` kombiniert werden.

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Umgebungsvariable für Daemon-Prozesse">
    <Warning>
    Ein nur in einer interaktiven Shell exportierter Schlüssel ist für einen
    als launchd-/systemd-Daemon ausgeführten Gateway nicht sichtbar, sofern diese Umgebung nicht ausdrücklich
    importiert wird. Legen Sie den Schlüssel in `~/.openclaw/.env` oder über `env.shellEnv` fest, damit der
    Gateway-Prozess darauf zugreifen kann. Die vollständige Rangfolge finden Sie unter [Umgebungsvariablen](/de/help/environment).
    </Warning>
  </Accordion>

  <Accordion title="OpenRouter-Proxy einrichten">
    Um Perplexity-Suchen über OpenRouter weiterzuleiten, legen Sie einen `OPENROUTER_API_KEY`
    (Präfix `sk-or-`) anstelle eines nativen Perplexity-Schlüssels fest. OpenClaw erkennt den
    Schlüssel und wechselt automatisch zum Sonar-Transportweg. Dies ist nützlich, wenn Sie bereits
    die Abrechnung über OpenRouter eingerichtet haben und Provider dort konsolidieren möchten.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Perplexity-Suchtool" href="/de/tools/perplexity-search" icon="magnifying-glass">
    Wie der Agent Perplexity-Suchen aufruft und Ergebnisse interpretiert.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/configuration-reference" icon="gear">
    Vollständige Konfigurationsreferenz einschließlich Plugin-Einträgen.
  </Card>
</CardGroup>
