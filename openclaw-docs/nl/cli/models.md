---
read_when:
    - Je wilt de standaardmodellen wijzigen of de authenticatiestatus van de provider bekijken
    - Je wilt beschikbare modellen/providers scannen en authenticatieprofielen debuggen
summary: CLI-referentie voor `openclaw models` (status/lijst/instellen/scannen, aliassen, terugvalopties, authenticatie)
title: Modellen
x-i18n:
    generated_at: "2026-07-27T06:08:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f7405c25694f04afe9c3029a8af64ae3ae7e1bdcf4c4ac31b8b84ff512d6a90e
    source_path: cli/models.md
    workflow: 16
---

# `openclaw models`

Modeldetectie, scannen en configuratie (standaardmodel, fallbacks, auth-profielen).

Gerelateerd:

- Providers + modellen: [Modellen](/nl/providers/models)
- Concepten voor modelselectie + `/models`-slashopdracht: [Modelconcept](/nl/concepts/models)
- Auth-configuratie voor providers: [Aan de slag](/nl/start/getting-started)

## Veelgebruikte opdrachten

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
openclaw models scan
```

De subopdrachten `status` en `auth` accepteren `--agent <id>` om een geconfigureerde agent te kiezen; `list`, `scan`, `aliases` en `fallbacks`/`image-fallbacks` gebruiken altijd de geconfigureerde standaardagent, en `set`/`set-image` weigeren `--agent` zonder meer. Wanneer dit wordt weggelaten, gebruiken opdrachten die rekening houden met `--agent` `OPENCLAW_AGENT_DIR` als die is ingesteld, en anders de geconfigureerde standaardagent.

### Status

`openclaw models status` toont het bepaalde standaardmodel en de fallbacks, plus een auth-overzicht. Voor agentruntimes die eigendom zijn van een plugin, zoals Codex, controleert het ook of de betreffende plugin is ingeschakeld en de verificatie van de opstartpayload heeft doorstaan. Een route met geldige inloggegevens maar een niet-beschikbare runtime meldt `status: unavailable` in plaats van `usable`; JSON-uitvoer bevat afzonderlijke `authStatus`, `runtimeStatus` en begrensde runtimediagnostiek. Wanneer snapshots van providergebruik beschikbaar zijn, bevat het statusgedeelte voor OAuth/API-sleutels gebruiksperioden en quotasnapshots van providers. Huidige providers voor gebruiksperioden: Anthropic, GitHub Copilot, Gemini CLI, OpenAI, MiniMax, Xiaomi en z.ai. Auth voor gebruik is waar mogelijk afkomstig van providerspecifieke hooks; anders valt OpenClaw terug op overeenkomende OAuth-/API-sleutelgegevens uit auth-profielen, omgevingsvariabelen of de configuratie.

In de uitvoer van `--json` is `auth.providers` het provideroverzicht dat rekening houdt met omgeving/configuratie/opslag, terwijl `auth.oauth` alleen de status van profielen in de auth-opslag weergeeft.

Opties:

| Vlag                      | Effect                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`                  | JSON-uitvoer; diagnostiek voor auth-profielen, providers en opstart gaat naar stderr, zodat stdout via een pipe naar `jq` kan worden geleid.                            |
| `--plain`                 | Uitvoer als platte tekst.                                                                                                                       |
| `--check`                 | Sluit af met een niet-nulstatus als auth bijna verloopt/verlopen is of een geselecteerde agentruntime niet beschikbaar is: `1` = niet beschikbaar/verlopen/ontbrekend, `2` = bijna verlopen. |
| `--probe`                 | Livecontrole van geconfigureerde auth-profielen. Echte aanvragen; kan tokens verbruiken en snelheidslimieten activeren.                                       |
| `--probe-provider <name>` | Controleer slechts één provider.                                                                                                                 |
| `--probe-profile <id>`    | Controleer specifieke auth-profiel-id's (herhalen of door komma's gescheiden).                                                                             |
| `--probe-timeout <ms>`    | Time-out per controle.                                                                                                                       |
| `--probe-concurrency <n>` | Gelijktijdige controles.                                                                                                                       |
| `--probe-max-tokens <n>`  | Maximaal aantal tokens voor controle (naar beste vermogen).                                                                                                          |
| `--agent <id>`            | Id van geconfigureerde agent; overschrijft `OPENCLAW_AGENT_DIR`.                                                                                     |

Controleregels kunnen afkomstig zijn van auth-profielen, omgevingsinloggegevens of `models.json`. Statuscategorieën voor controles: `ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`.

Detail-/redencodes die je kunt verwachten wanneer een controle nooit tot een modelaanroep komt:

- `excluded_by_auth_order`: er bestaat een opgeslagen profiel, maar expliciete `auth.order.<provider>` heeft het weggelaten, zodat de controle de uitsluiting meldt in plaats van het profiel te proberen.
- `missing_credential`, `invalid_expires`, `expired`, `unresolved_ref`: het profiel is aanwezig, maar komt niet in aanmerking of kan niet worden bepaald.
- `ineligible_profile`: het profiel is om een andere reden niet compatibel met de providerconfiguratie.
- `no_model`: er bestaat provider-auth, maar OpenClaw kon voor die provider geen controleerbare modelkandidaat bepalen.

Voor probleemoplossing met OpenAI ChatGPT/Codex OAuth zijn `openclaw models status`, `openclaw models auth list --provider openai` en `openclaw config get agents.defaults.model --json` de snelste manier om te bevestigen of een agent een bruikbaar `openai` OAuth-profiel voor `openai/*` heeft via de native Codex-runtime. Zie [OpenAI-providerconfiguratie](/nl/providers/openai#check-and-recover-codex-oauth-routing).

### Lijst

`openclaw models list` is alleen-lezen: het leest de configuratie, auth-profielen, bestaande catalogusstatus en catalogusregels van providers, maar herschrijft `models.json` nooit.

Opties: `--all` (volledige catalogus), `--local` (filteren op lokale modellen), `--provider <id>`, `--json`, `--plain`.

Opmerkingen:

- De kolom `Auth` is alleen-lezen. Voor modelroutes die eigendom zijn van providers, zoals OpenAI, koppelt deze de API-/basis-URL-route van elke regel aan geschikte profielen in de effectieve `auth.order`, inloggegevens uit omgeving/configuratie en bepaalde opdrachtgebonden SecretRefs. Een concrete OpenAI-regel blijft onbekend wanneer het routebeleid niet beschikbaar is, in plaats van auth op providerniveau over te nemen; verouderde controles die alleen de provider gebruiken en andere providers behouden gedrag op providerniveau. Metadata voor synthetische auth van plugins is alleen een aanwijzing voor runtimecapaciteiten, geen bewijs van native accountauthenticatie, zodat accountafhankelijke routes onbekend blijven zonder positief bewijs uit het register. De opdracht laadt de providerruntime niet, leest geen sleutelbosgeheimen, roept geen provider-API's aan en bewijst niet dat de route exact gereed is voor uitvoering.
- `models list --all --provider <id>` kan statische catalogusregels van providers bevatten uit pluginmanifesten of gebundelde providercatalogusmetadata, zelfs als je je nog niet bij die provider hebt geauthenticeerd. Deze regels worden nog steeds als niet beschikbaar weergegeven totdat overeenkomende auth is geconfigureerd.
- `models list` houdt het besturingsvlak responsief wanneer het detecteren van providercatalogi traag is. De standaard- en geconfigureerde weergaven vallen na een korte wachttijd terug op geconfigureerde of synthetische modelregels en laten de detectie op de achtergrond voltooien. Gebruik `--all` wanneer je de exacte, volledig gedetecteerde catalogus nodig hebt en bereid bent te wachten op providerdetectie.
- Brede `models list --all` voegt manifestcatalogusregels samen boven registerregels zonder aanvullende hooks van de providerruntime te laden. Providergefilterde snelle manifestpaden gebruiken alleen providers die zijn gemarkeerd als `static`; providers die zijn gemarkeerd als `refreshable` blijven register-/cachegestuurd en voegen manifestregels toe als aanvulling, terwijl providers die zijn gemarkeerd als `runtime` register-/runtimedetectie blijven gebruiken.
- `models list` houdt native modelmetadata en runtimelimieten gescheiden. In tabeluitvoer toont `Ctx` `contextTokens/contextWindow` wanneer een effectieve runtimelimiet afwijkt van het native contextvenster; JSON-regels bevatten `contextTokens` wanneer een provider die limiet beschikbaar stelt.
- Voor routes die eigendom zijn van providers, projecteert `models list` één logische provider-/modelregel op de geselecteerde route. `Input` en `Ctx` zijn alleen afkomstig van een exact overeenkomende catalogusregel voor de fysieke route, waarbij expliciet geconfigureerde logische overschrijvingen als laatste worden toegepast; bij een niet-bepaalde routeselectie worden onbekende capaciteitsvelden weergegeven in plaats van metadata van zusterroutes over te nemen.
- `models list --provider <id>` filtert op provider-id, zoals `moonshot` of `openai`. Het accepteert geen weergavelabels uit interactieve providerkeuzelijsten, zoals `Moonshot AI`.
- Modelverwijzingen worden geparseerd door ze te splitsen op de **eerste** `/`. Als de model-id `/` bevat (OpenRouter-stijl), neem dan het providervoorvoegsel op (voorbeeld: `openrouter/moonshotai/kimi-k2`).
- Als je de provider weglaat, bepaalt OpenClaw de invoer eerst als alias, vervolgens als een unieke overeenkomst met een geconfigureerde provider voor die exacte model-id, en valt pas daarna met een afschaffingswaarschuwing terug op de geconfigureerde standaardprovider. Als die provider het geconfigureerde standaardmodel niet langer beschikbaar stelt, valt OpenClaw terug op het eerste geconfigureerde provider/model in plaats van een verouderde standaardwaarde van een verwijderde provider te tonen.
- `models status` kan `marker(<value>)` in auth-uitvoer tonen voor niet-geheime tijdelijke aanduidingen (bijvoorbeeld `OPENAI_API_KEY`, `secretref-managed`, `minimax-oauth`, `oauth:chutes`, `ollama-local`) in plaats van ze als geheimen te maskeren.

### Standaard-/afbeeldingsmodel instellen

```bash
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
```

`set` schrijft `agents.defaults.model.primary`; `set-image` schrijft `agents.defaults.imageModel.primary`. Beide accepteren `provider/model` of een geconfigureerde alias. `set` herstelt ook installaties van Codex-/Copilot-runtimeplugins wanneer het nieuw geselecteerde model er een nodig heeft; `set-image` doet dat niet. Geen van beide opdrachten accepteert `--agent`; ze schrijven altijd agentstandaardwaarden.

### Scannen

`models scan` leest de openbare `:free`-catalogus van OpenRouter en rangschikt kandidaten voor gebruik als fallback. De catalogus zelf is openbaar, dus scans die alleen metadata gebruiken, hebben geen OpenRouter-sleutel nodig.

OpenClaw probeert standaard ondersteuning voor tools en afbeeldingen te controleren met live modelaanroepen. Als er geen OpenRouter-sleutel is geconfigureerd, valt de opdracht terug op uitvoer met alleen metadata en legt uit dat `:free`-modellen nog steeds `OPENROUTER_API_KEY` vereisen voor controles en inferentie.

Opties:

- `--no-probe` (alleen metadata; geen opzoekactie in configuratie/geheimen)
- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>` (time-out voor catalogusaanvraag en per controle)
- `--concurrency <n>`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

`--set-default` en `--set-image` vereisen livecontroles; scanresultaten met alleen metadata zijn informatief en worden niet op de configuratie toegepast.

## Aliassen

```bash
openclaw models aliases list [--json] [--plain]
openclaw models aliases add <alias> <model-or-alias>
openclaw models aliases remove <alias>
```

Aliassen worden per modelvermelding opgeslagen als `agents.defaults.models.<key>.alias`. `add` bepaalt `<model-or-alias>` eerst als een canonieke provider-/modelsleutel, zodat het toewijzen van een alias aan een alias deze opnieuw laat verwijzen in plaats van een keten te vormen.
Het toevoegen van een alias wijzigt `agents.defaults.modelPolicy.allow` niet en beperkt modeloverschrijvingen niet.

## Fallbacks

```bash
openclaw models fallbacks list [--json] [--plain]
openclaw models fallbacks add <model-or-alias>
openclaw models fallbacks remove <model-or-alias>
openclaw models fallbacks clear
```

Beheert `agents.defaults.model.fallbacks`. `openclaw models image-fallbacks list|add|remove|clear` beheert de parallelle `agents.defaults.imageModel.fallbacks`-lijst met dezelfde subopdrachtstructuur.

## Auth-profielen

```bash
openclaw models auth add
openclaw models auth list [--provider <id>] [--json]
openclaw models auth login --provider <id>
openclaw models auth login --provider openai --profile-id openai:work
openclaw models auth login-github-copilot
openclaw models auth paste-api-key --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token --provider <id>
openclaw models auth order get --provider <id>
openclaw models auth order set --provider <id> <profileIds...>
openclaw models auth order clear --provider <id>
```

`models auth add` is de interactieve authenticatiehelper. Deze kan een authenticatiestroom van een provider starten (OAuth/API-sleutel) of je begeleiden bij het handmatig plakken van een token, afhankelijk van de provider die je kiest.

`models auth list` vermeldt opgeslagen authenticatieprofielen voor de geselecteerde agent zonder tokens, API-sleutels of geheim OAuth-materiaal weer te geven. Gebruik `--provider <id>` om op één provider te filteren, zoals `openai`, en `--json` voor scripts.

`models auth login` voert de authenticatiestroom van een providerplugin uit (OAuth/API-sleutel). Gebruik `openclaw plugins list` om te zien welke providers zijn geïnstalleerd. `login` accepteert `--profile-id <id>` voor providers die tijdens het aanmelden benoemde profielen ondersteunen (gebruik dit om meerdere aanmeldingen voor dezelfde provider gescheiden te houden), `--method <id>` om een specifieke authenticatiemethode te kiezen, `--device-code` als snelkoppeling voor `--method device-code`, `--set-default` om het aanbevolen standaardmodel van de provider toe te passen en `--force` om eerst bestaande profielen voor die provider te verwijderen (gebruik dit wanneer een OAuth-profiel in de cache vastzit of je van account wilt wisselen).

`models auth login-github-copilot` is een snelkoppeling voor `models auth login --provider github-copilot --method device` (GitHub-apparaatstroom); deze accepteert `--yes` om een bestaand profiel zonder bevestigingsvraag te overschrijven.

Gebruik `openclaw models auth --agent <id> <subcommand>` om authenticatieresultaten naar de opslag van een specifieke geconfigureerde agent te schrijven. De bovenliggende vlag `--agent` wordt gerespecteerd door `add`, `list`, `login`, `paste-api-key`, `setup-token`, `paste-token`, `login-github-copilot` en `order get`/`set`/`clear`.

Voor OpenAI-modellen gebruikt `--provider openai` standaard aanmelding met een ChatGPT-/Codex-account. Gebruik `--method api-key` alleen wanneer je een OpenAI-API-sleutelprofiel wilt toevoegen, doorgaans als reserve voor de abonnementslimieten van Codex. Voer `openclaw doctor --fix` uit om oudere verouderde authenticatie-/profielstatus met het OpenAI Codex-voorvoegsel naar `openai` te migreren.

Voorbeelden:

```bash
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider openai --method api-key
openclaw models auth paste-api-key --provider openai
openclaw models auth list --provider openai
```

Opmerkingen:

- `paste-api-key` accepteert API-sleutels die elders zijn gegenereerd, vraagt om de sleutelwaarde en schrijft deze naar de standaardprofiel-id `<provider>:manual`, tenzij je `--profile-id` meegeeft. Stuur bij automatisering de sleutel via stdin, bijvoorbeeld `printf "%s\n" "$OPENAI_API_KEY" | openclaw models auth paste-api-key --provider openai`.
- `setup-token` en `paste-token` blijven algemene tokenopdrachten voor providers die tokenauthenticatiemethoden aanbieden.
- `setup-token` vereist een interactieve TTY en voert de tokenauthenticatiemethode van de provider uit (standaard de methode `setup-token` van die provider wanneer deze er een aanbiedt).
- `paste-token` vereist `--provider`, vraagt standaard om de tokenwaarde en schrijft deze naar de standaardprofiel-id `<provider>:manual`, tenzij je `--profile-id` meegeeft. Stuur bij automatisering het token via stdin in plaats van het als argument mee te geven, zodat providerreferenties niet in de shellgeschiedenis of proceslijsten verschijnen.
- `paste-token --expires-in <duration>` slaat een absolute vervaldatum van het token op, berekend aan de hand van een relatieve duur zoals `365d` of `12h`.
- Voor `openai` hebben OpenAI-API-sleutels en ChatGPT-/OAuth-tokenmateriaal verschillende authenticatiestructuren. Gebruik `paste-api-key` voor OpenAI-API-sleutels van `sk-...` en `paste-token` alleen voor tokenauthenticatiemateriaal.
- Anthropic: `setup-token`/`paste-token` zijn ondersteunde OpenClaw-authenticatiepaden voor `anthropic`, maar OpenClaw geeft er de voorkeur aan de Claude CLI (`claude -p`) op de host te hergebruiken wanneer deze beschikbaar is.
- `auth order get/set/clear` beheert voor één provider een overschrijving van de volgorde van authenticatieprofielen per agent, opgeslagen in `auth-state.json` (los van de configuratiesleutel `auth.order.<provider>`). `set` accepteert één of meer profiel-id's in prioriteitsvolgorde; `clear` valt terug op de configuratie-/round-robinvolgorde.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Modelselectie](/nl/concepts/model-providers)
- [Model-failover](/nl/concepts/model-failover)
