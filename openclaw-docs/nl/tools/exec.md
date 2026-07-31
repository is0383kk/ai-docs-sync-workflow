---
read_when:
    - De exec-tool gebruiken of aanpassen
    - Problemen met stdin- of TTY-gedrag opsporen
summary: Gebruik van de exec-tool, stdin-modi en TTY-ondersteuning
title: Uitvoeringstool
x-i18n:
    generated_at: "2026-07-27T05:36:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

Voer shellopdrachten uit in de werkruimte. `exec` is een muterend shelloppervlak: opdrachten kunnen bestanden maken, bewerken of verwijderen waar het bestandssysteem van de geselecteerde host of sandbox dit toestaat. Het uitschakelen van OpenClaw-bestandssysteemtools zoals `write`, `edit` of `apply_patch` maakt `exec` niet alleen-lezen.

Ondersteunt uitvoering op de voorgrond en achtergrond via `process`. Als `process` niet is toegestaan, wordt `exec` synchroon uitgevoerd en worden `yieldMs`/`background` genegeerd. Achtergrondsessies zijn per agent afgebakend; `process` ziet alleen sessies van dezelfde agent.

## Parameters

<ParamField path="command" type="string" required>
Uit te voeren shellopdracht.
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
Werkmap voor de opdracht.
</ParamField>

<ParamField path="env" type="object">
Omgevingsoverschrijvingen met sleutel/waarde-paren die boven op de overgenomen omgeving worden samengevoegd.
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
Plaats de opdracht na deze vertraging (ms) automatisch op de achtergrond.
</ParamField>

<ParamField path="background" type="boolean" default="false">
Plaats de opdracht onmiddellijk op de achtergrond in plaats van te wachten op `yieldMs`.
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
Overschrijf voor deze aanroep de geconfigureerde uitvoeringstime-out, in seconden. Geldt voor uitvoering op de voorgrond, achtergrond, `yieldMs`, Gateway, sandbox en Node `system.run`. `timeout: 0` schakelt de time-out van het uitvoeringsproces voor die aanroep uit.
</ParamField>

<ParamField path="pty" type="boolean" default="false">
Voer indien beschikbaar uit in een pseudoterminal. Gebruik dit voor CLI's die alleen met een TTY werken, programmeeragenten en terminalinterfaces.
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
Waar de uitvoering plaatsvindt. `auto` wordt omgezet naar `sandbox` wanneer een sandboxruntime actief is, en anders naar `gateway`.
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
Genegeerd voor normale toolaanroepen. De beveiliging van `gateway`/`node` wordt afgeleid van `tools.exec.mode` en het hostgoedkeuringsbestand; de verhoogde modus kan alleen volledige toegang afdwingen wanneer de operator expliciet verhoogde toegang verleent.
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
De basisvraagmodus wordt afgeleid van `tools.exec.mode` en hostgoedkeuringen. Voor modelaanroepen die vanuit een kanaal afkomstig zijn, wordt `ask` per aanroep genegeerd wanneer de effectieve hostvraagmodus `off` is; anders kan deze alleen worden aangescherpt naar een strengere modus.
</ParamField>

<ParamField path="node" type="string">
Node-id/-naam wanneer `host=node`.
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
Vraag de verhoogde modus aan: verlaat de sandbox naar het geconfigureerde hostpad. `security=full` wordt alleen afgedwongen wanneer verhoogd wordt omgezet naar `full`.
</ParamField>

Opmerkingen:

- `host` accepteert alleen `auto`, `sandbox`, `gateway` of `node`. Het is geen hostnaamkiezer; waarden die op hostnamen lijken, worden geweigerd voordat de opdracht wordt uitgevoerd.
- `host=node` per aanroep is toegestaan vanuit `auto`; `host=gateway` per aanroep is alleen toegestaan wanneer er geen sandboxruntime actief is.
- Zonder aanvullende configuratie werkt `host=auto` nog steeds direct: zonder sandbox wordt het omgezet naar `gateway`; met een actieve sandbox blijft het in de sandbox.
- `elevated` verlaat de sandbox naar het geconfigureerde hostpad: standaard `gateway`, of `node` wanneer `tools.exec.host=node` (of de sessiestandaard `host=node` is). Dit is alleen beschikbaar wanneer verhoogde toegang is ingeschakeld voor de huidige sessie/provider.
- Goedkeuringen voor `gateway`/`node` worden beheerd door het hostgoedkeuringsbestand.
- `node` vereist een gekoppelde Node (begeleidende app of headless Node-host). Als er meerdere Nodes beschikbaar zijn, stel je `exec.node` of `tools.exec.node` in om er één te selecteren.
- `exec host=node` is het enige pad voor shelluitvoering op Nodes; de verouderde wrapper `nodes.run` is verwijderd.
- Op niet-Windows-hosts gebruikt exec `SHELL` wanneer dit is ingesteld; als `SHELL` gelijk is aan `fish`, geeft het de voorkeur aan `bash` (of `sh`) uit `PATH` om bash-constructies te vermijden die niet compatibel zijn met fish, en valt het vervolgens terug op `SHELL` als geen van beide bestaat.
- Op Windows-hosts geeft exec de voorkeur aan detectie van PowerShell 7 (`pwsh`) (Program Files, ProgramW6432 en daarna PATH), en valt het vervolgens terug op Windows PowerShell 5.1.
- Op niet-Windows-Gateway-hosts gebruiken exec-opdrachten van bash en zsh een opstartmomentopname. OpenClaw legt sourcebare aliassen/functies en een kleine, veilige omgevingsset uit shellopstartbestanden vast in `$OPENCLAW_STATE_DIR/cache/shell-snapshots/` en sourcet die momentopname vervolgens vóór elke exec-opdracht. Variabelen die op geheimen lijken, worden uitgesloten; exec in sandbox en Node gebruikt deze momentopname niet. Stel `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in de procesomgeving van de Gateway in om dit momentopnamepad uit te schakelen.
- Hostuitvoering (`gateway`/`node`) weigert `env.PATH` en loaderoverschrijvingen (`LD_*`/`DYLD_*`) om kaping van binaire bestanden of geïnjecteerde code te voorkomen.
- OpenClaw stelt `OPENCLAW_SHELL=exec` in de omgeving van de gestarte opdracht in (inclusief PTY- en sandboxuitvoering), zodat shell-/profielregels de context van de exec-tool kunnen detecteren.
- Voor uitvoeringen die vanuit een kanaal afkomstig zijn, stelt OpenClaw ook een beperkte JSON-payload met de identiteit van de afzender/chat beschikbaar in `OPENCLAW_CHANNEL_CONTEXT` wanneer het kanaal die id's heeft verstrekt.
- `exec` kan de shellopdrachten `openclaw channels login` of `/approve` niet uitvoeren: `openclaw channels login` is een interactieve kanaalauthenticatiestroom en `/approve` moet via de opdrachtverwerker voor goedkeuringen verlopen, niet via een shell. Voer kanaalaanmelding uit in een terminal op de Gateway-host, of gebruik een kanaalspecifieke agenttool voor aanmelding wanneer die bestaat (bijvoorbeeld `whatsapp_login`).
- Belangrijk: sandboxing is **standaard uitgeschakeld**. Als sandboxing is uitgeschakeld, wordt impliciete `host=auto` omgezet naar `gateway`. Expliciete `host=sandbox` blijft veilig falen in plaats van stilzwijgend op de Gateway-host te worden uitgevoerd. Schakel sandboxing in of gebruik `host=gateway` met goedkeuringen.
- Voorafgaande scriptcontroles (voor veelvoorkomende fouten in Python-/Node-shellsyntaxis) inspecteren alleen bestanden binnen de effectieve grens van `workdir`. Als een scriptpad buiten `workdir` wordt omgezet, wordt de voorafgaande controle voor dat bestand overgeslagen. De voorafgaande controle wordt ook volledig overgeslagen wanneer `host=gateway` en het effectieve beleid `security=full` met `ask=off` is.
- Voor langdurig werk dat nu begint, start je het eenmaal en vertrouw je op de automatische voltooiingswake-up wanneer die is ingeschakeld en de opdracht uitvoer produceert of mislukt. Gebruik `process` voor logboeken, status, invoer of interventie; boots planning niet na met slaaplussen, time-outlussen of herhaald pollen.
- Door een agent gestarte achtergrondopdrachten verschijnen in de achtergrondtaakweergaven van Web, iOS en Android totdat ze zijn voltooid. Het taaklogboek wordt afgerond voordat de Heartbeat voor voltooiing de agent opnieuw activeert.
- Gebruik Cron in plaats van slaap-/vertragingspatronen met `exec` voor werk dat later of volgens een planning moet plaatsvinden.

## Configuratie

| Sleutel                               | Standaard                | Opmerkingen                                                                                                                                             |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | Standaardtime-out per exec-opdracht in seconden. `timeout` per aanroep overschrijft deze; `timeout: 0` per aanroep schakelt de time-out van het exec-proces uit. |
| `tools.exec.host`                    | `auto`                   | Wordt omgezet naar `sandbox` wanneer een sandboxruntime actief is, en anders naar `gateway`.                                                     |
| `tools.exec.mode`                    | afgeleid van host        | Canonieke beleidsinstelling. Zie [Modi](#modes) hieronder.                                                                                              |
| `tools.exec.reviewer.model`          | geconfigureerd primair agentmodel | Optionele provider-/modeloverschrijving voor beoordeling door `mode=auto`.                                                                  |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | Time-out per fase voor voorbereiding en voltooiing door het beoordelaarsmodel voordat wordt teruggevallen op een mens.                                  |
| `tools.exec.node`                    | niet ingesteld           |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | Indien waar, plaatsen exec-sessies op de achtergrond bij afsluiten een systeemgebeurtenis in de wachtrij en vragen ze een Heartbeat aan.                 |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | Geef één melding "wordt uitgevoerd" wanneer een exec waarvoor goedkeuring nodig is langer duurt dan dit (`0` schakelt dit uit).          |
| `tools.exec.strictInlineEval`        | `false`                  | Zie [Inline-evaluatie](#inline-eval-strictinlineeval).                                                                                                  |
| `tools.exec.commandHighlighting`     | `false`                  | Indien waar, kunnen goedkeuringsprompts door de parser afgeleide opdrachtsegmenten in de opdrachttekst markeren. Stel dit globaal of per agent in; dit wijzigt het goedkeuringsbeleid niet. |
| `tools.exec.pathPrepend`             | niet ingesteld           | Lijst met mappen die vóór `PATH` moeten worden geplaatst voor exec-uitvoeringen (alleen Gateway + sandbox).                                 |
| `tools.exec.safeBins`                | niet ingesteld           | Veilige binaire bestanden die alleen stdin gebruiken en zonder expliciete vermeldingen in de toelatingslijst kunnen worden uitgevoerd. Zie [Veilige binaire bestanden](/nl/tools/exec-approvals-advanced#safe-bins-stdin-only). |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | Aanvullende expliciete mappen die worden vertrouwd voor padcontroles van `safeBins`. Vermeldingen in `PATH` worden nooit automatisch vertrouwd. |
| `tools.exec.safeBinProfiles`         | niet ingesteld           | Optioneel aangepast argv-beleid per veilig binair bestand (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).             |

Host-exec zonder goedkeuring is de standaard voor Gateway en Node (`mode=full`) — dit komt voort uit de standaardwaarden van het hostbeleid, niet uit `host=auto`. Als je gedrag met goedkeuringen/toelatingslijsten wilt, stel je `tools.exec.mode` in en verscherp je het hostgoedkeuringsbestand; zie [Exec-goedkeuringen](/nl/tools/exec-approvals#yolo-mode-no-approval). Stel `tools.exec.host` in of gebruik `/exec host=...` om routering naar Gateway of Node af te dwingen, ongeacht de sandboxstatus.

Voorbeeld:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Modi

`tools.exec.mode` is de canonieke persistente beleidsinstelling. Runtimebeveiliging en goedkeuringsgedrag worden ervan afgeleid.

| Modus       | beveiliging | vragen    | Gedrag                                                                                                                                |
| ----------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | `deny`      | `off`     | Uitvoering wordt geweigerd.                                                                                                           |
| `allowlist` | `allowlist` | `off`     | Alleen opdrachten op de toelatingslijst of opdrachten met veilige binaire bestanden worden uitgevoerd; voor niets anders wordt toestemming gevraagd. |
| `ask`       | `allowlist` | `on-miss` | Overeenkomsten met de toelatingslijst worden direct uitgevoerd; voor al het overige wordt een mens om toestemming gevraagd.            |
| `auto`      | `allowlist` | `on-miss` | Overeenkomsten met de toelatingslijst of veilige binaire bestanden worden direct uitgevoerd; al het overige gaat eerst door de ingebouwde automatische reviewer van OpenClaw voordat een mens om toestemming wordt gevraagd. |
| `full`      | `full`      | `off`     | Geen goedkeuringspoort.                                                                                                               |

`/exec ask=always` per sessie vraagt nog steeds elke keer een mens om toestemming, ongeacht de opgeslagen modus.

Goedkeuring door automatische review is eenmalig. Op de Gateway geeft OpenClaw het opgeloste pad van het uitvoerbare bestand door aan de reviewer en zet het de uitvoering vast op datzelfde pad. Opdrachten die niet tot één afdwingbaar uitvoeringsplan kunnen worden teruggebracht, zoals heredocs, shelluitbreidingen of niet-ondersteunde aanhalingstekens voor wrappers, vallen terug op menselijke goedkeuring, zelfs als het model ze anders zou toestaan.

Goedkeuringen voor opdrachten van de Codex-appserver waarover nog niet door expliciet runtime- of ingebouwd beleid is beslist, gebruiken het menselijke goedkeuringstraject. OpenClaw voert de geconfigureerde uitvoeringsreviewer niet uit voor deze verzoeken, omdat Codex geen afdwingbaar opgelost uitvoerbaar bestand beschikbaar stelt waarmee de reviewbeslissing kan worden gekoppeld aan de opdracht die Codex uitvoert.

### Inline-evaluatie (`strictInlineEval`)

Wanneer `tools.exec.strictInlineEval` `true` is, vereisen inline-evaluatievormen voor interpreters goedkeuring door een reviewer of expliciete goedkeuring: `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, `osascript -e` en vergelijkbare vormen in andere ondersteunde interpreters en opdrachtdragers (`awk`, `find -exec`, `make`, `sed`, `xargs` en meer). In `mode=auto` kan het normale goedkeuringstraject voor uitvoering de ingebouwde automatische reviewer een duidelijk eenmalige opdracht met laag risico laten toestaan; rechtstreekse `system.run`-aanroepen naar de nodehost vereisen nog steeds expliciete goedkeuring, omdat ze de opdracht niet aan een menselijk goedkeuringstraject kunnen doorgeven. Als de reviewer daarom vraagt, gaat het verzoek naar een mens. `allow-always` kan onschuldige aanroepen van interpreters/scripts nog steeds permanent opslaan, maar inline-evaluatievormen worden geen permanente toelatingsregels.

### PATH-verwerking

- `host=gateway`: voegt de `PATH` van je loginshell samen met de uitvoeringsomgeving. Overschrijvingen van `env.PATH` worden geweigerd voor uitvoering op de host. De daemon zelf wordt nog steeds uitgevoerd met een minimale `PATH`:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
  - Om te voorkomen dat de shellconfiguratie van de gebruiker (zoals `~/.zshenv` of `/etc/zshenv`) prioriteitspaden tijdens het opstarten overschrijft, worden `tools.exec.pathPrepend`-items vlak voor uitvoering veilig vooraan aan de uiteindelijke `PATH` in de shellopdracht toegevoegd.
- `host=sandbox`: voert `sh -lc` (loginshell) uit in de container, waardoor `/etc/profile` mogelijk `PATH` opnieuw instelt. OpenClaw voegt `env.PATH` na het laden van het profiel vooraan toe via een interne omgevingsvariabele (zonder shellinterpolatie); `tools.exec.pathPrepend` is hier ook van toepassing.
- `host=node`: alleen niet-geblokkeerde omgevingsoverschrijvingen die je doorgeeft, worden naar de node verzonden. Overschrijvingen van `env.PATH` worden geweigerd voor uitvoering op de host en genegeerd door nodehosts. Als je aanvullende PATH-items op een node nodig hebt, configureer je de omgeving van de nodehostservice (systemd/launchd) of installeer je hulpprogramma's op standaardlocaties.

Nodebinding per agent (gebruik de agent-ID als sleutel in de configuratie):

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Control UI: de pagina **Apparaten** bevat een klein paneel 'Binding van uitvoeringsnode' voor dezelfde instellingen.

## Sessieoverschrijvingen (`/exec`)

Gebruik `/exec` om **per sessie** standaardwaarden in te stellen voor `host`, `security`, `ask` en `node`. Stuur `/exec` zonder argumenten om de huidige waarden weer te geven.

Voorbeeld:

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

`/exec` wordt alleen gehonoreerd voor **geautoriseerde afzenders** via kanaaltoelatingslijsten/koppeling en toegangsgroepen. Handhaving van toegangsgroepen is altijd actief. Hiermee wordt alleen de **sessiestatus** bijgewerkt en geen configuratie geschreven. Geautoriseerde afzenders van externe kanalen mogen deze standaardsessie-instellingen bepalen. Interne Gateway-/webchatclients hebben `operator.admin` nodig om ze permanent op te slaan.

Om uitvoering volledig uit te schakelen, weiger je deze via het hulpmiddelenbeleid (`tools.deny: ["exec"]` of per agent). Hostgoedkeuringen blijven van toepassing, tenzij je `security=full` en `ask=off` expliciet instelt.

## Uitvoeringsgoedkeuringen (begeleidende app/nodehost)

Agents in een sandbox kunnen goedkeuring per verzoek vereisen voordat `exec` op de Gateway of nodehost wordt uitgevoerd. Zie [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals) voor het beleid, de toelatingslijst en de UI-flow.

Wanneer menselijke goedkeuring vereist is, retourneren nodehost- en niet-ingebouwde Gateway-flows onmiddellijk `status: "approval-pending"` en een goedkeurings-ID. Ingebouwde chat- en Web UI-Gateway-flows kunnen in plaats daarvan inline wachten en na goedkeuring het uiteindelijke opdrachtresultaat retourneren. Een `approval-pending`-resultaat betekent dat de opdracht niet is gestart, zodat waarschuwingen over terugval naar de voorgrond alleen verschijnen als de goedgekeurde opdracht daadwerkelijk inline wordt uitgevoerd. Goedgekeurde asynchrone uitvoeringen genereren systeemgebeurtenissen voor opdrachtvoortgang en -voltooiing (`Exec running` / `Exec finished`); geweigerde of verlopen goedkeuringen zijn definitief en activeren de agentsessie niet met een systeemgebeurtenis over de weigering.

Op kanalen met ingebouwde goedkeuringskaarten/-knoppen moet de agent eerst op die ingebouwde UI vertrouwen en alleen een handmatige `/approve`-opdracht opnemen wanneer het hulpmiddelresultaat expliciet aangeeft dat chatgoedkeuringen niet beschikbaar zijn of dat handmatige goedkeuring de enige mogelijkheid is.

## Toelatingslijst + veilige binaire bestanden

Handmatige handhaving van de toelatingslijst vergelijkt globs van opgeloste paden naar binaire bestanden en globs van kale opdrachtnamen. Kale namen komen alleen overeen met opdrachten die via PATH worden aangeroepen. `rg` kan dus overeenkomen met `/opt/homebrew/bin/rg` wanneer de opdracht `rg` is, maar niet met `./rg` of `/tmp/rg`.

Wanneer `security=allowlist`, worden shellopdrachten alleen automatisch toegestaan als elk pijplijnsegment op de toelatingslijst staat of een veilig binair bestand is. Koppelingen (`;`, `&&`, `||`) en omleidingen worden in de toelatingslijstmodus geweigerd, tenzij elk segment op het hoogste niveau aan de toelatingslijst voldoet (inclusief veilige binaire bestanden). Omleidingen blijven niet ondersteund. Permanent `allow-always`-vertrouwen omzeilt die regel niet: voor een gekoppelde opdracht moet nog steeds elk segment op het hoogste niveau overeenkomen.

`autoAllowSkills` is een afzonderlijk gemakstraject in uitvoeringsgoedkeuringen en niet hetzelfde als handmatige paditems in de toelatingslijst. Houd `autoAllowSkills` uitgeschakeld voor strikt expliciet vertrouwen.

Gebruik de twee besturingselementen voor verschillende doeleinden:

- `tools.exec.safeBins`: kleine streamfilters die alleen stdin gebruiken.
- `tools.exec.safeBinTrustedDirs`: expliciete aanvullende vertrouwde mappen voor paden naar uitvoerbare veilige binaire bestanden.
- `tools.exec.safeBinProfiles`: expliciet argv-beleid voor aangepaste veilige binaire bestanden.
- toelatingslijst: expliciet vertrouwen voor paden naar uitvoerbare bestanden.

Beschouw `safeBins` niet als een algemene toelatingslijst en voeg geen binaire bestanden van interpreters/runtimes toe (bijvoorbeeld `python3`, `node`, `ruby`, `bash`). Als je die nodig hebt, gebruik je expliciete toelatingslijstitems en houd je goedkeuringsprompts ingeschakeld.

`openclaw security audit` waarschuwt wanneer `safeBins`-items voor interpreters/runtimes geen expliciete profielen hebben, en `openclaw doctor --fix` kan ontbrekende aangepaste `safeBinProfiles`-items opzetten. `openclaw security audit` en `openclaw doctor` waarschuwen ook wanneer je expliciet binaire bestanden met breed gedrag, zoals `jq`, opnieuw toevoegt aan `safeBins` (`jq` kan omgevingsgegevens lezen en jq-code uit modules of opstartbestanden laden; geef daarom de voorkeur aan expliciete toelatingslijstitems of uitvoeringen met een goedkeuringspoort). `jq` wordt als veilig binair bestand geweigerd, zelfs wanneer het expliciet in de lijst staat. Als je interpreters expliciet op de toelatingslijst zet, schakel je `tools.exec.strictInlineEval` in, zodat inline code-evaluatievormen nog steeds goedkeuring door een reviewer of expliciete goedkeuring vereisen.

Zie [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals-advanced#safe-bins-stdin-only) en [Veilige binaire bestanden versus toelatingslijst](/nl/tools/exec-approvals-advanced#safe-bins-versus-allowlist) voor volledige beleidsdetails en voorbeelden.

## Voorbeelden

Voorgrond:

```json
{ "tool": "exec", "command": "ls -la" }
```

Achtergrond + pollen:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Pollen is bedoeld voor status op aanvraag, niet voor wachtlussen. Als automatisch activeren bij voltooiing is ingeschakeld, kan de opdracht de sessie activeren wanneer deze uitvoer genereert of mislukt.

Toetsaanslagen verzenden (tmux-stijl):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Indienen (alleen CR verzenden):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Plakken (standaard tussen haakjes):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` is een subhulpmiddel van `exec` voor gestructureerde bewerkingen van meerdere bestanden. Het is standaard ingeschakeld en beschikbaar voor elke modelprovider; `allowModels` kan het beperken. Gebruik de configuratie alleen wanneer je het wilt uitschakelen of tot specifieke modellen wilt beperken:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

Opmerkingen:

- Het hulpmiddelenbeleid blijft van toepassing; `allow: ["write"]` staat `apply_patch` impliciet toe.
- `deny: ["write"]` weigert `apply_patch` niet; weiger `apply_patch` expliciet of gebruik `deny: ["group:fs"]` wanneer patchbewerkingen ook moeten worden geblokkeerd.
- De configuratie bevindt zich onder `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` is standaard `true`; stel dit in op `false` om het hulpmiddel uit te schakelen.
- `tools.exec.applyPatch.workspaceOnly` is standaard `true` (beperkt tot de werkruimte). Stel dit alleen in op `false` als je bewust wilt dat `apply_patch` buiten de werkruimtemap schrijft/verwijdert.
- `tools.exec.applyPatch.allowModels` is een optionele toelatingslijst met model-ID's (onbewerkt, zoals `gpt-5.4`, of volledig, zoals `openai/gpt-5.4`). Wanneer deze is ingesteld, krijgen alleen overeenkomende modellen het hulpmiddel; wanneer deze niet is ingesteld, krijgen alle modellen het.

## Gerelateerd

- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals) — goedkeuringspoorten voor shellopdrachten
- [Sandboxing](/nl/gateway/sandboxing) — opdrachten uitvoeren in sandboxomgevingen
- [Achtergrondproces](/nl/gateway/background-process) — hulpmiddelen voor langdurige uitvoeringen en processen
- [Beveiliging](/nl/gateway/security) — hulpmiddelenbeleid en verhoogde toegang
