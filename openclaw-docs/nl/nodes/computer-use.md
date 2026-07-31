---
read_when:
    - De Gateway-agent een gekoppelde desktop laten bekijken en bedienen
    - Activering, machtigingen of veiligheid voor computergebruik
    - De node-opdracht computer.act of de uitvoerders ervan uitbreiden
summary: Desktopbediening op basis van mogelijkheden via de computertool en de node-opdracht computer.act
title: Computergebruik
x-i18n:
    generated_at: "2026-07-27T05:19:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df8ce87e607ce1b22d91e4ed8702d500bccd4d4f59dab7b0eafac565e730d48a
    source_path: nodes/computer-use.md
    workflow: 16
---

Met computergebruik kan de Gateway-agent een gekoppelde desktop met uitgebreide mogelijkheden zien en bedienen. Geschiktheid is gebaseerd op mogelijkheden: de verbonden Node moet zowel `computer.act` als `screen.snapshot` adverteren, waarbij het resultaat van laatstgenoemde een `displayFrameId` moet bevatten. De tool maakt een schermafbeelding als referentiekader en bestuurt vervolgens de aanwijzer en het toetsenbord via de gevaarlijke opdracht `computer.act`. De actieset volgt de kernacties voor computergebruik van Anthropic; optionele `computer_20251124`-zoom wordt niet beschikbaar gesteld. Een model met visuele mogelijkheden bestuurt dit via de ingebouwde agenttool `computer`.

De agent verzendt één uniforme opdracht, `computer.act`; de agent kan niet bepalen hoe een Node deze uitvoert. De meegeleverde macOS-app verwerkt de opdracht in hetzelfde proces met ingebedde Peekaboo-services en beperkte CoreGraphics-primitieven (juiste TCC-machtigingen, geen extra proces). Windows en Linux kunnen de optionele, experimentele Plugin `cua-computer` gebruiken met een afzonderlijk geïnstalleerd binair bestand `cua-driver`. Beide uitvoerders gebruiken hetzelfde beleid voor koppelen en activeren.

## Vereisten

- Een gekoppelde, verbonden Node die zowel `computer.act` als `screen.snapshot` adverteert, waarbij `screen.snapshot` `displayFrameId` retourneert.
- **macOS-uitvoerder:** appinstelling **Allow Computer Control** ingeschakeld (standaard: uit).
- **macOS-uitvoerder:** machtiging **Accessibility** verleend aan OpenClaw (voor invoer via aanwijzer/toetsenbord) en machtiging **Screen Recording** (voor `screen.snapshot`).
- **Windows/Linux-uitvoerder:** meegeleverde Plugin `cua-computer` ingeschakeld en een compatibel uitvoerbaar bestand `cua-driver` 0.10.x geïnstalleerd.
- De opdracht `computer.act` geactiveerd op de Gateway (deze is gevaarlijk en standaard gedeactiveerd).
- Een agentmodel met visuele mogelijkheden.
- Toolbeleid dat `computer` beschikbaar stelt. Het standaardprofiel `coding` doet dit niet. Voeg `computer` toe aan `tools.alsoAllow`; gesandboxte agents hebben dit ook nodig in `tools.sandbox.tools.alsoAllow`.

## De agenttool `computer`

De ingebouwde tool `computer` voert één actie per aanroep uit. Coördinaten zijn niet-negatieve gehele pixels in de meest recente schermafbeelding; de Node zet ze om naar beeldschermpunten. Coördinaatacties moeten de `frameId` uit het resultaat van de schermafbeelding herhalen en een expliciete `screenIndex` moet met dat kader overeenkomen. OpenClaw neemt ook een door de Node uitgegeven beeldschermidentiteit uit de schermafbeelding mee in de actie, zodat een opnieuw verbonden beeldscherm of gewijzigde geometrie veilig mislukt in plaats van dezelfde index stilzwijgend opnieuw als doel te gebruiken. Deze controles weigeren gegokte tokens en tokens uit een ander afgeleverd kader of beeldscherm. Een token garandeert geen actualiteit: apps kunnen pixels op hetzelfde beeldscherm na de opname wijzigen, dus maak een nieuwe schermafbeelding wanneer de scène mogelijk is gewijzigd.

- Lezen: `screenshot`.
- Aanwijzer: `left_click`, `right_click`, `middle_click`, `double_click`, `triple_click`, `mouse_move`, `left_click_drag` (met `startCoordinate`), `left_mouse_down`, `left_mouse_up`.
- Scrollen: `scroll` met `scrollDirection` (`up|down|left|right`) en `scrollAmount` (muiswielstappen).
- Toetsenbord: `type` (tekst), `key` (combinatie zoals `cmd+shift+t` of `Return`), `hold_key` (`text`-combinatie gedurende `duration` seconden ingedrukt).
- Tempo: `wait` (`duration` seconden).

Modificatietoetsen worden via het veld `text` doorgegeven bij klik- en scrollacties (`shift`, `ctrl`, `alt`, `cmd`). Na een invoeractie retourneert de tool een nieuwe schermafbeelding, zodat het model het resultaat kan waarnemen. Als er meer dan één Node met computerbesturingsmogelijkheden is verbonden, geef je `node` expliciet door.

Schermafbeeldingen blijven **alleen voor het model**: ze worden nooit automatisch aan het chatkanaal geleverd. Behandel alle inhoud op het scherm als niet-vertrouwde invoer; de tool waarschuwt het model om geen instructies op het scherm te volgen die strijdig zijn met het verzoek van de gebruiker.

## Windows en Linux (experimenteel, via cua-driver)

De meegeleverde Plugin `cua-computer` biedt een experimentele uitvoerder voor Windows- en Linux-Node-hosts. Deze is standaard uitgeschakeld en vereist het prerelease-drivercontract 0.10.x:

1. Installeer een binair bestand `cua-driver` 0.10.x uit de [upstream-releases](https://github.com/trycua/cua/releases) en zorg dat dit beschikbaar is op `PATH`. Stel `plugins.entries.cua-computer.config.driverPath` in om een andere locatie voor het uitvoerbare bestand te gebruiken.
2. Schakel de Plugin in:

   ```bash
   openclaw plugins enable cua-computer
   ```

3. Start `openclaw node run` vanuit de interactieve desktopsessie. De Plugin start de lokale driverdaemon uitgesteld wanneer de eerste opname of actie binnenkomt.

Deze uitvoerder bestuurt momenteel alleen het primaire beeldscherm. X11/XWayland is de voorkeursroute voor Linux. Native Wayland blijft een upstream-opt-in: stel `CUA_DRIVER_RS_ENABLE_WAYLAND` zelf in voordat je de Node start; OpenClaw stelt dit nooit automatisch in. KDE/KWin wordt niet ondersteund door het upstream-invoerpad voor native Wayland. `hold_key`, `left_mouse_down` en `left_mouse_up` zijn niet beschikbaar omdat cua-driver 0.10.x geen platformoverschrijdend contract voor ingedrukte invoer op desktopniveau heeft. Scrollen en slepen met ingedrukte modificatietoetsen zijn op beide platforms niet beschikbaar, en klikken met ingedrukte modificatietoetsen is niet beschikbaar op Linux. De actie `key` accepteert benoemde toetsen, letters en modificatietoetscombinaties (bijvoorbeeld `cmd+c` of `Return`); cijfer- en leestekentoetsen worden geweigerd omdat de driver hun indelingsafhankelijke Shift-status laat vallen, dus verzend die tekst in plaats daarvan via de actie `type`. Het typen van tekst kan niet halverwege een driveraanroep `type_text` worden geannuleerd.

Omdat cua-driver geen stabiele beeldschermidentiteit rapporteert, wordt kaderautorisatie gekoppeld aan de driververbinding plus de actuele geometrie van het primaire beeldscherm. Een opnieuw verbonden daemon of sessie maakt openstaande kaders ongeldig, maar vervanging van het primaire beeldscherm door een beeldscherm met dezelfde geometrie terwijl de verbinding open blijft, kan niet worden gedetecteerd; gebruik bij voorkeur een stabiele sessie met één beeldscherm voor deze uitvoerder.

OpenClaw schakelt telemetrie en updatecontroles van cua-driver uit voor de processen `mcp` en `serve` die het beheert. Het downloadt of actualiseert het binaire driverbestand niet.

### Probleemoplossing

De uitvoerder `cua-computer` geeft getypeerde foutcodes weer in het toolresultaat en de Node-logboeken. Veelvoorkomende codes:

| Code                                                 | Oorzaak                                                                                                                                                           | Oplossing                                                                                                                                                                                                                                  |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `COMPUTER_DRIVER_UNAVAILABLE`                        | Het binaire bestand `cua-driver` staat niet op `PATH` (of `driverPath` is onjuist), de daemon was niet op tijd gereed of de Node gebruikt geen Windows/Linux.                 | Installeer `cua-driver` 0.10.x op `PATH` of stel `driverPath` in. Voer `openclaw node run` uit binnen de interactieve desktopsessie; zorg op Linux dat er een X11-`DISPLAY` (of een `WAYLAND_DISPLAY` met `CUA_DRIVER_RS_ENABLE_WAYLAND`) aanwezig is. |
| `COMPUTER_DRIVER_UNSUPPORTED`                        | De verbonden driver is niet `cua-driver` 0.10.x of de versie van de mogelijkheden/het schema wijkt af.                                                                      | Installeer een ondersteunde 0.10.x-build. De Plugin voert ongeveer 30 seconden nadat je dit hebt gecorrigeerd opnieuw een detectie uit, zodat de Node niet opnieuw hoeft te worden gestart.                                                                                                          |
| `COMPUTER_REFUSED_<code>`                            | De driver heeft de actie geweigerd met een gestructureerde code zoals `background_unavailable`, `background_occluded` of `foreground_unavailable` (KDE/KWin Wayland).   | Breng het doelvenster naar de voorgrond, schakel over naar X11 of gebruik een ondersteunde compositor. Zie de compatibiliteitsopmerkingen hierboven.                                                                                                                    |
| `COMPUTER_STALE_FRAME`                               | De coördinaten verwezen naar een schermafbeelding die niet meer actueel is (contextcompactie, een wijziging in de beeldschermgeometrie of een wijziging in de referentiebreedte).                 | Maak vóór de coördinaatactie een nieuwe `screenshot`.                                                                                                                                                                              |
| `COMPUTER_UNSUPPORTED_ACTION`                        | Een actie die deze uitvoerder niet getrouw kan uitvoeren: `hold_key`, `left_mouse_down`, `left_mouse_up`, slepen/scrollen met ingedrukte modificatietoets of klikken met ingedrukte modificatietoets op Linux. | Gebruik een ondersteunde actie. cua-driver 0.10.x heeft geen contract voor ingedrukte invoer op desktopniveau.                                                                                                                                                  |
| `COMPUTER_UNSUPPORTED_DISPLAY`                       | Een niet-primair `screenIndex`, een verschil tussen de opname- en schermgeometrie of een cursor buiten het primaire beeldscherm.                                                       | Bedien alleen het primaire beeldscherm.                                                                                                                                                                                                      |
| `COMPUTER_UNSUPPORTED_KEY`                           | Een `key`-waarde die de driver niet betrouwbaar kan reproduceren: een cijfer- of leestekentoets waarvan de Shift-status afhankelijk is van de indeling, of een onbekende toets.                        | Verzend die tekst in plaats daarvan via de actie `type`.                                                                                                                                                                                    |
| `COMPUTER_DRIVER_ERROR` / `COMPUTER_INVALID_REQUEST` | De driver is zonder gestructureerde code mislukt of de actieargumenten waren onjuist opgebouwd.                                                                            | Controleer de driverstatus en maak opnieuw een schermafbeelding; corrigeer de actieargumenten.                                                                                                                                                        |

## De Node-opdracht `computer.act`

`computer.act` is de enige Node-opdracht waarlangs de tool invoer routeert (`node.invoke` met `command: "computer.act"`). Deze is:

- **Standaard gevaarlijk**: opgenomen in de ingebouwde gevaarlijke Node-opdrachten en uitgesloten van de runtime-toelatingslijst totdat deze expliciet wordt geactiveerd. Desktop-Nodes voor macOS, Windows en Linux mogen deze nog steeds tijdens het koppelen declareren, zodat het oppervlak één keer wordt goedgekeurd.
- **Gebaseerd op mogelijkheden**: de tool vereist dat een verbonden Node zowel `computer.act` als `screen.snapshot` adverteert. De meegeleverde macOS-app en de experimentele opt-in-Plugin `cua-computer` voeren hetzelfde opdrachtenpaar uit.

Leesacties hergebruiken `screen.snapshot`; er is geen tweede opnamepad. Zie [Camera- en scherm-Nodes](/nl/nodes/camera) voor de gedeelde opnameopdracht.

## Inschakelen en activeren

1. Schakel de platformuitvoerder in: schakel op macOS **Settings → Allow Computer Control** in en verleen vervolgens **Accessibility** en **Screen Recording** onder **Settings → Permissions**; volg op Windows/Linux de experimentele configuratie voor `cua-computer` hierboven.
2. Keur de koppelingsupdate op de Gateway goed (een nieuwe opdracht dwingt opnieuw koppelen af).
3. Stel de tool beschikbaar aan de agent met visuele mogelijkheden. Voor het standaardprofiel `coding`:

   ```json5
   {
     tools: {
       alsoAllow: ["computer"],
       // Agents in een sandbox hebben deze tweede controle ook nodig:
       sandbox: { tools: { alsoAllow: ["computer"] } },
     },
   }
   ```

4. Activeer `computer.act` voor een begrensde periode. De Plugin `phone-control` stelt een groep `computer` beschikbaar:

   ```text
   /phone arm computer 30m
   /phone status
   /phone disarm
   ```

   Voor activering is `operator.admin` (of de eigenaar) vereist en de activering verloopt automatisch. De verouderde groep `/phone arm all` sluit desktopbesturing bewust uit; gebruik de expliciete groep `computer`. Activering bepaalt alleen wat de Gateway mag aanroepen; de Node-app blijft de platformspecifieke instellingen en OS-machtigingen afdwingen, waaronder **Allow Computer Control**, Accessibility en Screen Recording op macOS.

Voeg voor permanente autorisatie `computer.act` toe aan `gateway.nodes.commands.allow` **en verwijder het uit** `gateway.nodes.commands.deny`; de weigeringslijst heeft voorrang. Permanente autorisatie verloopt niet automatisch. Vermeldingen die al vóór `/phone arm` aanwezig waren, blijven na `/phone disarm` behouden; zet een tijdelijke toekenning niet om in een permanente zolang deze actief is.

Autorisatie is bewust opgesplitst in inschakeling en gebruik. Voor het activeren of
permanent configureren van `computer.act` is administratieve bevoegdheid vereist.
Na activering kan een geverifieerde operator met `operator.write`
`computer.act` via `node.invoke` aanroepen totdat de toekenning verloopt of wordt gedeactiveerd;
er is geen beheerderscontrole per actie. Het goedkeuren van een Node die
`computer.act` declareert, registreert alleen het oppervlak zodat dit later kan worden geactiveerd en
schakelt de aanroep niet zelfstandig in.

## Veiligheid

- Vóór autorisatie moeten alle lagen (toolbeleid, opdrachtbeleid van de Gateway, instelling van de Node-app en platformmachtigingen) overeenstemmen. Voor de huidige macOS-uitvoerder omvat dit **Allow Computer Control**, Accessibility en Screen Recording. Na activering worden acties zonder bevestiging per actie uitgevoerd totdat de activering verloopt of `/phone disarm` plaatsvindt.
- De macOS-uitvoerder plaatst tekst grafeem voor grafeem, zodat annulering, verbreking van de verbinding, pauzering, uitschakeling of vervanging van het eindpunt het proces vóór het volgende grafeem stopt. De experimentele cua-driver-uitvoerder kan een aanroep van `type_text` niet tijdens het typen annuleren.
- Schermafbeeldingen zijn uitsluitend voor het model en worden nooit automatisch naar de chat verzonden (issue [#44759](https://github.com/openclaw/openclaw/issues/44759)).
- Behandel scherminhoud als onvertrouwd; deze kan promptinjectie bevatten.

## Relatie tot andere methoden voor desktopbesturing

Dit is de door de agent aangestuurde methode. Zie [Peekaboo-bridge](/nl/platforms/mac/peekaboo) voor de relatie met de PeekabooBridge-host, Codex Computer Use en de directe MCP `cua-driver`.
