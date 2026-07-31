---
doc-schema-version: 1
read_when:
    - Plugins installeren of configureren
    - Inzicht in regels voor het ontdekken en laden van plugins
    - Werken met Codex-/Claude-compatibele pluginbundels
sidebarTitle: Getting Started
summary: OpenClaw-plugins installeren, configureren en beheren
title: Plugins
x-i18n:
    generated_at: "2026-07-27T05:27:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f210dccab059527192eeb0aa2e780dcea243959273938ffaacc867ec96f5085e
    source_path: tools/plugin.md
    workflow: 16
---

Plugins breiden OpenClaw uit met kanalen, modelproviders, agentharnassen, tools,
Skills, spraak, realtime transcriptie, stem, mediabegrip, generatie,
webophaling, webzoeken en andere runtimemogelijkheden.

Gebruik deze pagina om een Plugin te installeren, de Gateway opnieuw te starten, te controleren of de runtime
de Plugin heeft geladen en veelvoorkomende configuratiefouten op te lossen. Zie voor voorbeelden met alleen opdrachten
[Plugins beheren](/nl/plugins/manage-plugins). Zie voor de gegenereerde inventaris van
gebundelde, officiële externe en uitsluitend in de broncode aanwezige Plugins
[Plugininventaris](/nl/plugins/plugin-inventory).

## Vereisten

- een OpenClaw-checkout of -installatie waarin de `openclaw` CLI beschikbaar is
- netwerktoegang tot de geselecteerde bron (ClawHub, npm of een git-host)
- alle Pluginspecifieke inloggegevens, configuratiesleutels of OS-tools die in de
  installatiedocumentatie van die Plugin worden genoemd
- toestemming om de Gateway die jouw kanalen bedient opnieuw te laden of te starten

## Snel aan de slag

<Steps>
  <Step title="Zoek de Plugin">
    Zoek in [ClawHub](/nl/clawhub) naar openbare Pluginpakketten:

    ```bash
    openclaw plugins search "calendar"
    ```

    ClawHub is het belangrijkste ontdekkingsplatform voor community-Plugins. Tijdens de
    overgang bij de lancering worden gewone kale pakketspecificaties nog steeds vanuit npm geïnstalleerd, tenzij
    ze overeenkomen met een officiële Plugin-id. Kale `@openclaw/*`-specificaties die overeenkomen met een
    gebundelde Plugin verwijzen naar die gebundelde kopie. Gebruik een expliciet bronvoorvoegsel
    wanneer je specifiek één bron nodig hebt.

  </Step>

  <Step title="Installeer de Plugin">
    ```bash
    # Vanuit ClawHub.
    openclaw plugins install clawhub:<package>

    # Vanuit npm.
    openclaw plugins install npm:<package>

    # Vanuit git.
    openclaw plugins install git:github.com/<owner>/<repo>@<ref>

    # Vanuit een lokale ontwikkelcheckout.
    openclaw plugins install ./my-plugin
    openclaw plugins install --link ./my-plugin
    ```

    Behandel Plugininstallaties alsof je code uitvoert. Geef voor
    reproduceerbare productie-installaties de voorkeur aan vastgezette versies. ClawHub-pakketten en de
    gebundelde/officiële catalogus van OpenClaw zijn vertrouwde bronnen. Voor nieuwe willekeurige npm-, git-,
    lokale pad-/archief-, `npm-pack:`- of marketplace-bronnen is
    `--force` vereist bij niet-interactieve installaties nadat je
    de bron hebt beoordeeld en vertrouwd.

  </Step>

  <Step title="Configureer en schakel de Plugin in">
    Configureer Pluginspecifieke instellingen onder `plugins.entries.<id>.config`.
    Schakel de Plugin in als deze nog niet is ingeschakeld:

    ```bash
    openclaw plugins enable <plugin-id>
    ```

    Als `plugins.allow` is ingesteld, moet de id van de geïnstalleerde Plugin in die lijst staan
    voordat de Plugin kan worden geladen. `openclaw plugins install` voegt de geïnstalleerde
    id toe aan een bestaande `plugins.allow`-lijst en verwijdert dezelfde id uit
    `plugins.deny`, zodat de expliciete installatie na het opnieuw starten kan worden geladen.

  </Step>

  <Step title="Laat de Gateway opnieuw laden">
    Voor het installeren, bijwerken of verwijderen van Plugincode moet de Gateway
    opnieuw worden gestart. Een beheerde Gateway waarvoor het opnieuw laden van configuratie is ingeschakeld, detecteert de gewijzigde
    Plugininstallatierecord en start automatisch opnieuw. Start de Gateway anders
    zelf opnieuw:

    ```bash
    openclaw gateway restart
    ```

    In-/uitschakelen werkt de configuratie en het koude register bij. Een runtime-inspectie is
    nog steeds het duidelijkste bewijs van actieve runtime-oppervlakken.

  </Step>

  <Step title="Controleer runtimeregistratie">
    ```bash
    openclaw plugins inspect <plugin-id> --runtime --json
    ```

    Gebruik `--runtime` om geregistreerde tools, hooks, services, Gateway-
    methoden of CLI-opdrachten van de Plugin aan te tonen. Gewone `inspect` is alleen een koude controle
    van het manifest en register.

  </Step>
</Steps>

## Configuratie

### Kies een installatiebron

| Bron        | Gebruiken wanneer                                                                 | Voorbeeld                                                      |
| ----------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub     | Je OpenClaw-eigen ontdekking, scans, versiemetadata en installatietips wilt        | `openclaw plugins install clawhub:<package>`                   |
| npm         | Je directe workflows voor het npm-register of dist-tags nodig hebt                | `openclaw plugins install npm:<package>`                       |
| git         | Je een branch, tag of commit uit een repository nodig hebt                        | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| lokaal pad  | Je een Plugin op dezelfde machine ontwikkelt of test                              | `openclaw plugins install --link ./my-plugin`                  |
| marketplace | Je een Claude-compatibele marketplace-Plugin installeert                          | `openclaw plugins install <plugin> --marketplace <source>`     |

Kale pakketspecificaties hebben speciaal compatibiliteitsgedrag: een kale naam die
overeenkomt met de id van een gebundelde Plugin gebruikt die gebundelde bron; een kale naam die overeenkomt
met de id van een officiële externe Plugin gebruikt de officiële pakketcatalogus; elke andere
kale specificatie wordt tijdens de overgang bij de lancering via npm geïnstalleerd. Kale `@openclaw/*`-
specificaties die overeenkomen met gebundelde Plugins verwijzen ook naar de gebundelde kopie vóór de
terugval op npm. Gebruik `npm:@openclaw/<plugin>@<version>` om bewust het
externe npm-pakket te installeren in plaats van de gebundelde kopie. Gebruik `clawhub:`, `npm:`,
`git:` of `npm-pack:` voor deterministische bronselectie. Zie
[`openclaw plugins`](/nl/cli/plugins#install) voor het volledige opdrachtcontract.

Voor npm-installaties kiezen niet-vastgezette specificaties en `@latest` het nieuwste stabiele
pakket dat compatibiliteit met deze OpenClaw-build vermeldt. Als de
huidige nieuwste npm-release een nieuwere `openclaw.compat.pluginApi` of
`openclaw.install.minHostVersion` declareert dan deze build ondersteunt, scant OpenClaw
oudere stabiele versies en installeert het de nieuwste passende versie. Exacte versies
en expliciete kanaaltags zoals `@beta` blijven aan het geselecteerde pakket vastgezet
en mislukken wanneer ze incompatibel zijn.

### Installatiebeleid voor beheerders

Configureer `security.installPolicy` om een vertrouwde lokale beleidsopdracht uit te voeren
voordat een Plugininstallatie of -update doorgaat. Het beleid ontvangt metadata plus
het klaargezette bronpad en kan de installatie toestaan of blokkeren. Dit geldt voor zowel CLI-
als Gateway-gebaseerde installatie-/updatepaden. Pluginhooks voor `before_install` worden
later uitgevoerd, en alleen in OpenClaw-processen waarin Pluginhooks zijn geladen, dus gebruik
in plaats daarvan `security.installPolicy` voor installatiebeslissingen van de beheerder. De
verouderde vlag `--dangerously-force-unsafe-install` wordt voor
compatibiliteit geaccepteerd, maar doet niets: deze omzeilt het installatiebeleid of de ingebouwde
blokkeerlijst voor Pluginafhankelijkheden van OpenClaw niet.

Zie [Skills-configuratie](/nl/tools/skills-config#operator-install-policy-securityinstallpolicy)
voor het gedeelde `security.installPolicy`-execschema dat door zowel Skills als
Plugins wordt gebruikt.

### Pluginbeleid configureren

De algemene vorm van de Pluginconfiguratie is:

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-plugin"] },
    slots: { memory: "memory-core" },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

Belangrijkste beleidsregels:

- `plugins.enabled: false` schakelt alle Plugins uit en slaat ontdekkings-/laadwerk
  over. Verouderde Pluginverwijzingen blijven inactief zolang dit actief is; schakel
  Plugins opnieuw in voordat je doctor-opruiming uitvoert als je verouderde id's wilt verwijderen.
- `plugins.deny` heeft voorrang op de toelatingslijst en inschakeling per Plugin.
- `plugins.allow` is een exclusieve toelatingslijst. Tools van Plugins buiten de
  toelatingslijst blijven niet beschikbaar, zelfs wanneer `tools.allow` `"*"` bevat.
- `plugins.entries.<id>.enabled: false` schakelt één Plugin uit terwijl de
  configuratie behouden blijft.
- `plugins.load.paths` voegt expliciete lokale Pluginbestanden of -mappen toe.
  Beheerde lokale paden in `plugins install` moeten Pluginmappen of
  -archieven zijn; gebruik `plugins.load.paths` voor zelfstandige Pluginbestanden.
- Plugins uit de workspace zijn standaard uitgeschakeld; schakel ze expliciet in of
  voeg ze toe aan de toelatingslijst voordat je lokale workspacecode gebruikt.
- Gebundelde Plugins volgen hun ingebouwde metadata voor standaard aan/uit,
  tenzij de configuratie dit expliciet overschrijft.
- `plugins.slots.<slot>` (`memory` of `contextEngine`) kiest één Plugin voor een
  exclusieve categorie. Slotselectie telt als expliciete activering en
  schakelt de geselecteerde Plugin voor dat slot geforceerd in, zelfs als deze anders
  aanmelding vereist. `plugins.deny` en `plugins.entries.<id>.enabled: false` blokkeren
  de Plugin nog steeds.
- Gebundelde opt-in-Plugins kunnen automatisch worden geactiveerd wanneer de configuratie een van hun
  eigen oppervlakken noemt, zoals een provider-/modelverwijzing, kanaalconfiguratie, CLI-backend
  of agentharnasruntime.
- Codex-routering binnen de OpenAI-familie houdt de grenzen tussen provider- en runtime-Plugins
  gescheiden: verouderde Codex-modelverwijzingen zijn verouderde configuratie die doctor herstelt,
  terwijl de gebundelde Plugin `codex` eigenaar is van de Codex-app-serverruntime voor
  canonieke `openai/*`-agentverwijzingen, expliciete `agentRuntime.id: "codex"` en
  verouderde `codex/*`-verwijzingen.

Wanneer `plugins.allow` niet is ingesteld en niet-gebundelde Plugins automatisch worden ontdekt vanuit
de workspace of algemene Pluginroots, registreert het opstarten
`plugins.allow is empty; discovered non-bundled plugins may auto-load: ...`
met de ontdekte Plugin-id's en, voor korte lijsten, een minimaal `plugins.allow`-
fragment. Voer [`openclaw plugins list --enabled --verbose`](/nl/cli/plugins#list)
of [`openclaw plugins inspect <id>`](/nl/cli/plugins#inspect) uit voor de vermelde
Plugin-id voordat je vertrouwde Plugins naar `openclaw.json` kopieert. Dezelfde
vertrouwensvastzetting geldt wanneer diagnostiek meldt dat een Plugin is geladen
`without install/load-path provenance`: inspecteer die Plugin-id en zet deze vervolgens vast in
`plugins.allow` of installeer opnieuw vanuit een vertrouwde bron, zodat OpenClaw de installatieherkomst
vastlegt.

Voer `openclaw doctor` of `openclaw doctor --fix` uit wanneer configuratievalidatie
verouderde Plugin-id's, discrepanties in toelatingslijsten/tools of verouderde paden van gebundelde Plugins
meldt.

## Pluginindelingen begrijpen

OpenClaw herkent twee Pluginindelingen:

| Indeling                    | Hoe deze wordt geladen                                                        | Gebruiken wanneer                                                        |
| --------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Systeemeigen OpenClaw-Plugin | `openclaw.plugin.json` plus een runtimemodule die in het proces wordt geladen     | Je OpenClaw-specifieke runtimemogelijkheden installeert of bouwt          |
| Compatibele bundel          | Codex-, Claude- of Cursor-Pluginindeling toegewezen aan de OpenClaw-Plugininventaris | Je compatibele Skills, opdrachten, hooks of bundelmetadata hergebruikt |

Beide indelingen verschijnen in `openclaw plugins list`, `openclaw plugins inspect`,
`openclaw plugins enable` en `openclaw plugins disable`. Zie
[Pluginbundels](/nl/plugins/bundles) voor de compatibiliteitsgrens van bundels en
[Plugins bouwen](/nl/plugins/building-plugins) voor het maken van systeemeigen Plugins.

## Pluginhooks

Plugins kunnen tijdens runtime hooks registreren via twee verschillende API's:

- `api.on(...)` getypeerde hooks voor levenscyclusgebeurtenissen van de runtime. Dit is het
  voorkeursoppervlak voor middleware, beleid, het herschrijven van berichten, het vormgeven van prompts
  en toolbeheer.
- `api.registerHook(...)` voor het interne hooksysteem dat wordt beschreven in
  [Hooks](/nl/automation/hooks). Dit is voornamelijk bedoeld voor grove neveneffecten van opdrachten/de levenscyclus
  en compatibiliteit met bestaande automatisering in HOOK-stijl.

Vuistregel: als de handler prioriteit, samenvoegsemantiek of
blokkeer-/annuleergedrag nodig heeft, gebruik dan getypeerde hooks. Als de handler alleen reageert op `command:new`,
`command:reset`, `message:sent` of vergelijkbare grove gebeurtenissen, volstaat `api.registerHook`.

Door Plugins beheerde interne hooks verschijnen in `openclaw hooks list` met
`plugin:<id>`. Je kunt ze niet in- of uitschakelen via `openclaw hooks`;
schakel in plaats daarvan de Plugin in of uit.

## De actieve Gateway controleren

`openclaw plugins list` en gewone `openclaw plugins inspect` lezen koude configuratie-,
manifest- en registerstatus. Ze bewijzen niet dat een reeds actieve
Gateway dezelfde plugincode heeft geïmporteerd.

Wanneer een plugin geïnstalleerd lijkt, maar livechatverkeer deze niet gebruikt:

```bash
openclaw gateway status --deep --require-rpc
openclaw plugins inspect <plugin-id> --runtime --json
openclaw gateway restart
```

Beheerde Gateways worden automatisch opnieuw gestart na installatie-, update- en
verwijderingswijzigingen van plugins die de pluginbron veranderen. Zorg er bij VPS- of containerinstallaties
voor dat een handmatige herstart gericht is op het daadwerkelijke onderliggende proces `openclaw gateway run` dat
je kanalen bedient, en niet alleen op een wrapper of supervisor.

## Probleemoplossing

| Symptoom                                                        | Controle                                                                                                                                      | Oplossing                                                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| Plugin verschijnt in `plugins list`, maar runtime-hooks worden niet uitgevoerd  | Gebruik `openclaw plugins inspect <id> --runtime --json` en bevestig de actieve Gateway met `gateway status --deep --require-rpc`             | Start de live Gateway opnieuw na installatie-, update-, configuratie- of bronwijzigingen                               |
| Diagnostiek over dubbel eigenaarschap van kanalen of tools verschijnt         | Voer `openclaw plugins list --enabled --verbose` uit, inspecteer elke verdachte plugin met `--runtime --json` en vergelijk het eigenaarschap van kanalen/tools | Schakel één eigenaar uit, verwijder verouderde installaties of gebruik manifest `preferOver` voor opzettelijke vervanging      |
| Configuratie meldt dat een plugin ontbreekt                                | Controleer [Plugininventaris](/nl/plugins/plugin-inventory) om te bepalen of deze gebundeld, officieel extern of alleen als bron beschikbaar is                           | Installeer het externe pakket, schakel de gebundelde plugin in of verwijder verouderde configuratie                         |
| Configuratie is ongeldig tijdens installatie                               | Lees het validatiebericht en voer `openclaw doctor --fix` uit als het naar een verouderde pluginstatus verwijst                                             | Doctor kan ongeldige pluginconfiguratie in quarantaine plaatsen door de vermelding uit te schakelen en de ongeldige payload te verwijderen     |
| Pluginpad is geblokkeerd wegens verdacht eigenaarschap of verdachte machtigingen | Inspecteer de diagnostiek vóór de configuratiefout                                                                                             | Herstel het eigenaarschap/de machtigingen van het bestandssysteem en voer daarna `openclaw plugins registry --refresh` uit                    |
| `OPENCLAW_NIX_MODE=1` blokkeert levenscyclusopdrachten                | Bevestig dat de installatie door Nix wordt beheerd                                                                                                      | Wijzig de pluginselectie in de Nix-bron in plaats van mutatieopdrachten voor plugins te gebruiken                      |
| Importeren van afhankelijkheid mislukt tijdens runtime                             | Controleer of de plugin via npm/git/ClawHub is geïnstalleerd of vanuit een lokaal pad is geladen                                                 | Voer `openclaw plugins update <id>` uit, installeer de bron opnieuw of installeer zelf de lokale plugin-afhankelijkheden |

Wanneer de payloadverificatie van een ingeschakelde beheerde plugin mislukt tijdens het
opstarten van de Gateway, plaatst OpenClaw precies die geïnstalleerde pluginroot voor deze opstart
in quarantaine en blijft het andere plugins bedienen. `openclaw status --all`, `openclaw health`
en `openclaw doctor` rapporteren deze als `configured-unavailable`. Herstel of installeer
de plugin opnieuw en start daarna de Gateway opnieuw. Een gezonde expliciete `plugins.load.paths`-override
met dezelfde plugin-id wordt niet in quarantaine geplaatst door een verouderde defecte installatie.

Wanneer verouderde pluginconfiguratie nog steeds een kanaalplugin noemt die niet langer detecteerbaar is,
verlaagt configuratievalidatie die kanaalsleutel tot een waarschuwing in plaats van een harde
fout, zodat de Gateway bij het opstarten nog steeds alle andere kanalen kan bedienen. Voer
`openclaw doctor --fix` uit om verouderde plugin- en kanaalvermeldingen te verwijderen. Onbekende
kanaalsleutels zonder bewijs van een verouderde plugin blijven validatie afkeuren, zodat typefouten
zichtbaar blijven.

Voor opzettelijke kanaalvervanging moet de voorkeursplugin
`channelConfigs.<channel-id>.preferOver` declareren met de id van de oudere plugin of de plugin met lagere prioriteit.
Als beide plugins expliciet zijn ingeschakeld, respecteert OpenClaw dat verzoek
en rapporteert het diagnostiek over dubbele kanalen/tools in plaats van stilzwijgend
één eigenaar te kiezen.

Als een geïnstalleerd pakket meldt dat het `requires compiled runtime output for
TypeScript entry ...`, is het pakket gepubliceerd zonder de JavaScript-bestanden
die OpenClaw tijdens runtime nodig heeft. Werk het bij of installeer het opnieuw nadat de uitgever
gecompileerd JavaScript heeft uitgebracht, of schakel de plugin uit/verwijder deze tot die tijd.

### Geblokkeerd eigenaarschap van pluginpad

Als de diagnostiek
`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`
meldt en validatie wordt gevolgd door `plugin present but blocked`, heeft OpenClaw
pluginbestanden gevonden die eigendom zijn van een andere Unix-gebruiker dan het proces dat ze laadt.
Laat de pluginconfiguratie staan; herstel het eigenaarschap van het bestandssysteem of voer OpenClaw
uit als dezelfde gebruiker die eigenaar is van de statusmap.

Voor Docker-installaties wordt de officiële image uitgevoerd als `node` (uid `1000`), dus de
vanaf de host als bind-mount gekoppelde OpenClaw-configuratie- en werkruimtemappen moeten normaal gesproken
eigendom zijn van uid `1000`:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

Als je OpenClaw opzettelijk als root uitvoert, herstel je in plaats daarvan de beheerde pluginroot
naar eigenaarschap van root:

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
```

Voer na het herstellen van het eigenaarschap `openclaw doctor --fix` of
`openclaw plugins registry --refresh` opnieuw uit, zodat het persistente pluginregister
overeenkomt met de herstelde bestanden.

### Trage installatie van plugintools

Als agentbeurten lijken vast te lopen tijdens het voorbereiden van tools, schakel dan trace-logging in
en controleer op timingregels van plugintoolfactories:

```bash
openclaw config set logging.level trace
openclaw logs --follow
```

Zoek naar:

```text
[trace:plugin-tools] factory timings ...
```

Het overzicht vermeldt de totale factorytijd en de traagste factories voor plugintools,
inclusief plugin-id, gedeclareerde toolnamen, resultaatvorm en of de tool
optioneel is. Trage regels worden tot waarschuwingen gepromoveerd wanneer één factory
minstens 1s duurt of de totale voorbereiding van factories voor plugintools minstens 5s duurt.

OpenClaw cachet succesvolle resultaten van factories voor plugintools voor herhaalde
resoluties met dezelfde effectieve aanvraagcontext. De cachesleutel omvat
de effectieve runtimeconfiguratie, werkruimte- en agent-id, sandboxbeleid, browserinstellingen,
bezorgingscontext, identiteit van de aanvrager en eigendomsstatus, zodat
factories die van deze vertrouwde velden afhankelijk zijn opnieuw worden uitgevoerd wanneer de context
verandert. Als de tijden hoog blijven, voert de plugin mogelijk kostbaar werk uit voordat
de tooldefinities worden geretourneerd.

Als één plugin de timing domineert, inspecteer dan de runtimeregistraties ervan:

```bash
openclaw plugins inspect <plugin-id> --runtime --json
```

Werk die plugin vervolgens bij, installeer deze opnieuw of schakel deze uit. Pluginauteurs moeten het
kostbaar laden van afhankelijkheden verplaatsen naar het uitvoeringspad van de tool in plaats van dit
binnen de toolfactory te doen.

Zie voor afhankelijkheidsroots, validatie van pakketmetadata, registerrecords, herlaadgedrag bij het opstarten
en opschoning van verouderde gegevens:
[Resolutie van plugin-afhankelijkheden](/nl/plugins/dependency-resolution).

## Gerelateerd

- [Plugins beheren](/nl/plugins/manage-plugins) - opdrachtvoorbeelden voor weergeven, installeren, bijwerken, verwijderen en publiceren
- [`openclaw plugins`](/nl/cli/plugins) - volledige CLI-referentie
- [Plugininventaris](/nl/plugins/plugin-inventory) - gegenereerde lijst met gebundelde en externe plugins
- [Pluginreferentie](/nl/plugins/reference) - gegenereerde referentiepagina's per plugin
- [Communityplugins](/nl/plugins/community) - ClawHub-detectie en beleid voor documentatie-PR's
- [Resolutie van plugin-afhankelijkheden](/nl/plugins/dependency-resolution) - installatieroots, registerrecords en runtimegrenzen
- [Plugins bouwen](/nl/plugins/building-plugins) - handleiding voor het ontwikkelen van native plugins
- [Overzicht van de Plugin SDK](/nl/plugins/sdk-overview) - runtimeregistratie, hooks en API-velden
- [Pluginmanifest](/nl/plugins/manifest) - manifest- en pakketmetadata
