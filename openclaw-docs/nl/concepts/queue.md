---
read_when:
    - Uitvoering of gelijktijdigheid van automatische antwoorden wijzigen
    - Uitleg over `/queue`-modi of het sturen van berichten
summary: Wachtrijmodi voor automatische antwoorden, standaardinstellingen en overrides per sessie
title: Opdrachtenwachtrij
x-i18n:
    generated_at: "2026-07-27T06:13:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 69b40f67146226b0315492b27fc9d2218cace8bbd1eaff6514f7efb33b69d763
    source_path: concepts/queue.md
    workflow: 16
---

OpenClaw serialiseert inkomende automatische antwoordruns (alle kanalen) via een kleine wachtrij binnen het proces om te voorkomen dat meerdere agentruns met elkaar botsen, terwijl veilige parallelliteit tussen sessies mogelijk blijft.

## Waarom

- Automatische antwoordruns kunnen kostbaar zijn (LLM-aanroepen) en kunnen botsen wanneer meerdere inkomende berichten kort na elkaar arriveren.
- Serialisatie voorkomt concurrentie om gedeelde resources (sessiebestanden, logboeken, CLI-stdin) en verkleint de kans op upstream-snelheidslimieten.

## Hoe het werkt

- Een rijstrookbewuste FIFO-wachtrij verwerkt elke rijstrook met een configureerbare gelijktijdigheidslimiet (standaard 1 voor niet-geconfigureerde rijstroken; `main` is standaard 4, `subagent` 8).
- `runEmbeddedAgent` plaatst op basis van de **sessiesleutel** in de wachtrij (rijstrook `session:<key>`) om te garanderen dat er slechts één actieve run per sessie is.
- Elke sessierun wordt vervolgens in een **globale rijstrook** geplaatst (standaard `main`), zodat de totale parallelliteit wordt begrensd door `agents.defaults.maxConcurrent`.
- Wanneer uitgebreide logboekregistratie is ingeschakeld, geven runs in de wachtrij een korte melding als ze vóór het starten langer dan ~2s hebben gewacht.
- Typindicatoren worden nog steeds onmiddellijk bij plaatsing in de wachtrij geactiveerd (wanneer het kanaal dit ondersteunt), zodat de gebruikerservaring ongewijzigd blijft terwijl de run op zijn beurt wacht.

## Standaardwaarden

Wanneer niets is ingesteld, gebruiken alle oppervlakken voor inkomende kanalen:

- `mode: "steer"`
- `debounceMs: 500`
- `cap: 20`
- `drop: "summarize"`

Aansturing binnen dezelfde beurt is de standaard. Een prompt die tijdens een run binnenkomt, wordt in de actieve runtime geïnjecteerd wanneer de run aansturing kan accepteren, zodat geen tweede sessierun wordt gestart. Als de actieve run geen aansturing kan accepteren, wacht OpenClaw tot de actieve run is voltooid voordat de prompt wordt gestart.

## Wachtrijmodi

`/queue` bepaalt wat normale inkomende berichten doen wanneer een sessie al een actieve run heeft:

- `steer`: injecteer berichten in de actieve runtime. OpenClaw levert alle wachtende aansturingsberichten **nadat de huidige assistentbeurt de uitvoering van zijn toolaanroepen heeft voltooid**, vóór de volgende LLM-aanroep; de Codex-app-server ontvangt één gebundelde `turn/steer`. Als de run niet actief streamt of aansturing niet beschikbaar is, wacht OpenClaw tot de actieve run eindigt voordat de prompt wordt gestart.
- `followup`: stuur niet aan. Plaats elk bericht in de wachtrij voor een latere agentbeurt nadat de huidige run is beëindigd.
- `collect`: stuur niet aan. Voeg berichten in de wachtrij samen tot **één** vervolgbeurt na het stiltevenster. Als berichten op verschillende kanalen/threads zijn gericht, worden ze afzonderlijk verwerkt om de routering te behouden.
- `interrupt`: breek de actieve run voor die sessie af en voer vervolgens het nieuwste bericht uit.

Zie [Aansturingswachtrij](/nl/concepts/queue-steering) voor runtimespecifieke timing en afhankelijkheidsgedrag. Zie [Aansturen](/nl/tools/steer) voor de expliciete opdracht `/steer <message>`.

Configureer dit globaal of per kanaal via `messages.queue`:

```json5
{
  messages: {
    queue: {
      mode: "steer",
      debounceMs: 500,
      cap: 20,
      drop: "summarize",
      byChannel: { discord: "collect" },
    },
  },
}
```

## Wachtrijopties

Opties zijn van toepassing op levering vanuit de wachtrij. `debounceMs` stelt ook het stiltevenster voor Codex-aansturing in de modus `steer` in:

- `debounceMs`: stiltevenster voordat vervolgberichten of verzamelbundels uit de wachtrij worden verwerkt; in de Codex-modus `steer` is dit het stiltevenster voordat de gebundelde `turn/steer` wordt verzonden. Kale getallen zijn milliseconden; de eenheden `ms`, `s`, `m`, `h` en `d` worden geaccepteerd door `/queue`-opties.
- `cap`: maximaal aantal berichten in de wachtrij per sessie. Waarden lager dan `1` worden genegeerd.
- `drop: "summarize"` (standaard): verwijder indien nodig de oudste items uit de wachtrij, bewaar compacte samenvattingen en injecteer deze als een synthetische vervolgprompt.
- `drop: "old"`: verwijder indien nodig de oudste items uit de wachtrij zonder samenvattingen te bewaren.
- `drop: "new"`: wijs het nieuwste bericht af wanneer de wachtrij al vol is.

Standaardwaarden: `debounceMs: 500`, `cap: 20`, `drop: summarize`.

## Aansturing en streaming

Wanneer kanaalstreaming `partial` of `block` is, kan aansturing eruitzien als meerdere korte zichtbare antwoorden terwijl de actieve run runtimegrenzen bereikt:

- `partial`: de voorvertoning kan vroegtijdig worden afgerond, waarna een nieuwe voorvertoning begint zodra de aansturing is geaccepteerd.
- `block`: blokken ter grootte van een concept kunnen dezelfde opeenvolgende weergave veroorzaken.
- Zonder streaming valt aansturing terug op een vervolg nadat de actieve run is voltooid wanneer de runtime geen aansturing binnen dezelfde beurt kan accepteren.

`steer` breekt actieve tools niet af. Gebruik `/queue interrupt` wanneer het nieuwste bericht de huidige run moet afbreken.

## Prioriteit

Voor de modusselectie hanteert OpenClaw:

1. Inline of opgeslagen overschrijving van `/queue` per sessie.
2. `messages.queue.byChannel.<channel>`.
3. `messages.queue.mode`.
4. Standaard `steer`.

Voor opties hebben inline of opgeslagen `/queue`-opties voorrang op de configuratie. Vervolgens worden in deze volgorde de kanaalspecifieke debounce (`messages.queue.debounceMsByChannel`), standaardwaarden voor plugin-debounce, globale `messages.queue`-opties en ingebouwde standaardwaarden toegepast. `cap` en `drop` zijn globale/sessieopties, geen configuratiesleutels per kanaal.

## Overschrijvingen per sessie

- Verzend `/queue <steer|followup|collect|interrupt>` als zelfstandige opdracht om de wachtrijmodus voor de huidige sessie op te slaan.
- Opties kunnen worden gecombineerd: `/queue collect debounce:0.5s cap:25 drop:summarize`
- `/queue default` of `/queue reset` wist de sessieoverschrijving.

## Annulering van beurten in de wachtrij

Terwijl een prompt in de vervolg-/verzamelwachtrij staat (bijvoorbeeld een TUI- of
webchat-`chat.send` die binnenkomt terwijl een andere beurt actief is), bewaart Gateway een
**door Gateway beheerde annuleringsidentiteit** voor die client-`runId` totdat de inhoud in de wachtrij
wordt uitgevoerd of verwijderd. De identiteit volgt inhoud die in een
overloopsamenvatting is samengevoegd.

- `chat.abort` met een specifieke `runId` annuleert die beurt terwijl deze nog
  in de wachtrij staat, als de aanvrager geautoriseerd is (dezelfde eigendomsregels als voor actieve runs).
- `chat.abort` voor een sessie zonder `runId` annuleert **eerst geautoriseerde beurten
  in de wachtrij** en breekt vervolgens geautoriseerde actieve runs af. Die volgorde voorkomt dat verwerking van de wachtrij
  werk promoveert naar een half gestopte sessie.
- Het wissen van de volledige sessiewachtrij zonder controles per aanvrager is niet het
  stoppad voor sessies met meerdere eigenaren.
- Wachttijden in de wachtrij worden voor `sessions.list` niet weergegeven als actieve agentruns en
  vallen niet onder de time-outsemantiek van actieve runs; alleen de actieve fase doet dat.

Clients die door Gateway worden ondersteund (waaronder `openclaw tui`) sturen prompts tijdens een run door en
laten Gateway de wachtrijmodus toepassen. Esc/`/stop` gebruikt een sessiegebonden afbreking,
zodat verloren lokale handles er niet toe kunnen leiden dat een prompt in de wachtrij alsnog wordt uitgevoerd.

`openclaw chat` en `openclaw tui --local` passen dezelfde vier modi toe in de
ingesloten runtime. Lokale `steer` injecteert in een actieve ingesloten run wanneer die
runtime aansturing accepteert en wordt anders een vervolg; `followup` en
`collect` blijven lokaal wachtend werk; `interrupt` breekt de actieve lokale run af
voordat het nieuwste bericht wordt gestart. De expliciete opdracht `/steer <message>` is
geen opdracht voor de lokale modus.

## Bereik en garanties

- Van toepassing op automatische antwoordruns van agents voor alle inkomende kanalen die de antwoordpijplijn van Gateway gebruiken (WhatsApp-web, Telegram, Slack, Discord, Signal, iMessage, webchat enzovoort).
- De standaardrijstrook (`main`) geldt voor het hele proces voor inkomende berichten en hoofdheartbeats; stel `agents.defaults.maxConcurrent` in om meerdere sessies parallel toe te staan.
- Er kunnen aanvullende rijstroken bestaan (bijvoorbeeld `cron`, `cron-nested`, `nested`, `subagent`), zodat achtergrondtaken parallel kunnen worden uitgevoerd zonder inkomende antwoorden te blokkeren. Geïsoleerde Cron-agentbeurten houden een `cron`-slot bezet terwijl hun interne agentuitvoering `cron-nested` gebruikt. Gedeelde niet-Cron-`nested`-stromen behouden hun eigen rijstrookgedrag. Deze losgekoppelde runs worden bijgehouden als [achtergrondtaken](/nl/automation/tasks).
- Rijstroken per sessie garanderen dat slechts één agentrun tegelijk een bepaalde sessie gebruikt.
- Geen externe afhankelijkheden of achtergrondwerkthreads; uitsluitend TypeScript + promises.

## Probleemoplossing

- Als opdrachten lijken vast te lopen, schakel je uitgebreide logboeken in en zoek je naar regels met "queued for ...ms" om te bevestigen dat de wachtrij wordt verwerkt.
- Runs van de Codex-app-server die een beurt accepteren en vervolgens geen voortgang meer melden, worden door de Codex-adapter onderbroken, zodat de actieve sessierijstrook kan worden vrijgegeven in plaats van op de time-out van de buitenste run te wachten.
- Wanneer diagnostiek is ingeschakeld, worden sessies die na de ingebouwde waarschuwingsdrempel in `processing` blijven zonder waargenomen antwoord-, tool-, status-, blok- of ACP-voortgang, geclassificeerd op basis van de huidige activiteit:
  - Actief werk met recente voortgang wordt gelogd als `session.long_running`. Stille modelaanroepen met een eigenaar blijven ook `session.long_running` tot de ingebouwde afbreekdrempel, zodat trage of niet-streamende providers niet te vroeg als vastgelopen worden gemeld.
  - Actief werk zonder recente voortgang wordt gelogd als `session.stalled`; modelaanroepen met een eigenaar, geblokkeerde toolaanroepen en vastgelopen ingesloten runs schakelen bij of na de afbreekdrempel over naar `session.stalled`. Verouderde model-/toolactiviteit zonder eigenaar wordt niet verborgen als langdurig actief.
  - `session.stuck` is gereserveerd voor herstelbare verouderde sessieboekhouding, waaronder inactieve sessies in de wachtrij met verouderde model-/toolactiviteit zonder eigenaar.
  - `session.stuck` activeert altijd herstel dat de getroffen sessierijstrook kan vrijgeven. Een classificatie als `session.stalled` na de afbreekdrempel (geblokkeerde toolaanroep, vastgelopen modelaanroep of vastgelopen ingesloten run) kan ook actief afbreekherstel activeren, zodat beide classificaties een wachtrij kunnen deblokkeren, niet alleen `session.stuck`.
  - Herhaalde waarschuwingsregels voor `session.stuck` en `session.long_running` in het logboek worden exponentieel minder vaak weergegeven zolang de sessie ongewijzigd blijft; herstelpogingen worden ongeacht die vertraging nog steeds bij elke heartbeat-tik uitgevoerd.

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session)
- [Aansturingswachtrij](/nl/concepts/queue-steering)
- [Aansturen](/nl/tools/steer)
- [Beleid voor nieuwe pogingen](/nl/concepts/retry)
