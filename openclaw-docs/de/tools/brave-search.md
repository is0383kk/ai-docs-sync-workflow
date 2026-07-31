---
read_when:
    - Sie möchten Brave Search für `web_search` verwenden
    - Sie benötigen einen `BRAVE_API_KEY` oder Tarifdetails
summary: Einrichtung der Brave Search API für web_search
title: Brave-Suche
x-i18n:
    generated_at: "2026-07-26T18:51:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw unterstützt die Brave Search API als `web_search`-Provider.

## API-Schlüssel abrufen

1. Erstellen Sie unter [https://brave.com/search/api/](https://brave.com/search/api/) ein Brave-Search-API-Konto.
2. Wählen Sie im Dashboard den **Search**-Tarif aus und generieren Sie einen API-Schlüssel.
3. Speichern Sie den Schlüssel in der Konfiguration oder setzen Sie `BRAVE_API_KEY` in der Gateway-Umgebung.

## Konfigurationsbeispiel

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // oder "llm-context"
            baseUrl: "https://api.search.brave.com", // optionale Überschreibung der Proxy-/Basis-URL
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Providerspezifische Brave-Sucheinstellungen befinden sich unter `plugins.entries.brave.config.webSearch.*`; dies ist der kanonische Konfigurationspfad.

`webSearch.mode` steuert die Brave-Übertragung:

- `web` (Standard): normale Brave-Websuche mit Titeln, URLs und Textausschnitten
- `llm-context`: Brave LLM Context API mit vorab extrahierten Textabschnitten und Quellen zur Fundierung

`webSearch.baseUrl` kann Brave-Anfragen an einen vertrauenswürdigen, Brave-kompatiblen Proxy
oder ein entsprechendes Gateway leiten. OpenClaw hängt `/res/v1/web/search` oder `/res/v1/llm/context` an
die konfigurierte Basis-URL an und nimmt die Basis-URL in den Cache-Schlüssel auf. Öffentliche
Endpunkte müssen `https://` verwenden; `http://` wird nur für vertrauenswürdige Loopback-
oder private Netzwerk-Proxyhosts akzeptiert.

## Tool-Parameter

<ParamField path="query" type="string" required>
Suchanfrage.
</ParamField>

<ParamField path="count" type="number" default="5">
Anzahl der zurückzugebenden Ergebnisse (1–10).
</ParamField>

<ParamField path="country" type="string">
Zweistelliger ISO-Ländercode (z. B. `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
ISO-639-1-Sprachcode für Suchergebnisse (z. B. `en`, `de`, `fr`).
</ParamField>

<ParamField path="search_lang" type="string">
Brave-Suchsprachcode (z. B. `en`, `en-gb`, `zh-hans`).
</ParamField>

<ParamField path="ui_lang" type="string">
ISO-Sprachcode für UI-Elemente.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Zeitfilter – `day` entspricht 24 Stunden.
</ParamField>

<ParamField path="date_after" type="string">
Nur Ergebnisse, die nach diesem Datum veröffentlicht wurden (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Nur Ergebnisse, die vor diesem Datum veröffentlicht wurden (`YYYY-MM-DD`).
</ParamField>

**Beispiele:**

```javascript
// Länder- und sprachspezifische Suche
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// Aktuelle Ergebnisse (vergangene Woche)
await web_search({
  query: "AI news",
  freshness: "week",
});

// Suche nach Datumsbereich
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## Hinweise

- OpenClaw verwendet den Brave-Tarif **Search**. Wenn Sie ein älteres Abonnement haben (z. B. den ursprünglichen Free-Tarif mit 2.000 Anfragen/Monat), bleibt es gültig, enthält jedoch keine neueren Funktionen wie LLM Context oder höhere Ratenlimits.
- Jeder Brave-Tarif umfasst **ein monatlich erneuertes Gratisguthaben von 5 \$**. Der Search-Tarif kostet 5 \$ pro 1.000 Anfragen, sodass das Guthaben 1.000 Anfragen/Monat abdeckt. Legen Sie im Brave-Dashboard Ihr Nutzungslimit fest, um unerwartete Kosten zu vermeiden. Aktuelle Tarife finden Sie im [Brave-API-Portal](https://brave.com/search/api/).
- Der Search-Tarif umfasst den LLM-Context-Endpunkt und Rechte zur KI-Inferenz. Das Speichern von Ergebnissen zum Trainieren oder Abstimmen von Modellen erfordert einen Tarif mit ausdrücklichen Speicherrechten. Siehe die Brave-[Nutzungsbedingungen](https://api-dashboard.search.brave.com/terms-of-service).
- Der Modus `llm-context` gibt fundierte Quelleinträge anstelle der normalen Textausschnittstruktur der Websuche zurück.
- Der Modus `llm-context` unterstützt `freshness` und begrenzte Bereiche mit `date_after` + `date_before`. Er unterstützt `ui_lang` nicht; `date_before` ohne `date_after` wird abgelehnt, da Brave bei benutzerdefinierten Aktualitätsbereichen sowohl ein Start- als auch ein Enddatum verlangt.
- `ui_lang` muss ein Regions-Subtag wie `en-US` enthalten.
- Ergebnisse werden standardmäßig 15 Minuten lang zwischengespeichert (konfigurierbar über `cacheTtlMinutes`).
- Benutzerdefinierte `webSearch.baseUrl`-Werte werden in die Brave-Cache-Identität aufgenommen, sodass
  proxyspezifische Antworten nicht kollidieren.
- Aktivieren Sie das Diagnose-Flag `brave.http`, um bei der Fehlerbehebung Brave-Anfrage-URLs/-Abfrageparameter, Antwortstatus/-dauer sowie Treffer-, Fehlschlag- und Schreibereignisse des Such-Caches zu protokollieren. Das Flag protokolliert niemals den API-Schlüssel oder Antwortinhalte, Suchanfragen können jedoch vertraulich sein.

## Verwandte Themen

- [Übersicht zur Websuche](/de/tools/web) -- alle Provider und automatische Erkennung
- [Perplexity Search](/de/tools/perplexity-search) -- strukturierte Ergebnisse mit Domainfilterung
- [Exa Search](/de/tools/exa-search) -- neuronale Suche mit Inhaltsextraktion
