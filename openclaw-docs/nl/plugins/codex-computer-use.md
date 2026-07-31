---
read_when:
    - Je wilt dat OpenClaw-agenten in Codex-modus Codex Computer Use gebruiken
    - Je kiest tussen Codex Computer Use, PeekabooBridge en directe cua-driver-MCP
    - Je configureert computerUse voor de meegeleverde Codex-plugin
    - Je lost problemen met de status of installatie van computergebruik in /codex op
summary: Codex Computer Use instellen voor OpenClaw-agents in Codex-modus
title: Codex-computergebruik
x-i18n:
    generated_at: "2026-07-27T06:00:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use is een Codex-native MCP-plugin voor lokale desktopbesturing. OpenClaw
levert de desktopapp niet mee, voert zelf geen desktopacties uit en omzeilt
Codex-machtigingen niet. De meegeleverde `codex`-plugin bereidt alleen Codex app-server voor:
deze schakelt ondersteuning voor Codex-plugins in, zoekt of installeert de geconfigureerde Computer Use-
plugin, controleert of de `computer-use` MCP-server beschikbaar is en laat vervolgens
Codex de native MCP-toolaanroepen beheren tijdens beurten in Codex-modus.

Gebruik deze pagina wanneer OpenClaw de native Codex-harness al gebruikt. Zie
[Codex-harness](/nl/plugins/codex-harness) voor het instellen van de runtime zelf.

Dit verschilt van de ingebouwde [door een Node ondersteunde computertool](/nl/nodes/computer-use) van OpenClaw. Gebruik de ingebouwde tool wanneer hetzelfde agentcontract een gekoppelde Mac moet besturen, ongeacht of de agent op de Gateway of een andere Node draait. Gebruik Codex Computer Use wanneer Codex app-server de lokale MCP-installatie, machtigingen en native toolaanroepen moet beheren.

## OpenClaw.app en Peekaboo

De Peekaboo-integratie van OpenClaw.app staat los van Codex Computer Use. De
macOS-app kan een PeekabooBridge-socket hosten, zodat de `peekaboo` CLI de
lokale toestemmingen van de app voor Accessibility en Screen Recording kan hergebruiken voor de eigen
automatiseringstools van Peekaboo. Die bridge installeert of proxyt Codex Computer Use niet, en
Codex Computer Use roept niets aan via de PeekabooBridge-socket.

Gebruik [Peekaboo-bridge](/nl/platforms/mac/peekaboo) wanneer OpenClaw.app
een machtigingsbewuste host voor automatisering met de Peekaboo CLI moet zijn. Gebruik deze pagina wanneer een
OpenClaw-agent in Codex-modus vóór het begin van de beurt over de native `computer-use` MCP-plugin van Codex
moet beschikken.

## iOS-app

De iOS-app staat los van Codex Computer Use. Deze installeert of proxyt
de Codex `computer-use` MCP-server niet en is geen backend voor desktopbesturing.
In plaats daarvan maakt de iOS-app verbinding als een OpenClaw-Node en stelt deze mobiele
mogelijkheden beschikbaar via Node-opdrachten zoals `canvas.*`, `camera.*`, `screen.*`,
`location.*` en `talk.*`.

Gebruik [iOS](/nl/platforms/ios) wanneer een agent een iPhone-Node
via de Gateway moet aansturen. Gebruik deze pagina wanneer een agent in Codex-modus de
lokale macOS-desktop moet besturen via de native Computer Use-plugin van Codex.

## Directe cua-driver-MCP

Codex Computer Use is niet de enige manier om desktopbesturing beschikbaar te stellen. Als
door OpenClaw beheerde runtimes de driver van TryCua rechtstreeks moeten aanroepen, gebruik je de upstream
`cua-driver mcp`-server via het MCP-register van OpenClaw in plaats van de
Codex-specifieke marketplaceflow.

Vraag na installatie van `cua-driver` om de OpenClaw-opdracht:

```bash
cua-driver mcp-config --client openclaw
```

of registreer de stdio-server rechtstreeks:

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

Dat pad houdt het upstream MCP-tooloppervlak intact, inclusief de driver-
schema's en gestructureerde MCP-reacties. Gebruik het wanneer de CUA-driver
beschikbaar moet zijn als een normale OpenClaw MCP-server. Gebruik de configuratie voor Codex Computer Use op
deze pagina wanneer Codex app-server de plugininstallatie, het herladen van MCP-servers
en native toolaanroepen binnen beurten in Codex-modus moet beheren.

De driver van CUA levert prereleaseversies voor macOS, Windows (x64 en ARM64) en
Linux (x64 en ARM64, previewniveau). Deze vereist nog steeds de lokale
besturingssysteemmachtigingen waarom de app vraagt, zoals Accessibility en Screen Recording op
macOS. OpenClaw installeert `cua-driver` niet, verleent die machtigingen niet en
omzeilt het veiligheidsmodel van de upstream driver niet.

## Snelle configuratie

Stel `plugins.entries.codex.config.computerUse` in wanneer Computer Use voor beurten in Codex-modus
beschikbaar moet zijn voordat een thread start. Met `autoInstall: true` schakel je
Computer Use in en kan OpenClaw het vóór de beurt installeren of opnieuw inschakelen:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Met deze configuratie controleert OpenClaw Codex app-server vóór elke beurt in Codex-
modus. Als Computer Use ontbreekt, maar Codex app-server al een
installeerbare marketplace heeft gevonden, vraagt OpenClaw Codex app-server om de plugin te installeren of
opnieuw in te schakelen en de MCP-servers opnieuw te laden. Voordat op macOS een geïsoleerde
Codex app-server wordt gestart, kopieert automatische installatie ook de officiële ondertekende
Computer Use-serviceapp uit de geselecteerde desktopappbundel naar de map
`computer-use` van die Codex-home wanneer de native client ontbreekt.
Wanneer op macOS geen overeenkomende
marketplace is geregistreerd en er een standaarddesktopappbundel bestaat, probeert OpenClaw
ook de meegeleverde Codex-marketplace te registreren vanuit
`/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled`, waarbij
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled` behouden blijft
als terugvaloptie voor verouderde zelfstandige installaties. Als de configuratie de
MCP-server nog steeds niet beschikbaar kan maken, mislukt de beurt voordat de thread start.
Strikte gereedheidsfouten zijn preflightfouten van de harness, zodat modelterugval
niet voor elke modelkandidaat dezelfde lokale gereedheidsreeks herhaalt.

Gebruik na het wijzigen van de Computer Use-configuratie `/new` of `/reset` in de betreffende
chat voordat je test als er al een bestaande Codex-thread is gestart.

Voor beheerd opstarten van Computer Use op macOS heeft het binaire bestand van de desktopapp op
`/Applications/ChatGPT.app/Contents/Resources/codex` de voorkeur, waarna wordt
teruggevallen op `/Applications/Codex.app/Contents/Resources/codex` voor verouderde
zelfstandige installaties. Dit geldt ook voor eenmalige status- en
installatieopdrachten voor Computer Use die hun eigen client starten. Zo blijft desktopbesturing onder
de appbundel die de lokale macOS-machtigingen bezit. Als de desktopapp niet
is geïnstalleerd, valt OpenClaw terug op het beheerde Codex-binaire bestand dat naast de
plugin is geïnstalleerd. Normale beheerde Codex-beurten met de standaard geïsoleerde agent-home geven
eerst de voorkeur aan dat vastgezette pakket, zodat een oudere desktopapp de huidige model-
ondersteuning niet kan overschaduwen. Gebruikersgebonden homes blijven eerst de desktopapp gebruiken omdat ze native
Computer Use-status kunnen laden. Een geïsoleerde agent-home waarvan de effectieve Codex-configuratie
Computer Use inschakelt, blijft ook eerst de desktopapp gebruiken. Expliciete
`appServer.command`-configuratie of `OPENCLAW_CODEX_APP_SERVER_BIN` overschrijft
deze beheerde selectie nog steeds.

OpenClaw serialiseert het lezen van de native Codex-configuratie en de installatie van Computer Use
binnen één actieve Gateway. Een afzonderlijk Codex-proces of een andere Gateway valt niet
binnen die afscherming. Start de Gateway opnieuw en start een nieuwe chat nadat je de native
Codex-pluginconfiguratie buiten de Gateway hebt gewijzigd, voordat je op de nieuwe
selectie vertrouwt.

## Opdrachten

Gebruik de `/codex computer-use`-opdrachten vanaf elk chatoppervlak waar het
`codex`-pluginopdrachtoppervlak beschikbaar is. Dit zijn OpenClaw-chat-/runtime-
opdrachten, geen `openclaw codex ...` CLI-subopdrachten:

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` is de standaardactie en is alleen-lezen: deze voegt geen marketplace-
bronnen toe, installeert geen plugins en schakelt ondersteuning voor Codex-plugins niet in. Als geen configuratie
Computer Use inschakelt, kan `status` uitgeschakeld melden, zelfs na een eenmalige installatie-
opdracht.

`install` schakelt ondersteuning voor Codex app-server-plugins in, voegt optioneel een
geconfigureerde marketplacebron toe, installeert de geconfigureerde plugin of schakelt deze opnieuw in
via Codex app-server, herlaadt MCP-servers en verifieert dat de MCP-
server tools beschikbaar stelt. Omdat de installatie vertrouwde hostresources wijzigt,
kan alleen een eigenaar of een `operator.admin` Gateway-client `install` uitvoeren. Andere
geautoriseerde afzenders kunnen de alleen-lezen opdracht `status` blijven gebruiken,
ook met overschrijvingen.

Oudere releases accepteerden eenmalige identiteitsoverschrijvingen voor `--plugin`, `--server` en `--mcp-server`.
Configureer in plaats daarvan `computerUse.pluginName` en
`computerUse.mcpServerName` permanent. Wanneer een verouderde identiteitsvlag
wordt gebruikt, vermeldt de opdracht de exacte instelling die permanent moet worden gemaakt en herhaalt deze
de gevraagde actie plus eventuele ondersteunde marketplacevlaggen in de migratie-instructies.

## Marketplacekeuzes

OpenClaw gebruikt dezelfde app-server-API die Codex zelf beschikbaar stelt. De
marketplacevelden bepalen waar Codex `computer-use` moet vinden.

| Veld                 | Gebruiken wanneer                                                 | Installatieondersteuning                                  |
| -------------------- | ----------------------------------------------------------------- | --------------------------------------------------------- |
| Geen marketplaceveld | Je wilt dat Codex app-server marketplaces gebruikt die al bekend zijn. | Ja, wanneer app-server een lokale marketplace retourneert. |
| `marketplaceSource`  | Je hebt een Codex-marketplacebron die app-server kan toevoegen.   | Ja, voor expliciete `/codex computer-use install`.         |
| `marketplacePath`    | Je kent het lokale bestandspad van de marketplace op de host al.  | Ja, voor expliciete installatie en automatische installatie bij het starten van een beurt. |
| `marketplaceName`    | Je wilt één reeds geregistreerde marketplace op naam selecteren.  | Alleen ja wanneer de geselecteerde marketplace een lokaal pad heeft. |

Nieuwe Codex-homes hebben mogelijk even tijd nodig om hun officiële
marketplaces te initialiseren. Tijdens de installatie peilt OpenClaw `plugin/list` maximaal
`marketplaceDiscoveryTimeoutMs` milliseconden lang (standaard 60 seconden).

Als meerdere bekende marketplaces Computer Use bevatten, geeft OpenClaw de voorkeur aan
`openai-bundled`, vervolgens `openai-curated` en daarna `local`. Onbekende ambigue
overeenkomsten worden standaard geweigerd en vragen je `marketplaceName` of
`marketplacePath` in te stellen.

## Meegeleverde macOS-marketplace

Huidige ChatGPT-desktopversies leveren Computer Use hier mee; verouderde zelfstandige
Codex-desktopversies gebruiken dezelfde indeling onder `Codex.app`:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

Wanneer `computerUse.autoInstall` waar is en geen marketplace met
`computer-use` is geregistreerd, probeert OpenClaw de eerste bestaande standaard
meegeleverde marketplaceroot toe te voegen:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

Je kunt deze ook expliciet vanuit een shell registreren met Codex:

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

Als je een niet-standaardpad voor de Codex-app gebruikt, voer je `/codex computer-use install
--source <marketplace-root>` eenmaal uit of stel je `computerUse.marketplacePath` in op een
lokaal marketplacebestandspad. Gebruik `--marketplace-path` alleen wanneer je het pad naar het
marketplace-JSON-bestand hebt, niet de meegeleverde marketplaceroot.

### Gedeelde plugincache

De standaardinstelling `pluginCacheMode: "independent"` laat elke Codex-home en de bijbehorende
plugincache onbeheerd. Stel `pluginCacheMode: "shared"` in om de meegeleverde
Computer Use-plugin vóór het opstarten van app-server naar de vindbare plugincache van de actieve Codex-home
te kopiëren. De gedeelde modus behoudt oudere gecachte versies omdat
actieve Codex-clients nog steeds naar hun geversioneerde pluginmappen kunnen verwijzen; bij een
mislukte vervangende kopie blijft de actieve cache eveneens behouden. Expliciete configuratie van
`marketplaceName` of `marketplacePath` schakelt deze
afstemming uit, zodat OpenClaw die selectie niet overschrijft.

## Beperking van externe catalogus

Codex app-server kan uitsluitend externe catalogusvermeldingen weergeven en lezen, maar ondersteunt
momenteel geen externe `plugin/install`. Dit betekent dat `marketplaceName`
een uitsluitend externe marketplace kan selecteren voor statuscontroles, maar voor installaties en
opnieuw inschakelen nog steeds een lokale marketplace via `marketplaceSource` of
`marketplacePath` nodig is.

Als de status aangeeft dat de plugin beschikbaar is in een externe Codex-marketplace, maar
externe installatie niet wordt ondersteund, voer je de installatie uit met een lokale bron of een lokaal pad:

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## Configuratiereferentie

| Veld                           | Standaardwaarde | Betekenis                                                                      |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| `enabled`                       | afgeleid        | Computer Use vereisen. Is standaard true wanneer een ander Computer Use-veld is ingesteld. |
| `autoInstall`                   | false          | De native client inrichten en de Plugin installeren of opnieuw inschakelen bij het begin van de beurt. |
| `marketplaceDiscoveryTimeoutMs` | 60000          | Hoelang de installatie wacht op marketplace-detectie door de Codex-app-server. |
| `liveTestTimeoutMs`             | 60000          | Time-out voor de tijdelijke gereedheidsthread en de bijbehorende opschoningsverzoeken. |
| `toolCallTimeoutMs`             | 60000          | Time-out voor de Computer Use-aanroep van het gereedheidshulpmiddel `list_apps`. |
| `healthCheckEnabled`            | false          | Periodieke gereedheidscontroles uitvoeren zolang de bijbehorende app-serverclient actief is. |
| `healthCheckIntervalMinutes`    | 60             | Controle-interval; geaccepteerde waarden zijn 30, 60, 120 of 240 minuten. |
| `pluginCacheMode`               | `independent`  | `shared` gebruiken om de Codex-home-cache vanuit de meegeleverde desktop-Plugin te vernieuwen. |
| `strictReadiness`               | false          | Het opstarten stoppen bij een mislukte livecontrole in plaats van door te gaan met een waarschuwing. |
| `autoRepair`                    | false          | Verouderde, bereikgebonden Computer Use-MCP-subprocessen beëindigen en een mislukte controle eenmaal opnieuw proberen. |
| `marketplaceSource`             | niet ingesteld | Bronstring die wordt doorgegeven aan `marketplace/add` van de Codex-app-server. |
| `marketplacePath`               | niet ingesteld | Lokaal bestandspad van de Codex-marketplace die de Plugin bevat. |
| `marketplaceName`               | niet ingesteld | Naam van de geregistreerde Codex-marketplace die moet worden geselecteerd. |
| `pluginName`                    | `computer-use` | Naam van de Codex-marketplace-Plugin. |
| `mcpServerName`                 | `computer-use` | Naam van de MCP-server die door de geïnstalleerde Plugin beschikbaar wordt gesteld. |

Automatische installatie bij het begin van een beurt weigert bewust geconfigureerde
`marketplaceSource`-waarden. Het toevoegen van een nieuwe bron is een expliciete
installatiehandeling, dus gebruik `/codex computer-use install --source <marketplace-source>` eenmaal en laat daarna
`autoInstall` toekomstige herinschakelingen vanuit gedetecteerde lokale marketplaces
afhandelen. Automatische installatie bij het begin van een beurt kan een geconfigureerde
`marketplacePath` gebruiken, omdat dit al een lokaal pad op de host is.

Elk veld accepteert ook een overschrijving via een omgevingsvariabele, die wordt
gecontroleerd wanneer de overeenkomende configuratiesleutel niet is ingesteld:

| Veld                           | Omgevingsvariabele                                               |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## Wat OpenClaw controleert

OpenClaw rapporteert intern een stabiele installatiereden en formatteert de
gebruikersgerichte status voor de chat:

| Reden                        | Betekenis                                              | Volgende stap                                  |
| ---------------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `disabled`                   | `computerUse.enabled` is herleid tot false.            | Stel `enabled` of een ander Computer Use-veld in. |
| `marketplace_missing`        | Er was geen overeenkomende marketplace beschikbaar.    | Configureer een bron, pad of marketplacenaam. |
| `plugin_not_installed`       | De marketplace bestaat, maar de Plugin is niet geïnstalleerd. | Voer de installatie uit of schakel `autoInstall` in. |
| `plugin_disabled`            | De Plugin is geïnstalleerd, maar uitgeschakeld in de Codex-configuratie. | Voer de installatie uit om deze opnieuw in te schakelen. |
| `remote_install_unsupported` | De geselecteerde marketplace is uitsluitend extern.    | Gebruik `marketplaceSource` of `marketplacePath`. |
| `mcp_missing`                | De Plugin is ingeschakeld, maar de MCP-server is niet beschikbaar. | Controleer Codex Computer Use en de OS-machtigingen. |
| `ready`                      | De Plugin en MCP-hulpmiddelen zijn beschikbaar.        | Start de beurt in Codex-modus.                |
| `check_failed`               | Een verzoek aan de Codex-app-server is mislukt tijdens de statuscontrole. | Controleer de verbinding met de app-server en de logboeken. |
| `auto_install_blocked`       | Voor de installatie bij het begin van een beurt zou een nieuwe bron moeten worden toegevoegd. | Voer eerst een expliciete installatie uit. |

De chatuitvoer bevat de Pluginstatus, de MCP-serverstatus, de marketplace,
hulpmiddelen wanneer die beschikbaar zijn en het specifieke bericht voor de
mislukte installatiestap.

## macOS-machtigingen

Dit door Codex beheerde Computer Use-pad draait op macOS, waar de MCP-server
mogelijk lokale OS-machtigingen nodig heeft voordat deze apps kan inspecteren
of bedienen. (Zie voor platformoverschrijdende desktopbediening op Windows- en
Linux-Node-hosts de
[cua-computer-uitvoerder](/nl/nodes/computer-use#windows-and-linux-experimental-via-cua-driver).)
Als OpenClaw meldt dat Computer Use is geïnstalleerd maar de MCP-server niet
beschikbaar is, controleer dan eerst de Computer Use-installatie aan de Codex-zijde:

- De Codex-app-server draait op dezelfde host waarop de desktopbediening moet
  plaatsvinden.
- De Computer Use-Plugin is ingeschakeld in de Codex-configuratie.
- De MCP-server `computer-use` verschijnt in de MCP-status van de Codex-app-server.
- macOS heeft de vereiste machtigingen verleend aan de app voor desktopbediening.
- De huidige hostsessie heeft toegang tot de desktop die wordt bediend.

OpenClaw stopt bewust veilig wanneer `computerUse.enabled` true is. Een beurt in
Codex-modus mag niet stilzwijgend doorgaan zonder de native desktophulpmiddelen
die volgens de configuratie vereist zijn.

## Probleemoplossing

**De status meldt dat de Plugin niet is geïnstalleerd.** Voer `/codex computer-use install`
uit. Als de marketplace niet wordt gedetecteerd, geef dan `--source` of
`--marketplace-path` door.

**De status meldt dat de Plugin is geïnstalleerd maar uitgeschakeld.** Voer
`/codex computer-use install` opnieuw uit. De installatie via de Codex-app-server schrijft
de Pluginconfiguratie terug als ingeschakeld.

**De status meldt dat externe installatie niet wordt ondersteund.** Gebruik
een lokale marketplacebron of een lokaal pad. Catalogusitems die uitsluitend
extern beschikbaar zijn, kunnen worden bekeken maar niet via de huidige
app-server-API worden geïnstalleerd.

**De status meldt dat de MCP-server niet beschikbaar is.** Voer de installatie
eenmaal opnieuw uit zodat de MCP-servers opnieuw worden geladen. Als de server
niet beschikbaar blijft, herstel dan de Codex Computer Use-app, de MCP-status
van de Codex-app-server of de macOS-machtigingen.

**De status of een controle bereikt een time-out bij `computer-use.list_apps`.**
De Plugin en MCP-server zijn aanwezig, maar de lokale Computer Use-bridge
antwoordde niet. Sluit of herstart Codex Computer Use, start indien nodig
Codex Desktop opnieuw en probeer het vervolgens opnieuw in een nieuwe
OpenClaw-sessie. Als Computer Use eerder op de host werd uitgevoerd via een
oudere beheerde Codex-app-server, vernieuw dan de geïnstalleerde Plugin vanuit
de meegeleverde marketplace van de desktopapp (gebruik het pad
`Codex.app` voor zelfstandige Codex-desktopinstallaties):

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**Een Computer Use-hulpmiddel meldt `Native hook relay unavailable`.** De native
Codex-hulpmiddelkoppeling kon geen actieve OpenClaw-relay bereiken via de
lokale bridge of de Gateway-terugval. Start een nieuwe OpenClaw-sessie met
`/new` of `/reset`. Als het eenmaal werkt en bij een latere
hulpmiddelaanroep opnieuw mislukt, wist `/new` alleen de huidige poging;
herstart de Codex-app-server of OpenClaw Gateway zodat oude threads en
koppelingsregistraties worden verwijderd en probeer het daarna opnieuw in een
nieuwe sessie.

**Automatische installatie bij het begin van een beurt weigert een bron.**
Dit is opzettelijk. Voeg de bron eerst toe met een expliciete
`/codex computer-use install --source
<marketplace-source>`; daarna kan toekomstige automatische installatie bij het begin
van een beurt de gedetecteerde lokale marketplace gebruiken.

## Gerelateerd

- [Codex-harnas](/nl/plugins/codex-harness)
- [Peekaboo-bridge](/nl/platforms/mac/peekaboo)
- [iOS-app](/nl/platforms/ios)
