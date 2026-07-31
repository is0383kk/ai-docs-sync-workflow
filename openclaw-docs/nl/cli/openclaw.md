---
read_when:
    - Je hebt de inferentie ingesteld en wilt dat OpenClaw de rest configureert
    - Je moet OpenClaw inspecteren of repareren met de lokale installatieagent
    - Je ontwerpt of activeert de reddingsmodus voor berichtkanalen
summary: CLI-referentie en beveiligingsmodel voor de door inferentie ondersteunde OpenClaw-helper voor installatie en herstel
title: OpenClaw-installatieagent
x-i18n:
    generated_at: "2026-07-27T05:28:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw wordt geleverd met een ingebouwde systeemagent — deze spreekt als "OpenClaw" — voor
lokale installatie, reparatie en configuratie (voorheen Crestodian genoemd). Deze start pas nadat het effectieve standaardmodel een echte beurt heeft voltooid.
Bij nieuwe installaties wordt eerst inferentie ingesteld; een ongeldige configuratie blijft het
klassieke doctor-pad volgen.

## Wanneer deze start

Het uitvoeren van `openclaw` zonder subcommando kiest een route op basis van de configuratiestatus:

- Configuratie ontbreekt of bestaat zonder door de gebruiker opgegeven instellingen (leeg of alleen de sleutels `$schema`/`meta`): start begeleide onboarding met live AI-verificatie.
- Configuratie bestaat maar slaagt niet voor validatie: start klassieke onboarding, die de problemen meldt en je doorverwijst naar `openclaw doctor`.
- Configuratie bestaat en is geldig: opent de normale agent-TUI. Een bereikbare
  geconfigureerde Gateway waarvan de standaardagent een model heeft, gaat rechtstreeks naar die UI
  zonder onboarding of OpenClaw. Gebruik `/openclaw` in de TUI of voer
  `openclaw setup` rechtstreeks uit om OpenClaw later te openen.

Bij het uitvoeren van `openclaw setup` wordt eerst het geconfigureerde standaardmodel live getest. Na een geslaagde beurt start OpenClaw. Bij een interactieve fout wordt de begeleide inferentie-instelling geopend en wordt na goedkeuring van een kandidaat overgeschakeld naar OpenClaw. Eenmalige, JSON- en andere niet-interactieve verzoeken mislukken met instructies om `openclaw onboard` uit te voeren wanneer inferentie niet beschikbaar is. `openclaw --help` en `openclaw --version` behouden hun normale snelle paden.

Niet-interactief uitvoeren van alleen `openclaw` (zonder TTY) sluit af met een kort bericht in plaats van de hoofdhulp weer te geven: het verwijst naar niet-interactieve onboarding bij een nieuwe of ongeldige installatie, of naar `openclaw agent --local ...` wanneer de configuratie geldig is.

`openclaw onboard --modern` blijft een compatibiliteitsalias voor OpenClaw, maar gebruikt dezelfde inferentiecontrole: werkende inferentie opent de chat, interactieve fouten starten de begeleide inferentie-instelling en niet-interactieve fouten sluiten af met onboardinginstructies. `openclaw onboard --classic` opent de volledige stapsgewijze wizard.

## Wat OpenClaw toont

Interactieve OpenClaw opent dezelfde TUI-shell als `openclaw tui`, met een OpenClaw-chatbackend. De welkomsttekst bij het starten behandelt:

- de geldigheid van de configuratie en de standaardagent
- het geverifieerde model dat OpenClaw gebruikt
- de bereikbaarheid van de Gateway volgens de eerste controle bij het starten
- de volgende aanbevolen foutopsporingsactie

Er worden geen geheimen weergegeven en er worden niet uitsluitend voor het starten CLI-commando's van plugins geladen.

Gebruik `status` voor de gedetailleerde inventaris: het configuratiepad, documentatie-/bronpaden, lokale CLI-controles, de aanwezigheid van sleutels/tokens, agents, het model en Gateway-details.

OpenClaw gebruikt dezelfde referentiedetectie als reguliere agents: in een Git-checkout verwijst het naar de lokale `docs/` en de broncodeboom; in een npm-installatie gebruikt het gebundelde documentatie en verwijst het naar [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw), met het advies om de broncode te raadplegen wanneer de documentatie niet volstaat.

## Voorbeelden

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "models"
openclaw setup --message "validate config"
openclaw setup --message "setup workspace ~/Projects/work" --yes
openclaw setup --message "set default model openai/gpt-5.6" --yes
openclaw onboard --modern
```

In de OpenClaw-TUI:

```text
status
health
doctor
configuratie valideren
installatie
werkruimte ~/Projects/work instellen
config set gateway.port 19001
config set-ref gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
gateway-status
gateway opnieuw starten
agents
agent work maken met werkruimte ~/Projects/work
modellen
modelprovider configureren
standaardmodel instellen op openai/gpt-5.6
kanalen
kanaalinfo slack
slack verbinden
kanaalwizard voor slack openen
plugins weergeven
plugins zoeken slack
plugin install clawhub:openclaw-codex-app-server
praten met agent work
praten met agent voor ~/Projects/work
audit
afsluiten
```

## Bewerkingen en goedkeuring

OpenClaw gebruikt getypeerde bewerkingen in plaats van de configuratie ad hoc te bewerken.

Alleen-lezenbewerkingen worden onmiddellijk uitgevoerd: overzicht tonen, agents weergeven, geïnstalleerde plugins weergeven, ClawHub-plugins zoeken, model-/backendstatus tonen, status-/gezondheidscontroles uitvoeren, bereikbaarheid van de Gateway controleren, doctor uitvoeren zonder interactieve reparaties, configuratie valideren en het pad van het auditlogboek tonen.

Het starten van begeleide kanaalinstelling (`connect telegram`) wordt ook onmiddellijk uitgevoerd. De wizard verzamelt expliciete antwoorden en beheert de resulterende schrijfbewerkingen.

Permanente bewerkingen vereisen goedkeuring in het gesprek (of `--yes` voor een rechtstreeks commando): configuratie schrijven, `config set`, `config set-ref`, bootstrap van installatie/onboarding, het standaardmodel wijzigen, de Gateway starten/stoppen/opnieuw starten, agents maken en plugins installeren.

Doctor-reparaties zijn niet beschikbaar in OpenClaw, omdat ze de provider, authenticatie of inferentieroute van de standaardagent waarop de sessie draait, kunnen herschrijven. Sluit OpenClaw af en voer `openclaw doctor --fix` uit in een terminal. Alleen-lezen `doctor` blijft beschikbaar in OpenClaw.

Nieuwe agents nemen de live geverifieerde standaardinferentieroute over. De agent-id's `openclaw` en `crestodian` zijn gereserveerd voor de systeemagent en kunnen niet als normale agents worden gemaakt. De buiten gebruik gestelde id blijft geblokkeerd, zodat een oude configuratie deze niet kan opeisen.

`config set` en `config set-ref` kunnen elke instelling wijzigen die een gebruiker kan wijzigen,
met een korte, uitsluitend voor mensen geldende blokkeerlijst: `$include`, `auth.*`, `env.*`, `models.*`
en `secrets.*` blijven geweigerd, omdat ze referentiegegevens,
opname van alternatieve configuraties of de provider-/catalogusdefinities bevatten die
de inferentieroutering aansturen. De inferentieroutering zelf is ook beschermd: routes van het standaardmodel
(model-/parameter-/runtimevelden van `agents.defaults`) en de routeringsvelden
van de agent die de actieve standaardroute ondersteunt, worden geweigerd, evenals velden voor
agentidentiteit/-topologie (`id`, `agentDir`, `default`). Routeringsvelden voor
andere agents blijven na goedkeuring beschrijfbaar. Gateway- en kanaalauthenticatie blijven
normale configuratieoppervlakken. Gebruik `set default model <provider/model>` voor een
reeds geconfigureerde route; de route wordt live getest voordat deze wordt opgeslagen. Om
provider-/authenticatietoegang te configureren of repareren, sluit je OpenClaw af en voer je
`openclaw onboard` uit.

Schrijfbewerkingen via `plugins.entries.<id>.*` (inschakelen/uitschakelen/configureren van geïnstalleerde plugins)
zijn toegestaan, tenzij die plugin de actieve inferentieroute ondersteunt. Installatiebronnen
en laadbeleid van plugins behouden hun vertrouwensgrens in de getypeerde
plugininstallatieworkflow. Het verwijderen van de plugin die de route ondersteunt, wordt
om dezelfde reden geweigerd; sluit OpenClaw af en voer
`openclaw plugins uninstall <id>` uit vanuit een terminal.

Je geeft goedkeuring in je eigen woorden: ondubbelzinnige antwoorden ("ja", "zeker", "ga je gang", "nu niet") worden herkend aan de hand van een gesloten deterministische lijst. Wanneer de geconfigureerde route een afzonderlijke voltooiingsaanroep ondersteunt, kunnen andere antwoorden uitsluitend op basis van jouw bericht en het openstaande voorstel worden geclassificeerd — nooit door het gespreksmodel zelf, dat zichzelf niet kan goedkeuren. Niet-geclassificeerde of dubbelzinnige antwoorden laten het voorstel openstaan en er wordt in het gesprek opnieuw om goedkeuring gevraagd.

### Wijzigingsgeschiedenis

De pagina Ask OpenClaw kan recente toegepaste bewerkingen van de systeemagent, Doctor-
migraties, configuratieschrijfbewerkingen via Settings en de CLI en handmatige bewerkingen van
`openclaw.json` tonen. Het configuratiejournaal detecteert externe bewerkingen terwijl de Gateway
de configuratie bewaakt, tijdens een schrijfbewerking van OpenClaw of bij de volgende start na een
offline bewerking.

De geschiedenis wordt opgeslagen in de tabel `diagnostic_events` van de gedeelde
database `~/.openclaw/state/openclaw.sqlite`, binnen de bereiken `system-agent-audit`
en `config-audit`. Elk bereik bewaart de nieuwste 50,000 records.
Detectie- en alleen-lezenbewerkingen worden niet opgenomen. Geheimen verschijnen nooit in
de wijzigingsgeschiedenis; records in het configuratiejournaal bevatten gewijzigde paden in plaats van configuratie-
waarden en waarden worden vergeleken met behulp van beschermde vingerafdrukken.

Kanaalinstelling kan als gehost gesprek worden uitgevoerd totdat een geheim moet worden ingevoerd. De
lokale OpenClaw-TUI accepteert geen gevoelige antwoorden in de wizard, omdat chatinvoer in de terminal
zichtbaar is. Deze biedt onmiddellijk `open channel wizard` aan, waarbij
het geselecteerde kanaal wordt overgedragen naar de gemaskeerde terminalwizard; je kunt ook later
`openclaw channels add --channel <channel>` uitvoeren.

### Overschakelen naar gemaskeerde kanaalinstelling

De lokale chat kan de besturing overdragen aan de gemaskeerde kanaalwizard:

```text
kanaalwizard voor slack openen
kanaalinfo slack
```

`open channel wizard for <channel>` opent de gemaskeerde kanaalinstelling nadat de chat-
TUI is gesloten. Gebruik eerst `channel info <channel>` voor het kanaallabel, de instellings-
status, een samenvatting van de vereisten en de documentatielink.

OpenClaw wijzigt provider-/authenticatietoegang nooit vanuit zijn eigen sessie: de
sessie is al afhankelijk van die inferentieroute. Voor het instellen of
repareren van de modelprovider retourneert `configure model provider` instructies om af te sluiten/onboarding uit te voeren zonder
een wizard te starten of de configuratie te schrijven. Sluit OpenClaw af en voer `openclaw
onboard` uit; onboarding bereidt de referentiegegevens voor en slaat alleen een route op die
een echte live beurt voltooit. Start OpenClaw opnieuw nadat onboarding is geslaagd.

## Bootstrap van de installatie

`setup` configureert de resterende werkruimte- en Gateway-status nadat begeleide onboarding de inferentie al heeft ingesteld. Er wordt uitsluitend via getypeerde configuratiebewerkingen geschreven en eerst om goedkeuring gevraagd.

```text
installatie
werkruimte ~/Projects/work instellen
```

`setup` behoudt het geverifieerde effectieve model. Deze configureert of
vervangt inferentie niet.

Als inferentie ontbreekt of de live controle ervan mislukt, sluit je OpenClaw af en voer je `openclaw onboard` uit. Begeleide onboarding probeert eerst het geconfigureerde model, daarna geauthenticeerde abonnements-CLI's, API-sleutels en de overige ondersteunde CLI's; elke kandidaat wordt om een echt antwoord gevraagd en alleen een geslaagde route wordt opgeslagen. OpenClaw start onmiddellijk na die grens en kan vervolgens de werkruimte, Gateway, kanalen, agents, plugins en andere optionele functies configureren.

De macOS-app slaat deze reeks volledig over wanneer deze een geconfigureerde Gateway
bereikt waarvan de standaardagent al een geconfigureerd model heeft; de normale agent-
UI wordt geopend.
Voor een nieuwe of onvolledige Gateway doorloopt de app de inferentiereeks via
de Gateway-methoden `openclaw.setup.detect` en `openclaw.setup.activate`:
detecteren geeft elke gevonden kandidaat-backend weer, activeren test één
kandidaat live (een echte voltooiing met "antwoord met OK") en slaat pas nadat de test is geslaagd het model,
de referentiegegevens en de provider-/runtimestatus op die voor die route nodig zijn. De standaardwaarden voor werkruimte en Gateway blijven voor OpenClaw bestemd. Een mislukte kandidaat
wijzigt de configuratie nooit; de app doorloopt de reeks automatisch en biedt uiteindelijk
een handmatige sleutel-/tokenstap aan, ingevuld op basis van de actieve
tekstinferentieproviderplugins van de Gateway. De geselecteerde provider beheert het bijbehorende startmodel
en de configuratie, en de referentiegegevens worden op dezelfde manier geverifieerd voordat ze worden opgeslagen.

Codex-supervisie en andere optionele pluginfuncties blijven buiten deze
inferentieactiveringstransactie. Configureer ze pas nadat inferentie
werkt en OpenClaw is gestart; bestaand pluginbeleid en expliciete
afmeldingen voor supervisie blijven tijdens de inferentie-instelling ongewijzigd.

## AI-gesprek

Het vrije gesprek van interactieve OpenClaw loopt via dezelfde agentlus als reguliere OpenClaw-agents, beperkt tot één OpenClaw-bevoegdheidstool op ring zero, `openclaw`, die de getypeerde bewerkingen omvat. Leesacties worden vrij uitgevoerd, mutaties vereisen jouw goedkeuring in het gesprek voor precies die bewerking (zie Bewerkingen en goedkeuring) en elke toegepaste schrijfbewerking wordt geaudit en opnieuw gevalideerd. De agentsessie blijft bestaan, zodat OpenClaw echt geheugen voor meerdere beurten heeft. Als de geverifieerde inferentieroute later niet meer werkt, ga je terug naar `openclaw onboard` en repareer je deze voordat je doorgaat.

De host zet verzoeken in natuurlijke taal niet zelf om in bewerkingen. Vrije
berichten — waaronder tekst die op een commando lijkt en vragen zoals "waarom is mijn
gateway gestopt?" — gaan naar de AI, die het verzoek via de tool
`openclaw` aan een getypeerde bewerking kan koppelen.

Wanneer een mutatie in behandeling is, worden alleen ondubbelzinnige goedkeurings- of afwijzingszinnen uit een
gesloten lijst zonder gevolgtrekking verwerkt. Dubbelzinnige instemming gaat naar een
afzonderlijke geconfigureerde voltooiingsaanroep en wordt anders standaard geweigerd. Gestructureerde
wizardvelden en exacte hostnavigatie zijn UI-besturingselementen, geen parsering van
bewerkingen in natuurlijke taal. Eén uitzondering voor geheimhygiëne is bijzonder belangrijk: een
exacte `config set` op een gevoelig pad (tokens, sleutels, wachtwoorden) bereikt nooit
een model. De host maakt een geredigeerd voorstel en de waarde wordt gemaskeerd in de
voor AI zichtbare geschiedenis. Geef voor geheimen de voorkeur aan `config set-ref <path> env <ENV_VAR>`.

De herstelmodus voor berichtkanalen gebruikt nooit de modelondersteunde planner. Herstel op afstand blijft deterministisch, zodat een defect of gecompromitteerd normaal agentpad niet als configuratie-editor kan worden gebruikt.

### Vertrouwensmodel van de CLI-harnas

Ingebedde runtimes en het Codex-app-serverharnas handhaven de ring-zero-
beperking rechtstreeks: de uitvoering bevat een OpenClaw-toestaanlijst voor tools met alleen
de tool `openclaw`. Voor Codex schakelt OpenClaw voor die uitvoering ook omgevingen, native
uitvoering, multi-agent, doelen, apps/plugins, Skills/MCP, zoeken op het web en
`request_user_input`-oppervlakken uit. Codex injecteert nog steeds het inerte native hulpprogramma `update_plan`;
dit kan de tijdelijke checklist van het model bijwerken, maar kan geen bestanden
of OpenClaw-configuratie schrijven. CLI-harnassen gebruiken de toestaanlijst van OpenClaw niet,
dus laat OpenClaw alleen backends toe waarvan het eigen toolselectiecontract
dezelfde beperking kan aantonen:

- Selecteerbare backends, waaronder Claude Code, worden gestart met een lege selectie
  van native tools en één MCP-tool, `openclaw`. De gegenereerde MCP-configuratie van Claude wordt
  toegepast met `--strict-mcp-config`, zodat geen andere MCP-servers worden geladen.
- Backends die geen native tools declareren, krijgen dezelfde speciale OpenClaw-
  MCP-server.
- Backends met altijd actieve of onbekende native tools worden vóór gevolgtrekking standaard geweigerd;
  ze kunnen geen OpenClaw-sessie hosten.

Alleen OpenClaw-sessies krijgen de openclaw-MCP-server; normale agentuitvoeringen
zien deze tool nooit. Selecteerbare CLI-backends/backends zonder native tools en modellen
met API-sleutels handhaven daarom de letterlijke lus met één tool. Codex-app-servermodellen handhaven
één OpenClaw-autoriteitstool plus het inerte native planningshulpprogramma. In alle
drie gevallen blijven schrijfbewerkingen voor de installatie beperkt tot het gecontroleerde goedkeuringscontract
van OpenClaw.

Gemini CLI blijft beschikbaar voor normale agents, maar kan de
toolvrije controle die door de gevolgtrekkingspoort wordt vereist niet afdwingen en kan daarom OpenClaw niet hosten.

## Overschakelen naar een agent

Gebruik een selector in natuurlijke taal om OpenClaw te verlaten en de normale TUI te openen:

```text
praat met agent
praat met werkagent
schakel over naar hoofdagent
```

`openclaw tui`, `openclaw chat` en `openclaw terminal` openen de normale agent-TUI rechtstreeks; ze starten OpenClaw niet. Nadat je naar de normale TUI bent overgeschakeld, keer je met `/openclaw` terug naar OpenClaw, eventueel met een vervolgverzoek:

```text
/openclaw
/openclaw restart gateway
```

## Herstelmodus voor berichten

De herstelmodus voor berichten is het toegangspunt via berichtkanalen voor OpenClaw: gebruik deze wanneer je normale agent niet werkt, maar een vertrouwd kanaal (bijvoorbeeld WhatsApp) nog steeds opdrachten ontvangt.

Dit is een deterministische handler voor noodopdrachten, niet de conversationele
OpenClaw-agent. Deze initialiseert geen nieuwe installatie en versoepelt de gevolgtrekkingspoort
voor OpenClaw-chat niet.

Ondersteunde opdracht: `/openclaw <request>`. Herstel accepteert alleen de exact getypte opdrachtgrammatica — natuurlijke taal wordt met een aanwijzing afgewezen, nooit als een bewerking geïnterpreteerd, en er wordt nooit een model geraadpleegd.

```text
Jij, in een vertrouwde DM van de eigenaar: /openclaw status
OpenClaw: OpenClaw-herstelmodus. Gateway bereikbaar: nee. Configuratie geldig: nee.
Jij: /openclaw restart gateway
OpenClaw: Plan: start de Gateway opnieuw. Antwoord met /openclaw yes om toe te passen.
Jij: /openclaw yes
OpenClaw: Toegepast. Auditvermelding geschreven.
```

Het maken van een agent kan ook lokaal of via herstel in de wachtrij worden geplaatst:

```text
create agent work workspace ~/Projects/work model openai/gpt-5.6-sol
/openclaw create agent work workspace ~/Projects/work
```

Bij het maken van een agent mag alleen het huidige, live geverifieerde standaardmodel worden genoemd. Laat het
model weg om die route over te nemen.

Herstel op afstand is een beheerdersoppervlak en moet worden behandeld als configuratieherstel op afstand, niet als normale chat.

Beveiligingscontract voor herstel op afstand:

- Uitgeschakeld wanneer sandboxing actief is voor de agent/sessie; OpenClaw weigert herstel op afstand en verwijst naar lokaal herstel via de CLI.
- De standaard effectieve status is `auto`: sta herstel op afstand alleen toe bij vertrouwde YOLO-werking, waarbij de runtime al lokale autoriteit zonder sandbox heeft (`tools.exec.security` wordt omgezet in `full` en `tools.exec.ask` wordt omgezet in `off`, met sandboxmodus `off`).
- Vereist een expliciete eigenaarsidentiteit; geen wildcardregels voor afzenders, open groepsbeleid, niet-geverifieerde webhooks of anonieme kanalen.
- Herstel is beperkt tot DM's van de eigenaar.
- Zoeken naar en weergeven van plugins is alleen-lezen. Installatie van plugins is altijd alleen lokaal (geblokkeerd in herstel, zelfs wanneer dit anders is ingeschakeld), omdat hierbij uitvoerbare code wordt gedownload. Verwijdering van plugins wordt zowel in lokale OpenClaw als in herstel geweigerd; voer `openclaw plugins uninstall <id>` uit vanuit een terminal.
- Herstel op afstand kan de lokale TUI niet openen of overschakelen naar een interactieve agentsessie; gebruik lokaal `openclaw` voor de overdracht aan een agent.
- Permanente schrijfbewerkingen vereisen nog steeds goedkeuring, ook in de herstelmodus.
- Openstaande goedkeuringen zijn eenmalig. Elke nieuwere herstelopdracht voor hetzelfde account, kanaal en dezelfde afzender trekt het oudere plan in; een mislukte uitvoering verbruikt de goedkeuring ook, dus verstuur de opdracht opnieuw om het opnieuw te proberen.
- Elke toegepaste herstelbewerking wordt gecontroleerd. Herstel via berichtkanalen registreert metadata over kanaal, account, afzender en bronadres; configuratiewijzigende bewerkingen registreren ook configuratiehashes vóór en na de wijziging.
- Geheimen worden nooit weergegeven. Inspectie van SecretRef rapporteert beschikbaarheid, geen waarden.
- Als de Gateway actief is, geeft herstel de voorkeur aan getypeerde Gateway-bewerkingen; als deze niet actief is, gebruikt herstel alleen het minimale lokale hersteloppervlak dat niet afhankelijk is van de normale agentlus.

Het herstelbeleid is ingebouwd: het is alleen beschikbaar wanneer de effectieve runtime
YOLO is, sandboxing is uitgeschakeld en het verzoek een DM van de eigenaar is. Openstaande goedkeuringen voor schrijfbewerkingen
verlopen na 15 minuten. `openclaw doctor --fix` verwijdert de buiten gebruik gestelde
configuratieblokken `systemAgent` en `crestodian`.

Herstel op afstand wordt gedekt door de Docker-lane:

```bash
pnpm test:docker:system-agent-rescue
```

Een optionele live rooktest voor het opdrachtoppervlak van het kanaal controleert `/openclaw status` plus een permanente goedkeuringsrondgang via de herstelhandler:

```bash
pnpm test:live:system-agent-rescue-channel
```

Verpakte eenmalige installatie met gevolgtrekkingspoort wordt gedekt door:

```bash
pnpm test:docker:system-agent-first-run
```

Die verpakte CLI-lane begint met een lege statusmap en toont aan dat OpenClaw
zonder gevolgtrekking standaard weigert. Vervolgens test en activeert deze een nepversie van Claude via
de verpakte activeringsmodule. Pas daarna bereikt een vaag verzoek de
planner en wordt het omgezet in een getypeerde installatie, gevolgd door eenmalige opdrachten die een
extra agent maken, Discord configureren via inschakeling van een plugin plus een
SecretRef voor het token, de configuratie valideren en het auditlogboek controleren. Deze lane levert ondersteunend
bewijs voor poorten/bewerkingen; de lane test geen interactieve onboarding of het
OpenClaw-gesprek over agent/tool/goedkeuring. Het onderstaande QA Lab-scenario verwijst
naar dezelfde Docker-lane:

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Doctor](/nl/cli/doctor)
- [TUI](/nl/cli/tui)
- [Sandbox](/nl/cli/sandbox)
- [Beveiliging](/nl/cli/security)
