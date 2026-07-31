---
read_when:
    - De macOS-app installeren
    - Kiezen tussen lokale en externe Gateway-modus op macOS
    - Op zoek naar downloads van releases van de macOS-app
summary: Installeer en gebruik de OpenClaw-menubalkapp voor macOS
title: macOS-app
x-i18n:
    generated_at: "2026-07-27T06:22:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b319d72bcbffcf91b6bc012d352c2cf647abd66e08ab0146cf98f5edfae3bca1
    source_path: platforms/macos.md
    workflow: 16
---

De macOS-app is de **menubalkassistent** van OpenClaw: een systeemeigen menu-interface, macOS-
toestemmingsverzoeken, meldingen, WebChat, spraakinvoer, Canvas en
op de Mac gehoste Node-tools zoals `system.run`.

Gebruik **Snelle chat** voor een Spotlight-achtige editor voor de hoofdsessie zonder een volledig venster te openen. Druk standaard op Option-Space (⌥Space), kies deze optie in het menubalkmenu of stel een andere sneltoets in via **Instellingen → Algemeen**.

Alleen de CLI en Gateway nodig? Begin met [Aan de slag](/nl/start/getting-started).

## Downloaden

Download builds van de macOS-app via [OpenClaw-releases op GitHub](https://github.com/openclaw/openclaw/releases).
Als een release bestanden voor de macOS-app bevat, zoek dan naar:

- `OpenClaw-<version>.dmg` (aanbevolen)
- `OpenClaw-<version>.zip`

Sommige releases bevatten alleen bestanden voor de CLI, bewijsmateriaal of Windows. Als de nieuwste release
geen bestand voor de macOS-app bevat, gebruik dan de nieuwste release die dat wel heeft of bouw vanuit de broncode met
[macOS-ontwikkelomgeving](/nl/platforms/mac/dev-setup).

## Eerste gebruik

1. Installeer en start **OpenClaw.app**.
2. Kies **Deze Mac** voor een lokale Gateway of maak verbinding met een externe Gateway.
3. Wacht terwijl de app de bijbehorende CLI-runtime installeert. In de lokale modus
   installeert en start de app ook de Gateway.
4. Breng inferentie tot stand met een live modelcontrole. Nadat deze is geslaagd, handelt OpenClaw
   de resterende configuratie af.
5. Voltooi de controlelijst met macOS-toestemmingen en verstuur het testbericht voor de ingebruikname.

Als de app een bestaande Gateway bereikt waarvan de standaardagent een geconfigureerd
model heeft, beschouwt de app die Gateway als reeds geconfigureerd, slaat deze de ingebruikname van de provider en
OpenClaw over en opent deze het dashboard. Als de Gateway geen verbinding kan maken of de
standaardagent geen model heeft, blijft de ingebruikname voor inferentie beschikbaar voor
herstel.

Gebruik [Aan de slag](/nl/start/getting-started) voor het configuratiepad van de CLI/Gateway.
Gebruik [macOS-toestemmingen](/nl/platforms/mac/permissions) voor herstel van toestemmingen.

## Updates

De updatekaart van het dashboard vermeldt wat de app zal bijwerken:

- **Mac-app + Gateway bijwerken** betekent dat de ondertekende app eigenaar is van de lokale, door launchd beheerde
  Gateway. Sparkle werkt eerst de app bij; na het opnieuw starten werkt de app automatisch
  de Gateway bij naar de bijbehorende versie, start deze opnieuw en controleert vervolgens de
  verbinding.
- **Gateway bijwerken** betekent dat de app is verbonden met een externe Gateway, een handmatig
  beheerde lokale Gateway of een andere installatie waarvan de app geen eigenaar is. De knop
  voert de normale updatestroom van die Gateway uit in plaats van de Mac-app te wijzigen.

Een mislukte gecoördineerde update blijft in het configuratievenster staan met opties om het opnieuw te proberen,
de [updatehandleiding](/nl/install/updating) te openen en Discord-acties uit te voeren. Automatisch herstel
downgradet nooit een nieuwere Gateway en overschrijft nooit een kanaalpin voor `extended-stable`.

Na een geslaagde update zoekt de app de meest recent door een mens gebruikte,
rechtstreekse sessie op het hoogste niveau en stuurt die agent een eenmalige updategebeurtenis. Heartbeat-
en Cron-activiteit hebben geen invloed op deze keuze. De agent kan je vervolgens weer welkom heten
vanuit het gesprek dat je waarschijnlijk het laatst gebruikte. In de externe modus
werkt de app alleen de lokale runtime van de Mac-Node bij en slaat deze de melding over wanneer de
externe Gateway ouder is dan de app.

Sparkle volgt de instelling `update.channel` van de Gateway. `beta` en `dev` schakelen
bètaversies van de app in; `stable`, `extended-stable` en ontbrekende of onbekende waarden
blijven stabiele versies van de app gebruiken.

## Dashboardlinks openen

Als je in het ingebedde dashboard van de macOS-app op een externe weblink klikt, wordt deze geopend in een zijbalk met browser waarvan de grootte kan worden aangepast en die de helft van de vensterbreedte beslaat, terwijl de dashboardnavigatie zichtbaar blijft. Sleep de scheidingslijn om een andere breedte te kiezen; de app onthoudt deze. Elke link wordt in een eigen tabblad geopend, de tabbladbalk verschijnt wanneer meerdere pagina's geopend zijn en als je opnieuw op dezelfde link klikt, wordt het bestaande tabblad hergebruikt. Sleep tabbladen om ze opnieuw te ordenen, sluit ze met de sluitknop van het tabblad of met een middelklik en klik met de rechtermuisknop op een tabblad voor **Openen in standaardbrowser**, **Link kopiëren**, **Opnieuw laden**, **Tabblad sluiten** en **Andere tabbladen sluiten**. Met de knoppen voor terug en vooruit in de titelbalk van het venster en veegbewegingen op het trackpad navigeer je door de geschiedenis van het dashboard; met de eigen knoppen voor terug en vooruit van de zijbalk navigeer je door de geschiedenis van het actieve tabblad. De zijbalk bevat ook knoppen voor opnieuw laden, openen in de standaardbrowser en sluiten.

De knoppen in de titelbalk volgen de appzijbalk: als deze is uitgeklapt, staan terug en vooruit aan de rechterrand naast de schakelknop voor de zijbalk; als deze is ingeklapt, maken ze plaats voor een zoekknop (opent het opdrachtenpalet) en een knop voor een nieuwe sessie.

Klik met de rechtermuisknop op een externe link om **Openen in zijbalk**, **Openen in standaardbrowser** of **Link kopiëren** te kiezen. Klikken met modificatietoetsen en door de gebruiker geactiveerde koppelingen voor een nieuw venster vanuit het dashboard blijven in de standaardbrowser openen; koppelingen voor een nieuw venster in de zijbalk worden als nieuwe zijbalktabbladen geopend. Gewone pagina's van de Control UI die in een browser worden gehost, behouden het normale gedrag van de browser voor links en contextmenu's.

## Browseraanmeldingen importeren

De eerste keer dat de browserzijbalk wordt geopend terwijl de app met een lokale Gateway werkt, toont het dashboard een sluitbare banner als er op de Mac een profiel van een Chrome-achtige browser met cookies bestaat. De banner biedt aan om die cookies te kopiëren naar een geïsoleerd beheerd profiel dat agents gebruiken om te browsen. Kies een profiel via het bedieningselement **Import** (Touch ID kan vereist zijn); de voortgang en het aantal geïmporteerde cookies worden direct weergegeven, en alleen cookies worden gekopieerd — wachtwoorden verlaten de bronbrowser nooit. Als je de banner sluit, wordt die keuze vastgelegd; via **Instellingen → Algemeen → Browseraanmelding → Importeren…** kun je de optie op elk gewenst moment opnieuw tonen. Zie [Browser](/nl/cli/browser) voor de onderliggende importstroom en de `browser.allowSystemProfileImport`-poort.

## Een Gateway-modus kiezen

| Modus   | Gebruik deze wanneer                                                          | Detailpagina                                       |
| ------- | ----------------------------------------------------------------------------- | ------------------------------------------------- |
| Lokaal  | Deze Mac de Gateway moet uitvoeren en actief moet houden met launchd.         | [Gateway op macOS](/nl/platforms/mac/bundled-gateway) |
| Extern  | Een andere host de Gateway uitvoert; deze Mac deze bestuurt via SSH, LAN of Tailnet. | [Besturing op afstand](/nl/platforms/mac/remote) |

Beide modi vereisen een geïnstalleerde `openclaw`-CLI, omdat de app de runtime van de Node-host
hergebruikt. Op een nieuwe Mac installeert de app automatisch de bijbehorende CLI; in de lokale
modus wordt vervolgens de Gateway-wizard gestart, terwijl de externe modus verbinding maakt met de geselecteerde
Gateway zonder een tweede lokale Gateway te starten.
Zie [Gateway op macOS](/nl/platforms/mac/bundled-gateway) voor handmatig herstel.

## Waarvoor de app verantwoordelijk is

- Menubalkstatus, meldingen, statuscontrole, WebChat en de zwevende balk voor Snelle chat.
- macOS-toestemmingsverzoeken voor scherm, microfoon, spraak, automatisering en toegankelijkheid.
- Eén Mac-Node die systeemeigen Canvas, camera-/schermopname, meldingen,
  locatie en computerbesturing combineert met de systeem-, browser-,
  Plugin-, Skill- en MCP-opdrachten van de CLI-Node-host.
- Verzoeken om uitvoering goed te keuren voor op de Mac gehoste opdrachten.
- Uitvoering binnen de appcontext voor goedgekeurde shellopdrachten, waarbij de macOS-
  toewijzing van toestemmingen aan de app behouden blijft terwijl de CLI-runtime eigenaar is van het gedeelde Node-beleid.
- SSH-tunnels in de externe modus of rechtstreekse Gateway-verbindingen.

In de ingebedde Control UI toont **Instellingen → Meldingen** de systeemeigen
meldingstoestemming van de app in plaats van browserpushmeldingen, omdat de app meldingen systeemeigen aflevert.

De app vervangt de documentatie voor de Gateway of de algemene CLI **niet**. De configuratie van de Gateway,
providers, plugins, kanalen, tools en beveiliging worden in hun
eigen documentatie beschreven.

## macOS-detailpagina's

| Taak                                     | Lezen                                                                                       |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| De CLI/Gateway-service installeren of debuggen | [Gateway op macOS](/nl/platforms/mac/bundled-gateway)                                    |
| Toestand buiten met de cloud gesynchroniseerde mappen houden | [Gateway op macOS](/nl/platforms/mac/bundled-gateway#state-directory-on-macos)    |
| Appdetectie en connectiviteit debuggen   | [Gateway op macOS](/nl/platforms/mac/bundled-gateway#debug-app-connectivity)                   |
| Gedrag van launchd begrijpen             | [Levenscyclus van de Gateway](/nl/platforms/mac/child-process)                                 |
| Problemen met toestemmingen, ondertekening of TCC oplossen | [macOS-toestemmingen](/nl/platforms/mac/permissions)                       |
| De Mac detecteren die je het laatst hebt gebruikt | [Aanwezigheid van actieve computer](/nl/nodes/presence)                               |
| Verbinding maken met een externe Gateway | [Besturing op afstand](/nl/platforms/mac/remote)                                               |
| Menubalkstatus en statuscontroles bekijken | [Menubalk](/nl/platforms/mac/menu-bar), [Statuscontroles](/nl/platforms/mac/health)              |
| De ingebedde chatinterface gebruiken     | [WebChat](/nl/platforms/mac/webchat)                                                           |
| Activering met stem of indrukken-om-te-praten gebruiken | [Stemactivering](/nl/platforms/mac/voicewake)                                      |
| Canvas en Canvas-deeplinks gebruiken     | [Canvas](/nl/platforms/mac/canvas)                                                             |
| PeekabooBridge hosten voor UI-automatisering | [Peekaboo-brug](/nl/platforms/mac/peekaboo)                                                |
| Goedkeuringen voor opdrachten configureren | [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals), [geavanceerde details](/nl/tools/exec-approvals-advanced) |
| Opdrachten van de Mac-Node en IPC van de app inspecteren | [macOS-IPC](/nl/platforms/mac/xpc)                                      |
| Logboeken vastleggen                     | [macOS-logboekregistratie](/nl/platforms/mac/logging)                                          |
| Vanuit de broncode bouwen                | [macOS-ontwikkelomgeving](/nl/platforms/mac/dev-setup)                                         |

## Gerelateerd

- [Platformen](/nl/platforms)
- [Aan de slag](/nl/start/getting-started)
- [Gateway](/nl/gateway)
- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals)
