---
read_when:
    - Heartbeat-frequentie of berichtgeving aanpassen
    - Kiezen tussen Heartbeat en Cron voor geplande taken
sidebarTitle: Heartbeat
summary: Heartbeat-pollingberichten en meldingsregels
title: Heartbeat
x-i18n:
    generated_at: "2026-07-27T05:46:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**Heartbeat versus cron?** Zie [Automatisering](/nl/automation) voor richtlijnen over wanneer je welke gebruikt.
</Note>

Heartbeat voert **periodieke agentbeurten** uit in de hoofdsessie, zodat het model alles wat aandacht nodig heeft onder de aandacht kan brengen zonder je te overspoelen.

Heartbeat is een geplande beurt in de hoofdsessie; er worden **geen** records voor [achtergrondtaken](/nl/automation/tasks) aangemaakt. Taakrecords zijn bedoeld voor losgekoppeld werk (ACP-runs, subagents, geïsoleerde cron-taken).

Onder de motorkap wordt het Heartbeat-ritme beheerd door de cron-planner: de Gateway onderhoudt één cron-taak in systeemeigendom per agent waarvoor Heartbeat is ingeschakeld (zichtbaar in `openclaw cron list --all` als `Heartbeat (agent-id)`). De Heartbeat-configuratie blijft de invoer voor de gewenste toestand, terwijl het opgeslagen monitorschema de daadwerkelijke tik en de daaropvolgende afkoelperiode van de runner beheert. De Gateway schrijft configuratiewijzigingen door bij het opstarten en wanneer de configuratie opnieuw wordt geladen; `openclaw doctor --fix` kan ontbrekende of verouderde monitorrijen aanmaken vóór de volgende start van de Gateway. Bewerk `agents.*.heartbeat`, niet de cron-taak.

Geplande Heartbeats vereisen cron. Wanneer `cron.enabled` `false` of `OPENCLAW_SKIP_CRON=1` is, registreert de Gateway bij het opstarten een waarschuwing en worden geplande Heartbeats niet uitgevoerd; handmatige en gebeurtenisgestuurde Heartbeat-activeringen blijven beschikbaar. Er is geen afzonderlijke reservetimer voor Heartbeat.

Probleemoplossing: [Geplande taken](/nl/automation/cron-jobs#troubleshooting)

## Snel aan de slag (beginner)

<Steps>
  <Step title="Kies een ritme">
    Laat Heartbeats ingeschakeld (standaard is `30m`, of `1h` wanneer Anthropic OAuth-/tokenauthenticatie is geconfigureerd, inclusief hergebruik van de Claude CLI) of stel je eigen ritme in.
  </Step>
  <Step title="Voeg monitorkladruimte toe (optioneel)">
    Sla met `openclaw cron scratch <jobId> --set "..."` een korte controlelijst op in de kladruimte van de Heartbeat-monitor.
  </Step>
  <Step title="Bepaal waar Heartbeat-berichten naartoe moeten">
    `target: "none"` is de standaard; stel `target: "last"` in om berichten naar het laatste contact te routeren.
  </Step>
  <Step title="Optionele afstemming">
    - Gebruik een lichtgewicht bootstrapcontext als Heartbeat-runs alleen de monitorkladruimte nodig hebben.
    - Schakel geïsoleerde sessies in om te voorkomen dat bij elke Heartbeat de volledige gespreksgeschiedenis wordt verzonden.
    - Beperk Heartbeats tot actieve uren (lokale tijd).

  </Step>
</Steps>

Voorbeeldconfiguratie:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // expliciete bezorging bij het laatste contact (standaard is "none")
        directPolicy: "allow", // standaard: directe/DM-doelen toestaan; stel "block" in om ze te onderdrukken
        lightContext: true, // optioneel: bootstrapbestanden van de werkruimte overslaan voor Heartbeat-runs
        isolatedSession: true, // optioneel: elke run een nieuwe sessie (geen gespreksgeschiedenis)
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## Standaardwaarden

- Interval: `30m`. Door de standaardwaarden van de Anthropic-provider toe te passen, wordt dit verhoogd naar `1h` wanneer de vastgestelde authenticatiemodus OAuth/token is (inclusief hergebruik van de Claude CLI), maar alleen zolang `heartbeat.every` niet is ingesteld. Stel `agents.defaults.heartbeat.every` of `agents.entries.*.heartbeat.every` per agent in; gebruik `0m` om dit uit te schakelen.
- Prompttekst (configureerbaar via `agents.defaults.heartbeat.prompt`): `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Time-out: Heartbeat-beurten zonder ingestelde waarde gebruiken `agents.defaults.timeoutSeconds` als die is ingesteld. Anders gebruiken ze het Heartbeat-ritme, met een maximum van 600 seconden. Stel `agents.defaults.heartbeat.timeoutSeconds` of `agents.entries.*.heartbeat.timeoutSeconds` per agent in voor langer Heartbeat-werk.
- De Heartbeat-prompt wordt **letterlijk** als gebruikersbericht verzonden. De systeemprompt bevat een sectie ‘Heartbeats’ wanneer Heartbeats voor de standaardagent zijn ingeschakeld, en de run wordt intern gemarkeerd.
- Wanneer Heartbeats met `0m` worden uitgeschakeld, blijft de cron-taak van de monitor bestaan maar wordt deze uitgeschakeld. De kladruimte blijft behouden voor wanneer je het ritme opnieuw inschakelt.
- Wanneer cron zelf is uitgeschakeld, worden geplande Heartbeats niet uitgevoerd, zelfs als het Heartbeat-ritme ingeschakeld blijft.
- Actieve uren (`heartbeat.activeHours`) worden gecontroleerd in de geconfigureerde tijdzone. Buiten het tijdvenster worden Heartbeats overgeslagen tot de volgende tik binnen het venster.
- Heartbeats worden automatisch uitgesteld zolang cron-werk actief is of in de wachtrij staat, of zolang de sessiesleutelgebonden subagent- of geneste opdrachtbanen van die agent bezet zijn. Parallelle agents pauzeren elkaar niet.

## Waarvoor de Heartbeat-prompt dient

De standaardprompt is bewust ruim geformuleerd:

- **Achtergrondtaken**: ‘Houd rekening met openstaande taken’ spoort de agent aan om vervolgacties te bekijken (inbox, agenda, herinneringen, werk in de wachtrij) en alles wat dringend is onder de aandacht te brengen.
- **Contact met de gebruiker**: ‘Vraag overdag soms hoe het met je gebruiker gaat’ spoort aan tot af en toe een kort bericht als ‘heb je iets nodig?’, maar voorkomt berichtenoverlast 's nachts door je geconfigureerde lokale tijdzone te gebruiken (zie [Tijdzone](/nl/concepts/timezone)).

Heartbeat kan reageren op voltooide [achtergrondtaken](/nl/automation/tasks), maar een Heartbeat-run zelf maakt geen taakrecord aan.

Als je wilt dat een Heartbeat iets heel specifieks doet (bijvoorbeeld ‘controleer Gmail PubSub-statistieken’ of ‘controleer de status van de Gateway’), stel je `agents.defaults.heartbeat.prompt` (of `agents.entries.*.heartbeat.prompt`) in op een aangepaste tekst (die letterlijk wordt verzonden).

## Responscontract

- Als niets aandacht nodig heeft, antwoord je met **`HEARTBEAT_OK`**.
- Heartbeat-runs kunnen in plaats daarvan `heartbeat_respond` aanroepen met `notify: false` voor geen zichtbare update, of `notify: true` plus `notificationText` voor een waarschuwing. Wanneer aanwezig, heeft het gestructureerde toolantwoord voorrang op de tekstuele terugvaloptie.
- Een betekenisvol resultaat van `heartbeat_respond` met `notify: false` blijft stil, maar wordt onthouden als begrensde interne context voor de volgende gebruikersbeurt in die sessie. Bevestigingen met `no_change` en zichtbare meldingen worden niet op deze manier opgeslagen.
- Tijdens Heartbeat-runs behandelt OpenClaw `HEARTBEAT_OK` als een bevestiging wanneer het aan het **begin of einde** van het antwoord staat. Het token wordt verwijderd en het antwoord wordt verworpen als de resterende inhoud maximaal 300 tekens bevat.
- Als `HEARTBEAT_OK` in het **midden** van een antwoord staat, wordt het niet speciaal behandeld.
- Neem bij waarschuwingen `HEARTBEAT_OK` **niet** op; retourneer alleen de waarschuwingstekst.

Buiten Heartbeats wordt een losstaande `HEARTBEAT_OK` aan het begin/einde van een bericht verwijderd en geregistreerd; een bericht dat alleen uit `HEARTBEAT_OK` bestaat, wordt verworpen.

## Configuratie

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // standaard: 30m (0m schakelt uit)
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // standaard: false; true slaat bootstrapbestanden van de werkruimte over voor Heartbeat-runs
        isolatedSession: false, // standaard: false; true voert elke Heartbeat uit in een nieuwe sessie (geen gespreksgeschiedenis)
        target: "last", // standaard: none | opties: last | none | <channel id> (core of Plugin, bijvoorbeeld "imessage")
        to: "+15551234567", // optionele kanaalspecifieke overschrijving
        accountId: "ops-bot", // optionele kanaal-id voor meerdere accounts
        prompt: "Volg de kladcontext van de Heartbeat-monitor wanneer die beschikbaar is. Terugkerende taken zijn cron-taken; maak of wijzig hun schema's met cron-tools of de openclaw cron CLI, niet met de Heartbeat-kladruimte. Leid geen oude taken af uit eerdere chats en herhaal ze niet. Als niets aandacht nodig heeft, antwoord dan HEARTBEAT_OK.",
      },
    },
  },
}
```

### Bereik en voorrang

- `agents.defaults.heartbeat` stelt het algemene Heartbeat-gedrag in.
- `agents.entries.*.heartbeat` wordt daar bovenop samengevoegd; als een agent een `heartbeat`-blok heeft, voeren **alleen die agents** Heartbeats uit.
- `channels.defaults.heartbeatVisibility` stelt de standaardinstellingen voor zichtbaarheid voor alle kanalen in.
- `channels.<channel>.heartbeatVisibility` overschrijft de standaardinstellingen van het kanaal.
- `channels.<channel>.accounts.<id>.heartbeatVisibility` (kanalen met meerdere accounts) overschrijft de instellingen per kanaal.

### Heartbeats per agent

Als een item in `agents.entries.*` een `heartbeat`-blok bevat, voeren **alleen die agents** Heartbeats uit. Het blok per agent wordt boven op `agents.defaults.heartbeat` samengevoegd (zodat je gedeelde standaardwaarden één keer kunt instellen en ze per agent kunt overschrijven).

Voorbeeld: twee agents, waarbij alleen de tweede agent Heartbeats uitvoert.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // expliciete bezorging bij het laatste contact (standaard is "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Volg de kladcontext van de Heartbeat-monitor wanneer die beschikbaar is. Terugkerende taken zijn cron-taken; maak of wijzig hun schema's met cron-tools of de openclaw cron CLI, niet met de Heartbeat-kladruimte. Leid geen oude taken af uit eerdere chats en herhaal ze niet. Als niets aandacht nodig heeft, antwoord dan HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Voorbeeld van actieve uren

Beperk Heartbeats tot kantooruren in een specifieke tijdzone:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // expliciete bezorging bij het laatste contact (standaard is "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // optioneel; gebruikt je userTimezone indien ingesteld, anders de tijdzone van de host
        },
      },
    },
  },
}
```

Buiten dit venster (vóór 9 uur 's ochtends of na 10 uur 's avonds Eastern Time) worden Heartbeats overgeslagen. De volgende geplande tik binnen het venster wordt normaal uitgevoerd.

### Configuratie voor 24/7

Als je wilt dat Heartbeats de hele dag worden uitgevoerd, gebruik je een van deze patronen:

- Laat `activeHours` volledig weg (geen beperking door een tijdvenster; dit is het standaardgedrag).
- Stel een venster voor de hele dag in: `activeHours: { start: "00:00", end: "24:00" }`.

<Warning>
Stel niet dezelfde tijd in voor `start` en `end` (bijvoorbeeld `08:00` tot `08:00`). Dit wordt behandeld als een venster met een breedte van nul, waardoor Heartbeats altijd worden overgeslagen.
</Warning>

### Voorbeeld met meerdere accounts

Gebruik `accountId` om een specifiek account te kiezen op kanalen met meerdere accounts, zoals Telegram:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // optioneel: routeren naar een specifiek onderwerp/thread
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Opmerkingen bij velden

<ParamField path="every" type="string">
  Heartbeat-interval (tekenreeks voor de duur; standaardeenheid = minuten).
</ParamField>
<ParamField path="model" type="string">
  Optionele modeloverschrijving voor Heartbeat-runs (`provider/model`).
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  Indien true gebruiken Heartbeat-runs een lichtgewicht bootstrapcontext en slaan ze bootstrapbestanden van de werkruimte over. De monitorkladruimte wordt hoe dan ook door de Heartbeat-runner geïnjecteerd.
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  Indien true wordt elke Heartbeat uitgevoerd in een nieuwe sessie zonder eerdere gespreksgeschiedenis. Gebruikt hetzelfde isolatiepatroon als cron `sessionTarget: "isolated"`. Verlaagt de tokenkosten per Heartbeat drastisch. Combineer dit met `lightContext: true` voor maximale besparing. De bezorgingsroutering gebruikt nog steeds de context van de hoofdsessie.
</ParamField>
<ParamField path="session" type="string">
  Optionele sessiesleutel voor Heartbeat-runs.

- `main` (standaard): hoofdsessie van de agent.
- Expliciete sessiesleutel (kopieer deze uit `openclaw sessions --json` of de [sessie-CLI](/nl/cli/sessions)).
- Indelingen van sessiesleutels: zie [Sessies](/nl/concepts/session) en [Groepen](/nl/channels/groups).

</ParamField>
<ParamField path="target" type="string">
- `last`: afleveren bij het laatst gebruikte externe kanaal.
- expliciet kanaal: elk geconfigureerd kanaal of elke geconfigureerde plugin-id, bijvoorbeeld `discord`, `matrix`, `telegram` of `whatsapp`.
- `none` (standaard): voer de heartbeat uit, maar lever deze **niet extern af**.

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  Bepaalt het aflevergedrag voor directe berichten/DM's. `allow`: sta aflevering van heartbeats via directe berichten/DM's toe. `block`: onderdruk aflevering via directe berichten/DM's (`reason=dm-blocked`).

</ParamField>
<ParamField path="to" type="string">
  Optionele overschrijving van de ontvanger (kanaalspecifieke id, bijvoorbeeld E.164 voor WhatsApp of een Telegram-chat-id). Gebruik voor Telegram-onderwerpen/threads `<chatId>:topic:<messageThreadId>`.

</ParamField>
<ParamField path="accountId" type="string">
  Optionele account-id voor kanalen met meerdere accounts. Wanneer `target: "last"`, geldt de account-id voor het bepaalde laatste kanaal als dat accounts ondersteunt; anders wordt deze genegeerd. Als de account-id niet overeenkomt met een geconfigureerd account voor het bepaalde kanaal, wordt de aflevering overgeslagen.

</ParamField>
<ParamField path="prompt" type="string">
  Overschrijft de standaardinhoud van de prompt (wordt niet samengevoegd).

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Maximaal aantal seconden dat een heartbeat-beurt van de agent mag duren voordat deze wordt afgebroken. Laat dit oningesteld om `agents.defaults.timeoutSeconds` te gebruiken wanneer die is ingesteld; anders wordt het heartbeat-interval gebruikt, begrensd op 600 seconden.

</ParamField>
<ParamField path="activeHours" type="object">
  Beperkt heartbeat-uitvoeringen tot een tijdvenster. Object met `start` (UU:MM, inclusief; gebruik `00:00` voor het begin van de dag), `end` (UU:MM, exclusief; `24:00` toegestaan voor het einde van de dag) en optioneel `timezone`.

- Weggelaten of `"user"`: gebruikt jouw `agents.defaults.userTimezone` als die is ingesteld en valt anders terug op de tijdzone van het hostsysteem.
- `"local"`: gebruikt altijd de tijdzone van het hostsysteem.
- Elke IANA-identificatie (bijvoorbeeld `America/New_York`): wordt rechtstreeks gebruikt; als deze ongeldig is, wordt teruggevallen op het hierboven beschreven gedrag van `"user"`.
- `start` en `end` mogen voor een actief venster niet gelijk zijn; gelijke waarden worden behandeld als een venster zonder breedte (altijd buiten het venster).
- Buiten het actieve venster worden heartbeats overgeslagen tot de volgende tick binnen het venster.

</ParamField>

## Aflevergedrag

<AccordionGroup>
  <Accordion title="Sessie- en doelroutering">
    - Heartbeats worden standaard uitgevoerd in de hoofdsessie van de agent (`agent:<id>:<mainKey>`), of in `global` wanneer `session.scope = "global"`. Stel `session` in om dit te overschrijven met een specifieke kanaalsessie (Discord/WhatsApp/enzovoort).
    - `session` beïnvloedt alleen de uitvoeringscontext; de aflevering wordt bepaald door `target` en `to`.
    - Stel `target` + `to` in om bij een specifiek kanaal of een specifieke ontvanger af te leveren. Met `target: "last"` gebruikt de aflevering het laatste externe kanaal voor die sessie.
    - Heartbeat-afleveringen staan standaard directe doelen/DM-doelen toe. Stel `directPolicy: "block"` in om verzending naar directe doelen te onderdrukken terwijl de heartbeat-beurt nog steeds wordt uitgevoerd.
    - Als de hoofdwachtrij, de sessiebaan van het doel, de Cron-baan of een actieve Cron-taak bezet is, wordt de heartbeat overgeslagen en later opnieuw geprobeerd.
    - Als `target` geen externe bestemming oplevert, wordt de uitvoering nog steeds uitgevoerd, maar wordt er geen uitgaand bericht verzonden.

  </Accordion>
  <Accordion title="Zichtbaarheid en overslaggedrag">
    - Als `showOk`, `showAlerts` en `useIndicator` allemaal zijn uitgeschakeld, wordt de uitvoering vooraf overgeslagen als `reason=alerts-disabled`.
    - Als alleen de aflevering van waarschuwingen is uitgeschakeld, kan OpenClaw de heartbeat nog steeds uitvoeren, tijdstempels van vervallen taken bijwerken, het tijdstempel voor de inactiviteit van de sessie herstellen en de uitgaande waarschuwingsinhoud onderdrukken.
    - Als het bepaalde heartbeat-doel typen ondersteunt, toont OpenClaw een typindicator terwijl de heartbeat wordt uitgevoerd. Hiervoor wordt hetzelfde doel gebruikt waarnaar de heartbeat chatuitvoer zou verzenden; dit wordt uitgeschakeld door `typingMode: "never"`.

  </Accordion>
  <Accordion title="Sessielevenscyclus en audit">
    - Antwoorden die uitsluitend uit een heartbeat bestaan, houden de sessie **niet** actief. Heartbeat-metagegevens kunnen de sessierij bijwerken, maar voor verval wegens inactiviteit wordt `lastInteractionAt` van het laatste echte gebruikers-/kanaalbericht gebruikt en voor dagelijks verval wordt `sessionStartedAt` gebruikt.
    - De geschiedenis van de Control UI en WebChat verbergt heartbeat-prompts en bevestigingen die alleen uit OK bestaan. Het onderliggende sessietranscript kan die beurten nog steeds bevatten voor audits/herhalingen.
    - Losgekoppelde [achtergrondtaken](/nl/automation/tasks) kunnen een systeemgebeurtenis in de wachtrij plaatsen en de heartbeat activeren wanneer de hoofdsessie snel ergens van op de hoogte moet worden gebracht. Die activering maakt van de heartbeat-uitvoering geen achtergrondtaak.

  </Accordion>
</AccordionGroup>

## Zichtbaarheidsinstellingen

Standaard worden `HEARTBEAT_OK`-bevestigingen onderdrukt terwijl waarschuwingsinhoud wordt afgeleverd. Je kunt dit per kanaal of per account aanpassen:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OK verbergen (standaard)
      showAlerts: true # Waarschuwingsberichten tonen (standaard)
      useIndicator: true # Indicatorgebeurtenissen uitzenden (standaard)
  telegram:
    heartbeat:
      showOk: true # OK-bevestigingen tonen op Telegram
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Aflevering van waarschuwingen voor dit account onderdrukken
```

Prioriteit: per account → per kanaal → standaardinstellingen voor kanalen → ingebouwde standaardinstellingen.

### Wat elke vlag doet

- `showOk`: verzendt een `HEARTBEAT_OK`-bevestiging wanneer het model een antwoord retourneert dat alleen uit OK bestaat.
- `showAlerts`: verzendt de waarschuwingsinhoud wanneer het model een antwoord retourneert dat niet alleen uit OK bestaat.
- `useIndicator`: zendt indicatorgebeurtenissen uit voor UI-statusweergaven.

Als **alle drie** false zijn, slaat OpenClaw de heartbeat-uitvoering volledig over (geen modelaanroep).

### Voorbeelden per kanaal en per account

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # alle Slack-accounts
    accounts:
      ops:
        heartbeat:
          showAlerts: false # waarschuwingen alleen voor het ops-account onderdrukken
  telegram:
    heartbeat:
      showOk: true
```

### Veelvoorkomende patronen

| Doel                                             | Configuratie                                                                              |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| Standaardgedrag (stille OK's, waarschuwingen aan) | _(geen configuratie nodig)_                                                               |
| Volledig stil (geen berichten, geen indicator)    | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Alleen indicator (geen berichten)                 | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| Alleen OK's in één kanaal                         | `channels.telegram.heartbeat: { showOk: true }`                                          |

## Monitorkladblok (optioneel)

Elke Cron-taak voor heartbeat-bewaking heeft een eigen privékladblokdocument dat in de gedeelde statusdatabase wordt opgeslagen. Beschouw het als jouw 'heartbeat-checklist': klein, stabiel en veilig om elke 30 minuten te bekijken. Als het kladblok bestaat, wordt de inhoud ervan aan de heartbeat-prompt toegevoegd.

Beheer het met de Cron-CLI (de taak-id is afkomstig uit `openclaw cron list --all`):

```bash
openclaw cron scratch <jobId>                 # het huidige kladblok weergeven
openclaw cron scratch <jobId> --set "..."     # vervangen door exact deze tekst
openclaw cron scratch <jobId> --file notes.md # vervangen vanuit een bestand (- voor stdin)
openclaw cron scratch <jobId> --unset         # verwijderen
```

Schrijfbewerkingen worden beschermd met compare-and-swap: geef `--expected-revision <n>` door om te mislukken in plaats van een gelijktijdige bewerking te overschrijven. Het kladblok is begrensd op 256 KiB en verschijnt nooit in de uitvoer van `cron list`/`cron runs`.

De agent kan ook zijn eigen kladblok bijwerken: tijdens een heartbeat-beurt accepteert `heartbeat_respond` een optionele tekenreeks `scratch` die het kladblok van de monitor volledig vervangt voor toekomstige heartbeats.

<Note>
**Migreren vanuit HEARTBEAT.md of een interval dat alleen in de configuratie staat?** Voer `openclaw doctor --fix` uit. Doctor maakt eerst de systeemeigen monitorrijen vanuit `agents.*.heartbeat` of werkt deze bij, importeert vervolgens de `HEARTBEAT.md` uit de werkruimte van elke agent in het kladblok van de monitor, zet geldige verouderde `tasks:`-vermeldingen om in Cron-taken, archiveert het origineel onder de statusmap (`backups/heartbeat-migration/`) en verwijdert het bestand. Heartbeat-instructies tijdens runtime zijn uitsluitend afkomstig uit het databasekladblok; de runtime leest `HEARTBEAT.md` nooit.
</Note>

Als het kladblok bestaat maar feitelijk leeg is (alleen lege regels, Markdown-/HTML-opmerkingen, Markdown-koppen zoals `# Heading`, fence-markeringen of lege checklistsjablonen), slaat OpenClaw de heartbeat-uitvoering over om API-aanroepen te besparen. Die overslag wordt gerapporteerd als `reason=empty-heartbeat-file`. Als er geen kladblok bestaat, wordt de heartbeat nog steeds uitgevoerd en bepaalt het model wat er moet gebeuren.

Houd het klein (een korte checklist of herinneringen) om te voorkomen dat de prompt onnodig groot wordt.

Voorbeeldkladblok:

```md
# Heartbeat-checklist

- Snelle controle: staat er iets dringends in de inboxen?
- Als het overdag is, neem dan kort contact op als er verder niets openstaat.
- Als een taak geblokkeerd is, noteer dan _wat er ontbreekt_ en vraag het de volgende keer aan Peter.
```

### Terugkerende controles plannen met Cron

Het heartbeat-kladblok is promptcontext, geen planner. Maak elke terugkerende controle als een [Cron-taak](/nl/automation/cron-jobs), zodat deze een eigen interval, in-/uitgeschakelde status en uitvoeringsgeschiedenis heeft. Cron-taken kunnen nog steeds op de hoofdsessie worden gericht wanneer de controle de normale gesprekscontext moet gebruiken.

Oudere kladblokken kunnen een gestructureerd `tasks:`-blok bevatten. Voer `openclaw doctor --fix` eenmaal uit na de upgrade: Doctor zet elke geldige vermelding om in een onafhankelijk geplande Cron-taak, behoudt het interval en het eerdere tijdstip van de laatste uitvoering en verwijdert het buiten gebruik gestelde blok terwijl de omringende tekst in het kladblok behouden blijft. Heartbeat-beurten tijdens runtime interpreteren `tasks:`-tekst niet als planningen.

Door Doctor gemaakte heartbeat-taken behouden de actieve uren en de beveiligingen voor afkoeling, overbelasting en bezetting van de heartbeat. Taken die tegelijk moeten worden uitgevoerd, kunnen worden samengevoegd tot één heartbeat-beurt. Een uitvoering buiten de actieve uren wordt overgeslagen en bij de volgende geplande Cron-uitvoering opnieuw geprobeerd.

### Kan de agent zijn kladblok bijwerken?

Ja. Tijdens een heartbeat-beurt kan de agent een `scratch`-waarde doorgeven aan `heartbeat_respond` om de monitortekst voor toekomstige heartbeats volledig te vervangen. Je kunt de agent ook in een normaal gesprek vragen om `openclaw cron scratch <jobId> --set ...` uit te voeren, of het kladblok zelf met dezelfde opdracht bewerken. Beheer terugkerende planningen met Cron in plaats van plannersyntaxis in het kladblok te schrijven.

<Warning>
Plaats geen geheimen (API-sleutels, telefoonnummers, privétokens) in het monitorkladblok; het wordt onderdeel van de promptcontext.
</Warning>

## Handmatige activering (op aanvraag)

Gebruik `openclaw system event` om een systeemgebeurtenis in de wachtrij te plaatsen en optioneel onmiddellijk een heartbeat te activeren:

```bash
openclaw system event --text "Controleer op dringende vervolgacties" --mode now
```

| Vlag                         | Beschrijving                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | Tekst van de systeemgebeurtenis (verplicht).                                                                    |
| `--mode <mode>`              | `now` voert onmiddellijk een heartbeat uit; `next-heartbeat` (standaard) wacht op de volgende geplande tick. |
| `--session-key <sessionKey>` | Richt de gebeurtenis op een specifieke sessie; standaard wordt de hoofdsessie van de agent gebruikt.                   |
| `--json`                     | Voer JSON uit.                                                                                     |

Als geen `--session-key` is opgegeven en voor meerdere agents `heartbeat` is geconfigureerd, voert `--mode now` de heartbeats van elk van die agents onmiddellijk uit.

Gerelateerde heartbeat-besturingselementen in dezelfde CLI-groep:

```bash
openclaw system heartbeat last     # toon de laatste heartbeat-gebeurtenis
openclaw system heartbeat enable   # schakel heartbeats in
openclaw system heartbeat disable  # schakel heartbeats uit
```

## Kostenbewustzijn

Heartbeats voeren volledige agentbeurten uit. Kortere intervallen verbruiken meer tokens. Om de kosten te verlagen:

- Gebruik `isolatedSession: true` om te voorkomen dat de volledige gespreksgeschiedenis wordt verzonden (van ~100K tokens naar ~2-5K per uitvoering).
- Gebruik `lightContext: true` om bootstrapbestanden van de werkruimte over te slaan bij heartbeat-uitvoeringen.
- Stel een goedkoper `model` in (bijv. `ollama/llama3.2:1b`).
- Houd het kladgebied van de monitor klein.
- Gebruik `target: "none"` als je alleen interne statusupdates wilt.

## Contextoverloop na een heartbeat

Heartbeats behouden na voltooiing van de uitvoering het bestaande runtimemodel van de gedeelde sessie. Daardoor kan een heartbeat die een sessie heeft overgeschakeld naar een kleiner lokaal model (bijvoorbeeld een Ollama-model met een venster van 32k) dat model actief laten voor de volgende beurt in de hoofdsessie. Als die volgende beurt vervolgens een contextoverloop meldt en het laatst gebruikte runtimemodel van de sessie overeenkomt met de geconfigureerde `heartbeat.model`, vermeldt het herstelbericht van OpenClaw dat het doorsijpelen van het heartbeat-model waarschijnlijk de oorzaak is en stelt het een oplossing voor.

Om dit te voorkomen: gebruik `isolatedSession: true` om heartbeats in een nieuwe sessie uit te voeren (eventueel gecombineerd met `lightContext: true` voor de kleinste prompt), of kies een heartbeat-model met een contextvenster dat groot genoeg is voor de gedeelde sessie.

## Gerelateerd

- [Automatisering](/nl/automation) - alle automatiseringsmechanismen in één oogopslag
- [Achtergrondtaken](/nl/automation/tasks) - hoe losgekoppeld werk wordt bijgehouden
- [Tijdzone](/nl/concepts/timezone) - hoe de tijdzone de heartbeat-planning beïnvloedt
- [Problemen oplossen](/nl/automation/cron-jobs#troubleshooting) - automatiseringsproblemen opsporen
