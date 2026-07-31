---
read_when:
    - Je koppelt gebruiks- en quotuminterfaces van providers aan elkaar
    - Je moet het gedrag van gebruiksregistratie of de authenticatievereisten uitleggen
summary: Oppervlakken voor gebruiksregistratie en vereisten voor inloggegevens
title: Gebruiksregistratie
x-i18n:
    generated_at: "2026-07-27T05:49:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## Wat het is

- Haalt gebruik en quota van providers rechtstreeks op via het gebruikseindpunt van elke provider. Geen geschatte providerfacturering; alleen door de provider gerapporteerde abonnementsnamen, quotavensters, saldo's, uitgaven, budgetten, dagelijkse kostengeschiedenis, toewijzing aan tokens/modellen of samenvattingen van de accountstatus.
- Voor mensen leesbare uitvoer van quotavensters wordt genormaliseerd naar `X% left`, zelfs wanneer een provider verbruikt quota, resterend quota of alleen ruwe aantallen rapporteert. Providers zonder opnieuw instelbare quotavensters tonen in plaats daarvan samenvattingstekst van de provider (bijvoorbeeld een saldo).
- `/status` op sessieniveau en de tool `session_status` vallen terug op het transcriptlogboek van de sessie wanneer de live momentopname van de sessie geen token-/modelgegevens bevat. Die terugval vult ontbrekende token-/cachetellers aan, kan het actieve label van het runtimemodel herstellen en geeft de voorkeur aan het hogere promptgerichte totaal wanneer sessiemetadata ontbreekt of lager is (`totalTokensFresh !== true`, nul of lager dan de uit het transcript afgeleide waarde). Live waarden die niet nul zijn, hebben altijd voorrang op de terugval.

## Waar het verschijnt

- `/status` in chats: statuskaart met sessietokens en geschatte kosten (alleen modellen met API-sleutel). Providergebruik wordt, indien beschikbaar, getoond voor de **provider van het huidige model**, als een genormaliseerd `X% left`-venster of als samenvattingstekst van de provider.
- `/usage off|tokens|full` in chats: gebruiksvoettekst per antwoord.
- `/usage cost` in chats: lokaal kostenoverzicht, samengevoegd uit OpenClaw-sessielogboeken.
- CLI: `openclaw status --usage` toont een volledige uitsplitsing van gebruik en quota per provider.
- CLI: `openclaw models status` vermeldt OAuth-/tokenauthenticatieprofielen en toont naast elke provider die er een heeft een samenvatting van het gebruiksvenster.
- Besturingsinterface: **Gebruik** toont kaarten voor providerabonnementen en facturering boven OpenClaws op sessies gebaseerde analyse van tokens en geschatte kosten. Referenties voor de Anthropic- en OpenAI Admin API voegen door de provider gerapporteerde uitgaven van vandaag, 7 dagen en 30 dagen toe, evenals dagelijkse trends, tokentotalen, populairste modellen en kostencategorieën.
- Besturingsinterface: de pop-over van de contextring in de chatcomponist toont **abonnementsgebruik** voor abonnementsproviders — balken per venster (5 uur, wekelijks, modelspecifiek) met hersteltijden, het providerabonnement indien bekend (bijvoorbeeld `Max (20x)`) en tegoeden voor extra gebruik. Sessies die via een abonnement worden gefactureerd, verbergen dollarschattingen per token; via de API gefactureerde sessies behouden `Est. cost` en de uitsplitsing van kosten per type. Configuraties met de Claude Code CLI (`claude-cli`) gebruiken hetzelfde Anthropic-abonnementsgebruik.
- macOS-menubalk: wanneer momentopnamen van providergebruik beschikbaar zijn, verschijnt onder Context een hoofdsectie ‘Gebruik’. Zie [Menubalk](/nl/platforms/mac/menu-bar).

`openclaw channels list` toont geen providergebruik meer; het verwijst gebruikers in plaats daarvan naar `openclaw status` of `openclaw models list`.

## Kostengeschiedenis van Anthropic en OpenAI

Abonnementsquota en API-facturering zijn verschillende provideroppervlakken:

- Referenties voor Anthropic-abonnementen/configuraties blijven Claude-quotavensters en optionele budgetten voor extra gebruik tonen. Stel `ANTHROPIC_ADMIN_KEY` of `ANTHROPIC_ADMIN_API_KEY` in om in plaats daarvan de geschiedenis van de Usage and Cost API van de organisatie te tonen. Een Anthropic-providerreferentie die begint met `sk-ant-admin` wordt automatisch gedetecteerd.
- OAuth voor OpenAI ChatGPT/Codex blijft het abonnement, quotavensters en tegoedsaldo tonen. Stel `OPENAI_ADMIN_KEY` in om in plaats daarvan de kosten- en voltooiingsgebruiksgeschiedenis van de organisatie te tonen; stel eventueel `OPENAI_PROJECT_ID` in om deze tot één project te beperken. OpenClaw verzendt nooit inferentiereferenties uit `OPENAI_API_KEY`, providerconfiguratie of authenticatieprofielen naar organisatie-API's, omdat die sleutels bij aangepaste eindpunten kunnen horen.

Beheerdersreferenties hebben voorrang omdat ze de werkelijke organisatiefacturering leveren. OpenClaw combineert deze door de provider gerapporteerde totalen niet met zijn lokale sessieschattingen; de twee secties beantwoorden bewust verschillende vragen.

## Standaardmodus voor de gebruiksvoettekst

`/usage off|tokens|full` stelt de voettekst voor een sessie in en wordt voor die
sessie onthouden. `messages.responseUsage` initialiseert die modus voor sessies die er nog
geen hebben gekozen, zodat de voettekst standaard ingeschakeld kan zijn zonder elke keer `/usage` te typen.

Stel één modus in voor elk kanaal, of een kaart per kanaal met een `default`-terugval:

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // of: { "default": "off", "discord": "full" }
  },
}
```

Geaccepteerde waarden: `"off"`, `"tokens"`, `"full"` en de verouderde alias `"on"` (behandeld als `"tokens"`).

### Drie afzonderlijke sessiestatussen

Het veld `responseUsage` van een sessie heeft drie weergeefbare statussen, elk met
een andere semantiek:

| Status                       | Opgeslagen waarde                        | Effectieve modus                                                                     |
| ---------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------ |
| **Niet ingesteld / overnemen** | `undefined` (afwezig)             | Valt terug op de configuratiestandaard `messages.responseUsage` en daarna `off`. |
| **Expliciet uit**            | `"off"` (opgeslagen)          | Altijd uit; een configuratiestandaard die niet ‘uit’ is, kan de voettekst niet opnieuw inschakelen. |
| **Expliciet aan**            | `"tokens"` of `"full"` (opgeslagen) | Die modus, ongeacht de configuratiestandaard.                                         |

### Voorrang

Effectieve modus = sessieoverschrijving → kanaalconfiguratie-item → `default` → `off`.

Een expliciete `/usage off` wordt in de sessie **permanent opgeslagen** als de letterlijke waarde `"off"`,
en is niet hetzelfde als ‘niet ingesteld’. Een standaardwaarde voor `messages.responseUsage`
die niet ‘uit’ is, kan de voettekst niet opnieuw inschakelen nadat de gebruiker deze expliciet heeft uitgeschakeld.

### Opnieuw instellen versus uitschakelen

- `/usage off` schakelt de voettekst gedwongen uit en slaat die keuze permanent op. Een geconfigureerde
  standaardwaarde die niet ‘uit’ is, kan dit niet overschrijven.
- `/usage reset` (aliassen: `default`, `inherit`, `inherited`, `clear`, `unpin`) wist de sessieoverschrijving.
  De sessie **neemt** vervolgens de effectieve configuratiestandaard
  (`messages.responseUsage`) **over**. Als er geen standaardwaarde is geconfigureerd, blijft de voettekst uitgeschakeld.
- Een volledige sessiereset (`/reset` of `/new`) of een sessieovergang **behoudt**
  de expliciete voorkeur voor de gebruiksmodus, zodat de weergavekeuze van de gebruiker
  sessieovergangen overleeft. Alleen `/usage reset` (en de bijbehorende aliassen) wist de overschrijving.

### Schakelgedrag

`/usage` zonder argumenten doorloopt: uit → tokens → volledig → uit. Het beginpunt
van de cyclus is de **effectieve** huidige modus (de sessieoverschrijving die, wanneer deze
niet is ingesteld, terugvalt op de configuratiestandaard), zodat de cyclus altijd overeenkomt met wat
de gebruiker momenteel in de voettekst ziet.

### Configuratie

Zonder configuratie blijft het eerdere gedrag gelden (voettekst uit tot `/usage`). Gebruik
`/usage reset` om een sessieoverschrijving te wissen en de geconfigureerde standaardwaarde opnieuw over te nemen.

## Aangepaste `/usage full`-voettekst

`/usage tokens` geeft altijd een eenvoudige `Usage: X in / Y out`-regel weer (plus achtervoegsels voor cache en
geschatte kosten wanneer beschikbaar). Alleen `/usage full` geeft de hieronder beschreven uitgebreidere
voettekst weer.

`/usage full` toont een ingebouwde compacte voettekst met model, redenering, snel/langzaam,
contextvenster en kosten wanneer die velden beschikbaar zijn. Voor de ingebouwde voettekst is
geen sjabloonbestand vereist.

`messages.usageTemplate` is alleen bedoeld voor geavanceerde aangepaste indelingen. De waarde is een
pad naar een JSON-bestand (ondersteunt `~`) of een inline object, en vervangt de ingebouwde
voettekst wanneer deze geldig is. Een bestandspad wordt bewaakt en bij wijzigingen direct opnieuw geladen.

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

Ontbrekende of lege sjablonen vallen zonder melding terug op de ingebouwde voettekst. Onleesbare
of ongeldige geconfigureerde sjablonen (ongeldige JSON of een structuur zonder weergeefbare
uitvoeronderdelen) vallen eveneens terug op de ingebouwde voettekst en geven een waarschuwing voor de beheerder.

Baseer aangepaste sjablonen eerst op de ingebouwde structuur en bewerk daarna de onderdelen die je wilt
wijzigen:

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### Structuur

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "tekens van laag naar hoog" }, // tekenreeks (1 teken/karakter) of array
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // voegt resterende onderdelen samen
    "default": [/* pieces */], // terugval voor elk oppervlak
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

Elk oppervlak is een geordende lijst met **onderdelen**; de engine geeft elk onderdeel weer, verwijdert
lege onderdelen en voegt de resterende samen met `sep`. Een oppervlak zonder item gebruikt
`output.default`.

### Contractpaden

Een onderdeel leest waarden uit het contract per beurt via een puntpad. Ontbrekende waarden zijn
leeg (zodat een `when`-voorwaarde of een `|fallback` het onderdeel schoon houdt).

| Pad                                                                                 | Betekenis                                                                                            |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | kanaal-id (`discord`/`telegram`/enz.)                                                               |
| `agentId` / `chat_type`                                                             | id van de beherende agent / type chatinterface                                                       |
| `model.id` / `model.display_name` / `model.provider`                                | model-id / weergavenaam / provider-id                                                               |
| `model.actual`, `model.resolved_ref`                                                | daadwerkelijk voor de beurt gebruikte provider-/modelreferentie                                      |
| `model.requested`                                                                   | aangevraagde provider-/modelreferentie (vóór fallback)                                               |
| `model.reasoning`                                                                   | inspanning (`off` tot en met `xhigh`)                                                                       |
| `model.is_fallback` / `model.is_override`                                           | booleaans: fallback gebruikt / model vastgezet                                                       |
| `model.override_source` / `model.auth_mode`                                         | label van de overschrijvingsbron / referentiemodus (`oauth`, `api-key`, `token`, `mixed`, `aws-sdk`, `unknown`) |
| `state.fast_mode`                                                                   | booleaans: snel versus langzaam                                                                      |
| `state.compactions`                                                                 | aantal Compactions voor de sessie                                                                    |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | vensterbudget / bezette tokens / 0-100 gebruikt                                                      |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | totaal voor de beurt                                                                                 |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | tokens voor cachelezingen en cacheschrijfbewerkingen voor de beurt                                   |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | voorwaarden voor tokenweergave                                                                       |
| `usage.cache_hit_pct`                                                               | aandeel cachelezingen van het totale aantal prompttokens                                             |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | alleen de laatste modelaanroep (heeft ook `cache_read_tokens`, `cache_write_tokens`, `total_tokens`) |
| `cost.turn_usd` / `cost.available`                                                  | geschatte kosten van de beurt / of een kostentabel is gevonden                                       |
| `timing.duration_ms`                                                                | duur van de beurt volgens de klok                                                                    |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | naam / emoji / avatar van de agentidentiteit                                                         |
| `session.id`                                                                        | sessie-id                                                                                            |

(Vensters voor snelheidslimieten van providers maken **geen** deel uit van dit contract; er is momenteel geen pad met een arraywaarde, dus een `each`-onderdeel heeft niets om over te itereren.)

### Werkwoorden

Leid een waarde van links naar rechts door werkwoorden; een segment dat geen werkwoord is, dient als fallback.

| Werkwoord       | Effect                                | Voorbeeld                         |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | compact aantal                        | `272000 -> 272k`                  |
| `fixed:N`       | N decimalen (`0..100`, standaard 2) | `0.0377`                          |
| `dur`           | seconden naar tijdsduur               | `14820 -> 4h07m`                  |
| `pct`           | voeg `%` toe                  | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | om gebruikt naar resterend om te zetten |
| `alias:TABLE`   | opzoeken in `aliases`, ongewijzigd weergeven indien niet vermeld | `medium -> 🌗` |
| `meter:W:SCALE` | glyphbalk van W cellen voor een waarde van 0-100 | `[⣿⣿⠐⠐⠐]` (`meter:1` = één glyph) |

`fixed:N` accepteert alleen een volledig decimaal geheel getal van 0 tot en met 100. Ongeldige
precisieargumenten maken die interpolatie leeg.

`meter:W:SCALE` accepteert alleen een volledige decimale gehele breedte van 1 tot en met 100. Laat de breedte leeg om de standaardwaarde 5 (`meter::braille`) te gebruiken; ongeldige
breedtes maken die interpolatie leeg.

### Onderdeelvormen

- `{ "text": "📚 {context.max_tokens|num}" }`: letterlijke tekst + interpolatie.
- `{ "when": "<path>", "text": "..." }`: alleen renderen als het pad een waarheidswaarde heeft.
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`: waarde naar glyph (een `_default`-geval dekt waarden zonder overeenkomst).
- `{ "each": "<array-path>", "item": "{label}" }`: itereren over een pad met een arraywaarde (geen enkel huidig contractpad is een array).

### Voorbeeld

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

rendert bijvoorbeeld `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k`.

## Providers + referenties

Gebruik wordt verborgen wanneer geen bruikbare providerreferenties voor gebruik kunnen worden gevonden. OpenClaw
ontdekt automatisch ingeschakelde providerplugins die
`contracts.usageProviders` declareren en zowel `resolveUsageAuth` als
`fetchUsageSnapshot` implementeren; er is geen afzonderlijke allowlist voor providers in de kern. Het statische
contract houdt de ontdekking afgebakend zonder elke providerplugin te importeren. Elke
plugin beheert zijn eigen upstream-eindpunt en responstoewijzing. De
gedeelde momentopname houdt plannamen, quotavensters, saldi, uitgaven en budgetten
providerneutraal voor gebruikers van de CLI, app en Control UI.

- **Anthropic (Claude)**: OAuth-tokens in authenticatieprofielen. Als het OAuth-token het
  bereik `user:profile` mist, wordt teruggevallen op een `claude.ai`-websessie (`CLAUDE_AI_SESSION_KEY`,
  `CLAUDE_WEB_SESSION_KEY` of een `sessionKey=`-cookie in `CLAUDE_WEB_COOKIE`) indien ingesteld.
  Modelspecifieke limieten en ingeschakelde extra maandelijkse gebruiksuitgaven/-budgetten worden opgenomen
  wanneer Anthropic deze rapporteert. Een expliciete Anthropic Admin API-sleutel, of een
  automatisch gedetecteerd `sk-ant-admin...`-providerprofiel, toont in plaats daarvan de organisatiekosten
  over 30 dagen en de geschiedenis van de Messages API.
- **ClawRouter**: API-sleutel (`CLAWROUTER_API_KEY`). Toont een maandelijks budgetvenster
  en een getypeerd budget in USD wanneer dit is geconfigureerd; anders worden de totale uitgaven en een
  overzicht van aanvragen/tokens/kosten weergegeven.
- **DeepSeek**: API-sleutel via omgeving/configuratie/authenticatieopslag (`DEEPSEEK_API_KEY`).
  Toont elk door de provider gerapporteerd valutasaldo.
- **GitHub Copilot**: OAuth-tokens in authenticatieprofielen.
- **Gemini CLI**: OAuth-tokens in authenticatieprofielen.
- **MiniMax**: API-sleutel of MiniMax OAuth-authenticatieprofiel. OpenClaw behandelt
  `minimax`, `minimax-cn` en `minimax-portal` als hetzelfde MiniMax-quotaoppervlak,
  geeft waar aanwezig de voorkeur aan opgeslagen MiniMax OAuth en valt anders terug
  op `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` of `MINIMAX_API_KEY`.
  Bij het opvragen van gebruik wordt de Coding Plan-host afgeleid van `models.providers.minimax-portal.baseUrl`
  of `models.providers.minimax.baseUrl` wanneer deze zijn geconfigureerd; anders wordt de
  MiniMax CN-host gebruikt.
  De onbewerkte velden `usage_percent` / `usagePercent` van MiniMax betekenen **resterend**
  quota, dus OpenClaw keert ze vóór weergave om; op aantallen gebaseerde velden hebben voorrang
  wanneer ze aanwezig zijn.
  - Vensterlabels zijn afkomstig van de uren-/minutenvelden van de provider wanneer deze aanwezig zijn en
    vallen vervolgens terug op het bereik `start_time` / `end_time`.
  - Als het coding-plan-eindpunt `model_remains` retourneert, geeft OpenClaw de voorkeur aan de
    chatmodelvermelding, leidt het vensterlabel af uit tijdstempels wanneer expliciete
    velden `window_hours` / `window_minutes` ontbreken en neemt de modelnaam
    op in het planlabel.
- **OpenAI (Codex/ChatGPT-abonnement)**: OAuth-tokens in authenticatieprofielen (`ChatGPT-Account-Id`-
  header wordt verzonden wanneer een account-id aanwezig is). Toont het ChatGPT-abonnement, resetbare
  Codex-vensters en een tegoedsaldo wanneer dit wordt gerapporteerd. Tegoeden blijven providertegoeden;
  OpenClaw labelt ze niet als dollars. `OPENAI_ADMIN_KEY` voegt
  organisatiekosten over 30 dagen en de gebruiksgeschiedenis van completions toe wanneer de sleutel toegang
  tot het Usage Dashboard heeft. Referenties voor inferentie worden nooit doorgestuurd naar organisatie-API's.
- **OpenRouter**: API-sleutel of door OAuth ondersteunde API-sleutel (`OPENROUTER_API_KEY` of een authenticatieprofiel).
  Combineert het eindpunt voor accounttegoeden met het eindpunt voor het sleutelquotum,
  zodat accountsaldo/-uitgaven, sleutelbudget en dagelijks/wekelijks/maandelijks gebruik worden weergegeven
  wanneer de referentie er toegang toe heeft. Elk eindpunt kan de momentopname
  onafhankelijk verrijken.
- **Venice**: API-sleutel via omgeving/configuratie/authenticatieopslag (`VENICE_API_KEY`). Toont USD- en
  DIEM-saldi plus het gebruik van DIEM-epochtoewijzingen wanneer dit wordt gerapporteerd.
- **Xiaomi MiMo**: twee afzonderlijke gebruiksoppervlakken. Betalen naar gebruik gebruikt een API-sleutel
  (`XIAOMI_API_KEY`); het Token Plan gebruikt een afzonderlijke sleutel (`XIAOMI_TOKEN_PLAN_API_KEY`).
  Geen van beide rapporteert momenteel quotavensters.
- **z.ai**: API-sleutel via omgeving/configuratie/authenticatieopslag (`ZAI_API_KEY` of `Z_AI_API_KEY`).

## Gerelateerd

- [Tokengebruik en kosten](/nl/reference/token-use)
- [API-gebruik en kosten](/nl/reference/api-usage-costs)
- [Promptcaching](/nl/reference/prompt-caching)
- [Menubalk](/nl/platforms/mac/menu-bar)
