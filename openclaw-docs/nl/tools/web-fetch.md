---
read_when:
    - Je wilt een URL ophalen en leesbare inhoud extraheren
    - Je moet web_fetch of de Firecrawl-fallback ervan configureren
    - Je wilt de limieten en caching van web_fetch begrijpen
sidebarTitle: Web Fetch
summary: web_fetch-tool -- HTTP-ophalen met extractie van leesbare inhoud
title: Web ophalen
x-i18n:
    generated_at: "2026-07-27T05:21:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` voert een gewone HTTP GET uit en extraheert leesbare inhoud (HTML naar
Markdown of tekst). JavaScript wordt **niet** uitgevoerd. Gebruik voor sites die sterk van JS afhankelijk zijn of
pagina's waarvoor aanmelding vereist is in plaats daarvan de [webbrowser](/nl/tools/browser).

## Snel aan de slag

Standaard ingeschakeld, geen configuratie nodig:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## Toolparameters

<ParamField path="url" type="string" required>
URL om op te halen. Alleen `http(s)`.
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
Uitvoerindeling na extractie van de hoofdinhoud.
</ParamField>

<ParamField path="maxChars" type="number">
Kort de uitvoer af tot dit aantal tekens. Begrensd op `tools.web.fetch.maxCharsCap`.
</ParamField>

## Resultaat

`web_fetch` retourneert een gesloten, gestructureerd resultaat met deze velden:

- Metagegevens van het verzoek: `url`, `finalUrl`, `status`, `extractMode` en `extractor`
- Optionele metagegevens van het antwoord: `contentType`, `title` en `warning` (weggelaten indien afwezig)
- Metagegevens van de verpakte inhoud: `externalContent`, `truncated`, `length`, `rawLength`,
  `fetchedAt`, `tookMs` en `text`
- Optionele `cached: true` bij een cachetreffer
- Optionele `spill: { path, chars, truncated? }` wanneer afgekorte inhoud naar
  een privé-tijdelijk bestand is geschreven; `truncated` is alleen aanwezig wanneer dat bestand
  gedeeltelijke broninhoud bevat

`length` is de lengte van de verpakte `text`. `rawLength` is de lengte van de geëxtraheerde inhoud
vóór het verpakken van externe inhoud.

## Werking

<Steps>
  <Step title="Ophalen">
    Verzendt een HTTP GET met een Chrome-achtige User-Agent en de header `Accept-Language`.
    Blokkeert privé-/interne hostnamen en controleert omleidingen opnieuw.
  </Step>
  <Step title="Extraheren">
    Voert Readability (extractie van hoofdinhoud) uit op het HTML-antwoord.
  </Step>
  <Step title="Terugval (optioneel)">
    Als Readability mislukt en er een ophaalprovider beschikbaar is, wordt het opnieuw geprobeerd via
    die provider (bijvoorbeeld Firecrawls modus om botdetectie te omzeilen).
  </Step>
  <Step title="Cache">
    Resultaten worden 15 minuten in de cache opgeslagen (configureerbaar) om herhaaldelijk
    ophalen van dezelfde URL te beperken.
  </Step>
</Steps>

## Voortgangsupdates

`web_fetch` geeft alleen een openbare voortgangsregel weer wanneer het ophalen na
vijf seconden nog steeds bezig is:

```text
Pagina-inhoud ophalen...
```

Snelle cachetreffers en snelle netwerkantwoorden zijn klaar voordat de timer afgaat en
tonen daarom nooit een voortgangsregel. Als de aanroep wordt geannuleerd, wordt de timer gewist. De
voortgangsregel is uitsluitend de UI-status van het kanaal en bevat nooit opgehaalde pagina-inhoud.

## Configuratie

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // standaard: true
        provider: "firecrawl", // optioneel; weglaten voor automatische detectie
        maxChars: 20000, // standaardaantal uitvoertekens; begrensd door maxCharsCap
        maxCharsCap: 20000, // harde limiet voor de parameter maxChars
        maxResponseBytes: 750000, // maximale downloadgrootte vóór afkapping (32000-10000000)
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // laat een vertrouwde HTTP(S)-omgevingsproxy DNS omzetten
        readability: true, // gebruik Readability-extractie
        userAgent: "Mozilla/5.0 ...", // overschrijf User-Agent
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // expliciete inschakeling voor vertrouwde proxy's met nep-IP's die 198.18.0.0/15 gebruiken
          allowIpv6UniqueLocalRange: true, // expliciete inschakeling voor vertrouwde proxy's met nep-IP's die fc00::/7 gebruiken
        },
      },
    },
  },
}
```

## Firecrawl-terugval

Als Readability-extractie mislukt, kan `web_fetch` terugvallen op
[Firecrawl](/nl/tools/firecrawl) om botdetectie te omzeilen en de extractie te verbeteren:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // optioneel; weglaten voor automatische detectie aan de hand van beschikbare referenties
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // optioneel; weglaten voor sleutelvrije starterstoegang
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // cacheduur (2 dagen)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` is optioneel en ondersteunt SecretRef-objecten.
Verouderde `tools.web.fetch.firecrawl.*`-configuratie wordt automatisch gemigreerd naar
`plugins.entries.firecrawl.config.webFetch` via `openclaw doctor --fix`.

<Note>
  Als je een SecretRef voor een Firecrawl-API-sleutel configureert en deze niet kan worden omgezet zonder
  terugval op de omgevingsvariabele `FIRECRAWL_API_KEY`, mislukt het opstarten van de Gateway onmiddellijk.
</Note>

<Note>
  Overschrijvingen van Firecrawl `baseUrl` zijn vergrendeld: gehost verkeer gebruikt
  `https://api.firecrawl.dev`; zelfgehoste overschrijvingen moeten naar privé- of
  interne eindpunten verwijzen en `http://` wordt alleen voor die privétargets geaccepteerd.
</Note>

Huidig runtimegedrag:

- `tools.web.fetch.provider` selecteert expliciet de terugvalprovider voor ophalen.
- Als `provider` wordt weggelaten, detecteert OpenClaw automatisch de eerste beschikbare provider voor ophalen van het web
  aan de hand van geconfigureerde referenties. `web_fetch` buiten een sandbox kan
  geïnstalleerde plugins gebruiken die `contracts.webFetchProviders` declareren en tijdens runtime een
  overeenkomende provider registreren. De officiële Firecrawl-plugin biedt deze
  terugval momenteel.
- Aanroepen van `web_fetch` in een sandbox staan gebundelde providers toe, plus geïnstalleerde providers
  waarvan de officiële npm- of ClawHub-herkomst is geverifieerd. Momenteel staat dit de
  officiële Firecrawl-plugin toe; externe ophaalplugins van derden blijven uitgesloten.
- Als Readability is uitgeschakeld, gaat `web_fetch` direct door naar de geselecteerde
  terugvalprovider. Als er geen provider beschikbaar is, wordt de aanroep veilig geweigerd.

## Vertrouwde omgevingsproxy

Als je implementatie vereist dat `web_fetch` via een vertrouwde uitgaande
HTTP(S)-proxy loopt, stel je `tools.web.fetch.useTrustedEnvProxy: true` in.

In deze modus past OpenClaw nog steeds SSRF-controles op basis van hostnamen toe voordat
het verzoek wordt verzonden, maar laat het de proxy DNS omzetten in plaats van lokale DNS-pinning
uit te voeren. Schakel dit alleen in wanneer de proxy door de beheerder wordt beheerd en
na DNS-resolutie beleid voor uitgaand verkeer afdwingt.

<Note>
  Als er geen omgevingsvariabele voor een HTTP(S)-proxy is geconfigureerd of de doelhost wordt uitgesloten door
  `NO_PROXY`, valt `web_fetch` terug op het normale strikte pad met lokale
  DNS-pinning.
</Note>

## Limieten en veiligheid

- `maxChars` wordt begrensd op `tools.web.fetch.maxCharsCap` (standaard `20000`)
- De hoofdtekst van het antwoord wordt vóór verwerking begrensd op `maxResponseBytes` (standaard `750000`, begrensd op
  32000-10000000); te grote antwoorden worden met een waarschuwing afgekort
- Privé-/interne hostnamen worden geblokkeerd
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` en
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` zijn beperkte expliciete inschakelingen
  voor vertrouwde proxystacks met nep-IP's; laat ze oningesteld, tenzij je proxy
  eigenaar is van die synthetische bereiken en zijn eigen bestemmingsbeleid afdwingt
- Omleidingen worden gecontroleerd en beperkt door `maxRedirects` (standaard `3`)
- `useTrustedEnvProxy` vereist expliciete inschakeling en mag alleen worden ingeschakeld voor
  proxy's die door de beheerder worden beheerd en na DNS-resolutie nog steeds beleid voor uitgaand verkeer
  afdwingen
- `web_fetch` werkt naar beste vermogen -- sommige sites vereisen de [webbrowser](/nl/tools/browser)

## Toolprofielen

Als je toolprofielen of toelatingslijsten gebruikt, voeg je `web_fetch` of `group:web` toe:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // of: allow: ["group:web"]  (omvat web_fetch, web_search en x_search)
  },
}
```

## Gerelateerd

- [Zoeken op het web](/nl/tools/web) -- doorzoek het web met meerdere providers
- [Webbrowser](/nl/tools/browser) -- volledige browserautomatisering voor sites die sterk van JS afhankelijk zijn
- [Firecrawl](/nl/tools/firecrawl) -- Firecrawl-tools voor zoeken en scrapen
