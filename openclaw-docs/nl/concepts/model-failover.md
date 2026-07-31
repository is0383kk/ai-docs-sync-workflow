---
read_when:
    - Problemen met rotatie van authenticatieprofielen, afkoelperiodes of model-fallbackgedrag vaststellen
    - Failoverregels voor authenticatieprofielen of modellen bijwerken
    - Begrijpen hoe modeloverschrijvingen per sessie samenwerken met nieuwe pogingen via terugvalmodellen
sidebarTitle: Model failover
summary: Hoe OpenClaw auth-profielen roteert en terugvalt op andere modellen
title: Model-failover
x-i18n:
    generated_at: "2026-07-27T05:02:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dfedbc85038eebb5be056a7b3ffa3275b4329a0b0d791e1a2b4701cbaa4b595
    source_path: concepts/model-failover.md
    workflow: 16
---

OpenClaw verwerkt fouten in twee fasen:

1. **Rotatie van authenticatieprofielen** binnen de huidige provider.
2. **Modelterugval** naar het volgende model in `agents.defaults.model.fallbacks`.

## Runtimeverloop

<Steps>
  <Step title="Sessiestatus bepalen">
    Bepaal het actieve sessiemodel en de voorkeur voor het authenticatieprofiel.
  </Step>
  <Step title="Kandidatenketen samenstellen">
    Stel de modelkandidatenketen samen op basis van de huidige modelselectie en het terugvalbeleid voor de bron van die selectie. Geconfigureerde standaardwaarden, primaire modellen van Cron-taken en automatisch geselecteerde terugvalmodellen kunnen geconfigureerde terugvalopties gebruiken; expliciete gebruikersselecties voor sessies zijn strikt.
  </Step>
  <Step title="De huidige provider proberen">
    Probeer de huidige provider met de regels voor rotatie en afkoelperioden van authenticatieprofielen.
  </Step>
  <Step title="Doorgaan bij fouten die failover rechtvaardigen">
    Als de opties van die provider zijn uitgeput door een fout die failover rechtvaardigt, ga je door naar de volgende modelkandidaat.
  </Step>
  <Step title="Terugval gebruiken voor de huidige beurt">
    Voer de succesvolle terugvalkandidaat uit zonder de geselecteerde provider of het geselecteerde model van de sessie te wijzigen.
  </Step>
  <Step title="Opnieuw proberen bij uitsluitend veilige uitputting door overbelasting">
    Als elke kandidaat alleen mislukt doordat providers overbelast zijn, probeer je de volledige beurtlokale keten maximaal 10 keer opnieuw met exponentiële back-off, zolang nog geen tooluitvoering of assistentuitvoer is gestart. Stuur na 30 seconden één statusmelding, zodat de gebruiker niet zonder bericht blijft wachten.
  </Step>
  <Step title="FallbackSummaryError genereren bij uitputting">
    Als elke kandidaat mislukt, genereer je een `FallbackSummaryError` met details per poging en het eerstvolgende einde van een afkoelperiode, indien bekend.
  </Step>
</Steps>

De uitvoering van terugval is beurtlokaal. De antwoordrunner slaat alleen de status van terugvalmeldingen persistent op, zodat `/status` en overgangsmeldingen onderscheid kunnen maken tussen het geselecteerde model en het model dat antwoordde; de terugval wordt niet opgeslagen als de modelselectie voor de volgende beurt.

## Beleid voor selectiebronnen

De selectiebron bepaalt of de terugvalketen is toegestaan:

- **Geconfigureerde standaardwaarde**: `agents.defaults.model.primary` gebruikt `agents.defaults.model.fallbacks`.
- **Primair model van agent**: `agents.entries.*.model` is strikt, tenzij het modelobject van die agent een eigen `fallbacks` bevat. Gebruik `fallbacks: []` om het strikte gedrag expliciet te maken, of een niet-lege lijst om modelterugval voor die agent in te schakelen.
- **Runtimeterugval**: de terugvalkandidaat geldt alleen voor de huidige beurt. De volgende beurt begint opnieuw met het geselecteerde primaire model. OpenClaw herkent nog steeds eerder opgeslagen `modelOverrideSource: "auto"`-vermeldingen, controleert elke 5 minuten hun geconfigureerde oorsprong en wist ze zodra de oorsprong is hersteld. `/new`, `/reset` en `sessions.reset` wissen deze vermeldingen eveneens.
- **Gebruikersoverschrijving voor sessie**: `/model`, de modelkiezer, `session_status(model=...)` en `sessions.patch` schrijven `modelOverrideSource: "user"`. Dit is een exacte sessieselectie. Als de geselecteerde provider of het geselecteerde model mislukt voordat er een antwoord is geproduceerd, meldt OpenClaw de fout in plaats van te antwoorden met een niet-gerelateerde geconfigureerde terugvaloptie.
- **Verouderde sessieoverschrijving**: oudere sessievermeldingen kunnen `modelOverride` zonder `modelOverrideSource` bevatten. OpenClaw behandelt deze als gebruikersoverschrijvingen, zodat een expliciete oude selectie niet stilzwijgend wordt omgezet in terugvalgedrag.
- **Model in Cron-payload**: een Cron-taak met `payload.model` / `--model` is een primair taakmodel, geen gebruikersoverschrijving voor een sessie. Het gebruikt geconfigureerde terugvalopties, tenzij de taak `payload.fallbacks` opgeeft; `payload.fallbacks: []` maakt de Cron-uitvoering strikt.

OpenClaw stuurt een zichtbare melding wanneer een beurt overschakelt op terugval en nog een melding wanneer een latere beurt slaagt met het geselecteerde primaire model. De persistent opgeslagen meldingsstatus voorkomt herhaalde meldingen wanneer opeenvolgende beurten hetzelfde geselecteerde/actieve paar gebruiken, terwijl de modelselectie zelf ongewijzigd blijft.

## Cache voor het overslaan van authenticatiefouten

Standaard behoudt elke nieuwe beurt het bestaande gedrag voor nieuwe terugvalpogingen: OpenClaw probeert elke geconfigureerde terugvalkandidaat opnieuw, inclusief niet-primaire kandidaten die onlangs zijn mislukt met `auth` of `auth_permanent`.

Schakel het onderdrukken van herhaalde authenticatiefouten in met:

```bash
OPENCLAW_FALLBACK_SKIP_TTL_MS=60000
```

Wanneer dit is ingeschakeld, registreert OpenClaw na een fout uit de authenticatieklasse een sessiegebonden oversla-marker in het geheugen voor een niet-primaire terugvalkandidaat, met sessie-id, provider en model als sleutel. Primaire kandidaten worden nooit overgeslagen, zodat bij een expliciete gebruikersselectie van een model nog steeds de werkelijke authenticatiefout wordt weergegeven. De cache is proceslokaal en wordt gewist wanneer de Gateway opnieuw wordt gestart.

De waarde is een TTL in milliseconden. `0` of niet ingesteld schakelt de cache uit. Positieve waarden worden begrensd tussen 1 seconde en 10 minuten.

## Voor de gebruiker zichtbare terugvalmeldingen

Wanneer een sessie overschakelt op een automatisch geselecteerde terugvaloptie, stuurt OpenClaw een statusmelding via hetzelfde antwoordoppervlak:

```text
↪️ Modelterugval: <fallback> (geselecteerd: <primary>; <reason>)
```

Wanneer een latere controle slaagt en de sessie terugkeert naar het geselecteerde primaire model, stuurt OpenClaw:

```text
↪️ Modelterugval opgeheven: <primary> (was <fallback>)
```

Deze meldingen zijn operationele berichten, geen inhoud van de assistent. Ze worden eenmaal per statuswijziging afgeleverd, indien haalbaar ook bij beurten met alleen neveneffecten, maar herhaalde beurtlokale terugvalovergangen zorgen niet voor herhaling. De aflevering omzeilt de normale onderdrukking van bronantwoorden, neemt bij kanalen met threads niet de positie van het eerste assistentantwoord in en wordt uitgesloten van tekst-naar-spraak en extractie van toezeggingen.

## Opslag van authenticatie (sleutels + OAuth)

OpenClaw gebruikt **authenticatieprofielen** voor zowel API-sleutels als OAuth-tokens.

- Geheimen en de runtime-status voor authenticatieroutering bevinden zich in `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.
- Configuratie `auth.profiles` / `auth.order` bevat **alleen metadata en routering** (geen geheimen).
- Verouderd OAuth-bestand dat alleen voor import wordt gebruikt: `~/.openclaw/credentials/oauth.json` (bij het eerste gebruik geïmporteerd in de authenticatieopslag per agent).
- Verouderde bestanden `auth-profiles.json`, `auth-state.json` en bestanden `auth.json` per agent worden geïmporteerd door `openclaw doctor --fix`.

Meer informatie: [OAuth](/nl/concepts/oauth)

Typen aanmeldgegevens:

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` voor sommige providers)
- `type: "token"` → statisch token van het bearer-type, eventueel met een vervaldatum; OpenClaw vernieuwt dit niet (gebruikt voor `aws-sdk` en andere authenticatiemodi met een keten van aanmeldgegevens)

## Profiel-ID's

OAuth-aanmeldingen maken afzonderlijke profielen, zodat meerdere accounts naast elkaar kunnen bestaan.

- Standaard: `provider:default` wanneer geen e-mailadres beschikbaar is.
- OAuth met e-mailadres: `provider:<email>` (bijvoorbeeld `google-antigravity:user@gmail.com`).

Profielen bevinden zich in de opslag voor authenticatieprofielen `openclaw-agent.sqlite` per agent.

## Rotatievolgorde

Wanneer een provider meerdere profielen heeft, kiest OpenClaw als volgt een volgorde:

<Steps>
  <Step title="Expliciete configuratie">
    `auth.order[provider]` (indien ingesteld).
  </Step>
  <Step title="Geconfigureerde profielen">
    `auth.profiles`, gefilterd op provider.
  </Step>
  <Step title="Opgeslagen profielen">
    SQLite-vermeldingen voor authenticatieprofielen per agent voor de provider.
  </Step>
</Steps>

Als er geen expliciete volgorde is geconfigureerd, gebruikt OpenClaw een round-robinvolgorde:

- **Primaire sleutel:** profieltype (**eerst OAuth, dan statisch token, dan API-sleutel**).
- **Secundaire sleutel voor OAuth:** profielen met een momenteel bruikbaar toegangstoken vóór
  profielen waarvan het toegangstoken is verlopen. Verlopen OAuth-profielen blijven in aanmerking komen, zodat
  de runtime ze kan vernieuwen wanneer er geen bruikbaar alternatief beschikbaar is.
- **Volgende sleutel:** `usageStats.lastUsed` (oudste eerst, binnen elke type-/statuscategorie).
- **Profielen in een afkoelperiode of uitgeschakelde profielen** worden naar het einde verplaatst, gerangschikt op het eerstvolgende einde.

### Sessiegebondenheid (cachevriendelijk)

OpenClaw **zet het gekozen authenticatieprofiel per sessie vast** om providercaches warm te houden. Het roteert **niet** bij elke aanvraag. Het vastgezette profiel wordt hergebruikt totdat:

- de sessie wordt gereset (`/new` / `/reset`)
- een Compaction is voltooid (het aantal compacties wordt verhoogd)
- het profiel in een afkoelperiode staat of is uitgeschakeld

Handmatige selectie via `/model …@<profileId>` stelt een **gebruikersoverschrijving** voor die sessie in en wordt niet automatisch geroteerd totdat een nieuwe sessie begint.

<Note>
Automatisch vastgezette profielen (geselecteerd door de sessierouter) worden behandeld als een **voorkeur**: ze worden als eerste geprobeerd, maar OpenClaw kan bij frequentielimieten of time-outs naar een ander profiel roteren. Wanneer het oorspronkelijke profiel weer beschikbaar is, kunnen nieuwe uitvoeringen er opnieuw de voorkeur aan geven zonder het geselecteerde model of de runtime te wijzigen. Door de gebruiker vastgezette profielen blijven aan dat profiel gebonden; als het mislukt en modelterugvalopties zijn geconfigureerd, gaat OpenClaw naar het volgende model in plaats van van profiel te wisselen.
</Note>

### OpenAI Codex-abonnement met API-sleutel als reserveoptie

Voor OpenAI-agentmodellen staan authenticatie en runtime los van elkaar. `openai/gpt-*` blijft op de Codex-harness, terwijl de authenticatie kan roteren tussen een Codex-abonnementsprofiel en een OpenAI API-sleutel als reserveoptie.

Gebruik `auth.order.openai` voor de volgorde die aan de gebruiker wordt getoond:

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Gebruik `openai:*` voor zowel ChatGPT/Codex OAuth-profielen als OpenAI API-sleutelprofielen. Wanneer het abonnement een Codex-gebruikslimiet bereikt, registreert OpenClaw het exacte resettijdstip als Codex dit verstrekt, probeert het volgende geordende authenticatieprofiel en houdt de uitvoering binnen de Codex-harness. Zodra het resettijdstip is verstreken, komt het abonnementsprofiel weer in aanmerking en kan de volgende automatische selectie ernaar terugkeren.

Gebruik alleen een door de gebruiker vastgezet profiel wanneer je voor die sessie één account/sleutel wilt afdwingen. Door de gebruiker vastgezette profielen zijn bewust strikt en schakelen niet stilzwijgend over naar een ander profiel.

## Afkoelperioden

Wanneer een profiel mislukt door authenticatie- of frequentielimietfouten (of een time-out die op frequentiebeperking lijkt), plaatst OpenClaw het in een afkoelperiode en gaat het door naar het volgende profiel.

<AccordionGroup>
  <Accordion title="Wat in de categorie frequentielimiet/time-out valt">
    Die categorie voor frequentielimieten is ruimer dan alleen `429`: deze omvat ook providerberichten zoals `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, `throttled`, `resource exhausted` en periodieke gebruiksvensterlimieten zoals `weekly limit reached` of `monthly limit exhausted`.

    Opmaak- en ongeldige-aanvraagfouten zijn meestal definitief, omdat een nieuwe poging met dezelfde payload op dezelfde manier zou mislukken. Daarom geeft OpenClaw deze weer in plaats van authenticatieprofielen te roteren. Bekende herstelpaden voor nieuwe pogingen kunnen expliciet worden ingeschakeld: fouten bij de validatie van toolaanroep-ID's van Cloud Code Assist worden bijvoorbeeld opgeschoond en eenmaal opnieuw geprobeerd via het beleid `allowFormatRetry`.

    OpenAI-compatibele **door de provider voltooide** stop-/eindredenen zoals `Unhandled stop reason: error`, `stop reason: error`, `reason: error` en `Provider finish_reason: error` worden geclassificeerd als **`server_error`** (HTTP-achtige status 500), niet als time-out. Ze blijven in aanmerking komen voor failover via model-/profielrotatie, maar diagnostische gegevens behouden de tekst van de eindreden van de provider in plaats van de voor de gebruiker bestemde tekst te herschrijven naar "Time-out bij LLM-aanvraag." Transportgerelateerde eindredenen zoals `Provider finish_reason: abort`, `network_error` en `malformed_response` blijven in de categorie time-out/failover (status 408).

    Algemene servertekst kan ook in die time-outcategorie vallen wanneer de bron overeenkomt met een bekend tijdelijk patroon. Het kale stream-wrapperbericht van de modelruntime `An unknown error occurred` wordt bijvoorbeeld voor elke provider beschouwd als een reden voor failover, omdat de gedeelde modelruntime dit produceert wanneer providerstreams eindigen met `stopReason: "aborted"` of `stopReason: "error"` zonder specifieke details. JSON-`api_error`-payloads met tijdelijke servertekst zoals `internal server error`, `unknown error, 520`, `upstream error` of `backend error` worden eveneens behandeld als time-outs die failover rechtvaardigen.

    OpenRouter-specifieke generieke upstreamtekst zoals alleen `Provider returned error` wordt uitsluitend als time-out behandeld wanneer de providercontext daadwerkelijk OpenRouter is. Generieke interne fallbacktekst zoals `LLM request failed with an unknown error.` blijft terughoudend en activeert niet op zichzelf failover.

  </Accordion>
  <Accordion title="Limieten voor retry-after van de SDK">
    Sommige provider-SDK's kunnen anders gedurende een lang `Retry-After`-venster wachten voordat ze de besturing teruggeven aan OpenClaw. Voor op Stainless gebaseerde SDK's zoals Anthropic en OpenAI beperkt OpenClaw interne wachttijden van de SDK voor `retry-after-ms` / `retry-after` standaard tot 60 seconden en geeft het langere opnieuw te proberen responsen onmiddellijk door, zodat dit failoverpad kan worden uitgevoerd. Pas de limiet aan of schakel deze uit met `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS`; zie [Gedrag bij opnieuw proberen](/nl/concepts/retry).
  </Accordion>
  <Accordion title="Modelgebonden cooldowns">
    Cooldowns voor snelheidslimieten kunnen ook modelgebonden zijn:

    - OpenClaw registreert `cooldownModel` voor fouten door snelheidslimieten wanneer de id van het falende model bekend is.
    - Een verwant model van dezelfde provider kan nog steeds worden geprobeerd wanneer de cooldown aan een ander model is gebonden.
    - Vensters voor facturering/uitschakeling blokkeren nog steeds het volledige profiel voor alle modellen.

  </Accordion>
</AccordionGroup>

Reguliere cooldowns (niet voor facturering en niet voor permanente authenticatiefouten) schalen met het recente aantal fouten van het profiel:

- 1e fout: 30 seconden
- 2e fout: 1 minuut
- 3e en volgende fouten: 5 minuten (maximum)

Tellers worden opnieuw ingesteld zodra het ingebouwde foutvenster van het profiel is verstreken.

De status wordt opgeslagen in de SQLite-authenticatiestatus per agent onder `usageStats`:

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## Uitschakeling wegens facturering

Fouten met facturering/tegoed (bijvoorbeeld "onvoldoende tegoed" / "tegoedsaldo te laag") worden beschouwd als reden voor failover, maar zijn meestal niet tijdelijk. In plaats van een korte cooldown markeert OpenClaw het profiel als **uitgeschakeld** (met een langere wachttijd) en schakelt het door naar het volgende profiel/de volgende provider.

<Note>
Niet elke respons die op een factureringsfout lijkt is `402`, en niet elke HTTP-`402` komt hier terecht. OpenClaw houdt expliciete factureringstekst in het factureringstraject, zelfs wanneer een provider in plaats daarvan `401` of `403` retourneert, maar providerspecifieke matchers blijven beperkt tot de provider die er eigenaar van is (bijvoorbeeld OpenRouter `403 Key limit exceeded`).

Tijdelijke fouten met `402`-gebruiksvensters en bestedingslimieten van organisaties/werkruimten worden ondertussen geclassificeerd als `rate_limit` wanneer het bericht opnieuw te proberen lijkt (bijvoorbeeld `weekly usage limit exhausted`, `daily limit reached, resets tomorrow` of `organization spending limit exceeded`). Deze blijven het pad voor korte cooldown/failover volgen in plaats van het pad voor langdurige uitschakeling wegens facturering.
</Note>

Permanente authenticatiefouten met hoge zekerheid (ingetrokken/gedeactiveerde sleutels, gedeactiveerde werkruimten) krijgen een vergelijkbaar uitschakelingstraject, maar herstellen veel sneller dan bij facturering, omdat sommige providers tijdens incidenten tijdelijk payloads tonen die op authenticatiefouten lijken.

De status wordt opgeslagen in de SQLite-authenticatiestatus per agent:

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

Overbelastings- en snelheidslimietfouten worden agressiever afgehandeld dan cooldowns voor facturering: standaard staat OpenClaw één nieuwe poging met een authenticatieprofiel van dezelfde provider toe en schakelt het daarna zonder te wachten over naar de volgende geconfigureerde modelfallback.

## Modelfallback

Als alle profielen voor een provider mislukken, gaat OpenClaw naar het volgende model in `agents.defaults.model.fallbacks`. Dit geldt voor authenticatiefouten, snelheidslimieten en time-outs waarbij profielrotatie is uitgeput (andere fouten gaan niet door naar de fallback). Providerfouten die onvoldoende details beschikbaar stellen, worden nog steeds nauwkeurig gelabeld in de fallbackstatus: `empty_response` betekent dat de provider geen bruikbaar bericht of status retourneerde, `no_error_details` betekent dat de provider expliciet `Unknown error (no error details in response)` retourneerde en `unclassified` betekent dat OpenClaw het onbewerkte voorbeeld heeft behouden, maar dat nog geen classificator ermee overeenkwam.

Signalen dat de provider bezet is, zoals `ModelNotReadyException`, komen in de categorie overbelasting terecht en volgen hetzelfde beleid van één rotatie en daarna fallback als snelheidslimieten (zie de tabel met standaardwaarden hierboven).

Als de volledige keten met kandidaten uitsluitend door overbelastingsfouten is uitgeput, probeert de uitvoerder van antwoorden de keten in dezelfde beurt maximaal 10 keer opnieuw. Opnieuw proberen van de volledige beurt is alleen toegestaan voordat de uitvoering van tools of de uitvoer van de assistent begint, om dubbele mutaties of berichten te voorkomen als overbelasting optreedt nadat waarneembaar werk is uitgevoerd. De wachttijd begint bij 2.5 seconden en verdubbelt tot een maximum van 30 seconden. Zodra de beurt 30 seconden heeft gewacht, verstuurt OpenClaw één tijdelijke statusmelding: `The AI service is temporarily overloaded. I’m still retrying; this may take a few minutes.` De nieuwe poging en een eventuele winnende fallback blijven beperkt tot de beurt; reguliere tijdelijke serverfouten behouden hun afzonderlijke beleid van één nieuwe poging.

Wanneer een uitvoering begint met de geconfigureerde standaard-primaire optie, de primaire optie van een Cron-taak, de primaire optie van een agent met expliciete fallbacks of een automatisch geselecteerde fallbackoverschrijving, kan OpenClaw de bijbehorende geconfigureerde fallbackketen doorlopen. Primaire opties van agents zonder expliciete fallbacks en expliciete gebruikersselecties (bijvoorbeeld `/model ollama/qwen3.5:27b`, de modelkiezer, `sessions.patch` of eenmalige CLI-overschrijvingen van provider/model) zijn strikt: als die provider/dat model onbereikbaar is of faalt voordat een antwoord wordt geproduceerd, meldt OpenClaw de fout in plaats van te antwoorden via een niet-gerelateerde fallback.

### Regels voor de kandidatenketen

OpenClaw stelt de kandidatenlijst samen uit de momenteel aangevraagde `provider/model` en de geconfigureerde fallbacks.

<AccordionGroup>
  <Accordion title="Regels">
    - Het aangevraagde model staat altijd als eerste.
    - Expliciet geconfigureerde fallbacks worden ontdubbeld, maar niet gefilterd op basis van de lijst met toegestane modellen. Ze worden behandeld als expliciete intentie van de beheerder.
    - Als de huidige uitvoering al een geconfigureerde fallback binnen dezelfde providerfamilie gebruikt, blijft OpenClaw de volledige geconfigureerde keten gebruiken.
    - Wanneer geen expliciete fallbackoverschrijving wordt opgegeven, worden geconfigureerde fallbacks vóór de geconfigureerde primaire optie geprobeerd, zelfs als het aangevraagde model een andere provider gebruikt.
    - Wanneer geen expliciete fallbackoverschrijving aan de fallbackuitvoerder wordt doorgegeven, wordt de geconfigureerde primaire optie aan het einde toegevoegd, zodat de keten kan terugkeren naar de normale standaardwaarde zodra eerdere kandidaten zijn uitgeput.
    - Wanneer een aanroeper `fallbacksOverride` doorgeeft, gebruikt de uitvoerder uitsluitend het aangevraagde model plus die overschrijvingslijst. Een lege lijst schakelt modelfallback uit en voorkomt dat de geconfigureerde primaire optie als verborgen doel voor een nieuwe poging wordt toegevoegd.

  </Accordion>
</AccordionGroup>

### Welke fouten doorgaan naar de fallback

<Tabs>
  <Tab title="Gaat door bij">
    - authenticatiefouten
    - snelheidslimieten en uitgeputte cooldowns
    - overbelastingsfouten/fouten doordat de provider bezet is
    - failoverfouten in de vorm van time-outs
    - uitschakeling wegens facturering
    - `LiveSessionModelSwitchError`, dat wordt genormaliseerd naar een failoverpad, zodat een verouderd permanent opgeslagen model geen buitenste lus voor nieuwe pogingen veroorzaakt
    - andere niet-herkende fouten wanneer er nog kandidaten over zijn

  </Tab>
  <Tab title="Gaat niet door bij">
    - expliciete afbrekingen die niet de vorm van een time-out/failover hebben
    - contextoverloopfouten die binnen de logica voor Compaction/opnieuw proberen moeten blijven (bijvoorbeeld `request_too_large`, `input token count exceeds the maximum number of input tokens`, `input exceeds the maximum number of tokens`, `input too long for the model` of `ollama error: context length exceeded`)
    - een laatste onbekende fout wanneer er geen kandidaten meer over zijn
    - veiligheidsweigeringen van Claude Fable 5; directe aanvragen met API-sleutels handelen deze in plaats daarvan op providerniveau af via de server-side fallback van Anthropic naar `claude-opus-4-8` (zie [Anthropic](/nl/providers/anthropic#safety-refusal-fallback-claude-fable-5))

  </Tab>
</Tabs>

### Gedrag voor cooldown overslaan versus testen

Wanneer elk authenticatieprofiel voor een provider al een cooldown heeft, slaat OpenClaw die provider niet automatisch voor altijd over. Het neemt een beslissing per kandidaat:

<AccordionGroup>
  <Accordion title="Beslissingen per kandidaat">
    - Bij permanente authenticatiefouten wordt de volledige provider onmiddellijk overgeslagen.
    - Uitschakelingen wegens facturering leiden meestal tot overslaan, maar de primaire kandidaat kan nog steeds met een begrensde frequentie worden getest, zodat herstel mogelijk is zonder opnieuw op te starten.
    - De primaire kandidaat kan vlak voor het verlopen van de cooldown worden getest, met een begrensde frequentie per provider.
    - Verwante fallbacks van dezelfde provider kunnen ondanks de cooldown worden geprobeerd wanneer de fout tijdelijk lijkt (`rate_limit`, `overloaded` of onbekend). Dit is vooral relevant wanneer een snelheidslimiet modelgebonden is en een verwant model mogelijk onmiddellijk kan herstellen.
    - Tijdelijke cooldowntests zijn beperkt tot één per provider per fallbackuitvoering, zodat één provider de fallback tussen providers niet vertraagt.

  </Accordion>
</AccordionGroup>

## Sessieoverschrijvingen en live van model wisselen

Wijzigingen van het sessiemodel zijn gedeelde status. De actieve uitvoerder, de opdracht `/model`, Compaction-/sessie-updates en reconciliatie van livesessies lezen of schrijven allemaal delen van dezelfde sessievermelding. De uitvoering van fallbacks schrijft geen modelselectievelden en kan daarom tijdens het opnieuw proberen geen recentere handmatige selectie vervangen.

Live van model wisselen volgt deze regels:

- Alleen expliciete, door de gebruiker aangestuurde modelwijzigingen markeren een wachtende livewissel. Dit omvat `/model`, `session_status(model=...)` en `sessions.patch`.
- Door het systeem aangestuurde modelwijzigingen zoals fallbackrotatie, Heartbeat-overschrijvingen of Compaction markeren nooit uit zichzelf een wachtende livewissel.
- Door gebruikers aangestuurde modeloverschrijvingen worden voor het fallbackbeleid behandeld als exacte selecties, zodat een onbereikbare geselecteerde provider als fout wordt weergegeven in plaats van te worden gemaskeerd door `agents.defaults.model.fallbacks`.
- Fallbackkandidaten tijdens runtime blijven beperkt tot de beurt. De volgende beurt begint met het momenteel geselecteerde model, inclusief een handmatige selectie die tijdens de vorige uitvoering is binnengekomen.
- Eerder opgeslagen automatische fallbackoverschrijvingen blijven ondersteund: OpenClaw test periodiek hun geconfigureerde oorsprong en wist de overschrijving wanneer die is hersteld; `/new`, `/reset` en `sessions.reset` wissen automatisch aangemaakte overschrijvingen onmiddellijk.
- Antwoorden aan gebruikers kondigen fallbackovergangen en herstel na het wissen van een fallback eenmaal per statuswijziging aan. Opeenvolgende beurten met hetzelfde geselecteerde/actieve paar herhalen de melding niet.
- `/status` toont het geselecteerde model en, wanneer de fallbackstatus afwijkt, het actieve fallbackmodel en de reden.
- Bij reconciliatie van livesessies krijgen permanent opgeslagen sessieoverschrijvingen voorrang boven verouderde modelvelden tijdens runtime.
- Als een fout bij live wisselen verwijst naar een latere kandidaat in de actieve fallbackketen, springt OpenClaw rechtstreeks naar dat geselecteerde model in plaats van eerst niet-gerelateerde kandidaten te doorlopen.

De actieve uitvoering draagt de gekozen kandidaat rechtstreeks mee. Live reconciliatie wijzigt die kandidaat alleen bij een expliciete wachtende gebruikerswissel, zodat geen tijdelijke fallbackoverschrijving of terugdraaiing nodig is.

## Observeerbaarheid en foutoverzichten

`runWithModelFallback(...)` registreert details per poging die worden gebruikt voor logboeken en gebruikersgerichte cooldownmeldingen:

- geprobeerde provider/model
- reden (`rate_limit`, `overloaded`, `billing`, `auth`, `model_not_found` en vergelijkbare failoverredenen)
- optionele status/code
- voor mensen leesbaar foutoverzicht

Gestructureerde `model_fallback_decision`-logboeken bevatten ook platte `fallbackStep*`-velden wanneer een kandidaat faalt, wordt overgeslagen of een latere fallback slaagt. Deze velden maken de geprobeerde overgang expliciet (`fallbackStepFromModel`, `fallbackStepToModel`, `fallbackStepFromFailureReason`, `fallbackStepFromFailureDetail`, `fallbackStepFinalOutcome`), zodat exporteurs van logboeken en diagnostische gegevens de primaire fout kunnen reconstrueren, zelfs wanneer de uiteindelijke fallback ook faalt.

Wanneer elke kandidaat mislukt, genereert OpenClaw `FallbackSummaryError`. De buitenste antwoordrunner kan dit gebruiken om een specifiekere melding op te stellen, zoals "voor alle modellen geldt tijdelijk een frequentielimiet", en de eerstvolgende vervaldatum van de afkoelperiode vermelden wanneer die bekend is.

Dat overzicht van de afkoelperiode houdt rekening met het model:

- niet-gerelateerde modelspecifieke frequentielimieten worden genegeerd voor de gebruikte provider-/modelketen
- als de resterende blokkering een overeenkomende modelspecifieke frequentielimiet is, rapporteert OpenClaw de laatste overeenkomende vervaldatum die dat model nog blokkeert

## Gerelateerde configuratie

Zie [Gateway-configuratie](/nl/gateway/configuration) voor:

- `auth.profiles` / `auth.order`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel`-routering

Zie [Modellen](/nl/concepts/models) voor het bredere overzicht van modelselectie en fallback.
