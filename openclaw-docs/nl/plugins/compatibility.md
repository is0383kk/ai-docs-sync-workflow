---
read_when:
    - Je onderhoudt een OpenClaw-plugin
    - Je ziet een compatibiliteitswaarschuwing voor een plugin
    - Je plant een migratie van een Plugin-SDK of manifest
summary: Plugincompatibiliteitscontracten, afschaffingsmetadata en migratieverwachtingen
title: Plugincompatibiliteit
x-i18n:
    generated_at: "2026-07-27T06:00:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw houdt oudere Plugin-contracten via benoemde compatibiliteitsadapters
aangesloten voordat ze worden verwijderd. Dit beschermt bestaande gebundelde en externe
plugins terwijl de contracten voor de SDK, het manifest, de installatie, de configuratie en de agentruntime
zich ontwikkelen.

## Compatibiliteitsregister

Plugin-compatibiliteitscontracten worden bijgehouden in het kernregister op
`src/plugins/compat/registry.ts`. Elke record bevat:

- een stabiele compatibiliteitscode
- status: `active`, `deprecated`, `removal-pending` of `removed`
- eigenaar: `sdk`, `config`, `setup`, `channel`, `provider`, `plugin-execution`,
  `agent-runtime` of `core`
- introductie- en afschrijvingsdatums indien van toepassing
- een exacte verwijderingsdatum zodra de verantwoordelijke maintainer deze goedkeurt; als
  `removeAfter` ontbreekt, komt een afgeschreven oppervlak niet in aanmerking voor verwijdering
- richtlijnen voor vervanging
- documentatie, diagnostiek en tests die het oude en nieuwe gedrag dekken

Het register is de bron voor de planning door maintainers en toekomstige controles door de
Plugin-inspector. Als gedrag voor plugins verandert, voeg dan de
compatibiliteitsrecord toe of werk deze bij in dezelfde wijziging waarin de adapter wordt toegevoegd.

Compatibiliteit voor reparaties en migraties door Doctor wordt afzonderlijk bijgehouden op
`src/commands/doctor/shared/deprecation-compat.ts`. Die records omvatten oude
configuratievormen, indelingen van installatiegrootboeken en reparatieshims die mogelijk
beschikbaar moeten blijven nadat het runtimecompatibiliteitspad is verwijderd.

Releasecontroles moeten beide registers controleren. Verwijder een Doctor-
migratie niet alleen omdat de overeenkomende runtime- of configuratiecompatibiliteitsrecord
is verlopen; controleer eerst of er geen ondersteund upgradepad is dat de
reparatie nog nodig heeft. Valideer ook elke vervangingsannotatie opnieuw tijdens de releaseplanning,
omdat het eigenaarschap van plugins en de configuratieomvang kunnen veranderen wanneer providers
en kanalen uit de kern worden verplaatst.

## Afschrijvingsbeleid

OpenClaw mag een gedocumenteerd Plugin-contract niet verwijderen in dezelfde release
waarin de vervanging ervan wordt geïntroduceerd. Migratievolgorde:

1. Voeg het nieuwe contract toe.
2. Houd het oude gedrag aangesloten via een benoemde compatibiliteitsadapter.
3. Geef diagnostiek of waarschuwingen wanneer Pluginauteurs actie kunnen ondernemen.
4. Documenteer de vervanging en het tijdschema.
5. Test zowel het oude als het nieuwe pad.
6. Wacht gedurende het aangekondigde migratievenster.
7. Verwijder alleen met expliciete goedkeuring voor een brekende release.

Afgeschreven records moeten een begindatum voor waarschuwingen, een vervanging, een link naar de
documentatie en een definitieve verwijderingsdatum bevatten die niet meer dan drie maanden na het begin van de waarschuwingen
ligt. Voeg geen afgeschreven compatibiliteitspad toe met een onbeperkt
verwijderingsvenster, tenzij maintainers expliciet besluiten dat het om permanente
compatibiliteit gaat en het in plaats daarvan als `active` markeren.

## Huidige compatibiliteitsgebieden

Bij de controle van juli 2026 zijn de verlopen aliassen voor de hoofd-SDK, het manifest, de provider, de runtime,
registervlaggen en de webconfiguratie in eigendom van plugins verwijderd. Doctor-migraties blijven
afzonderlijk bijgehouden, zodat ondersteunde upgradepaden oude configuraties nog steeds kunnen repareren.

De resterende gedateerde compatibiliteitsgebieden zijn:

- de SDK-subpadvensters van augustus en september die in de migratiehandleiding worden vermeld
- `api.on("deactivate", ...)`- en `api.on("subagent_spawning", ...)`-hookaliassen
- geheugenspecifieke registratie van embeddings en de sessieopslagbrug van beta.5
- de hieronder beschreven aliassen voor inkomende WhatsApp-callbacks
- expliciete parsering van kanaaldoelen en `openclaw/plugin-sdk/messaging-targets`
- ingebedde Pi-agentaliassen
- de uitgebrachte SDK-aliassen van de agentharnas, waarvan de verwijdering wacht op een nieuwe
  extern gedocumenteerde migratiebeslissing

Actieve registerrecords zonder datum omvatten ondersteund gedrag in plaats van
verwijderingsschuld, waaronder activeringshints, het vastleggen van plugins, het inschakelen van gebundelde plugins
en de gegenereerde terugvaloptie voor kanaalconfiguratie.

### Platte aliassen voor inkomende WhatsApp-callbacks

WhatsApp-runtimecallbacks leveren `WebInboundMessage`: de canonieke
geneste contexten `event`, `payload`, `quote`, `group` en `platform`, plus
afgeschreven platte aliassen voor de uitgebrachte callbackvelden. Nieuwe callbackcode
moet de geneste contexten lezen. Code die schone geneste callbackberichten
samenstelt, kan `WebInboundCallbackMessage` gebruiken; compatibiliteitslisteners die
nog steeds oude platte test- of Plugin-berichten invoegen, moeten
`LegacyFlatWebInboundMessage` of `WebInboundMessageInput` gebruiken.

De platte aliassen blijven beschikbaar tot **2026-08-30**; dat venster geldt
alleen voor toegang via platte aliassen, niet voor de geneste vorm, die het canonieke
runtimecontract is. De TypeScript-annotatie `@deprecated` van elke platte alias
noemt de exacte geneste vervanging. Veelvoorkomende voorbeelden:

- `id`, `timestamp` en `isBatched` worden verplaatst onder `event`.
- `body`, `mediaPath`, `mediaType`, `mediaFileName`, `mediaUrl`, `location`
  en `untrustedStructuredContext` worden verplaatst onder `payload`.
- `to`, `chatId`, afzender-/zelfvelden, `sendComposing`, `reply(...)` en
  `sendMedia(...)` worden verplaatst onder `platform`.
- `replyTo*`-velden worden verplaatst onder `quote`; velden voor groepsonderwerp/-deelnemer/-vermelding
  worden verplaatst onder `group`.

`payload.untrustedStructuredContext` wordt uit inkomende providerpayloads
geëxtraheerd. Plugins moeten `label`, `source` en `type` inspecteren voordat
ze de `payload` ervan als gezaghebbend beschouwen.

### Toelatingsvelden voor inkomende WhatsApp-berichten

Geaccepteerde WhatsApp-callbackberichten bevatten `admission`, een openbaar veilige
envelop voor de toegangscontrolebeslissing waarmee het bericht werd toegelaten. Nieuwe
callbackcode moet toelatingsfeiten uit `msg.admission` lezen in plaats van
uit de oudere toelatingsvelden op het hoogste niveau.

De velden op het hoogste niveau blijven beschikbaar tot **2026-08-30**. De
TypeScript-annotatie `@deprecated` van elk veld noemt de vervanging:

- `from` en `conversationId` worden verplaatst naar `admission.conversation.id`.
- `accountId` wordt verplaatst naar `admission.accountId`.
- `accessControlPassed` is een afgeleide compatibiliteitsweergave van
  `admission.ingress.decision === "allow"`; bij berichten die al
  `admission` bevatten, herschrijft het schrijven van de verouderde booleaanse waarde de ingress-
  graaf niet.
- `chatType` wordt verplaatst naar `admission.conversation.kind`.

## Pakket voor de Plugin-inspector

De Plugin-inspector moet buiten de kernrepository van OpenClaw bestaan als een
afzonderlijk pakket/repository dat wordt ondersteund door de geversioneerde compatibiliteits- en
manifestcontracten. De CLI voor de eerste dag moet zijn:

```sh
openclaw-plugin-inspector ./my-plugin
```

Deze moet manifest-/schemavalidatie, de contractcompatibiliteitsversie
die wordt gecontroleerd, controles van installatie-/bronmetadata, importcontroles
voor koude paden en waarschuwingen over afschrijving/compatibiliteit uitvoeren. Gebruik `--json` voor stabiele
machineleesbare uitvoer in CI-annotaties. De kern van OpenClaw moet
contracten en fixtures beschikbaar stellen die de inspector kan gebruiken, maar mag het
inspector-binaire bestand niet publiceren vanuit het hoofdpakket `openclaw`.

### Acceptatietraject voor maintainers

Gebruik door Crabbox ondersteunde Blacksmith Testbox voor het acceptatietraject
van installeerbare pakketten wanneer de externe inspector wordt gevalideerd met OpenClaw-Plugin-
pakketten. Voer dit uit vanuit een schone OpenClaw-checkout nadat het pakket is gebouwd:

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

Houd dit traject optioneel voor maintainers, omdat het een extern npm-
pakket installeert en mogelijk Plugin-pakketten inspecteert die buiten de repository zijn gekloond. De lokale
repositorycontroles dekken de SDK-exportmap, metadata van het compatibiliteitsregister,
de afbouw van afgeschreven SDK-importen en de importgrenzen van gebundelde extensies;
het inspectorbewijs uit Testbox dekt het pakket zoals externe Pluginauteurs
het gebruiken.

## Releaseopmerkingen

Releaseopmerkingen moeten aankomende Plugin-afschrijvingen met streefdatums
en links naar migratiedocumentatie bevatten voordat een compatibiliteitspad naar
`removal-pending` of `removed` wordt verplaatst.
