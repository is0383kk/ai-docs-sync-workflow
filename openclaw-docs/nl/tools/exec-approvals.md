---
read_when:
    - Exec-goedkeuringen of toelatingslijsten configureren
    - UX voor uitvoeringsgoedkeuring implementeren in de macOS-app
    - Sandbox-escape-prompts en de implicaties ervan beoordelen
sidebarTitle: Exec approvals
summary: 'Goedkeuringen voor uitvoering op de host: beleidsinstellingen, toelatingslijsten en de YOLO/strikte workflow'
title: Goedkeuringen voor uitvoeren
x-i18n:
    generated_at: "2026-07-27T05:24:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Exec-goedkeuringen zijn de **beveiliging van de begeleidende app / nodehost** waarmee een
gesandboxte agent opdrachten op een echte host kan uitvoeren (`gateway` of `node`). Opdrachten
worden alleen uitgevoerd wanneer beleid + toestemmingslijst + (optionele) gebruikersgoedkeuring allemaal overeenstemmen.
Goedkeuringen komen **boven op** het toolbeleid en de verhoogde toegangscontrole (verhoogde
`full` omzeilt ze).

Zie
[Toestemmingsmodi](/nl/tools/permission-modes) voor een modusgericht overzicht van `deny`, `allowlist`, `ask`, `auto`, `full`,
Codex Guardian-toewijzing en ACPX-harnastoestemmingen.

<Note>
Het effectieve beleid is het **strengste** van `tools.exec.*` en de standaardwaarden voor goedkeuringen:
goedkeuringen kunnen de uit de configuratie afgeleide beveiliging/vraaginstelling alleen aanscherpen, nooit
versoepelen. Als een goedkeuringsveld wordt weggelaten, wordt de waarde `tools.exec`
gebruikt. Host-exec gebruikt ook de lokale goedkeuringsstatus op die machine: een
hostlokale `ask: "always"` in het goedkeuringsbestand van de uitvoeringshost blijft
om goedkeuring vragen, zelfs als de sessie- of configuratiestandaarden om `ask: "on-miss"` vragen.
</Note>

## Waar dit van toepassing is

Exec-goedkeuringen worden lokaal afgedwongen op de uitvoeringshost:

- **Gateway-host** -> `openclaw`-proces op de gatewaymachine.
- **Nodehost** -> node-runner (begeleidende macOS-app of headless nodehost).

### Vertrouwensmodel

- Door de Gateway geauthenticeerde aanroepers zijn vertrouwde operators voor die Gateway.
- Gekoppelde nodes breiden die vertrouwde operatorbevoegdheid uit naar de nodehost.
- Goedkeuringen verminderen het risico op onbedoelde uitvoering, maar zijn **geen** authenticatiegrens per gebruiker of alleen-lezenbeleid voor het bestandssysteem.
- Na goedkeuring kan een opdracht bestanden wijzigen volgens de geselecteerde host- of sandboxbestandssysteemtoestemmingen.
- Goedgekeurde uitvoeringen op de nodehost leggen de canonieke uitvoeringscontext vast: cwd, exacte argv, omgevingsbinding indien aanwezig en een vastgezet pad naar het uitvoerbare bestand indien van toepassing.
- Voor shellscripts en directe aanroepen van interpreter-/runtimebestanden probeert OpenClaw ook één concreet lokaal bestandsoperand vast te leggen. Als dat bestand na goedkeuring maar vóór uitvoering verandert, wordt de uitvoering geweigerd in plaats van gewijzigde inhoud uit te voeren.
- Bestandsbinding gebeurt naar beste vermogen en vormt geen volledig model van elk laadpad van interpreters/runtimes. Als niet precies één concreet lokaal bestand kan worden geïdentificeerd, weigert OpenClaw een door goedkeuring gedekte uitvoering aan te maken in plaats van volledige dekking te veinzen.

### macOS-splitsing

- De **nodehostservice** stuurt `system.run` via lokale IPC door naar de **macOS-app**.
- De **macOS-app** dwingt goedkeuringen af en voert de opdracht uit in de UI-context.

## Het effectieve beleid inspecteren

| Opdracht                                                          | Wat deze toont                                                                          |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | Aangevraagd beleid, bronnen van hostbeleid en het effectieve resultaat.                       |
| `openclaw exec-policy show`                                      | Samengevoegde weergave van de lokale machine.                                                             |
| `openclaw exec-policy set` / `preset`                            | Synchroniseert het lokaal aangevraagde beleid in één stap met het lokale hostgoedkeuringsbestand. |

<Note>
`/exec`-overschrijvingen per sessie zijn niet inbegrepen. Voer `/exec` in de betreffende sessie uit om de huidige standaardwaarden ervan te inspecteren. Zie [sessieoverschrijvingen](/nl/tools/exec#session-overrides-exec).
</Note>

Volledige CLI-referentie (vlaggen, JSON-uitvoer, toevoegen aan/verwijderen uit de toestemmingslijst): [CLI voor goedkeuringen](/nl/cli/approvals).

Wanneer een lokaal bereik `host=node` aanvraagt, rapporteert `exec-policy show` dat
bereik tijdens runtime als door de node beheerd, in plaats van het lokale goedkeuringsbestand
als de bron van waarheid te behandelen.

Als de UI van de begeleidende app **niet beschikbaar is**, wordt elk verzoek dat
normaal om goedkeuring zou vragen, afgehandeld door de **vraagterugval** (standaard: `deny`).

<Tip>
Native chatgoedkeuringsclients kunnen kanaalspecifieke mogelijkheden toevoegen aan het
wachtende goedkeuringsbericht. Matrix voegt reactiesnelkoppelingen toe (`✅` eenmaal toestaan,
`♾️` altijd toestaan, `❌` weigeren), terwijl `/approve ...` als
terugval in het bericht blijft staan.
</Tip>

## Instellingen en opslag

Goedkeuringen bevinden zich in een lokaal JSON-bestand op de uitvoeringshost. Wanneer
`OPENCLAW_STATE_DIR` is ingesteld, volgt het bestand die statusmap;
anders gebruikt het de standaardstatusmap van OpenClaw:

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# anders
~/.openclaw/exec-approvals.json
```

De standaardgoedkeuringssocket volgt dezelfde hoofdmap:
`$OPENCLAW_STATE_DIR/exec-approvals.sock`, of
`~/.openclaw/exec-approvals.sock` wanneer de variabele niet is ingesteld.

Statusmappen zijn onafhankelijke vertrouwensbereiken. Wanneer `OPENCLAW_STATE_DIR`
naar een andere locatie verwijst, importeert of archiveert OpenClaw
`~/.openclaw/exec-approvals.json` nooit; configureer goedkeuringen afzonderlijk voor de
aangepaste statusmap. Doctor importeert ook de verouderde
`plugin-binding-approvals.json` alleen wanneer deze bij de actieve statusmap
hoort.

Voorbeeldschema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## Beleidsinstellingen

### `tools.exec.mode`

`tools.exec.mode` is het aanbevolen genormaliseerde beleidsoppervlak voor host-exec:

| Waarde       | Gedrag                                                                                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | Host-exec blokkeren.                                                                                                                                                          |
| `allowlist` | Alleen opdrachten op de toestemmingslijst uitvoeren zonder te vragen.                                                                                                                             |
| `ask`       | Toestemmingslijstbeleid gebruiken en bij ontbrekende overeenkomsten om goedkeuring vragen.                                                                                                                                   |
| `auto`      | Toestemmingslijstbeleid gebruiken, deterministische overeenkomsten direct uitvoeren en ontbrekende goedkeuringen via de native automatische reviewer van OpenClaw sturen voordat wordt teruggevallen op een menselijke goedkeuringsroute. |
| `full`      | Host-exec zonder goedkeuringsvragen uitvoeren.                                                                                                                                   |

Doctor migreert het buiten gebruik gestelde, opgeslagen paar `tools.exec.security` / `tools.exec.ask`
naar `tools.exec.mode`.

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - alle host-exec-verzoeken blokkeren.
  - `allowlist` - alleen opdrachten op de toestemmingslijst toestaan.
  - `full` - alles toestaan (gelijkwaardig aan verhoogd).

De standaardwaarde is `full` voor gateway-/nodehosts; een `sandbox`-host gebruikt in plaats daarvan standaard
`deny`.
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  Geconfigureerd vraagbeleid voor host-exec. Bepaalt het basisgedrag voor
  goedkeuringsvragen vanuit `tools.exec.ask` en de standaardwaarden voor hostgoedkeuringen.
  De standaardwaarde is `off`. De toolparameter `ask` per aanroep (zie
  [Exec-tool](/nl/tools/exec#parameters)) kan die basis alleen aanscherpen, en
  modelaanroepen vanuit kanalen negeren deze wanneer de effectieve hostvraaginstelling `off` is.

- `off` - nooit vragen.
- `on-miss` - alleen vragen wanneer de toestemmingslijst niet overeenkomt.
- `always` - bij elke opdracht vragen. Duurzaam vertrouwen via `allow-always` onderdrukt vragen **niet** wanneer de effectieve vraagmodus `always` is.

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  Afhandeling wanneer een vraag vereist is maar geen UI bereikbaar is (of de
  vraag verloopt). Standaard `deny` wanneer weggelaten.

- `deny` - blokkeren.
- `allowlist` - alleen toestaan als de toestemmingslijst overeenkomt.
- `full` - toestaan.

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  Wanneer `true`, worden vormen voor inline code-evaluatie behandeld als uitsluitend na goedkeuring, zelfs als het
  interpreterprogramma zelf op de toestemmingslijst staat. Verdediging in de diepte voor
  interpreterladers die niet eenduidig aan één stabiel bestandsoperand kunnen worden gekoppeld.
</ParamField>

Voorbeelden die door de strikte modus worden onderschept: `python -c`, `node -e`/`--eval`/`-p`,
`ruby -e`, `perl -e`/`-E`, `php -r`, `lua -e`, `osascript -e` (ook inlinevormen van `awk`,
`sed`, `make`, `find -exec` en `xargs`).

In de strikte modus vereisen deze opdrachten een reviewer of expliciete goedkeuring. Met
`tools.exec.mode: "auto"` kan de reviewer één uitvoering met laag risico toestaan wanneer
de opdracht een afdwingbaar plan heeft; anders vraagt OpenClaw een mens om goedkeuring.
`Codex app-server`-opdrachtgoedkeuringen die bij de reviewerterugval terechtkomen, vragen een
mens om goedkeuring omdat hun goedkeuringsverzoeken geen afdwingbaar, bepaald
uitvoerbaar bestand beschikbaar stellen.
`allow-always` slaat geen nieuwe vermeldingen in de toestemmingslijst op voor inline-evaluatieopdrachten.

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  Alleen voor presentatie: indien ingeschakeld, kan OpenClaw door de parser afgeleide
  opdrachtbereiken toevoegen, zodat webgoedkeuringsvragen opdrachttokens kunnen markeren. Dit
  wijzigt **niet** `security`, `ask`, overeenkomsten met de toestemmingslijst, strikt inline-evaluatiegedrag,
  het doorsturen van goedkeuringen of de uitvoering van opdrachten.
</ParamField>

Stel dit globaal in onder `tools.exec.commandHighlighting` of per agent onder
`agents.entries.*.tools.exec.commandHighlighting`.

## YOLO-modus (zonder goedkeuring)

Om host-exec zonder goedkeuringsvragen uit te voeren, open je **beide** beleidslagen:
het aangevraagde exec-beleid in de OpenClaw-configuratie (`tools.exec.*`) **en**
het hostlokale goedkeuringsbeleid in het goedkeuringsbestand van de uitvoeringshost.

Een weggelaten `askFallback` gebruikt standaard `deny`. Stel host-`askFallback` expliciet in op `full`
wanneer een goedkeuringsvraag zonder UI moet terugvallen op toestaan.

| Laag              | YOLO-instelling               |
| ------------------ | -------------------------- |
| `tools.exec.mode`  | `full` op `gateway`/`node` |
| Host-`askFallback` | `full`                     |

<Warning>
**Belangrijke verschillen:**

- `tools.exec.host=auto` bepaalt **waar** exec wordt uitgevoerd: in de sandbox wanneer die beschikbaar is, anders op de Gateway.
- YOLO bepaalt **hoe** exec op de host wordt goedgekeurd: `security=full` plus `ask=off`.
- YOLO voegt **geen** afzonderlijke heuristische goedkeuringspoort voor opdrachtverhulling of afwijzingslaag voor scriptvoorcontrole toe boven op het geconfigureerde exec-beleid van de host.
- `auto` maakt Gateway-routering niet tot een vrij te overschrijven instelling vanuit een sessie in een sandbox. Een `host=node`-verzoek per aanroep is toegestaan vanuit `auto`; `host=gateway` is alleen toegestaan vanuit `auto` wanneer er geen sandboxruntime actief is. Stel voor een stabiele niet-automatische standaardwaarde `tools.exec.host` in of gebruik expliciet `/exec host=...`.

</Warning>

CLI-gebaseerde providers die hun eigen niet-interactieve machtigingsmodus aanbieden,
kunnen dit beleid volgen. Claude CLI voegt
`--permission-mode bypassPermissions` toe wanneer het effectieve exec-beleid
van OpenClaw YOLO is. Voor door OpenClaw beheerde live Claude-sessies is het
effectieve exec-beleid van OpenClaw bepalend ten opzichte van de eigen machtigingsmodus van Claude:
YOLO normaliseert live starts naar `--permission-mode bypassPermissions`, en
een beperkend effectief exec-beleid normaliseert live starts naar
`--permission-mode default`, zelfs als onbewerkte argumenten voor de Claude-backend een andere
modus opgeven.

Als je een conservatievere configuratie wilt, stel je het exec-beleid van OpenClaw weer strenger in op
`allowlist` / `on-miss` of `deny`.

### Permanente configuratie 'nooit vragen' voor de Gateway-host

<Steps>
  <Step title="Stel het gevraagde configuratiebeleid in">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="Stem het bestand met hostgoedkeuringen af">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### Lokale snelkoppeling

```bash
openclaw exec-policy preset yolo
```

Werkt zowel de lokale `tools.exec.host/security/ask` als de standaardwaarden in het lokale
goedkeuringsbestand bij (inclusief `askFallback: "full"`). Dit is bewust
uitsluitend lokaal. Gebruik `openclaw approvals set --gateway` of `openclaw approvals set --node
<id|name|ip>` om
goedkeuringen voor een Gateway-host of Node-host op afstand te wijzigen.

Andere ingebouwde voorinstellingen: `cautious` (`host=gateway`, `security=allowlist`,
`ask=on-miss`, `askFallback=deny`) en `deny-all` (`host=gateway`,
`security=deny`, `ask=off`, `askFallback=deny`). Pas ze op dezelfde manier toe:
`openclaw exec-policy preset cautious`.

Gebruik `openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>` met een willekeurige subset van die vlaggen
om afzonderlijke velden in te stellen in plaats van een volledige voorinstelling.

### Node-host

Pas in plaats daarvan hetzelfde goedkeuringsbestand toe op de Node:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**Beperkingen voor uitsluitend lokaal gebruik:**

- `openclaw exec-policy` synchroniseert geen Node-goedkeuringen.
- `openclaw exec-policy set --host node` wordt afgewezen.
- Exec-goedkeuringen voor een Node worden tijdens runtime van de Node opgehaald, dus updates die op een Node zijn gericht, moeten `openclaw approvals --node ...` gebruiken.

</Note>

### Snelkoppeling voor alleen de sessie

- `/exec security=full ask=off` wijzigt alleen de huidige sessie.
- `/elevated full` is een noodsnelkoppeling die exec-goedkeuringen alleen overslaat
  wanneer zowel het gevraagde beleid als het goedkeuringsbestand van de host worden herleid tot
  `security: "full"` en `ask: "off"`. Een strenger hostbestand, zoals `ask:
"always"`, blijft om goedkeuring vragen.

Als het goedkeuringsbestand van de host strenger blijft dan de configuratie, blijft het strengere
hostbeleid bepalend.

## Toestaanlijst (per agent)

Toestaanlijsten gelden **per agent**. Als er meerdere agents bestaan, wissel je in
de macOS-app van agent om te bepalen welke je bewerkt. Patronen zijn glob-overeenkomsten.

Patronen kunnen globs voor opgeloste paden naar binaire bestanden of globs met alleen opdrachtnamen zijn.
Losse namen komen alleen overeen met opdrachten die via `PATH` worden aangeroepen, zodat `rg` kan overeenkomen met
`/opt/homebrew/bin/rg` wanneer de opdracht `rg` is, maar **niet** met `./rg` of
`/tmp/rg`. Gebruik een padglob om één specifieke locatie van een binair bestand te vertrouwen.

Verouderde `agents.default`-vermeldingen worden tijdens het laden naar `agents.main` gemigreerd.
Bij shellketens zoals `echo ok && pwd` moet elk segment op het hoogste niveau nog steeds
aan de regels van de toestaanlijst voldoen.

Voorbeelden:

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### Argumenten beperken met argPattern

Voeg `argPattern` toe wanneer een vermelding in de toestaanlijst moet overeenkomen met een binair bestand en een
specifieke argumentstructuur. OpenClaw gebruikt op elke host de semantiek van reguliere
ECMAScript-expressies (JavaScript) en evalueert de expressie aan de hand van
de geparseerde opdrachtargumenten, exclusief het uitvoerbare token (`argv[0]`).
Bij handmatig gemaakte vermeldingen worden argumenten met één spatie samengevoegd; veranker
het patroon daarom wanneer je een exacte overeenkomst nodig hebt.

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

Die vermelding staat `python3 safe.py` toe; `python3 other.py` komt niet overeen met de toestaanlijst.
Als er ook een vermelding met alleen een pad voor hetzelfde binaire bestand aanwezig is, kunnen niet-overeenkomende
argumenten nog steeds terugvallen op die vermelding met alleen een pad. Laat de vermelding met alleen een pad
weg wanneer het doel is om het binaire bestand tot de opgegeven argumenten te beperken.

Vermeldingen die door goedkeuringsflows worden opgeslagen, gebruiken een interne scheidingsindeling voor exacte
argv-overeenkomsten. Gebruik bij voorkeur de gebruikersinterface of goedkeuringsflow om die vermeldingen opnieuw te genereren
in plaats van de gecodeerde waarde handmatig te bewerken. Als OpenClaw argv voor
een opdrachtsegment niet kan parseren, komen vermeldingen met `argPattern` niet overeen.

Gegenereerde `allow-always`-vermeldingen zijn aan argv gebonden. Nieuwe gegenereerde vermeldingen bevatten
`argPattern`; oudere gegenereerde vermeldingen met alleen een pad worden genegeerd en vereisen een nieuwe
goedkeuring. Laat voor een handmatige regel met alleen een pad zowel `source` als `argPattern` weg.

Elke vermelding in de toestaanlijst ondersteunt:

| Veld               | Betekenis                                                                |
| ------------------ | ------------------------------------------------------------------------ |
| `pattern`          | Glob voor het opgeloste pad naar het binaire bestand of glob met alleen de opdrachtnaam |
| `argPattern`       | ECMAScript-regex voor argv of gegenereerde hash voor exacte argv; weglaten betekent alleen pad |
| `id`               | Stabiele ondoorzichtige ID; wordt bij afwezigheid als UUID gegenereerd    |
| `source`           | Bron van de gegenereerde vermelding, zoals `allow-always`; weglaten voor handmatige vermeldingen |
| `commandText`      | Verouderde invoer als platte tekst; wordt tijdens het laden verwijderd    |
| `lastUsedAt`       | Tijdstempel van het laatste gebruik                                      |
| `lastUsedCommand`  | Laatste opdracht die overeenkwam; weggelaten bij gegenereerde vermeldingen met gehashte argv |
| `lastResolvedPath` | Laatst opgeloste pad naar het binaire bestand                             |

## CLI's van Skills automatisch toestaan

Wanneer **CLI's van Skills automatisch toestaan** (`autoAllowSkills`) is ingeschakeld, worden uitvoerbare bestanden
waarnaar bekende Skills verwijzen, op Nodes als toegestaan beschouwd (macOS-Node
of headless Node-host). Hiervoor wordt `skills.bins` via de Gateway-RPC gebruikt om
de lijst met binaire bestanden van Skills op te halen. Schakel dit uit als je strikt handmatige
toestaanlijsten wilt.

<Warning>
- Dit is een **impliciete toestaanlijst voor gebruiksgemak**, los van handmatige vermeldingen met toegestane paden.
- Deze is bedoeld voor vertrouwde beheeromgevingen waarin de Gateway en Node zich binnen dezelfde vertrouwensgrens bevinden.
- Als je strikt expliciet vertrouwen vereist, behoud dan `autoAllowSkills: false` en gebruik uitsluitend handmatige vermeldingen met toegestane paden.

</Warning>

## Veilige binaire bestanden en doorsturen van goedkeuringen

Zie voor veilige binaire bestanden (het snelle pad met uitsluitend stdin), details over het binden van interpreters en
het doorsturen van goedkeuringsvragen naar Slack/Discord/Telegram (of het uitvoeren ervan als
native goedkeuringsclients)
[Exec-goedkeuringen - geavanceerd](/nl/tools/exec-approvals-advanced).

## Bewerken in de Control UI

Gebruik de kaart **Control UI -> Nodes -> Exec approvals** om standaardwaarden,
overschrijvingen per agent en toestaanlijsten te bewerken. Kies een bereik (Defaults of een agent),
pas het beleid aan, voeg patronen aan de toestaanlijst toe of verwijder ze en kies vervolgens **Save**. De gebruikersinterface
toont per patroon metagegevens over het laatste gebruik, zodat je de lijst overzichtelijk kunt houden.

De doelselector kiest **Gateway** (lokale goedkeuringen) of een **Node**.
Nodes moeten `system.execApprovals.get/set` aankondigen (macOS-app of headless
Node-host). Als een Node nog geen exec-goedkeuringen aankondigt, bewerk je het
lokale goedkeuringsbestand rechtstreeks.

Sommige Node-hosts, waaronder de Windows-companion, gebruiken een andere indeling voor het
goedkeuringsbeleid. De Control UI toont dit systeemeigen hostbeleid als alleen-lezen. Gebruik de
companion-app of `openclaw approvals set --node <id|name|ip>` met de systeemeigen
beleidsstructuur om het te bewerken; zie [CLI voor goedkeuringen](/nl/cli/approvals).

CLI: `openclaw approvals` ondersteunt het bewerken van de Gateway of een Node — zie
[CLI voor goedkeuringen](/nl/cli/approvals).

## Goedkeuringsflow

Wanneer goedkeuring vereist is, zendt de Gateway
`exec.approval.requested` uit naar beheerclients. De Control UI en macOS-app
handelen dit af via `exec.approval.resolve`, waarna de Gateway het
goedgekeurde verzoek doorstuurt naar de Node-host.

Voor `host=node` bevatten goedkeuringsverzoeken een canonieke `systemRunPlan`-payload.
De Gateway gebruikt dat plan als de gezaghebbende context voor opdracht/cwd/sessie
wanneer goedgekeurde `system.run`-verzoeken worden doorgestuurd:

- Het exec-pad van de Node bereidt vooraf één canoniek plan voor.
- De goedkeuringsrecord slaat dat plan en de bijbehorende bindingsmetagegevens op.
- Na goedkeuring hergebruikt de uiteindelijke doorgestuurde `system.run`-aanroep het opgeslagen plan in plaats van latere wijzigingen van de aanroeper te vertrouwen.
- Als de aanroeper `command`, `rawCommand`, `cwd`, `agentId` of `sessionKey` wijzigt nadat het goedkeuringsverzoek is gemaakt, wijst de Gateway de doorgestuurde uitvoering af wegens een afwijking van de goedkeuring.

## Systeemgebeurtenissen en afwijzingen

De exec-levenscyclus plaatst een `Exec finished`-systeembericht in de sessie van de agent
nadat de Node voltooiing meldt. OpenClaw kan ook een melding over de voortgang verzenden zodra een goedkeuring is verleend, nadat
`tools.exec.approvalRunningNoticeMs` is verstreken (standaard `10000`; `0` schakelt
dit uit). Afgewezen exec-goedkeuringen beëindigen de hostopdracht definitief: de opdracht
wordt niet uitgevoerd.

- Bij asynchrone goedkeuringen voor de hoofdagent met een oorspronkelijke sessie plaatst OpenClaw
  de afwijzing als interne vervolgmelding terug in die sessie, zodat de
  agent niet langer op de asynchrone opdracht wacht en herstel wegens een ontbrekend resultaat
  wordt voorkomen.
- Als er geen sessie is of de sessie niet kan worden hervat, kan OpenClaw
  nog steeds een beknopte afwijzing melden aan de beheerder of via de rechtstreekse chatroute.
- Afwijzingen voor subagent- en Cron-sessies worden niet teruggeplaatst in die
  sessie.

Exec-goedkeuringen voor de Gateway-host zenden dezelfde gebeurtenis voor voltooiing van de levenscyclus uit.
Exec-opdrachten waarvoor goedkeuring vereist is, hergebruiken de goedkeurings-ID om het openstaande
verzoek aan het voltooiings- of afwijzingsbericht te koppelen (`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`).

## Gevolgen

- **`full`** is krachtig; geef waar mogelijk de voorkeur aan toestaanlijsten.
- **`ask`** houdt je op de hoogte en maakt snelle goedkeuringen toch mogelijk.
- Toestaanlijsten per agent voorkomen dat goedkeuringen van de ene agent naar andere agents doorlekken.
- Goedkeuringen zijn alleen van toepassing op exec-verzoeken voor de host van **geautoriseerde afzenders**. Niet-geautoriseerde afzenders kunnen geen `/exec` uitvoeren.
- `/exec security=full` is een hulpmiddel op sessieniveau voor geautoriseerde beheerders en slaat goedkeuringen bewust over. Stel de beveiliging voor goedkeuringen in op `deny` of weiger de tool `exec` via het toolbeleid om exec op de host volledig te blokkeren.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Exec-goedkeuringen - geavanceerd" href="/nl/tools/exec-approvals-advanced" icon="gear">
    Veilige binaire bestanden, interpreterkoppeling en doorsturen van goedkeuringen naar de chat.
  </Card>
  <Card title="Exec-tool" href="/nl/tools/exec" icon="terminal">
    Tool voor het uitvoeren van shell-opdrachten.
  </Card>
  <Card title="Verhoogde modus" href="/nl/tools/elevated" icon="shield-exclamation">
    Noodroute die ook goedkeuringen overslaat.
  </Card>
  <Card title="Sandboxing" href="/nl/gateway/sandboxing" icon="box">
    Sandboxmodi en toegang tot de werkruimte.
  </Card>
  <Card title="Beveiliging" href="/nl/gateway/security" icon="lock">
    Beveiligingsmodel en versterking.
  </Card>
  <Card title="Sandbox versus toolbeleid versus verhoogde modus" href="/nl/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    Wanneer je welk besturingselement gebruikt.
  </Card>
  <Card title="Skills" href="/nl/tools/skills" icon="sparkles">
    Door Skills ondersteund automatisch toestaan.
  </Card>
</CardGroup>
