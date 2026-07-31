---
read_when:
    - Je wilt geplande taken en wake-ups
    - Je debugt de uitvoering en logboeken van Cron
summary: CLI-referentie voor `openclaw cron` (achtergrondtaken plannen en uitvoeren)
title: Cron
x-i18n:
    generated_at: "2026-07-27T04:51:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5989a7558f4ae2f046480b6a52e3fa296c95d47b14b11c5bad709fea4af6af3e
    source_path: cli/cron.md
    workflow: 16
---

# `openclaw cron`

Beheer crontaken voor de Gateway-planner.

<Tip>
Voer `openclaw cron --help` uit voor het volledige opdrachtenaanbod. Zie [Crontaken](/nl/automation/cron-jobs) voor de conceptuele handleiding.
</Tip>

<Note>
Alle cronwijzigingen (`add`/`create`, `update`/`edit`, `remove`, `run`) vereisen `operator.admin`. Uitvoeringen met een opdrachtpayload worden rechtstreeks in het Gateway-proces uitgevoerd, niet als een `tools.exec`-toolaanroep van een agent; `tools.exec.*` en uitvoeringsgoedkeuringen blijven van toepassing op uitvoeringstools die zichtbaar zijn voor het model.
</Note>

## Taken snel aanmaken

`openclaw cron create` is een alias voor `openclaw cron add`. Plaats voor nieuwe taken eerst het schema en daarna de prompt:

```bash
openclaw cron create "0 7 * * *" \
  "Vat de updates van de afgelopen nacht samen." \
  --name "Ochtendoverzicht" \
  --agent ops
```

Gebruik `--webhook <url>` wanneer de taak de voltooide payload via POST moet verzenden in plaats van deze bij een chatdoel af te leveren:

```bash
openclaw cron create "0 18 * * 1-5" \
  "Vat de implementaties van vandaag samen als JSON." \
  --name "Implementatieoverzicht" \
  --webhook "https://example.invalid/openclaw/cron"
```

Gebruik `--command` voor deterministische shellachtige taken die binnen OpenClaw-cron worden uitgevoerd zonder een geïsoleerde agent-/modeluitvoering te starten:

```bash
openclaw cron create "*/15 * * * *" \
  --name "Wachtrijdieptecontrole" \
  --command "scripts/check-queue.sh" \
  --command-cwd "/srv/app" \
  --announce \
  --channel telegram \
  --to "-1001234567890"
```

`--command <shell>` slaat `argv: ["sh", "-lc", <shell>]` op. Gebruik `--command-argv '["node","scripts/report.mjs"]'` voor exacte argv-uitvoering. Opdrachttaken leggen stdout/stderr vast, registreren de normale crongeschiedenis en routeren uitvoer via dezelfde afleveringsmodi `announce`, `webhook` of `none` als geïsoleerde taken. Een opdracht die uitsluitend `NO_REPLY` afdrukt, wordt onderdrukt.

## Sessies

`--session` accepteert `main`, `isolated`, `current` of `session:<id>`.

<AccordionGroup>
  <Accordion title="Sessiesleutels">
    - `main` wordt gekoppeld aan de hoofdsessie van de agent.
    - `isolated` maakt voor elke uitvoering een nieuw transcript en een nieuwe sessie-id aan.
    - `current` wordt gekoppeld aan de actieve sessie op het moment van aanmaken.
    - `session:<id>` wordt vastgezet op een expliciete permanente sessiesleutel.

  </Accordion>
  <Accordion title="Semantiek van geïsoleerde sessies">
    Bij geïsoleerde uitvoeringen wordt de omringende gesprekscontext opnieuw ingesteld. Kanaal- en groepsroutering, verzend-/wachtrijbeleid, verhoging, oorsprong en ACP-runtimebinding worden voor de nieuwe uitvoering opnieuw ingesteld. Veilige voorkeuren en expliciet door de gebruiker geselecteerde model- of authenticatieoverschrijvingen kunnen tussen uitvoeringen behouden blijven.
  </Accordion>
</AccordionGroup>

## Aflevering

`openclaw cron list` en `openclaw cron show <job-id>` tonen een voorbeeld van de bepaalde afleveringsroute. Voor `channel: "last"` laat het voorbeeld zien of de route vanuit de hoofd- of huidige sessie is bepaald, of gesloten zal mislukken.

Doelen met een providervoorvoegsel kunnen onbepaalde aankondigingskanalen eenduidig maken. Zo selecteert `to: "telegram:123"` Telegram wanneer `delivery.channel` is weggelaten of `last` is. Alleen voorvoegsels die door de geladen Plugin worden geadverteerd, zijn providerselectoren. Als `delivery.channel` expliciet is, moet het voorvoegsel met dat kanaal overeenkomen; `channel: "whatsapp"` met `to: "telegram:123"` wordt geweigerd. Dienstvoorvoegsels zoals `imessage:` en `sms:` blijven doelsyntaxis die eigendom is van het kanaal.

<Note>
Geïsoleerde `cron add`-taken gebruiken standaard aflevering via `--announce`. Gebruik `--no-deliver` om uitvoer intern te houden. `--deliver` blijft beschikbaar als verouderde alias voor `--announce`.
</Note>

### Eigendom van aflevering

Aflevering van geïsoleerde cronchats wordt gedeeld tussen de agent en het uitvoeringsprogramma:

- De agent kan rechtstreeks verzenden met de tool `message` wanneer een chatroute beschikbaar is.
- `announce` levert het definitieve antwoord alleen als terugval af wanneer de agent niet rechtstreeks naar het bepaalde doel heeft verzonden.
- `webhook` plaatst de voltooide payload op een URL.
- `none` schakelt terugvalaflevering door het uitvoeringsprogramma uit.

Gebruik `cron add|create --webhook <url>` of `cron edit <job-id> --webhook <url>` om Webhook-aflevering in te stellen. Combineer `--webhook` niet met vlaggen voor chataflevering, zoals `--announce`, `--no-deliver`, `--channel`, `--to`, `--thread-id` of `--account`.

`cron edit <job-id>` kan afzonderlijke routeringsvelden voor aflevering uitschakelen met `--clear-channel`, `--clear-to`, `--clear-thread-id` en `--clear-account` (elk wordt geweigerd wanneer het met de bijbehorende instelvlag wordt gecombineerd). Anders dan `--no-deliver`, dat alleen terugvalaflevering door het uitvoeringsprogramma uitschakelt, verwijderen deze het opgeslagen veld, zodat de taak dat deel van de route weer op basis van de standaardwaarden bepaalt.

`--announce` is terugvalaflevering door het uitvoeringsprogramma voor het definitieve antwoord. `--no-deliver` schakelt die terugval uit, maar verwijdert de tool `message` van de agent niet wanneer een chatroute beschikbaar is.

Herinneringen die vanuit een actieve chat zijn aangemaakt, behouden het actuele afleveringsdoel van de chat voor terugvalaankondigingen. Interne sessiesleutels kunnen kleine letters bevatten; gebruik ze niet als gezaghebbende bron voor hoofdlettergevoelige provider-id's, zoals Matrix-ruimte-id's.

### Aflevering bij fouten

Foutmeldingen worden in deze volgorde bepaald:

1. `delivery.failureDestination` voor de taak.
2. Globale `cron.failureDestination`.
3. Het primaire aankondigingsdoel van de taak (wanneer geen van bovenstaande opties een concrete bestemming oplevert).

<Note>
Taken in de hoofdsessie mogen `delivery.failureDestination` alleen gebruiken wanneer de primaire afleveringsmodus `webhook` is. Geïsoleerde taken accepteren dit in alle modi.
</Note>

Geïsoleerde cronuitvoeringen behandelen fouten van de agent op uitvoeringsniveau als taakfouten, zelfs wanneer er geen antwoordpayload wordt geproduceerd. Hierdoor verhogen model-/providerfouten nog steeds de fouttellers en activeren ze foutmeldingen.

Cronopdrachttaken starten geen geïsoleerde agentbeurt. Een afsluitcode van nul registreert `ok`; een afsluitcode anders dan nul, een signaal, een time-out of een time-out wegens ontbrekende uitvoer registreert `error` en kan hetzelfde pad voor foutmeldingen activeren.

Als een geïsoleerde uitvoering een time-out bereikt vóór het eerste modelverzoek, bevatten `openclaw cron show` en `openclaw cron runs` een fasespecifieke fout, zoals `setup timed out before runner start`, of een vastloopmelding die de laatst bekende opstartfase noemt (bijvoorbeeld `context-engine`). Voor providers die door een CLI worden ondersteund, blijft de bewaking vóór het model actief totdat de externe CLI-beurt start. Daardoor worden vastlopers bij het opzoeken van de sessie, hooks, authenticatie, de prompt en CLI-instelling gemeld als cronfouten vóór het model.

## Planning

### Eenmalige taken

`--at <datetime>` plant een eenmalige uitvoering. Datum-tijdwaarden zonder offset worden als UTC behandeld, tenzij je ook `--tz <iana>` doorgeeft. Daarmee wordt de kloktijd in de opgegeven tijdzone geïnterpreteerd.

<Note>
Eenmalige taken worden na succes standaard verwijderd. Gebruik `--keep-after-run` om ze te behouden.
</Note>

### Terugkerende taken

Terugkerende taken gebruiken exponentiële terugvalvertraging na opeenvolgende fouten: 30s, 1m, 5m, 15m, 60m. Na de volgende geslaagde uitvoering wordt het normale schema hervat.

Overgeslagen uitvoeringen worden afzonderlijk van uitvoeringsfouten bijgehouden. Ze hebben geen invloed op de terugvalvertraging, maar met `openclaw cron edit <job-id> --failure-alert-include-skipped` kunnen foutwaarschuwingen ook herhaaldelijk melding maken van overgeslagen uitvoeringen.

Voor geïsoleerde taken die zijn gericht op een lokaal geconfigureerde modelprovider (basis-URL op loopback, een privénetwerk of `.local`) voert cron een lichte providercontrole uit voordat de agentbeurt wordt gestart: `api: "ollama"`-providers worden gecontroleerd op `/api/tags`; andere lokale OpenAI-compatibele providers (`api: "openai-completions"`, bijvoorbeeld vLLM, SGLang, LM Studio) worden gecontroleerd op `/models`. Als het eindpunt onbereikbaar is, wordt de uitvoering geregistreerd als `skipped` en bij een later schema opnieuw geprobeerd. Het bereikbaarheidsresultaat wordt per eindpunt 5 minuten in de cache opgeslagen, zodat veel taken die dezelfde lokale server gebruiken deze niet bestoken met herhaalde controles.

Crontaken, runtime-status in afwachting en uitvoeringsgeschiedenis bevinden zich in de gedeelde SQLite-statusdatabase. Verouderde bestanden `jobs.json`, `<name>-state.json` en `runs/*.jsonl` worden eenmaal geïmporteerd en hernoemd met het achtervoegsel `.migrated`. Bewerk na de import schema's met `openclaw cron add|edit|remove` in plaats van JSON-bestanden te bewerken.

### Handmatige uitvoeringen

`openclaw cron run <job-id>` forceert standaard een uitvoering en keert terug zodra de handmatige uitvoering in de wachtrij staat. Geslaagde antwoorden bevatten `{ ok: true, enqueued: true, runId }`. Gebruik de geretourneerde `runId` om het latere resultaat te bekijken:

```bash
openclaw cron run <job-id>
openclaw cron runs --id <job-id> --run-id <run-id>
```

Voeg `--wait` toe wanneer een script moet blokkeren totdat precies die uitvoering in de wachtrij een eindstatus registreert:

```bash
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
```

Met `--wait` roept de CLI nog steeds eerst `cron.run` aan en peilt daarna `cron.runs` voor de geretourneerde `runId`. De opdracht wordt alleen afgesloten met `0` wanneer de uitvoering eindigt met status `ok`. De opdracht wordt afgesloten met een niet-nulwaarde wanneer de uitvoering eindigt met `error` of `skipped`, wanneer het Gateway-antwoord geen `runId` bevat of wanneer `--wait-timeout` verloopt (standaard `10m`, standaard elke `2s` gepeild). `--poll-interval` moet groter zijn dan nul.

<Note>
Gebruik `--due` wanneer je wilt dat de handmatige opdracht alleen wordt uitgevoerd als de taak momenteel aan de beurt is. Als `--due --wait` geen uitvoering in de wachtrij plaatst, retourneert de opdracht het normale antwoord voor geen uitvoering in plaats van te peilen.
</Note>

## Modellen

`cron add|edit --model <ref>` selecteert een toegestaan model voor de taak. `cron add|edit --fallbacks <list>` stelt terugvalmodellen per taak in, bijvoorbeeld `--fallbacks openrouter/gpt-4.1-mini,openai/gpt-5`; geef `--fallbacks ""` door voor een strikte uitvoering zonder terugvalmodellen. `cron edit <job-id> --clear-fallbacks` verwijdert de terugvaloverschrijving per taak. `cron edit <job-id> --clear-model` verwijdert de modeloverschrijving per taak, zodat de taak de normale prioriteitsvolgorde voor cronmodelselectie volgt (een opgeslagen overschrijving voor de cronsessie indien aanwezig, anders het agent-/standaardmodel); dit kan niet worden gecombineerd met `--model`. `cron add|edit --thinking <level>` stelt een denkoverschrijving per taak in; `cron edit <job-id> --clear-thinking` verwijdert deze, zodat de taak de normale prioriteitsvolgorde voor crondenken volgt, en kan niet worden gecombineerd met `--thinking`.

<Warning>
Als het model niet is toegestaan of niet kan worden bepaald, laat cron de uitvoering mislukken met een expliciete validatiefout in plaats van terug te vallen op de agent- of standaardmodelselectie van de taak.
</Warning>

Cron-`--model` is een **primair taakmodel**, geen `/model`-overschrijving voor de chatsessie. Dat betekent:

- Geconfigureerde modelterugvallen blijven van toepassing wanneer het geselecteerde taakmodel mislukt.
- De `fallbacks` van de payload per taak vervangt de geconfigureerde lijst met terugvalmodellen wanneer deze aanwezig is.
- Een lege lijst met terugvalmodellen per taak (`--fallbacks ""` of `fallbacks: []` in de taakpayload/API) maakt de cronuitvoering strikt.
- Wanneer een taak `--model` heeft, maar er geen lijst met terugvalmodellen is geconfigureerd, geeft OpenClaw een expliciete lege terugvaloverschrijving door, zodat het primaire agentmodel niet als verborgen doel voor een nieuwe poging wordt toegevoegd.
- Controles vooraf voor lokale providers doorlopen geconfigureerde terugvalmodellen voordat een cronuitvoering als `skipped` wordt gemarkeerd.

`openclaw doctor` rapporteert taken waarvoor `payload.model` al is ingesteld, inclusief aantallen per providernamespace en afwijkingen ten opzichte van `agents.defaults.model`. Gebruik deze controle wanneer het gedrag voor authenticatie, providers of facturering verschilt tussen livechat en geplande taken.

### Prioriteitsvolgorde voor geïsoleerde cronmodellen

Geïsoleerde cron bepaalt het actieve model in deze volgorde:

1. Overschrijving door Gmail-hook.
2. `--model` per taak.
3. Opgeslagen modeloverschrijving voor de cronsessie (wanneer de gebruiker er een heeft geselecteerd).
4. Agent- of standaardmodelselectie.

### Snelle modus

De snelle modus voor geïsoleerde Cron-taken volgt de opgeloste live modelselectie. Modelconfiguratie `params.fastMode` wordt standaard toegepast, maar een opgeslagen sessie-override voor `fastMode` heeft nog steeds voorrang op de configuratie. Wanneer de opgeloste modus `auto` is, gebruikt de limiet de waarde `params.fastAutoOnSeconds` van het geselecteerde model, met standaard 60 seconden.

### Nieuwe pogingen na live modelwissels

Als een geïsoleerde uitvoering `LiveSessionModelSwitchError` genereert, slaat Cron vóór de nieuwe poging de gewisselde provider en het gewisselde model (en, indien aanwezig, de override van het gewisselde authenticatieprofiel) op voor de actieve uitvoering. De buitenste lus voor nieuwe pogingen is beperkt tot twee wisselpogingen na de eerste poging en wordt daarna afgebroken in plaats van eindeloos door te lopen.

## Uitvoer en weigeringen van uitvoeringen

### Onderdrukking van verouderde bevestigingen

Geïsoleerde Cron-beurten onderdrukken verouderde antwoorden die alleen een bevestiging bevatten. Als het eerste resultaat slechts een tussentijdse statusupdate is en geen uitvoering van een onderliggende subagent verantwoordelijk is voor het uiteindelijke antwoord, vraagt Cron eenmaal opnieuw om het werkelijke resultaat voordat het wordt afgeleverd.

### Onderdrukking van stille tokens

Als een geïsoleerde Cron-uitvoering alleen het stille token (`NO_REPLY` of `no_reply`) retourneert, onderdrukt Cron zowel directe uitgaande aflevering als het terugvalpad voor de samenvatting in de wachtrij, zodat er niets naar de chat wordt teruggestuurd.

### Gestructureerde weigeringen

Geïsoleerde Cron-uitvoeringen gebruiken gestructureerde metagegevens over uitvoeringsweigeringen uit de ingesloten uitvoering (fatale fouten van het uitvoeringshulpmiddel met code `SYSTEM_RUN_DENIED` of `INVALID_REQUEST`) als gezaghebbend weigeringssignaal. Ze herkennen ook `UNAVAILABLE`-wrappers van de Node-host rond een geneste gestructureerde fout met een van die codes.

Cron classificeert proza in de uiteindelijke uitvoer of op goedkeuringsverzoeken lijkende weigeringszinnen niet als weigeringen, tenzij de ingesloten uitvoering ook gestructureerde weigeringsmetagegevens levert. Gewone assistenttekst wordt dus niet als een geblokkeerde opdracht behandeld.

`cron list` en de uitvoeringsgeschiedenis tonen de reden voor de weigering in plaats van een geblokkeerde opdracht als `ok` te rapporteren.

## Bewaartermijn

Bewaartermijngedrag:

- `cron.sessionRetention` (standaard `24h`, of `false` om uit te schakelen) verwijdert voltooide sessies van geïsoleerde uitvoeringen.
- De uitvoeringsgeschiedenis bewaart de nieuwste 2000 terminalrijen per Cron-taak. Verloren rijen behouden het standaardopschoningsvenster van 24 uur voor verloren taken.

## Oudere taken migreren

<Note>
Als je Cron-taken hebt van vóór de huidige afleverings- en opslagindeling, voer je `openclaw doctor --fix` uit. Doctor normaliseert verouderde Cron-velden (`jobId`, `schedule.cron`, afleveringsvelden op het hoogste niveau, waaronder de verouderde `threadId`, en afleveringsaliassen voor payload `provider`) en migreert `notify: true`-taken met Webhook-terugval van de buiten gebruik gestelde onbewerkte waarde `cron.webhook` naar expliciete Webhook-aflevering voordat die configuratiesleutel wordt verwijderd. Taken die al een aankondiging naar een chat sturen, behouden die aflevering en krijgen een Webhook-bestemming voor voltooiing. Zonder een verouderde Webhook wordt de inactieve markering `notify` op het hoogste niveau verwijderd voor taken zonder migratiedoel (de bestaande aflevering blijft ongewijzigd behouden), zodat `doctor --fix` er niet meer herhaaldelijk voor waarschuwt.
</Note>

## Veelvoorkomende bewerkingen

Werk afleveringsinstellingen bij zonder het bericht te wijzigen:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

Schakel aflevering uit voor een geïsoleerde taak:

```bash
openclaw cron edit <job-id> --no-deliver
```

Schakel lichtgewicht bootstrapcontext in voor een geïsoleerde taak:

```bash
openclaw cron edit <job-id> --light-context
```

Kondig aan in een specifiek kanaal:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```

Kondig aan in een Telegram-forumonderwerp:

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "-1001234567890" --thread-id 42
```

Maak een geïsoleerde taak met lichtgewicht bootstrapcontext:

```bash
openclaw cron create "0 7 * * *" \
  "Vat de updates van afgelopen nacht samen." \
  --name "Lichtgewicht ochtendoverzicht" \
  --session isolated \
  --light-context \
  --no-deliver
```

`--light-context` is alleen van toepassing op geïsoleerde taken voor agentbeurten. Voor Cron-uitvoeringen houdt de lichtgewicht modus de bootstrapcontext leeg in plaats van de volledige bootstrapset van de werkruimte in te voegen.

Maak een opdrachttaak met exacte argv, cwd, env, stdin en uitvoerlimieten:

```bash
openclaw cron create "*/30 * * * *" \
  --name "Positie-export" \
  --command-argv '["node","scripts/export-position.mjs"]' \
  --command-cwd "/srv/app" \
  --command-env "NODE_ENV=production" \
  --command-input '{"mode":"summary"}' \
  --timeout-seconds 120 \
  --no-output-timeout-seconds 30 \
  --output-max-bytes 65536 \
  --webhook "https://example.invalid/openclaw/cron"
```

## Veelvoorkomende beheeropdrachten

Handmatige uitvoering en inspectie:

```bash
openclaw cron list
openclaw cron list --agent ops
openclaw cron get <job-id>
openclaw cron show <job-id>
openclaw cron run <job-id>
openclaw cron run <job-id> --due
openclaw cron run <job-id> --wait --wait-timeout 10m
openclaw cron run <job-id> --wait --wait-timeout 10m --poll-interval 2s
openclaw cron runs --id <job-id> --limit 50
openclaw cron runs --id <job-id> --run-id <run-id>
```

`openclaw cron list` toont standaard ingeschakelde taken. Geef `--all` door om uitgeschakelde taken op te nemen, of `--agent <id>` om alleen taken te tonen waarvan de effectieve genormaliseerde agent-id overeenkomt; taken zonder opgeslagen agent-id tellen als de geconfigureerde standaardagent.

`openclaw cron get <job-id>` retourneert rechtstreeks de opgeslagen JSON van de taak. Gebruik `cron show <job-id>` wanneer je de voor mensen leesbare weergave met een voorbeeld van de afleveringsroute wilt.

`cron list --json` en `cron show <job-id> --json` bevatten voor elke taak een veld `status` op het hoogste niveau, berekend op basis van `enabled`, `state.runningAtMs` en `state.lastRunStatus`. Waarden: `disabled`, `running`, `ok`, `error`, `skipped` of `idle`. De JSON-status blijft canoniek en onopgesmukt, zodat externe hulpmiddelen de taakstatus kunnen lezen zonder deze opnieuw af te leiden; voor mensen leesbare uitvoer kan herhaalde statussen `error` voorzien van een aantal mislukkingen.

`cron runs`-vermeldingen bevatten afleveringsdiagnostiek met het beoogde Cron-doel, het opgeloste doel, verzendingen via het berichthulpmiddel, terugvalgebruik en de afleveringsstatus.

Privékladruimte per taak (Heartbeat-controlelijsten en vergelijkbare monitorcontext):

```bash
openclaw cron scratch <job-id>                  # huidige kladinhoud afdrukken
openclaw cron scratch <job-id> --json           # kladinhoud plus revisiemetagegevens
openclaw cron scratch <job-id> --set "text"     # kladinhoud vervangen door exacte tekst
openclaw cron scratch <job-id> --file notes.md  # kladinhoud vervangen vanuit een bestand (- voor stdin)
openclaw cron scratch <job-id> --unset          # kladrij verwijderen
```

De kladruimte wordt opgeslagen in de gedeelde statusdatabase, is beperkt tot 256 KiB en wordt nooit opgenomen in de uitvoer van `cron list`/`cron get`/`cron runs`. Schrijfbewerkingen worden met compare-and-swap beschermd tegen de revisie die bij het starten van de opdracht is gelezen; geef in plaats daarvan `--expected-revision <n>` door om een expliciete revisie vast te zetten. Zie [Heartbeat](/nl/gateway/heartbeat#monitor-scratch-optional) voor informatie over hoe Heartbeat-monitors de kladruimte gebruiken.

Agent- en sessiedoelen wijzigen:

```bash
openclaw cron edit <job-id> --agent ops
openclaw cron edit <job-id> --clear-agent
openclaw cron edit <job-id> --session current
openclaw cron edit <job-id> --session "session:daily-brief"
```

`openclaw cron add` waarschuwt wanneer `--agent` bij taken voor agentbeurten is weggelaten en valt terug op de standaardagent (`main`). Geef bij het maken `--agent <id>` door om een specifieke agent vast te zetten.

Aflevering aanpassen:

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
openclaw cron edit <job-id> --webhook "https://example.invalid/openclaw/cron"
openclaw cron edit <job-id> --best-effort-deliver
openclaw cron edit <job-id> --no-best-effort-deliver
openclaw cron edit <job-id> --no-deliver
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Geplande taken](/nl/automation/cron-jobs)
