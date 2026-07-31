---
read_when:
    - Je wilt een tijdlijn in Dayflow-stijl van je dag in de Control UI
    - Je schakelt de meegeleverde Logbook-plugin in of configureert deze
    - Je wilt stand-upsamenvattingen of een terugblik op je dag op basis van schermactiviteit
summary: Optioneel automatisch werklogboek opgebouwd uit periodieke schermafbeeldingen
title: Logboek-Plugin
x-i18n:
    generated_at: "2026-07-27T06:01:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

De Logbook-plugin zet schermactiviteit om in een automatisch werkjournaal. De plugin
legt periodiek schermafbeeldingen van een gekoppelde Node vast, vat deze samen als
waarnemingen met tijdstempels en bouwt tijdlijnkaarten in de
[Control UI](/nl/web/control-ui). De plugin kan ook dagelijkse stand-upnotities genereren en
vragen over een bijgehouden dag beantwoorden.

Door OpenClaw beheerde status blijft op de Gateway onder `<state-dir>/logbook/`, maar
modelverwerking vindt niet noodzakelijk lokaal plaats. Bemonsterde schermafbeeldingen gaan naar de
geconfigureerde visieroute; waarnemingen en tijdlijntekst gaan naar het standaardmodel
van de agent. Gebruik voor beide fasen lokale modelroutes als scherminhoud en
afgeleide activiteitstekst op de machine moeten blijven.

Logbook is meegeleverd en standaard uitgeschakeld. Als je de plugin inschakelt, stemt de
Gateway in met schermopname, omdat `captureEnabled` standaard `true` is.

## Voordat je begint

Je hebt het volgende nodig:

- Een verbonden Node die `screen.snapshot` of `logbook.snapshot` beschikbaar stelt. De
  macOS-app-Node heeft toestemming voor Screen Recording nodig. Een headless macOS-Node-host
  (`openclaw node host run`) krijgt de door de plugin geleverde opdracht `logbook.snapshot`,
  die gebruikmaakt van het systeemhulpmiddel `screencapture`.
- De meegeleverde Codex-plugin, ingeschakeld en geauthenticeerd. Codex biedt momenteel
  het contract voor gestructureerde afbeeldingsextractie dat Logbook vereist. Meld je aan met
  `openclaw models auth login --provider openai`; zie
  [Codex-harnas](/nl/plugins/codex-harness) voor andere authenticatieroutes.
- Een werkend standaardmodel voor de agent. Logbook gebruikt dit na de visiefase om kaarten,
  stand-upnotities en vragen en antwoorden over een dag samen te stellen.

## Snelstart

Schakel de Codex- en Logbook-plugins in:

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

Configureer een expliciet visiemodel voor deterministisch opstarten:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

Als je `plugins.allow` gebruikt, neem dan zowel `codex` als `logbook` op. Start de
Gateway opnieuw nadat je de pluginconfiguratie hebt gewijzigd, inspecteer vervolgens de registraties
en open het dashboard:

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

De Node-beschrijving moet `screen.snapshot` of `logbook.snapshot` bevatten.
Headless Nodes adverteren `logbook.snapshot` pas nadat de plugin actief is.
Zie [Problemen met Nodes oplossen](/nl/nodes/troubleshooting) als de opdracht ontbreekt.

Het tabblad Logbook verschijnt alleen voor een ingeschakelde plugin en een `operator.write`-
sessie van de Control UI. De statusrij moet zonder fout **Vastleggen** tonen.
Er verschijnt een tijdlijnkaart wanneer het analysevenster sluit, of je kunt
**Nu analyseren** selecteren nadat activiteit is vastgelegd.

## Werking

1. **Vastleggen**: elke `captureIntervalSeconds` (standaard 30s) roept Logbook
   de vastlegopdracht van de geselecteerde Node aan en slaat het een geschaald JPEG-frame op.
   Opeenvolgende identieke frames worden als inactief gemarkeerd en van analyse uitgesloten.
2. **Waarnemen**: zodra een analysevenster (standaard 15 minuten) is verstreken, bemonstert de
   plugin maximaal 16 actieve frames en stuurt deze naar het visiemodel,
   dat activiteitswaarnemingen met tijdstempels retourneert ("VS Code: bezig met het bewerken van
   store.ts en het oplossen van een typefout"). Een onderbreking in het vastleggen van meer dan twee minuten of
   lokale middernacht sluit ook het huidige venster.
3. **Samenstellen**: waarnemingen plus de laatste 45 minuten aan bestaande kaarten worden
   herzien tot tijdlijnkaarten (elk 10-60 minuten) met een titel, samenvatting,
   categorie, hoofdapp en eventuele korte afleidingen.
4. **Opschonen**: frames ouder dan `retentionDays` (standaard 14) worden verwijderd.
   Kaarten, waarnemingen en gecachte stand-ups blijven bewaard.

Daggrenzen en tijdlijnklokken gebruiken de lokale tijdzone van de Gateway, niet de
tijdzone van de browser. Frames en de SQLite-tijdlijndatabase bevinden zich onder
`<state-dir>/logbook/`.

## Model- en gegevensstroom

Logbook gebruikt twee afzonderlijke modelroutes:

| Fase             | Verzonden gegevens                                         | Modelroute                                                        |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| Waarnemen        | Maximaal 16 bemonsterde JPEG-frames plus hun vastlegtijden | `visionModel`, of een compatibel geleend Codex-item `tools.media` |
| Kaarten samenstellen | Waarnemingen met tijdstempels en recente tijdlijnkaarten | Standaardmodel van de agent via de LLM-runtime van de plugin      |
| Stand-up genereren | Kaarten voor de geselecteerde dag en de vorige dag       | Standaardmodel van de agent via de LLM-runtime van de plugin      |
| Vragen over je dag | De vraag, kaarten van de geselecteerde dag en recente waarnemingen | Standaardmodel van de agent via de LLM-runtime van de plugin |

De volledige SQLite-database wordt niet naar een van beide modellen verzonden. Onbewerkte schermafbeeldingen gaan alleen
naar de waarnemingsfase; kaartsamenstelling, stand-up en vragen en antwoorden ontvangen afgeleide
tekst.

## Configuratie

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

Alle Logbook-configuratiesleutels zijn optioneel. Numerieke waarden worden afgerond op gehele getallen
en begrensd tot het ondersteunde bereik.

| Sleutel                   | Standaard | Bereik of waarden        | Gedrag                                                                                       |
| ------------------------- | --------- | ------------------------ | -------------------------------------------------------------------------------------------- |
| `captureEnabled`        | `true` | booleaans                | Permanente hoofdschakelaar voor nieuwe momentopnamen; de tijdlijn blijft beschikbaar wanneer `false` |
| `captureIntervalSeconds`        | `30` | `5`-`600` | Vertraging tussen vastlegpogingen                                                            |
| `analysisIntervalMinutes`        | `15` | `3`-`120` | Beoogd waarnemingsvenster; onderbrekingen en middernacht kunnen het eerder sluiten            |
| `nodeId`        | niet ingesteld | Node-id of weergavenaam | Zet vastleggen vast op één verbonden Node; vergelijking is niet hoofdlettergevoelig          |
| `screenIndex`        | `0` | `0`-`16` | Nulgebaseerde schermindex                                                                    |
| `maxWidth`        | `1440` | `480`-`3840` | Aangevraagde limiet voor vastlegformaat; headless macOS past deze toe op de grootste dimensie |
| `visionModel`        | niet ingesteld | `provider/model`       | Expliciete gestructureerde route; onjuist gevormde verwijzingen pauzeren de analyse, niet-ondersteunde providers laten batches mislukken |
| `retentionDays`        | `14` | `1`-`365` | Verwijdert oude frames; kaarten, waarnemingen en stand-ups blijven behouden                   |

Zonder `nodeId` geeft Logbook de voorkeur aan een verbonden app-Node die
`screen.snapshot` beschikbaar stelt en valt het daarna terug op een headless Node die
`logbook.snapshot` beschikbaar stelt. In een niet-vastgezette configuratie wordt een mislukte Node achter andere
geschikte Nodes geplaatst. De pauzeschakelaar op het dashboard geldt alleen voor de sessie en wordt opnieuw ingesteld wanneer de
Gateway opnieuw start; gebruik `captureEnabled: false` voor een permanente stop.

### Selectie van visiemodel

Logbook bepaalt het waarnemingsmodel in deze volgorde:

1. `plugins.entries.logbook.config.visionModel`
2. het eerste Codex-item met afbeeldingsondersteuning onder `tools.media.models`

Andere mediaproviders worden overgeslagen omdat ze momenteel niet het
contract voor gestructureerde extractie beschikbaar stellen dat Logbook vereist. Het instellen van
`tools.media.image.enabled: false` schakelt geleende mediastandaarden uit, maar een
expliciete Logbook-instelling `visionModel` blijft van toepassing.

## Dashboardtabblad

- **Tijdlijn**: uitvouwbare kaarten per activiteit met categoriekleuren, de hoofdapp,
  afleidingslabels en een sleutelframe van een momentopname.
- **Dag in één oogopslag**: focusverhouding, categorieverdeling, meest gebruikte apps.
- **Dagelijkse stand-up**: zet gisteren plus vandaag om in een kant-en-klare update.
- **Vragen over je dag**: vragen in natuurlijke taal, beantwoord vanuit de bijgehouden
  tijdlijn ("wanneer heb ik de pull request voor de Gateway beoordeeld?").
- **Nu analyseren**: sluit het huidige vastlegvenster onmiddellijk in plaats van
  op het analyse-interval te wachten.

## Gateway-methoden

Logbook registreert deze RPC-methoden van de Gateway:

| Methode               | Parameters               | Bereik           | Resultaat                                                                |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`    | geen                     | `operator.read` | Status van vastleggen, analyse, model, Node, Gateway-dag en Gateway-tijdzone |
| `logbook.days`    | geen                     | `operator.read` | Dagen met aantallen tijdlijnkaarten en tijdsgrenzen van kaarten          |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }`       | `operator.read` | Afgeleide kaarten en dagstatistieken; standaard de huidige dag van de Gateway |
| `logbook.frames`    | `{ startMs, endMs }`       | `operator.write` | Framemetadata binnen het aangevraagde bereik in epoch-milliseconden      |
| `logbook.frame`    | `{ frameId }`       | `operator.write` | Eén onbewerkt JPEG-frame als base64                                      |
| `logbook.standup`    | `{ day?, refresh? }`       | `operator.write` | Gecachte of opnieuw gegenereerde stand-uptekst voor een dag              |
| `logbook.ask`    | `{ day?, question }`       | `operator.write` | Op de tijdlijn gebaseerd antwoord voor een dag                           |
| `logbook.capture.set`    | `{ paused }`       | `operator.write` | Pauzestatus die alleen voor de sessie geldt en bijgewerkte status        |
| `logbook.analyze.now`    | geen                     | `operator.write` | Start een wachtende analyse of retourneert waarom deze niet kon starten  |

De leesmethoden retourneren de operationele status of afgeleide tekst. Onbewerkte schermafbeeldingspixels,
acties die modelkosten veroorzaken en runtimewijzigingen vereisen
`operator.write`. Het tabblad van de Control UI vereist ook `operator.write`, omdat het
deze acties en voorbeelden van onbewerkte frames beschikbaar stelt; een alleen-lezenclient kan de
methoden voor afgeleide tekst nog steeds rechtstreeks aanroepen.

## Privacyopmerkingen

- Momentopnamen kunnen alles bevatten wat op het scherm staat, inclusief geheimen. Frames verlaten
  de machine nooit, behalve als bemonsterde invoer voor het geconfigureerde
  observatiemodel.
- Observaties, recente kaarten en vragen kunnen de machine verlaten via het
  standaard agentmodel tijdens het samenstellen van kaarten, het genereren van stand-ups of vraag-en-antwoordinteracties. Pas
  het gegevensverwerkingsbeleid van de provider toe op beide modelroutes.
- Gebruik lokale routes voor zowel het gestructureerde observatiemodel als het standaard
  agentmodel wanneer je een volledig lokale pijplijn nodig hebt.
- Frames, de tijdlijndatabase en tijdelijke opnamen worden opgeslagen met
  bestandsmachtigingen die alleen de eigenaar toegang geven.
- Het toevoegen van `screen.snapshot` aan `gateway.nodes.commands.deny` is de
  noodstop voor schermopnamen: hiermee worden zowel opnamen door app-nodes als Logbooks eigen
  opdracht `logbook.snapshot` geblokkeerd.
- Het instellen van `tools.media.image.enabled: false` voorkomt ook dat Logbook
  de media-afbeeldingsmodellen voor analyse gebruikt; dan wordt alleen een expliciete `visionModel` in de
  Pluginconfiguratie gebruikt.

## Problemen oplossen

### Het tabblad Logbook ontbreekt

Controleer alle drie de voorwaarden:

1. `openclaw plugins list --enabled` bevat `logbook`.
2. De Gateway is opnieuw gestart na de wijziging van de Plugin of toelatingslijst.
3. De Control UI-verbinding heeft `operator.write`; alleen-lezen-sessies ontvangen
   de interactieve tabbladbeschrijving niet.

Als `plugins.allow` is ingesteld, moet deze voor de
aanbevolen configuratie zowel `logbook` als `codex` bevatten.

### Er wordt een fout gemeld bij de opname

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- Controleer of de node `screen.snapshot` of `logbook.snapshot` beschikbaar stelt.
- Verleen op de Mac die de opname maakt toestemming voor schermopname.
- Als `nodeId` is geconfigureerd, controleer dan of deze overeenkomt met de node-id of weergavenaam.
- Controleer of `gateway.nodes.commands.deny` niet
  `screen.snapshot` bevat.

Na drie opeenvolgende fouten wacht Logbook tien opname-intervallen en
probeert het daarna opnieuw. Een niet-vastgezette configuratie kan overschakelen naar een andere geschikte node.

### Opnamen slagen, maar er verschijnen geen kaarten

- De status **Model ontbreekt** betekent dat er geen compatibele route voor gestructureerde visuele analyse is
  gevonden. Schakel de Codex-Plugin in en verifieer deze, of stel een geldige expliciete
  `visionModel` in. Opgenomen frames blijven in behandeling zolang het model ontbreekt en
  kunnen worden geanalyseerd nadat de configuratie is hersteld.
- Wacht op `analysisIntervalMinutes` of selecteer **Nu analyseren** nadat activiteit
  is opgenomen.
- Opeenvolgende identieke frames gelden als bewijs van inactiviteit en worden niet opgenomen in
  analysebatches. Wijzig de zichtbare scherminhoud voordat je test.
- Als de nieuwste batch een fout toont, los je het probleem met het model of de authenticatie op en selecteer je
  **Nu analyseren**. Mislukte batches worden alleen na die expliciete actie opnieuw geprobeerd om
  herhaaldelijke modelkosten te voorkomen.

## Gerelateerd

- [Plugins beheren](/nl/plugins/manage-plugins)
- [Codex-harnas](/nl/plugins/codex-harness)
- [Mediabegrip](/nl/nodes/media-understanding)
- [Nodes](/nl/nodes)
- [Problemen met nodes oplossen](/nl/nodes/troubleshooting)
- [Control UI](/nl/web/control-ui)
