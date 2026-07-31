---
read_when:
    - Veilige bins of aangepaste veilige-binprofielen configureren
    - Goedkeuringen doorsturen naar Slack/Discord/Telegram of andere chatkanalen
    - Een native goedkeuringsclient voor een kanaal implementeren
summary: 'Geavanceerde uitvoeringsgoedkeuringen: veilige binaire bestanden, interpreterkoppeling, doorsturen van goedkeuringen, systeemeigen levering'
title: Uitvoeringsgoedkeuringen — geavanceerd
x-i18n:
    generated_at: "2026-07-27T05:18:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

Geavanceerde onderwerpen voor exec-goedkeuring: het snelle pad `safeBins`, koppeling van interpreter/runtime
en het doorsturen van goedkeuringen naar chatkanalen (inclusief native bezorging).
Zie [Exec-goedkeuringen](/nl/tools/exec-approvals) voor het kernbeleid en de goedkeuringsflow.

## Veilige binaire bestanden (alleen stdin)

`tools.exec.safeBins` benoemt binaire bestanden die **alleen stdin** gebruiken (bijvoorbeeld `cut`) en die
in allowlist-modus worden uitgevoerd **zonder** expliciete allowlist-vermeldingen. Veilige binaire bestanden weigeren
positionele bestandsargumenten en padachtige tokens, zodat ze alleen op de
inkomende stream kunnen werken. Beschouw dit als een beperkt snel pad voor streamfilters, niet als een
algemene lijst met vertrouwde bestanden.

<Warning>
Voeg **geen** interpreter- of runtime-binaire bestanden (bijvoorbeeld `python3`, `node`,
`ruby`, `bash`, `sh`, `zsh`) toe aan `safeBins`. Als een opdracht ontworpen is om code te evalueren,
subopdrachten uit te voeren of bestanden te lezen, gebruik dan bij voorkeur expliciete allowlist-vermeldingen
en houd goedkeuringsprompts ingeschakeld. Aangepaste veilige binaire bestanden moeten een expliciet
profiel definiëren in `tools.exec.safeBinProfiles.<bin>`.
</Warning>

Standaard veilige binaire bestanden:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` en `sort` staan niet in de standaardlijst. Als je ze inschakelt, behoud dan expliciete
allowlist-vermeldingen voor hun workflows die niet alleen stdin gebruiken. Geef voor `grep` in de veilige-binaire-bestandenmodus
het patroon op met `-e`/`--regexp`; de positionele patroonvorm wordt geweigerd,
zodat bestandsoperanden niet als dubbelzinnige positionele argumenten kunnen worden binnengesmokkeld.

### Argv-validatie en geweigerde vlaggen

Validatie is uitsluitend deterministisch op basis van de vorm van argv (zonder te controleren
of bestanden op het hostsysteem bestaan), wat voorkomt dat verschillen tussen toestaan en weigeren
als orakel voor het bestaan van bestanden kunnen dienen. Bestandsgerichte opties worden geweigerd voor standaard veilige binaire bestanden; lange
opties worden fail-closed gevalideerd (onbekende vlaggen en dubbelzinnige afkortingen worden
geweigerd). Herkende alleen-lezen Booleaanse vlaggen van de standaardbinaire bestanden (bijvoorbeeld
`wc -l`, `tr -d`, `uniq -c`) worden geaccepteerd, terwijl niet-herkende korte vlaggen
fail-closed blijven en terugvallen op handmatige goedkeuring.

Geweigerde vlaggen per profiel voor veilige binaire bestanden:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `tail`: `--follow`, `--retry`, `-F`, `-f`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Veilige binaire bestanden dwingen bovendien af dat argv-tokens tijdens uitvoering als **letterlijke tekst** worden behandeld
(geen globbing en geen uitbreiding van `$VARS`) voor segmenten die alleen stdin gebruiken, zodat
patronen zoals `*` of `$HOME/...` niet kunnen worden gebruikt om het lezen van bestanden binnen te smokkelen. `awk`,
`sed` en `jq` worden altijd geweigerd als veilige binaire bestanden, omdat niet kan worden
gevalideerd dat hun semantiek beperkt blijft tot stdin: `jq` kan omgevingsgegevens lezen en jq-code laden uit
modules of opstartbestanden. Gebruik voor die tools een expliciete allowlist-vermelding of goedkeuringsprompt
in plaats van `safeBins`.

### Vertrouwde mappen met binaire bestanden

Veilige binaire bestanden moeten worden gevonden in vertrouwde mappen met binaire bestanden (systeemstandaarden plus
optioneel `tools.exec.safeBinTrustedDirs`). Vermeldingen in `PATH` worden nooit automatisch vertrouwd.
De standaard vertrouwde mappen zijn bewust minimaal: `/bin`, `/usr/bin`. Als
je uitvoerbare veilige binaire bestand zich in paden van een pakketbeheerder of gebruiker bevindt (bijvoorbeeld
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), voeg deze dan
expliciet toe aan `tools.exec.safeBinTrustedDirs`.

### Shell-ketens, wrappers en multiplexers

Shell-ketens (`&&`, `||`, `;`) zijn toegestaan wanneer elk segment op het hoogste niveau
aan de allowlist voldoet (inclusief veilige binaire bestanden of automatisch toestaan door Skills). Omleidingen
blijven niet ondersteund in allowlist-modus. Opdrachtsubstitutie (`$()` / backticks) wordt
tijdens het parseren van de allowlist geweigerd, ook binnen dubbele aanhalingstekens; gebruik enkele
aanhalingstekens als je letterlijke `$()`-tekst nodig hebt.

Bij goedkeuringen via de begeleidende macOS-app wordt onbewerkte shelltekst met shellbesturings- of
uitbreidingssyntaxis (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`)
behandeld als een allowlist-misser, tenzij het shell-binaire bestand zelf op de allowlist staat.

Voor shell-wrappers (`bash|sh|zsh ... -c/-lc`) worden omgevingsoverschrijvingen die aan de aanvraag zijn gebonden
beperkt tot een kleine expliciete allowlist (`TERM`, `LANG`, `LC_*`, `COLORTERM`,
`NO_COLOR`, `FORCE_COLOR`).

Bij `allow-always`-beslissingen in allowlist-modus slaan transparante dispatch-wrappers
(bijvoorbeeld `env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`) het pad van het
interne uitvoerbare bestand op in plaats van het wrapperpad. Shell-multiplexers
(`busybox`, `toybox`) worden op dezelfde manier uitgepakt voor shell-applets (`sh`, `ash`, enzovoort).
Als een wrapper of multiplexer niet veilig kan worden uitgepakt, wordt er niet automatisch
een allowlist-vermelding opgeslagen.

Als je interpreters zoals `python3` of `node` op de allowlist zet, geef dan de voorkeur aan
`tools.exec.strictInlineEval=true`, zodat inline-evaluatie nog steeds expliciete
goedkeuring vereist. In de strikte modus kan `allow-always` nog steeds onschuldige
interpreter-/scriptaanroepen opslaan, maar dragers van inline-evaluatie worden niet
automatisch opgeslagen.

### Veilige binaire bestanden versus allowlist

| Onderwerp         | `tools.exec.safeBins`                                        | Allowlist (`exec-approvals.json`)                                                                  |
| ----------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Doel              | Beperkte stdin-filters automatisch toestaan               | Specifieke uitvoerbare bestanden expliciet vertrouwen                                           |
| Type overeenkomst | Naam van uitvoerbaar bestand + argv-beleid voor veilige binaire bestanden | Glob voor het gevonden pad van het uitvoerbare bestand, of glob voor alleen de opdrachtnaam bij via PATH aangeroepen opdrachten |
| Argumentbereik    | Beperkt door het profiel voor veilige binaire bestanden en regels voor letterlijke tokens | Standaard padovereenkomst; optioneel kan `argPattern` de geparste argv beperken                  |
| Typische voorbeelden | `head`, `tail`, `tr`, `wc`                          | `jq`, `python3`, `node`, `ffmpeg`, aangepaste CLI's                                 |
| Beste toepassing  | Teksttransformaties met laag risico in pijplijnen         | Elke tool met breder gedrag of neveneffecten                                                    |

Configuratielocatie:

- `safeBins` komt uit de configuratie (`tools.exec.safeBins` of `agents.entries.*.tools.exec.safeBins` per agent).
- `safeBinTrustedDirs` komt uit de configuratie (`tools.exec.safeBinTrustedDirs` of `agents.entries.*.tools.exec.safeBinTrustedDirs` per agent).
- `safeBinProfiles` komt uit de configuratie (`tools.exec.safeBinProfiles` of `agents.entries.*.tools.exec.safeBinProfiles` per agent). Profielsleutels per agent overschrijven globale sleutels.
- allowlist-vermeldingen bevinden zich in het hostlokale goedkeuringsbestand onder `agents.<id>.allowlist` (of via Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` waarschuwt met `tools.exec.safe_bins_interpreter_unprofiled` wanneer interpreter-/runtime-binaire bestanden zonder expliciete profielen in `safeBins` voorkomen.
- `openclaw doctor --fix` kan ontbrekende aangepaste `safeBinProfiles.<bin>`-vermeldingen als `{}` aanmaken (controleer en beperk ze daarna). Interpreter-/runtime-binaire bestanden worden niet automatisch aangemaakt.

Voorbeeld van een aangepast profiel:

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## Interpreter-/runtime-opdrachten

Door goedkeuring ondersteunde interpreter-/runtime-uitvoeringen zijn bewust conservatief:

- De exacte argv-/cwd-/env-context wordt altijd gebonden.
- Directe shellscripts en directe runtime-bestandsvormen worden naar beste vermogen aan één concrete lokale
  momentopname van een bestand gebonden.
- Gangbare wrappervormen voor pakketbeheerders die nog steeds naar één direct lokaal bestand verwijzen (bijvoorbeeld
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) worden vóór het binden uitgepakt.
- Als OpenClaw niet precies één concreet lokaal bestand voor een interpreter-/runtime-opdracht kan identificeren
  (bijvoorbeeld pakketscripts, evaluatievormen, runtime-specifieke loaderketens of dubbelzinnige vormen met meerdere bestanden),
  wordt door goedkeuring ondersteunde uitvoering geweigerd in plaats van semantische dekking te claimen die
  niet aanwezig is.
- Geef voor die workflows de voorkeur aan sandboxing, een afzonderlijke hostgrens of een expliciet vertrouwde
  allowlist/volledige workflow waarbij de operator de bredere runtimesemantiek accepteert.

Wanneer goedkeuring vereist is, retourneert de exec-tool onmiddellijk een goedkeurings-id. Gebruik dat id om
latere systeemgebeurtenissen van goedgekeurde uitvoeringen te correleren (`Exec finished`, en `Exec running` indien geconfigureerd).
Als er vóór de time-out geen beslissing binnenkomt, wordt de aanvraag behandeld als een time-out van de goedkeuring en
weergegeven als een definitieve weigering van de hostopdracht. Voor asynchrone goedkeuringen van de hoofdagent met een oorspronkelijke
sessie hervat OpenClaw die sessie ook met een interne vervolgactie, zodat de agent ziet dat
de opdracht niet is uitgevoerd in plaats van later een ontbrekend resultaat te herstellen. Openstaande exec-goedkeuringen verlopen
standaard na 30 minuten.

### Gedrag bij bezorging van vervolgacties

Nadat een goedgekeurde asynchrone exec is voltooid, stuurt OpenClaw een vervolgbeurt `agent` naar dezelfde sessie.
Geweigerde asynchrone goedkeuringen gebruiken voor de weigeringsstatus hetzelfde vervolgpad naar de hoofdsessie, maar
registreren geen verhoogde runtime-overdrachten en voeren de opdracht niet uit. Weigeringen zonder een hervatbare
hoofdsessie worden onderdrukt of gemeld via een veilige directe route, wanneer die beschikbaar is.

- Als er een geldig extern bezorgingsdoel bestaat (bezorgbaar kanaal plus doel `to`), gebruikt de vervolgbezorging dat kanaal.
- In uitsluitend webchat- of interne sessieflows zonder extern doel blijft de vervolgbezorging beperkt tot de sessie (`deliver: false`).
- Als een aanroeper expliciet strikte externe bezorging aanvraagt zonder een oplosbaar extern kanaal, mislukt de aanvraag met `INVALID_REQUEST`.
- Als `bestEffortDeliver` is ingeschakeld en er geen extern kanaal kan worden gevonden, wordt de bezorging teruggebracht tot alleen de sessie in plaats van te mislukken.

## Minimale bereiken voor clients van derden

Het oplossen van Gateway-goedkeuringen wordt beschermd door het specifieke bereik `operator.approvals`. Dit geldt zowel voor de eigenaarspecifieke methode `exec.approval.resolve` als voor de soortonafhankelijke methode `approval.resolve`; `operator.write` omvat dit bereik niet. Dashboards en integraties moeten alleen de bereiken aanvragen die vereist zijn voor de methoden die ze gebruiken. Behandel toegang voor het oplossen van goedkeuringen als bevoegdheid op het niveau van uitvoering op afstand en verleen `operator.approvals` weloverwogen, zelfs wanneer de client slechts een kleine goedkeuringsinterface toont.

## Goedkeuringen doorsturen naar chatkanalen

Je kunt prompts voor exec-goedkeuring doorsturen naar elk chatkanaal (inclusief pluginkanalen) en ze
goedkeuren met `/approve`. Hiervoor wordt de normale pijplijn voor uitgaande bezorging gebruikt.

Configuratie:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // substring or regex
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Antwoord in de chat:

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

De opdracht `/approve` verwerkt zowel exec-goedkeuringen als plugingoedkeuringen. Als de ID niet overeenkomt met een wachtende exec-goedkeuring, worden in plaats daarvan automatisch de plugingoedkeuringen gecontroleerd. Deze terugval is beperkt tot fouten van het type 'goedkeuring niet gevonden'; bij een daadwerkelijke weigering/fout van een exec-goedkeuring wordt niet stilzwijgend opnieuw geprobeerd als plugingoedkeuring.

### Doorsturen van plugingoedkeuringen

Voor het doorsturen van plugingoedkeuringen wordt dezelfde bezorgingspijplijn gebruikt als voor exec-goedkeuringen, maar met een eigen
onafhankelijke configuratie onder `approvals.plugin`. Het in- of uitschakelen van de ene heeft geen invloed op de andere.
Zie voor het gedrag bij het maken van plugins, aanvraagvelden en de semantiek van beslissingen
[Toestemmingsaanvragen van plugins](/plugins/plugin-permission-requests).

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

De configuratiestructuur is identiek aan `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` en `targets` werken op dezelfde manier.

Kanalen die gedeelde interactieve antwoorden ondersteunen, geven voor zowel exec- als
plugingoedkeuringen dezelfde goedkeuringsknoppen weer. Kanalen zonder gedeelde interactieve gebruikersinterface vallen terug op platte tekst met
instructies voor `/approve`. Aanvragen voor plugingoedkeuring kunnen de beschikbare beslissingen beperken: goedkeuringsinterfaces gebruiken
de in de aanvraag opgegeven verzameling beslissingen en de Gateway weigert pogingen om een beslissing in te dienen die
niet werd aangeboden.

### Goedkeuringen in dezelfde chat op elk kanaal

Wanneer een aanvraag voor een exec- of plugingoedkeuring afkomstig is van een chatinterface waarop berichten kunnen worden bezorgd, kan diezelfde chat
de aanvraag standaard goedkeuren met `/approve`. Dit geldt naast de bestaande flows van de webinterface en terminalinterface ook voor Slack, Matrix, Microsoft Teams en
vergelijkbare chats waarop berichten kunnen worden bezorgd, waarbij het
normale autorisatiemodel van het kanaal voor dat gesprek wordt gebruikt. Als de oorspronkelijke chat al opdrachten kan verzenden
en antwoorden kan ontvangen, hebben goedkeuringsaanvragen niet langer een afzonderlijke systeemeigen bezorgingsadapter nodig om
wachtend te blijven.

Discord, Telegram en QQ bot ondersteunen ook `/approve` in dezelfde chat, maar die kanalen gebruiken nog steeds hun
vastgestelde lijst met goedkeurders voor autorisatie, zelfs wanneer systeemeigen bezorging van goedkeuringen is uitgeschakeld.

### Systeemeigen bezorging van goedkeuringen

Sommige kanalen kunnen ook als systeemeigen goedkeuringsclients fungeren: Discord, Slack, Telegram, Matrix en QQ bot.
Systeemeigen clients voegen privéberichten aan goedkeurders, distributie naar de oorspronkelijke chat en een kanaalspecifieke interactieve goedkeuringsinterface toe
boven op de gedeelde flow voor `/approve` in dezelfde chat.

Wanneer systeemeigen goedkeuringskaarten/-knoppen beschikbaar zijn, is die systeemeigen gebruikersinterface het primaire pad voor de agent.
De agent mag niet ook een dubbele platte chatopdracht `/approve` herhalen, tenzij het toolresultaat aangeeft
dat chatgoedkeuringen niet beschikbaar zijn of handmatige goedkeuring het enige resterende pad is.

Als een systeemeigen goedkeuringsclient is geconfigureerd, maar er geen systeemeigen runtime actief is voor het oorspronkelijke
kanaal, houdt OpenClaw de lokale deterministische prompt `/approve` zichtbaar. Als de systeemeigen runtime
actief is en bezorging probeert, maar geen enkel doel de kaart ontvangt, stuurt OpenClaw in dezelfde chat een terugvalmelding
met de exacte opdracht `/approve <id> <decision>`, zodat de aanvraag alsnog kan worden afgehandeld.

Algemeen model:

- het exec-beleid van de host bepaalt nog steeds of exec-goedkeuring vereist is
- `approvals.exec` bepaalt of goedkeuringsprompts naar andere chatbestemmingen worden doorgestuurd
- `channels.<channel>.execApprovals` bepaalt of kanaalspecifieke systeemeigen clients voor Discord, Slack, Telegram, QQ bot en vergelijkbare
  kanalen zijn ingeschakeld
- Slack-plugingoedkeuringen kunnen de systeemeigen goedkeuringsclient van Slack gebruiken wanneer de aanvraag afkomstig is van Slack
  en Slack-plugingoedkeurders kunnen worden vastgesteld; `approvals.plugin` kan plugingoedkeuringen ook routeren naar Slack-
  sessies of -doelen, zelfs wanneer Slack-exec-goedkeuringen zijn uitgeschakeld
- Systeemeigen goedkeuringskaarten van Google Chat verwerken exec- en plugingoedkeuringen die afkomstig zijn uit Google
  Chat-ruimten of -threads wanneer stabiele `users/<id>`-goedkeurders worden vastgesteld via `dm.allowFrom` of
  `defaultTo`; ze gebruiken geen reactiegebeurtenissen voor beslissingen
- Bezorging van goedkeuringen via reacties in WhatsApp en Signal wordt beheerd door `approvals.exec` en
  `approvals.plugin`; ze hebben geen `channels.<channel>.execApprovals`-blokken

Systeemeigen goedkeuringsclients schakelen automatisch bezorging met privéberichten als eerste optie in wanneer aan al deze voorwaarden wordt voldaan:

- het kanaal ondersteunt systeemeigen bezorging van goedkeuringen
- goedkeurders kunnen worden vastgesteld via expliciete `execApprovals.approvers` of de identiteit van de eigenaar,
  zoals `commands.ownerAllowFrom`
- `channels.<channel>.execApprovals.enabled` is niet ingesteld of is `"auto"`

Stel `enabled: false` in om een systeemeigen goedkeuringsclient expliciet uit te schakelen. Stel `enabled: true` in om
deze geforceerd in te schakelen wanneer goedkeurders kunnen worden vastgesteld. Openbare bezorging in de oorspronkelijke chat blijft expliciet via
`channels.<channel>.execApprovals.target`. Wanneer systeemeigen `target` bezorging in de oorspronkelijke chat inschakelt,
bevatten goedkeuringsprompts de opdrachttekst.

Veelgestelde vraag: [Waarom zijn er twee configuraties voor exec-goedkeuringen voor chatgoedkeuringen?](/help/faq-first-run)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`
- QQ bot: `channels.qqbot.execApprovals.*`
- Google Chat: configureer stabiele goedkeurders met `channels.googlechat.dm.allowFrom` of
  `channels.googlechat.defaultTo`; er is geen `execApprovals`-blok vereist
- WhatsApp: gebruik `approvals.exec` en `approvals.plugin` om goedkeuringsprompts naar WhatsApp te routeren
- Signal: gebruik `approvals.exec` en `approvals.plugin` om goedkeuringsprompts naar Signal te routeren

Routering specifiek voor systeemeigen clients:

- Telegram gebruikt standaard privéberichten aan goedkeurders (`target: "dm"`). Schakel over naar `channel` of `both` om
  goedkeuringsprompts ook in de oorspronkelijke Telegram-chat/het oorspronkelijke Telegram-onderwerp weer te geven. Voor Telegram-forumonderwerpen behoudt OpenClaw
  het onderwerp voor de goedkeuringsprompt en het vervolgbericht na goedkeuring.
- Discord- en Telegram-goedkeurders kunnen expliciet zijn (`execApprovals.approvers`) of worden afgeleid uit
  `commands.ownerAllowFrom`; alleen vastgestelde goedkeurders kunnen goedkeuren of weigeren.
- Slack-goedkeurders kunnen expliciet zijn (`execApprovals.approvers`) of worden afgeleid uit
  `commands.ownerAllowFrom`. Privéberichten voor Slack-plugingoedkeuringen gebruiken Slack-plugingoedkeurders uit `allowFrom`
  en de standaardroutering van het account, niet Slack-exec-goedkeurders. Systeemeigen Slack-knoppen behouden het soort goedkeurings-ID,
  zodat `plugin:`-ID's plugingoedkeuringen kunnen afhandelen zonder een tweede lokale terugvallaag van Slack.
- Systeemeigen Google Chat-kaarten behouden de handmatige terugval naar `/approve` in de berichttekst, maar callbacks van kaartknoppen
  bevatten alleen ondoorzichtige actietokens; de goedkeurings-ID en beslissing worden opgehaald uit
  de wachtende status aan de serverzijde.
- Emoji-goedkeuringen van WhatsApp verwerken zowel exec- als pluginprompts wanneer de bijbehorende doorstuurfamilie op het hoogste niveau
  naar WhatsApp routeert. Prompts met een systeemeigen oorsprong worden rechtstreeks gekoppeld; gedeelde bezorging in de doelmodus
  koppelt dezelfde getypeerde goedkeuringsmetadata aan het geaccepteerde WhatsApp-berichtbewijs.
- Goedkeuringen via reacties in Signal verwerken zowel exec- als pluginprompts alleen wanneer de bijbehorende doorstuurfamilie op het hoogste niveau
  is ingeschakeld en naar Signal routeert. Rechtstreekse Signal-exec-goedkeuringen in dezelfde chat kunnen
  de lokale terugval naar `/approve` onderdrukken zonder expliciete goedkeurders; voor de afhandeling van Signal-reacties
  zijn nog steeds expliciete Signal-goedkeurders uit `channels.signal.allowFrom` of `defaultTo` vereist.
- Systeemeigen routering van Matrix via privéberichten/kanalen en snelkoppelingen via reacties verwerken zowel exec- als plugingoedkeuringen;
  pluginautorisatie is nog steeds afkomstig uit `channels.matrix.dm.allowFrom`. Systeemeigen Matrix-prompts
  bevatten bij de eerste promptgebeurtenis aangepaste `com.openclaw.approval`-gebeurtenisinhoud, zodat Matrix-clients die OpenClaw ondersteunen
  de gestructureerde goedkeuringsstatus kunnen lezen, terwijl standaardclients de plattetekstterugval
  `/approve` behouden.
- Systeemeigen goedkeuringsknoppen van Discord en Telegram bevatten in transportprivé-callbackgegevens expliciet het eigenaartype exec of plugin
  en handelen alleen die eigenaar af. Oudere `/approve`-besturingselementen zonder
  type blijven een begrensd compatibiliteitspad: ze proberen alleen eigenaartypen die de actor mag goedkeuren,
  gaan alleen door na het resultaat 'goedkeuring niet gevonden' en leiden nooit eigenaarschap af uit de goedkeurings-ID.
- De aanvrager hoeft geen goedkeurder te zijn.
- Als geen enkele operatorinterface of geconfigureerde goedkeuringsclient de aanvraag kan accepteren, valt de prompt terug op
  `askFallback`.

Gevoelige groepsopdrachten die alleen voor de eigenaar bestemd zijn, zoals `/diagnostics` en `/export-trajectory`, gebruiken privéroutering
naar de eigenaar voor goedkeuringsprompts en eindresultaten. OpenClaw probeert eerst een privéroute op hetzelfde
oppervlak waar de eigenaar de opdracht heeft uitgevoerd. Als dat oppervlak geen privéroute naar de eigenaar heeft, wordt
teruggevallen op de eerste beschikbare route naar de eigenaar uit `commands.ownerAllowFrom`, zodat een Discord-groepsopdracht
de goedkeuring en het resultaat nog steeds naar het privébericht van de eigenaar in Telegram kan sturen wanneer Telegram als
primaire privéinterface is geconfigureerd. De groepschat krijgt alleen een korte bevestiging.

Zie:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ bot](/channels/qqbot)

### Officiële mobiele operatorapps

De officiële iOS- en Android-apps kunnen ook wachtende exec-
goedkeuringen van de Gateway beoordelen wanneer een `operator.admin`-verbinding wordt gebruikt, of wanneer hun gekoppelde
`operator.approvals`-apparaat expliciet als doel van de aanvraag was ingesteld. Ze lezen
dezelfde opgeschoonde duurzame record die door de
bedieningsinterface wordt gebruikt, dienen een typebewuste beslissing in en tonen het canonieke
resultaat van het eerste antwoord van de Gateway. De Apple Watch spiegelt deze goedkeuringsprompts via
de gekoppelde iPhone, met acties voor eenmalig toestaan en weigeren. In de rechtstreekse Gateway-modus van de Watch
worden goedkeuringen niet beoordeeld.

Een verloren bevestiging van de afhandeling maakt de ingediende keuze niet gezaghebbend:
de app schakelt de bedieningselementen uit en leest de record opnieuw. Als een ander oppervlak
heeft gewonnen, toont de app die vastgelegde beslissing. Wachtende prompts blijven gekoppeld aan de
Gateway die ze heeft uitgegeven, zodat het wisselen van de actieve Gateway een
oude goedkeurings-ID niet kan omleiden.

### macOS-IPC-flow

```
Gateway -> Node-service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac-app (gebruikersinterface + goedkeuringen + system.run)
```

Beveiligingsopmerkingen:

- Unix-socketmodus `0600`, token opgeslagen in `exec-approvals.json`.
- Peercontrole voor dezelfde UID.
- Uitdaging/antwoord (nonce + HMAC-token + aanvraaghash) + korte TTL.

## Veelgestelde vragen

### Wanneer worden `accountId` en `threadId` gebruikt bij een goedkeuringsdoel?

Gebruik `accountId` wanneer het kanaal meerdere geconfigureerde identiteiten heeft en de goedkeuringsprompt
via één specifiek account moet worden verzonden. Gebruik `threadId` wanneer de bestemming onderwerpen of
threads ondersteunt en de prompt binnen die thread moet blijven in plaats van in de chat op het hoogste niveau.

Een concreet Telegram-geval is een supergroep voor beheer met forumonderwerpen en twee Telegram-botaccounts.
De waarde `to` benoemt de supergroep, `accountId` selecteert het botaccount en `threadId`
selecteert het forumonderwerp:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

Met deze configuratie worden doorgestuurde uitvoeringsgoedkeuringen door het Telegram-account `ops-bot` geplaatst in onderwerp
`77` van chat `-1001234567890`. Een doel zonder `accountId` gebruikt het standaardaccount van het kanaal, en
een doel zonder `threadId` plaatst het bericht op de hoofdbestemming.

### Kan iedereen in een sessie goedkeuringen verlenen wanneer ze naar die sessie worden verzonden?

Nee. Levering aan een sessie bepaalt alleen waar de prompt verschijnt. Hierdoor krijgt niet automatisch elke
deelnemer aan die chat toestemming om goedkeuring te verlenen.

Voor algemene `/approve` in dezelfde chat moet de afzender al gemachtigd zijn om opdrachten uit te voeren in die
kanaalsessie. Als het kanaal expliciete goedkeurders beschikbaar stelt, kunnen die goedkeurders de actie
`/approve` autoriseren, zelfs wanneer ze in die sessie niet anderszins gemachtigd zijn om opdrachten uit te voeren.

Sommige kanalen zijn strenger. Discord, Telegram, Matrix, systeemeigen goedkeurings-DM's van Slack en vergelijkbare
systeemeigen goedkeuringsclients gebruiken hun vastgestelde lijsten met goedkeurders voor goedkeuringsautorisatie. Een
goedkeuringsprompt in een Telegram-forumonderwerp kan bijvoorbeeld zichtbaar zijn voor iedereen in het onderwerp, maar alleen numerieke
Telegram-gebruikers-ID's die zijn vastgesteld via `channels.telegram.execApprovals.approvers` of
`commands.ownerAllowFrom` kunnen deze goedkeuren of afwijzen.

## Gerelateerd

- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals) — kernbeleid en goedkeuringsflow
- [Uitvoeringstool](/nl/tools/exec)
- [Verhoogde modus](/nl/tools/elevated)
- [Skills](/nl/tools/skills) — door Skills ondersteund gedrag voor automatisch toestaan
