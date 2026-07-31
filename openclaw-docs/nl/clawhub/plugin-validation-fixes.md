---
read_when:
    - Je hebt clawhub package validate uitgevoerd en moet de Plugin-bevindingen oplossen
    - ClawHub heeft de publicatie van een Plugin-pakket geweigerd of er een waarschuwing voor gegeven
    - Je werkt de metagegevens van het pluginpakket bij vóór de release
summary: Los bevindingen bij de validatie van ClawHub-pluginpakketten op vóór publicatie
title: Oplossingen voor Plugin-validatie
x-i18n:
    generated_at: "2026-07-27T05:38:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cb869e41c9a9f1c0725f514f6b48095eb3838bf61aaf06c2474a18192f0e819
    source_path: clawhub/plugin-validation-fixes.md
    workflow: 16
---

# Oplossingen voor pluginvalidatie

ClawHub valideert pluginpakketten vóór publicatie en kan ook bevindingen van
geautomatiseerde pakketscans tonen. Deze pagina behandelt bevindingen voor auteurs, dat wil zeggen
bevindingen die de pluginauteur kan oplossen in de pakketmetadata, het manifest, de SDK-
imports of het gepubliceerde artefact.

Interne dekkingsbevindingen van Plugin Inspector worden hier niet behandeld. Als een volledig rapport
onderhoudscodes voor de scanner bevat zonder herstelrichtlijnen voor auteurs, zijn deze
bestemd voor OpenClaw-beheerders en niet voor pluginauteurs.

Voer na elke toegepaste oplossing opnieuw het volgende uit:

```bash
clawhub package validate <path-to-plugin>
```

## Bevindingen voor auteurs

| Code                                    | Begin hier                                                                                                                  |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `package-json-missing`                  | [Pakketmetadata toevoegen](/nl/clawhub/plugin-validation-fixes#package-json-missing)                                                   |
| `package-openclaw-metadata-missing`     | [Het openclaw-blok aan het pakket toevoegen](/nl/clawhub/plugin-validation-fixes#package-openclaw-metadata-missing)                            |
| `package-openclaw-entry-missing`        | [OpenClaw-pakketingangspunten declareren](/nl/clawhub/plugin-validation-fixes#package-openclaw-entry-missing)                         |
| `package-entrypoint-missing`            | [Het gedeclareerde ingangspunt publiceren](/nl/clawhub/plugin-validation-fixes#package-entrypoint-missing)                                  |
| `package-install-metadata-incomplete`   | [Installatiemetadata voltooien](/nl/clawhub/plugin-validation-fixes#package-install-metadata-incomplete)                               |
| `package-plugin-api-compat-missing`     | [Compatibiliteit met de plugin-API declareren](/nl/clawhub/plugin-validation-fixes#package-plugin-api-compat-missing)                          |
| `package-min-host-version-drift`        | [Minimale hostversie afstemmen](/nl/clawhub/plugin-validation-fixes#package-min-host-version-drift)                                   |
| `package-manifest-version-drift`        | [Pakket- en manifestversies afstemmen](/nl/clawhub/plugin-validation-fixes#package-manifest-version-drift)                          |
| `package-openclaw-unsupported-metadata` | [Niet-ondersteunde OpenClaw-pakketmetadata verwijderen](/nl/clawhub/plugin-validation-fixes#package-openclaw-unsupported-metadata)          |
| `package-npm-pack-unavailable`          | [Zorgen dat het npm-artefact kan worden verpakt](/nl/clawhub/plugin-validation-fixes#package-npm-pack-unavailable)                                 |
| `package-npm-pack-entrypoint-missing`   | [Ingangspunten opnemen in de uitvoer van npm pack](/nl/clawhub/plugin-validation-fixes#package-npm-pack-entrypoint-missing)                  |
| `package-npm-pack-metadata-missing`     | [Metadata opnemen in de uitvoer van npm pack](/nl/clawhub/plugin-validation-fixes#package-npm-pack-metadata-missing)                       |
| `manifest-name-missing`                 | [Een weergavenaam aan het manifest toevoegen](/nl/clawhub/plugin-validation-fixes#manifest-name-missing)                                           |
| `manifest-unknown-fields`               | [Niet-ondersteunde manifestvelden verwijderen](/nl/clawhub/plugin-validation-fixes#manifest-unknown-fields)                                  |
| `manifest-unknown-contracts`            | [Niet-ondersteunde contractsleutels verwijderen](/nl/clawhub/plugin-validation-fixes#manifest-unknown-contracts)                                 |
| `legacy-root-sdk-import`                | [SDK-imports uit de hoofdmodule vervangen](/nl/clawhub/plugin-validation-fixes#legacy-root-sdk-import)                                             |
| `reserved-sdk-import`                   | [Gereserveerde SDK-imports verwijderen](/nl/clawhub/plugin-validation-fixes#reserved-sdk-import)                                             |
| `sdk-load-session-store`                | [Toegang tot de volledige sessieopslag vervangen](/nl/clawhub/plugin-validation-fixes#sdk-load-session-store)                                   |
| `sdk-session-store-write`               | [Schrijfbewerkingen naar de volledige sessieopslag vervangen](/nl/clawhub/plugin-validation-fixes#sdk-session-store-write)                                  |
| `sdk-session-file-helper`               | [Helpers voor sessiebestandspaden vervangen](/nl/clawhub/plugin-validation-fixes#sdk-session-file-helper)                                   |
| `sdk-session-transcript-file-target`    | [Verouderde transcriptbestandsdoelen vervangen](/nl/clawhub/plugin-validation-fixes#sdk-session-transcript-file-target)                   |
| `sdk-session-transcript-low-level`      | [Transcripthelpers op laag niveau vervangen](/nl/clawhub/plugin-validation-fixes#sdk-session-transcript-low-level)                       |
| `legacy-before-agent-start`             | [before_agent_start vervangen](/nl/clawhub/plugin-validation-fixes#legacy-before-agent-start)                                        |
| `provider-auth-env-vars`                | [Omgevingsvariabelen van de provider naar instellingsmetadata verplaatsen](/nl/clawhub/plugin-validation-fixes#provider-auth-env-vars)                             |
| `channel-env-vars`                      | [Omgevingsvariabelen van het kanaal in de huidige metadata overnemen](/nl/clawhub/plugin-validation-fixes#channel-env-vars)                                |
| `security-manifest-schema-unavailable`  | [Niet-beschikbare schemaverwijzingen voor beveiligingsmanifesten verwijderen](/nl/clawhub/plugin-validation-fixes#security-manifest-schema-unavailable) |
| `unrecognized-security-manifest`        | [Niet-ondersteunde beveiligingsmanifestbestanden verwijderen](/nl/clawhub/plugin-validation-fixes#unrecognized-security-manifest)                   |

## Pakketmetadata

### package-json-missing

De hoofdmap van het pakket bevat geen `package.json`, waardoor ClawHub het
npm-pakket, de versie, de ingangspunten of de OpenClaw-metadata niet kan identificeren.

- Voeg `package.json` toe met `name`, `version` en `type`.
- Voeg een `openclaw`-blok toe wanneer het pakket een OpenClaw-plugin bevat.
- Gebruik [Plugins bouwen](/nl/plugins/building-plugins) voor een minimaal pakketvoorbeeld
  en [Pluginmanifest](/nl/plugins/manifest#manifest-versus-packagejson)
  voor de scheiding tussen pakket en manifest.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-openclaw-metadata-missing

Het pakket bevat `package.json`, maar declareert geen OpenClaw-
pakketmetadata.

- Voeg `package.json#openclaw` toe.
- Neem ingangspuntmetadata op, zoals `openclaw.extensions` of
  `openclaw.runtimeExtensions`.
- Voeg compatibiliteits- en installatiemetadata toe wanneer het pakket via ClawHub wordt gepubliceerd of
  geïnstalleerd.
- Zie [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-openclaw-entry-missing

De pakketmetadata bestaat, maar declareert geen ingangspunt voor de OpenClaw-
runtime.

- Voeg `openclaw.extensions` toe voor ingangspunten van native plugins.
- Voeg `openclaw.runtimeExtensions` toe wanneer het gepubliceerde pakket gebouwde
  JavaScript moet laden.
- Houd alle ingangspuntpaden binnen de pakketmap.
- Zie [Pluginingangspunten](/nl/plugins/sdk-entrypoints) en
  [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-entrypoint-missing

Het pakket declareert een OpenClaw-ingangspunt, maar het bestand waarnaar wordt verwezen ontbreekt
in het pakket dat wordt gevalideerd.

- Controleer elk pad in `openclaw.extensions`, `openclaw.runtimeExtensions`,
  `openclaw.setupEntry` en `openclaw.runtimeSetupEntry`.
- Bouw het pakket als het ingangspunt in `dist` wordt gegenereerd.
- Werk de metadata bij als het ingangspunt is verplaatst.
- Zie [Pluginingangspunten](/nl/plugins/sdk-entrypoints).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-install-metadata-incomplete

ClawHub kan niet bepalen hoe het pakket moet worden geïnstalleerd of bijgewerkt.

- Vul `openclaw.install` in met de ondersteunde installatiebron, zoals
  `clawhubSpec`, `npmSpec` of `localPath`.
- Stel `openclaw.install.defaultChoice` in wanneer er meer dan één installatiebron
  beschikbaar is.
- Gebruik `openclaw.install.minHostVersion` voor de minimale OpenClaw-hostversie.
- Zie [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-plugin-api-compat-missing

Het pakket declareert niet welk bereik van de OpenClaw-plugin-API het ondersteunt.

- Voeg `openclaw.compat.pluginApi` toe aan `package.json`.
- Gebruik de versie of semver-ondergrens van de OpenClaw-plugin-API waartegen je hebt gebouwd en getest.
- Houd dit gescheiden van de pakketversie. De pakketversie beschrijft de
  pluginrelease; `openclaw.compat.pluginApi` beschrijft het host-API-contract.
- Zie [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-min-host-version-drift

De minimale hostversie van het pakket komt niet overeen met de OpenClaw-versiemetadata
waartegen het pakket is gebouwd.

- Controleer `openclaw.install.minHostVersion`.
- Controleer alle OpenClaw-buildmetadata in het pakket, zoals de OpenClaw-versie
  die tijdens de release is gebruikt.
- Stem de minimale hostversie af op het hostversiebereik dat het pakket
  daadwerkelijk ondersteunt.
- Zie [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-manifest-version-drift

De pakketversie en de versie van het pluginmanifest komen niet overeen.

- Gebruik bij voorkeur `package.json#version` als de releaseversie van het pakket.
- Als `openclaw.plugin.json` ook `version` bevat, werk je deze bij zodat deze overeenkomt, of verwijder je
  verouderde manifestversiemetadata wanneer de pakketmetadata leidend is.
- Publiceer een nieuwe pakketversie nadat je gepubliceerde metadata hebt gewijzigd.
- Zie [Pluginmanifest](/nl/plugins/manifest).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-openclaw-unsupported-metadata

Het `package.json#openclaw`-blok bevat velden die niet worden ondersteund als
OpenClaw-pakketmetadata.

- Verwijder niet-ondersteunde velden zoals `openclaw.bundle`.
- Bewaar metadata van native plugins in `openclaw.plugin.json`.
- Bewaar pakketingangspunten en compatibiliteits-, installatie-, instellings- en catalogusmetadata
  in ondersteunde `package.json#openclaw`-velden.
- Zie [package.json-velden die de detectie beïnvloeden](/nl/plugins/manifest#packagejson-fields-that-affect-discovery).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

## Gepubliceerd artefact

### package-npm-pack-unavailable

Het pakket kan niet worden verpakt in het artefact dat ClawHub zou inspecteren of
publiceren.

- Voer `npm pack --dry-run` uit vanuit de hoofdmap van het pakket.
- Herstel ongeldige pakketmetadata, defecte levenscyclusscripts of bestandsvermeldingen die
  het verpakken doen mislukken.
- Verwijder `private: true` als dit pakket bedoeld is voor openbare publicatie.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-npm-pack-entrypoint-missing

Het pakket kan worden verpakt, maar het verpakte artefact bevat niet de
ingangspuntbestanden die in `package.json#openclaw` zijn gedeclareerd.

- Voer `npm pack --dry-run` uit en inspecteer de bestanden die zouden worden opgenomen.
- Bouw gegenereerde ingangspunten voordat je het pakket verpakt.
- Werk `files`, `.npmignore` of de builduitvoer bij, zodat gedeclareerde ingangspunten worden
  opgenomen.
- Zie [Pluginingangspunten](/nl/plugins/sdk-entrypoints).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### package-npm-pack-metadata-missing

In het verpakte artefact ontbreekt OpenClaw-metadata die wel in je bronpakket
aanwezig is.

- Voer `npm pack --dry-run` uit en inspecteer de meegeleverde metadatabestanden.
- Zorg ervoor dat `package.json` het blok `openclaw` bevat in het verpakte artefact.
- Zorg ervoor dat `openclaw.plugin.json` wordt opgenomen wanneer het pakket een native
  OpenClaw-plugin is.
- Werk `files` of `.npmignore` bij, zodat pakketmetadata niet wordt uitgesloten.
- Zie [Plugins bouwen](/nl/plugins/building-plugins).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

## Manifestmetadata

### manifest-name-missing

Het manifest van de native plugin bevat geen weergavenaam.

- Voeg een niet-leeg veld `name` toe aan `openclaw.plugin.json`.
- Houd `name` leesbaar voor mensen en behoud `id` als de stabiele machine-id.
- Zie [Pluginmanifest](/nl/plugins/manifest).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### manifest-unknown-fields

Het pluginmanifest bevat velden op het hoogste niveau die OpenClaw niet ondersteunt.

- Vergelijk elk veld op het hoogste niveau met de
  [referentie voor manifestvelden](/nl/plugins/manifest#top-level-field-reference).
- Verwijder aangepaste velden uit `openclaw.plugin.json`.
- Verplaats pakket- of installatiemetadata naar ondersteunde velden in `package.json#openclaw`
  in plaats van naar het manifest.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### manifest-unknown-contracts

Het manifest declareert niet-ondersteunde sleutels binnen `contracts`.

- Vergelijk elke sleutel onder `contracts` met de
  [contractreferentie](/nl/plugins/manifest#contracts-reference).
- Verwijder niet-ondersteunde contractsleutels.
- Verplaats runtimegedrag naar de registratincode van de plugin en beperk `contracts`
  tot statische metadata over het eigenaarschap van mogelijkheden.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

## SDK- en compatibiliteitsmigratie

### legacy-root-sdk-import

De plugin importeert vanuit de verouderde SDK-barrel op rootniveau:
`openclaw/plugin-sdk`.

- Vervang imports uit de rootbarrel door gerichte imports uit openbare subpaden.
- Gebruik `openclaw/plugin-sdk/plugin-entry` voor `definePluginEntry`.
- Gebruik `openclaw/plugin-sdk/channel-core` voor helpers voor kanaalingangspunten.
- Gebruik [Importconventies](/nl/plugins/building-plugins#import-conventions) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths) om de specifieke import te vinden.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### reserved-sdk-import

De plugin importeert een SDK-pad dat is gereserveerd voor gebundelde plugins of interne
compatibiliteit.

- Vervang gereserveerde interne SDK-imports van OpenClaw door gedocumenteerde openbare
  subpaden van `openclaw/plugin-sdk/*`.
- Als het gedrag geen openbare SDK heeft, houd je de helper binnen je pakket of
  vraag je een openbare OpenClaw-API aan.
- Gebruik [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths) en
  [SDK-migratie](/nl/plugins/sdk-migration) om een ondersteunde import te kiezen.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### sdk-load-session-store

De plugin gebruikt nog steeds de verouderde helper voor het volledige sessiearchief
`loadSessionStore`.

- Gebruik `getSessionEntry(...)` of `listSessionEntries(...)` bij het lezen van de
  sessiestatus.
- Gebruik `patchSessionEntry(...)` of `upsertSessionEntry(...)` bij het schrijven van de
  sessiestatus.
- Vermijd het laden, wijzigen en opslaan van het volledige sessiearchiefobject.
- Behoud `loadSessionStore(...)` alleen zolang je gedeclareerde compatibiliteitsbereik
  nog oudere OpenClaw-versies ondersteunt die dit vereisen.
- Zie [Runtime-API](/nl/plugins/sdk-runtime#agent-session-state) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### sdk-session-store-write

De plugin gebruikt nog steeds een verouderde schrij helper voor het volledige sessiearchief, zoals
`saveSessionStore` of `updateSessionStore`.

- Gebruik `patchSessionEntry(...)` bij het bijwerken van velden in een bestaande
  sessievermelding.
- Gebruik `upsertSessionEntry(...)` bij het vervangen of maken van een sessievermelding.
- Vermijd het laden, wijzigen en opslaan van het volledige sessiearchiefobject.
- Behoud schrij helpers voor het volledige archief alleen zolang je gedeclareerde compatibiliteitsbereik
  nog oudere OpenClaw-versies ondersteunt die deze vereisen.
- Zie [Runtime-API](/nl/plugins/sdk-runtime#agent-session-state) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### sdk-session-file-helper

De plugin gebruikt nog steeds verouderde helpers voor bestandspaden van sessies, zoals
`resolveSessionFilePath` of `resolveAndPersistSessionFile`.

- Gebruik `getSessionEntry(...)` om sessiemetadata te lezen op basis van de identiteit
  van de agent en sessie.
- Gebruik `patchSessionEntry(...)` of `upsertSessionEntry(...)` om sessiemetadata
  op te slaan.
- Gebruik helpers voor transcriptidentiteit of -doelen wanneer de code een
  transcriptbewerking voorbereidt.
- Sla verouderde bestandspaden voor transcripten niet op en maak je er niet van afhankelijk.
- Zie [Runtime-API](/nl/plugins/sdk-runtime#agent-session-state) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### sdk-session-transcript-file-target

De plugin gebruikt nog steeds de verouderde doelhelper voor transcriptbestanden
`resolveSessionTranscriptLegacyFileTarget`.

- Gebruik `resolveSessionTranscriptIdentity(...)` wanneer de code alleen de openbare
  sessie-identiteit nodig heeft.
- Gebruik `resolveSessionTranscriptTarget(...)` wanneer de code een gestructureerd doel voor een
  transcriptbewerking nodig heeft.
- Vermijd het rechtstreeks lezen of samenstellen van verouderde doelen voor transcriptbestanden.
- Behoud de verouderde helper alleen zolang je gedeclareerde compatibiliteitsbereik nog
  oudere OpenClaw-versies ondersteunt die deze vereisen.
- Zie [Runtime-API](/nl/plugins/sdk-runtime#agent-session-state) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### sdk-session-transcript-low-level

De plugin gebruikt nog steeds verouderde low-level-transcripthelpers, zoals
`appendSessionTranscriptMessage` of `emitSessionTranscriptUpdate`.

- Gebruik `appendSessionTranscriptMessageByIdentity(...)` om transcripten aan te vullen.
- Gebruik `publishSessionTranscriptUpdateByIdentity(...)` voor meldingen over
  transcriptupdates.
- Geef de voorkeur aan het gestructureerde runtimeoppervlak voor transcripten, zodat OpenClaw de
  juiste transactiegrenzen en identiteitsafhandeling kan toepassen.
- Behoud low-level-transcripthelpers alleen zolang je gedeclareerde compatibiliteitsbereik
  nog oudere OpenClaw-versies ondersteunt die deze vereisen.
- Zie [Runtime-API](/nl/plugins/sdk-runtime#agent-session-state) en
  [Subpaden van de Plugin-SDK](/nl/plugins/sdk-subpaths).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### legacy-before-agent-start

De plugin gebruikt nog steeds de verouderde hook `before_agent_start`.

- Verplaats overschrijvingen van modellen of providers naar `before_model_resolve`.
- Verplaats wijzigingen aan prompts of context naar `before_prompt_build`.
- Behoud `before_agent_start` alleen zolang je gedeclareerde compatibiliteitsbereik nog
  oudere OpenClaw-versies ondersteunt die dit vereisen.
- Zie [Hooks](/nl/plugins/hooks) en
  [Plugincompatibiliteit](/nl/plugins/compatibility).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### provider-auth-env-vars

Het manifest gebruikt nog steeds verouderde authenticatiemetadata voor providers in `providerAuthEnvVars`.

- Neem metadata over provideromgevingsvariabelen ook op in `setup.providers[].envVars`.
- Behoud `providerAuthEnvVars` alleen als compatibiliteitsmetadata zolang je ondersteunde
  OpenClaw-bereik dit nog nodig heeft.
- Zie [setupreferentie](/nl/plugins/manifest#setup-reference) en
  [SDK-migratie](/nl/plugins/sdk-migration).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### channel-env-vars

Het manifest gebruikt verouderde of oudere metadata voor kanaalomgevingsvariabelen zonder de huidige
setup- of configuratiemetadata die ClawHub verwacht.

- Houd metadata voor kanaalomgevingsvariabelen declaratief, zodat OpenClaw de setupstatus kan inspecteren
  zonder de kanaalruntime te laden.
- Neem door omgevingsvariabelen aangestuurde kanaalsetup ook op in de huidige setup-, kanaalconfiguratie- of
  pakketmetadata voor kanalen die door de vorm van je plugin wordt gebruikt.
- Behoud `channelEnvVars` alleen als compatibiliteitsmetadata zolang oudere ondersteunde
  OpenClaw-versies dit nog vereisen.
- Zie [Pluginmanifest](/nl/plugins/manifest) en
  [Kanaalplugins](/nl/plugins/sdk-channel-plugins).
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

## Beveiligingsmanifest

### security-manifest-schema-unavailable

Het pakket bevat `openclaw.security.json` met een schemaverwijzing die ClawHub
niet als beschikbaar herkent.

- Verwijder de schema-URL als deze uitsluitend adviserend is.
- Gebruik pas een gedocumenteerd schema met versie nadat OpenClaw er een publiceert.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

### unrecognized-security-manifest

Het pakket bevat een niet-ondersteund beveiligingsmanifestbestand.

- Verwijder `openclaw.security.json` totdat OpenClaw een beveiligingsmanifestschema met versie
  en het gedrag van ClawHub documenteert.
- Blijf beveiligingsgevoelig gedrag documenteren in je openbare pakketdocumentatie of
  README totdat het manifestcontract bestaat.
- Voer `clawhub package validate <path-to-plugin>` opnieuw uit.

## Gerelateerd

- [ClawHub-CLI](/nl/clawhub/cli)
- [Publiceren op ClawHub](/nl/clawhub/publishing)
- [Plugins bouwen](/nl/plugins/building-plugins)
- [Pluginmanifest](/nl/plugins/manifest)
- [Ingangspunten van plugins](/nl/plugins/sdk-entrypoints)
- [Plugincompatibiliteit](/nl/plugins/compatibility)
