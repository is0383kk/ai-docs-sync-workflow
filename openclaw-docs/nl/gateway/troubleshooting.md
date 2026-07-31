---
read_when:
    - De hub voor probleemoplossing heeft je hierheen verwezen voor een diepgaandere diagnose
    - Je hebt stabiele, op symptomen gebaseerde runbooksecties met exacte opdrachten nodig
sidebarTitle: Troubleshooting
summary: Uitgebreid draaiboek voor probleemoplossing voor Gateway, kanalen, automatisering, Nodes en browser
title: Probleemoplossing
x-i18n:
    generated_at: "2026-07-27T05:01:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4bb1e061dbf2767118c24ad1ca2d2d1f7eeeff88e18ed0e6111aebe1cc99a26
    source_path: gateway/troubleshooting.md
    workflow: 16
---

Dit is het uitgebreide draaiboek. Begin eerst bij [/help/problemen-oplossen](/nl/help/troubleshooting) voor de snelle triageflow.

## Commandovolgorde

Voer de volgende opdrachten in deze volgorde uit:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Signalen van een gezonde werking:

- `openclaw gateway status` toont `Runtime: running`, `Connectivity probe: ok` en een `Capability: ...`-regel.
- `openclaw doctor` meldt geen blokkerende configuratie- of serviceproblemen.
- `openclaw channels status --probe` toont de actuele transportstatus per account en, waar ondersteund, `works` of `audit ok`.

## Na een update

Gebruik dit wanneer een update is voltooid, maar de Gateway niet actief is, kanalen leeg zijn of modelaanroepen mislukken met 401-fouten.

```bash
openclaw status --all
openclaw update status --json
openclaw gateway status --deep
openclaw doctor --fix
openclaw gateway restart
```

Let op:

- `Update restart` in `openclaw status` / `openclaw status --all`. Overdrachten die in behandeling zijn of zijn mislukt, bevatten de volgende opdracht die moet worden uitgevoerd.
- `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` onder Kanalen: de kanaalconfiguratie bestaat nog, maar de Plugin-registratie is mislukt voordat het kanaal kon worden geladen.
- 401-fouten van de provider na herauthenticatie: `openclaw doctor --fix` controleert op verouderde OAuth-authenticatieschaduwen per agent en verwijdert oude kopieën, zodat alle agents het huidige gedeelde profiel gebruiken.

## Gesplitste installaties en beveiliging tegen nieuwere configuraties

Gebruik dit wanneer een Gateway-service na een update onverwacht stopt, of wanneer uit de logboeken blijkt dat één `openclaw`-binair bestand ouder is dan de versie die `openclaw.json` voor het laatst heeft geschreven.

OpenClaw markeert configuratieschrijfbewerkingen met `meta.lastTouchedVersion`. Alleen-lezenopdrachten kunnen een configuratie inspecteren die door een nieuwere OpenClaw is geschreven, maar proces- en servicemutaties worden niet uitgevoerd vanuit een ouder binair bestand. Geblokkeerde acties: de Gateway-service starten, stoppen, herstarten of verwijderen, geforceerde herinstallatie van de service, het starten van de Gateway in servicemodus en het opschonen van de `gateway --force`-poort.

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

<Steps>
  <Step title="PATH herstellen">
    Herstel `PATH` zodat `openclaw` naar de nieuwere installatie verwijst en voer de actie vervolgens opnieuw uit.
  </Step>
  <Step title="De Gateway-service opnieuw installeren">
    Installeer de beoogde Gateway-service opnieuw vanuit de nieuwere installatie:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  </Step>
  <Step title="Verouderde wrappers verwijderen">
    Verwijder verouderde systeempakketten of oude wrappervermeldingen die nog naar een oud `openclaw`-binair bestand verwijzen.
  </Step>
</Steps>

<Warning>
Stel uitsluitend voor een opzettelijke downgrade of noodherstel `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` in voor de afzonderlijke opdracht. Laat dit bij normaal gebruik uitgeschakeld.
</Warning>

## Protocolverschil na terugdraaien

Gebruik dit wanneer de logboeken na een downgrade of rollback `protocol mismatch` blijven weergeven. Er wordt een oudere Gateway uitgevoerd, maar een nieuwer lokaal clientproces probeert nog steeds opnieuw verbinding te maken met een protocolbereik dat de oudere Gateway niet ondersteunt.

```bash
openclaw --version
which -a openclaw
openclaw gateway status --deep
openclaw doctor --deep
openclaw logs --follow
```

Let op:

- `protocol mismatch ... client=... v<version> min=<n> max=<n> expected=<n>` in de Gateway-logboeken.
- `Established clients:` in `openclaw gateway status --deep` of `Gateway clients` in `openclaw doctor --deep`: actieve TCP-clients die zijn verbonden met de Gateway-poort, met PID's en opdrachtregels wanneer het besturingssysteem dit toestaat.
- Een clientproces waarvan de opdrachtregel verwijst naar de nieuwere OpenClaw-installatie of wrapper waarvan je bent teruggegaan.

Oplossing:

1. Stop of herstart het verouderde OpenClaw-clientproces dat door `gateway status --deep` wordt weergegeven.
2. Herstart apps of wrappers waarin OpenClaw is ingebed: lokale dashboards, editors, appserverhelpers of langlopende `openclaw logs --follow`-shells.
3. Voer `openclaw gateway status --deep` of `openclaw doctor --deep` opnieuw uit en bevestig dat de PID van de verouderde client verdwenen is.

Zorg er niet voor dat een oudere Gateway een nieuwer, incompatibel protocol accepteert. Protocolverhogingen beschermen het communicatiecontract; herstel na een rollback is een probleem dat moet worden opgelost door processen en versies op te schonen.

## Symlink van Skill overgeslagen wegens ontsnapping uit pad

Gebruik dit wanneer de logboeken het volgende bevatten:

```text
Overgeslagen ontsnapt Skill-pad buiten de geconfigureerde hoofdmap: ... reason=symlink-escape
```

Elke hoofdmap voor Skills vormt een insluitingsgrens. Een symlink onder `~/.agents/skills`, `<workspace>/.agents/skills`, `<workspace>/skills` of `~/.openclaw/skills` wordt overgeslagen wanneer het werkelijke doel buiten die hoofdmap valt, tenzij het doel expliciet wordt vertrouwd.

Inspecteer de koppeling:

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

Als het doel opzettelijk is, configureer je zowel de directe hoofdmap voor Skills als het toegestane symlinkdoel:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Start vervolgens een nieuwe sessie of wacht totdat de Skills-watcher is vernieuwd. Herstart de Gateway als het actieve proces van vóór de configuratiewijziging dateert.

Gebruik geen brede doelen zoals `~`, `/` of een volledige gesynchroniseerde projectmap. Beperk `allowSymlinkTargets` tot de werkelijke hoofdmap voor Skills die vertrouwde `SKILL.md`-mappen bevat.

Als toepassen vanuit Skill Workshop ook moet schrijven via die vertrouwde, gesymlinkte paden voor werkruimte-Skills, schakel je `skills.workshop.allowSymlinkTargetWrites` in. Laat dit uitgeschakeld voor gedeelde hoofd­mappen voor Skills die alleen-lezen zijn.

Gerelateerd:

- [Configuratie van Skills](/nl/tools/skills-config#symlinked-skill-roots)
- [Configuratievoorbeelden](/nl/gateway/configuration-examples#symlinked-sibling-skill-repo)

## Anthropic 429: extra gebruik vereist voor lange context

Gebruik dit wanneer logboeken of fouten het volgende bevatten: `HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Let op:

- Het geselecteerde Anthropic-model is een algemeen beschikbaar Claude 4.x-model met ondersteuning voor 1M (Opus 4.6/4.7/4.8, Sonnet 4.6), of de modelconfiguratie bevat nog de verouderde `params.context1m: true`.
- De huidige Anthropic-referentie is niet geschikt voor gebruik met lange context.
- Aanvragen mislukken uitsluitend tijdens lange sessies of modeluitvoeringen waarvoor het 1M-contextpad nodig is.

Oplossingsopties:

<Steps>
  <Step title="Een standaard contextvenster gebruiken">
    Schakel over naar een model met een standaardvenster of verwijder de verouderde `context1m` uit een oudere
    modelconfiguratie die niet algemeen beschikbaar is voor een context van 1M.
  </Step>
  <Step title="Een geschikte referentie gebruiken">
    Gebruik een Anthropic-referentie die geschikt is voor aanvragen met lange context, of schakel over naar een Anthropic-API-sleutel.
  </Step>
  <Step title="Terugvalmodellen configureren">
    Configureer terugvalmodellen zodat uitvoeringen doorgaan wanneer Anthropic aanvragen met lange context weigert.
  </Step>
</Steps>

Gerelateerd:

- [Anthropic](/nl/providers/anthropic)
- [Tokengebruik en kosten](/nl/reference/token-use)
- [Waarom zie ik HTTP 429 van Anthropic?](/nl/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Geblokkeerde upstream-403-antwoorden

Gebruik dit wanneer een upstream-LLM-provider een algemene `403` retourneert, zoals `Your request was blocked`.

Ga er niet van uit dat dit altijd een OpenClaw-configuratieprobleem is. Het antwoord kan afkomstig zijn van een upstream-beveiligingslaag, zoals een CDN, WAF, botbeheerregel of reverse proxy vóór een OpenAI-compatibel eindpunt.

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

Let op:

- Meerdere modellen van dezelfde provider mislukken op dezelfde manier.
- HTML of algemene beveiligingstekst in plaats van een normale API-fout van de provider.
- Beveiligingsgebeurtenissen aan de kant van de provider voor hetzelfde tijdstip van de aanvraag.
- Een minimale directe `curl`-test slaagt, terwijl normale aanvragen met een SDK-vorm mislukken.

Herstel eerst de filtering aan de kant van de provider wanneer het bewijs op een WAF/CDN-blokkering wijst. Geef de voorkeur aan een nauwkeurig afgebakende toestemmings- of overslaregel voor het API-pad dat OpenClaw gebruikt en schakel de beveiliging niet voor de hele site uit.

<Warning>
Een geslaagde minimale `curl` garandeert niet dat echte SDK-achtige aanvragen door dezelfde upstream-beveiligingslaag komen.
</Warning>

Gerelateerd:

- [OpenAI-compatibele eindpunten](/nl/gateway/configuration-reference#openai-compatible-endpoints)
- [Providerconfiguratie](/nl/providers)
- [Logboeken](/nl/logging)

## Lokale OpenAI-compatibele backend slaagt voor directe tests, maar agentuitvoeringen mislukken

Gebruik dit wanneer:

- `curl ... /v1/models` werkt.
- Minimale directe `/v1/chat/completions`-aanroepen werken.
- OpenClaw-modeluitvoeringen mislukken alleen tijdens normale agentbeurten.

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Let op:

- Directe minimale aanroepen slagen, maar OpenClaw-uitvoeringen mislukken alleen bij grotere prompts.
- `model_not_found`- of 404-fouten, hoewel directe `/v1/chat/completions` werkt met dezelfde kale model-id.
- Backendfouten over `messages[].content` die een tekenreeks verwacht.
- Onregelmatige `incomplete turn detected ... stopReason=stop payloads=0`-waarschuwingen met een OpenAI-compatibele lokale backend.
- Backendcrashes die alleen optreden bij grotere aantallen prompttokens of volledige prompts van de agentruntime.

<AccordionGroup>
  <Accordion title="Veelvoorkomende kenmerken">
    - `model_not_found` met een lokale server in MLX/vLLM-stijl: controleer of `baseUrl` `/v1` bevat, `api` gelijk is aan `"openai-completions"` voor `/v1/chat/completions`-backends en `models.providers.<provider>.models[].id` de kale lokale provider-id is. Selecteer deze eenmaal met het providervoorvoegsel, bijvoorbeeld `mlx/mlx-community/Qwen3-30B-A3B-6bit`; behoud de catalogusvermelding als `mlx-community/Qwen3-30B-A3B-6bit`.
    - `messages[...].content: invalid type: sequence, expected a string`: de backend weigert gestructureerde inhoudsonderdelen voor Chat Completions. Oplossing: stel `models.providers.<provider>.models[].compat.requiresStringContent: true` in.
    - `validation.keys` of toegestane berichtsleutels zoals `["role","content"]`: de backend weigert OpenAI-achtige metadata voor het opnieuw afspelen van Chat Completions-berichten. Oplossing: stel `models.providers.<provider>.models[].compat.strictMessageKeys: true` in.
    - `incomplete turn detected ... stopReason=stop payloads=0`: de backend heeft de Chat Completions-aanvraag voltooid, maar voor die beurt geen voor de gebruiker zichtbare assistenttekst geretourneerd. OpenClaw probeert replay-veilige, lege OpenAI-compatibele beurten één keer opnieuw; aanhoudende fouten betekenen meestal dat de backend lege of niet-tekstuele inhoud uitvoert, of de tekst van het definitieve antwoord onderdrukt.
    - Directe minimale aanvragen slagen, maar OpenClaw-agentuitvoeringen mislukken door backend- of modelcrashes (bijvoorbeeld Gemma op sommige `inferrs`-builds): het OpenClaw-transport is waarschijnlijk al correct; de backend faalt bij de grotere promptvorm van de agentruntime.
    - Fouten nemen af nadat hulpmiddelen zijn uitgeschakeld, maar verdwijnen niet: hulpmiddelschema's droegen bij aan de belasting, maar het resterende probleem is nog steeds de capaciteit van het upstream-model of de server, of een backendbug.

  </Accordion>
  <Accordion title="Oplossingsopties">
    1. Stel `compat.requiresStringContent: true` in voor backends voor Chat Completions die uitsluitend tekenreeksen ondersteunen.
    2. Stel `compat.strictMessageKeys: true` in voor strikte backends voor Chat Completions die per bericht uitsluitend `role` en `content` accepteren.
    3. Stel `compat.supportsTools: false` in voor modellen of backends die het hulpmiddelschemaoppervlak van OpenClaw niet betrouwbaar kunnen verwerken.
    4. Verlaag waar mogelijk de promptbelasting: een kleinere werkruimtebootstrap, kortere sessiegeschiedenis, lichter lokaal model of een backend met betere ondersteuning voor lange context.
    5. Als minimale directe aanvragen blijven slagen terwijl OpenClaw-agentbeurten nog steeds binnen de backend crashen, behandel je dit als een beperking van de upstream-server of het model en dien je daar een reproduceerbaar voorbeeld in met de geaccepteerde payloadvorm.
  </Accordion>
</AccordionGroup>

Gerelateerd:

- [Configuratie](/nl/gateway/configuration)
- [Lokale modellen](/nl/gateway/local-models)
- [OpenAI-compatibele eindpunten](/nl/gateway/configuration-reference#openai-compatible-endpoints)

## Geen antwoorden

Als kanalen actief zijn maar niets antwoordt, controleer dan de routering en het beleid voordat je iets opnieuw verbindt.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Let op:

- Koppeling in afwachting voor afzenders van privéberichten.
- Vermeldingsbeperking voor groepen (`requireMention`, `mentionPatterns`).
- Niet-overeenkomende toestemmingslijsten voor kanalen/groepen.

Veelvoorkomende meldingen:

- `drop guild message (mention required` → groepsbericht genegeerd tot er een vermelding is.
- `pairing request` → afzender moet worden goedgekeurd.
- `blocked` / `allowlist` → afzender/kanaal is door beleid gefilterd.

Gerelateerd:

- [Problemen met kanalen oplossen](/nl/channels/troubleshooting)
- [Groepen](/nl/channels/groups)
- [Koppeling](/nl/channels/pairing)

## Connectiviteit van het dashboard en de bedieningsinterface

Als het dashboard/de bedieningsinterface geen verbinding maakt, controleer dan de URL, verificatiemodus en aannames over de beveiligde context.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Let op:

- Juiste URL voor de test en het dashboard.
- Niet-overeenkomende verificatiemodus/token tussen client en Gateway.
- Gebruik van HTTP waar apparaatidentiteit vereist is.

Als een lokale browser na een update geen verbinding kan maken met `127.0.0.1:18789`, herstel dan eerst de lokale Gateway-service en controleer of deze het dashboard aanbiedt:

```bash
openclaw gateway restart
lsof -i :18789
curl http://127.0.0.1:18789
```

Als `curl` OpenClaw-HTML retourneert, werkt de Gateway en is het resterende probleem waarschijnlijk de browsercache, een oude deeplink of verouderde tabbladstatus. Open `http://127.0.0.1:18789` rechtstreeks en navigeer vanaf het dashboard. Als de service na een herstart niet actief blijft, voer dan `openclaw gateway start` uit en controleer `openclaw gateway status` opnieuw.

<AccordionGroup>
  <Accordion title="Verbindings-/verificatiemeldingen">
    - `device identity required` → onbeveiligde context of ontbrekende apparaatverificatie.
    - `origin not allowed` → browser-`Origin` staat niet in `gateway.controlUi.allowedOrigins` (of je maakt verbinding vanaf een browseroorsprong die geen loopback gebruikt zonder expliciete toestemmingslijst).
    - `device nonce required` / `device nonce mismatch` → client voltooit de op uitdagingen gebaseerde apparaatverificatiestroom niet (`connect.challenge` + `device.nonce`).
    - `device signature invalid` / `device signature expired` → client heeft de verkeerde payload (of een verouderde tijdstempel) voor de huidige handshake ondertekend.
    - `AUTH_TOKEN_MISMATCH` met `canRetryWithDeviceToken=true` → client kan één vertrouwde nieuwe poging uitvoeren met een apparaat-token uit de cache.
    - Die nieuwe poging met een token uit de cache hergebruikt de scopeset die samen met het gekoppelde apparaat-token in de cache is opgeslagen. Aanroepers met expliciete `deviceToken` / expliciete `scopes` behouden in plaats daarvan hun aangevraagde scopeset.
    - `AUTH_SCOPE_MISMATCH` → het apparaat-token is herkend, maar de goedgekeurde scopes ervan dekken dit verbindingsverzoek niet; koppel opnieuw of keur het aangevraagde scopecontract goed in plaats van een gedeeld Gateway-token te roteren.
    - Buiten dat pad voor een nieuwe poging is de prioriteit voor verbindingsverificatie: eerst een expliciet gedeeld token/wachtwoord, daarna expliciete `deviceToken`, vervolgens het opgeslagen apparaat-token en ten slotte het bootstrap-token.
    - Op het asynchrone Tailscale Serve-pad voor de bedieningsinterface worden mislukte pogingen voor dezelfde `{scope, ip}` geserialiseerd voordat de begrenzer de mislukking registreert. Bij twee gelijktijdige mislukte pogingen van dezelfde client kan bij de tweede poging daarom `retry later` verschijnen in plaats van twee gewone niet-overeenkomende waarden.
    - `too many failed authentication attempts (retry later)` van een loopbackclient met browseroorsprong → herhaalde mislukkingen van diezelfde genormaliseerde `Origin` worden tijdelijk geblokkeerd; een andere localhost-oorsprong gebruikt een afzonderlijke groep.
    - Herhaalde `unauthorized` na die nieuwe poging → afwijking tussen gedeeld token en apparaat-token; vernieuw de tokenconfiguratie en keur het apparaat-token indien nodig opnieuw goed of roteer het.
    - `gateway connect failed:` → onjuist doel voor host/poort/URL.

  </Accordion>
</AccordionGroup>

### Sneloverzicht van detailcodes voor verificatie

Gebruik `error.details.code` uit het mislukte `connect`-antwoord om de volgende actie te kiezen:

| Detailcode                   | Betekenis                                                                                                                                                                                    | Aanbevolen actie                                                                                                                                                                                                                                                                          |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | Client heeft een vereist gedeeld token niet verzonden.                                                                                                                                       | Plak/stel het token in de client in en probeer het opnieuw. Voor dashboardpaden: `openclaw config get gateway.auth.token` en plak het vervolgens in de instellingen van de bedieningsinterface.                                                                                             |
| `AUTH_TOKEN_MISMATCH`        | Gedeeld token kwam niet overeen met het verificatietoken van de Gateway.                                                                                                                      | Als `canRetryWithDeviceToken=true`, sta dan één vertrouwde nieuwe poging toe. Nieuwe pogingen met een token uit de cache hergebruiken opgeslagen goedgekeurde scopes; aanroepers met expliciete `deviceToken` / `scopes` behouden aangevraagde scopes. Als dit nog steeds mislukt, voer dan de [controlelijst voor herstel van tokenafwijking](/nl/cli/devices#token-drift-recovery-checklist) uit. |
| `AUTH_DEVICE_TOKEN_MISMATCH` | Het apparaatgebonden token uit de cache is verouderd of ingetrokken.                                                                                                                         | Roteer/keur het apparaat-token opnieuw goed met de [apparaten-CLI](/nl/cli/devices) en maak vervolgens opnieuw verbinding.                                                                                                                                                                   |
| `AUTH_SCOPE_MISMATCH`        | Het apparaat-token is geldig, maar de goedgekeurde rol/scopes ervan dekken dit verbindingsverzoek niet.                                                                                       | Koppel het apparaat opnieuw of keur het aangevraagde scopecontract goed; behandel dit niet als een afwijking van het gedeelde token.                                                                                                                                                       |
| `PAIRING_REQUIRED`           | Apparaatidentiteit moet worden goedgekeurd. Controleer `error.details.reason` op `not-paired`, `scope-upgrade`, `role-upgrade` of `metadata-upgrade`, en gebruik `requestId` / `remediationHint` indien aanwezig. | Keur het openstaande verzoek goed: `openclaw devices list` en vervolgens `openclaw devices approve <requestId>`. Upgrades van scopes/rollen gebruiken dezelfde stroom nadat je de aangevraagde toegang hebt beoordeeld.                                                                         |

<Note>
Rechtstreekse loopback-RPC's naar de backend die met het gedeelde Gateway-token/wachtwoord zijn geverifieerd, mogen niet afhankelijk zijn van de scopebasislijn voor gekoppelde apparaten van de CLI. Als subagents of andere interne aanroepen nog steeds mislukken met `scope-upgrade`, controleer dan of de aanroeper `client.id: "gateway-client"` en `client.mode: "backend"` gebruikt en geen expliciete `deviceIdentity` of apparaat-token afdwingt.
</Note>

Migratiecontrole voor apparaatverificatie v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Als de logboeken nonce-/handtekeningfouten tonen, werk dan de verbindende client bij en controleer deze:

<Steps>
  <Step title="Wacht op connect.challenge">
    Client wacht op de door de Gateway uitgegeven `connect.challenge`.
  </Step>
  <Step title="Onderteken de payload">
    Client ondertekent de aan de uitdaging gebonden payload.
  </Step>
  <Step title="Verzend de apparaat-nonce">
    Client verzendt `connect.params.device.nonce` met dezelfde uitdaging-nonce.
  </Step>
</Steps>

Als `openclaw devices rotate` / `revoke` / `remove` onverwacht wordt geweigerd:

- Tokensessies van gekoppelde apparaten kunnen alleen **hun eigen** apparaat beheren, tenzij de aanroeper ook `operator.admin` heeft.
- `openclaw devices rotate --scope ...` kan alleen operatorscopes aanvragen die de sessie van de aanroeper al heeft.

Gerelateerd:

- [Configuratie](/nl/gateway/configuration) (Gateway-verificatiemodi)
- [Bedieningsinterface](/nl/web/control-ui)
- [Apparaten](/nl/cli/devices)
- [Externe toegang](/nl/gateway/remote)
- [Verificatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth)

## Gateway-service is niet actief

Gebruik dit wanneer de service is geïnstalleerd, maar het proces niet actief blijft.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # scan ook services op systeemniveau
```

Let op:

- `Runtime: stopped` met aanwijzingen over het afsluiten.
- Niet-overeenkomende serviceconfiguratie (`Config (cli)` versus `Config (service)`).
- Conflicten met poort/listener.
- Extra installaties van launchd/systemd/schtasks wanneer `--deep` wordt gebruikt.
- `Other gateway-like services detected (best effort)`-aanwijzingen voor opschoning.

<AccordionGroup>
  <Accordion title="Veelvoorkomende meldingen">
    - `Gateway start blocked: set gateway.mode=local` of `existing config is missing gateway.mode` → lokale Gateway-modus is niet ingeschakeld, of het configuratiebestand is overschreven en `gateway.mode` is verloren gegaan. Oplossing: stel `gateway.mode="local"` in je configuratie in, of voer `openclaw onboard --mode local` / `openclaw setup` opnieuw uit om de verwachte configuratie voor de lokale modus opnieuw vast te leggen. Als je OpenClaw via Podman uitvoert, is het standaardconfiguratiepad `~/.openclaw/openclaw.json`.
    - `refusing to bind gateway ... without auth` → binding zonder loopback zonder een geldig verificatiepad voor de Gateway (token/wachtwoord, of trusted-proxy waar geconfigureerd).
    - `another gateway instance is already listening` / `EADDRINUSE` → poortconflict.
    - `Other gateway-like services detected (best effort)` → er bestaan verouderde of parallelle launchd-/systemd-/schtasks-eenheden. De meeste configuraties moeten één Gateway per machine gebruiken; als je er meer dan één nodig hebt, isoleer dan poorten + configuratie/status/werkruimte. Zie [/gateway#multiple-gateways-same-host](/nl/gateway#multiple-gateways-same-host).
    - `System-level OpenClaw gateway service detected` van doctor → er bestaat een systemd-systeemeenheid terwijl de service op gebruikersniveau ontbreekt. Verwijder of deactiveer het duplicaat voordat je doctor toestaat een gebruikersservice te installeren, of stel `OPENCLAW_SERVICE_REPAIR_POLICY=external` in als de systeemeenheid de bedoelde supervisor is.
    - `Gateway service port does not match current gateway config` → de geïnstalleerde supervisor houdt nog steeds de oude `--port` vast. Voer `openclaw doctor --fix` of `openclaw gateway install --force` uit en start vervolgens de Gateway-service opnieuw.

  </Accordion>
</AccordionGroup>

Gerelateerd:

- [Uitvoering op de achtergrond en procestool](/nl/gateway/background-process)
- [Configuratie](/nl/gateway/configuration)
- [Doctor](/nl/gateway/doctor)

## macOS-Gateway reageert stilzwijgend niet meer en hervat wanneer je het dashboard aanraakt

Gebruik dit wanneer kanalen (Telegram, WhatsApp enz.) op een macOS-host telkens minuten tot uren stilvallen en de Gateway weer lijkt te werken zodra je de Control UI opent, via SSH verbinding maakt of anderszins interactie met de host hebt. In `openclaw status` is doorgaans geen duidelijk symptoom zichtbaar, omdat de Gateway alweer actief is tegen de tijd dat je kijkt.

```bash
ls ~/.openclaw/logs/stability/ | tail -5
openclaw gateway stability --bundle latest
pmset -g log | grep -iE "sleep|wake|maintenance" | tail -50
launchctl print gui/$UID/ai.openclaw.gateway | grep -E "state|last exit|runs"
```

Let op:

- Een of meer `*-uncaught_exception.json`-bundels in `~/.openclaw/logs/stability/` waarbij `error.code` is ingesteld op een tijdelijke netwerkcode, zoals `ENETDOWN`, `ENETUNREACH`, `EHOSTUNREACH` of `ECONNREFUSED`.
- `pmset -g log`-regels zoals `Entering Sleep state due to 'Maintenance Sleep'` of `en0 driver is slow (msg: WillChangeState to 0)` die samenvallen met de tijdstempels van de crashes. Power Nap / Maintenance Sleep zet het wifi-stuurprogramma kort in status 0; elke uitgaande `connect()` die in dat tijdsvenster valt, kan mislukken met `ENETDOWN`, zelfs op een host die verder volledige netwerkconnectiviteit heeft.
- `launchctl print`-uitvoer die `state = not running` toont met meerdere recente `runs` en een afsluitcode, vooral wanneer het interval tussen de crash en de volgende start eerder ongeveer een uur dan enkele seconden bedraagt. macOS launchd past na een reeks crashes een ongedocumenteerde beveiliging tegen herhaald starten toe, waardoor `KeepAlive=true` mogelijk niet meer wordt gehonoreerd totdat een externe trigger, zoals een interactieve aanmelding, dashboardverbinding of `launchctl kickstart`, deze opnieuw activeert.

Veelvoorkomende kenmerken:

- Een stabiliteitsbundel waarvan `error.code` gelijk is aan `ENETDOWN` of een verwante code, waarbij de aanroepstack verwijst naar Node `net` `lookupAndConnect` / `Socket.connect`. OpenClaw `2026.5.26` en nieuwer classificeren deze als onschuldige tijdelijke netwerkfouten, zodat ze niet meer worden doorgegeven aan de niet-afgevangen handler op het hoogste niveau; werk je met een oudere release, voer dan eerst een upgrade uit.
- Lange stille perioden die onmiddellijk eindigen zodra je verbinding maakt met de Control UI of via SSH met de host: de voor de gebruiker zichtbare activiteit activeert launchd's beveiliging tegen herhaald starten opnieuw, niet iets wat het dashboard met de Gateway doet.
- Het aantal `runs` neemt gedurende de dag toe zonder bijbehorende `received SIG*; shutting down`-regel in `~/Library/Logs/openclaw/gateway.log`: bij correct afsluiten wordt een signaal geregistreerd; bij tijdelijke crashes niet.

Wat je moet doen:

1. **Voer een upgrade van de Gateway uit** als je een release van vóór `2026.5.26` gebruikt. Na de upgrade worden toekomstige `ENETDOWN`-fouten als waarschuwingen geregistreerd in plaats van dat ze het proces beëindigen.
2. **Beperk activiteit tijdens de onderhoudsslaapstand** op Mac mini-/desktophosts die als permanent actieve servers moeten functioneren:

   ```bash
   sudo pmset -a sleep 0 disksleep 0 standby 0 powernap 0
   ```

   Dit vermindert de onderliggende uitval van het stuurprogramma aanzienlijk, maar voorkomt deze niet volledig. Het systeem kan ondanks deze vlaggen nog steeds bepaalde onderhoudsslaapstanden uitvoeren voor TCP-keepalive en mDNS-onderhoud.

3. **Voeg een bewakingsproces voor beschikbaarheid toe**, zodat een toekomstige reeks crashes die door launchd wordt geparkeerd snel wordt gedetecteerd:

   ```bash
   # Voorbeeld van een launchd-bewuste beschikbaarheidscontrole, geschikt voor een vijfminuten-Cron of LaunchAgent
   state=$(launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | awk -F'= ' '/state =/ {print $2; exit}')
   if [ "$state" != "running" ]; then
     launchctl kickstart -k gui/$UID/ai.openclaw.gateway
   fi
   ```

   Het doel is om de beveiliging tegen herhaald starten extern opnieuw te activeren; alleen `KeepAlive=true` is op macOS na een reeks crashes niet voldoende.

Gerelateerd:

- [macOS-platformnotities](/nl/platforms/macos)
- [Logboekregistratie](/nl/logging)
- [Doctor](/nl/gateway/doctor)

## macOS launchd-supervisorlus met dubbele Gateway/Node-LaunchAgents

Gebruik dit wanneer een macOS-installatie om de paar seconden opnieuw wordt gestart, `openclaw`
statuscontroles wisselen tussen beschikbaar en niet beschikbaar en de kanaalafhandeling vastloopt,
ook al lijkt de service actief te zijn.

Dit is waargenomen bij oudere installaties waarbij zowel `ai.openclaw.gateway` als
`ai.openclaw.node` LaunchAgents actief waren en beide
`OPENCLAW_LAUNCHD_LABEL` invoegden. In die toestand kan OpenClaw launchd-
supervisie detecteren, proberen het opnieuw starten weer aan launchd over te dragen en in een snelle
`EADDRINUSE`-/herstartlus terechtkomen in plaats van één stabiel Gateway-proces.

```bash
for i in 1 2 3 4; do
  ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
  sleep 10
done

openclaw gateway status --deep
openclaw node status
launchctl print gui/$UID/ai.openclaw.gateway | grep -E 'state|last exit|runs'
tail -n 80 ~/Library/Logs/openclaw/gateway.log
```

Let op:

- Meer dan één Gateway-PID tijdens de steekproef van 30 seconden in plaats van één stabiel
  proces.
- `EADDRINUSE`, `another gateway instance is already listening` of herhaalde
  regels over opnieuw starten/overdracht in `gateway.log`.
- Zowel `~/Library/LaunchAgents/ai.openclaw.gateway.plist` als
  `~/Library/LaunchAgents/ai.openclaw.node.plist` zijn tegelijkertijd geladen op een
  host waarop slechts één beheerde Gateway-service actief hoort te zijn.

Wat je moet doen:

1. Als op deze host alleen de Gateway-service actief hoort te zijn, verwijder je de beheerde Node-
   service via OpenClaw. **Sla deze stap over** als je de Node-
   service actief gebruikt voor functies van externe Nodes; als je deze verwijdert, stoppen die functies op
   deze host:

   ```bash
   openclaw node uninstall
   ```

2. Installeer een permanente Gateway-wrapper die de overgenomen launchd-
   markeringen wist voordat OpenClaw wordt gestart. Gebruik de ondersteunde optie `--wrapper`; bewerk
   het gegenereerde bestand onder `~/.openclaw/service-env/` niet, omdat dit bestand bij het opnieuw
   installeren of bijwerken van de service en bij herstel door Doctor opnieuw wordt gegenereerd:

   ```bash
   mkdir -p ~/.local/bin
   cat >~/.local/bin/openclaw-launchd-workaround <<'EOF'
   #!/bin/sh
   set -eu
   unset OPENCLAW_LAUNCHD_LABEL LAUNCH_JOB_LABEL LAUNCH_JOB_NAME XPC_SERVICE_NAME || true
   exec openclaw "$@"
   EOF
   chmod 700 ~/.local/bin/openclaw-launchd-workaround

   openclaw gateway install \
     --wrapper ~/.local/bin/openclaw-launchd-workaround \
     --force
   ```

   `gateway install` bewaart het wrapperpad bij gedwongen herinstallaties,
   updates en reparaties door Doctor.

3. Controleer of de Gateway stabiel is en RPC bedient, en niet alleen luistert:

   ```bash
   openclaw gateway status --deep --require-rpc

   for i in 1 2 3 4; do
     ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
     sleep 10
   done
   ```

   De PID-steekproef moet één stabiel proces tonen in plaats van een wisselende reeks
   PID's, en de afhandeling van inkomende kanalen moet worden hervat.

4. Verwijder na de upgrade naar een release waarin de onderliggende dubbele LaunchAgent-lus is
   opgelost de tijdelijke oplossing en installeer de normale beheerde service opnieuw:

   ```bash
   OPENCLAW_WRAPPER= openclaw gateway install --force
   rm ~/.local/bin/openclaw-launchd-workaround
   ```

Gerelateerd:

- [macOS-platformopmerkingen](/nl/platforms/mac/bundled-gateway)
- [Doctor](/nl/gateway/doctor)
- [Gateway-CLI](/nl/cli/gateway)

## Gateway wordt afgesloten bij hoog geheugengebruik

Gebruik dit wanneer de Gateway onder belasting verdwijnt, de supervisor een herstart in OOM-stijl meldt of de logs `critical memory pressure bundle written` vermelden.

```bash
openclaw gateway status --deep
openclaw logs --follow
openclaw gateway stability --bundle latest
openclaw gateway diagnostics export
```

Let op:

- `Reason: diagnostic.memory.pressure.critical` in de nieuwste stabiliteitsbundel.
- `Memory pressure:` met `critical/rss_threshold`, `critical/heap_threshold` of `critical/rss_growth`.
- `V8 heap:`-waarden nabij de heaplimiet.
- `Largest session files:`-vermeldingen zoals `agents/<agent>/sessions/<session>.jsonl` of `sessions/<session>.jsonl`.
- Linux-cgroup-geheugentellers wanneer de Gateway in een container of service met een geheugenlimiet draait.

Veelvoorkomende signalen:

- `critical memory pressure bundle written` verschijnt kort vóór de herstart → OpenClaw heeft een stabiliteitsbundel van vóór de OOM vastgelegd. Inspecteer deze met `openclaw gateway stability --bundle latest`.
- `memory pressure: level=critical` verschijnt in de Gateway-logs → OpenClaw heeft kritieke geheugendruk gedetecteerd en de beschikbare geheugengegevens binnen het proces vastgelegd.
- `Largest session files:` verwijst naar een zeer groot geredigeerd transcriptpad → verklein de bewaarde sessiegeschiedenis, inspecteer de sessiegroei of verplaats oude transcripten uit de actieve opslag voordat je opnieuw opstart.
- `V8 heap:` gebruikte bytes liggen dicht bij de heaplimiet → verlaag eerst de prompt-/sessiedruk of verminder gelijktijdig werk. Inspecteer voor een beheerde service `Gateway heap:` in `openclaw gateway status`; als er `not set` staat, genereer je oude servicemetadata opnieuw met `openclaw gateway install --force`. `NODE_OPTIONS` uit de omringende shell wordt bewust genegeerd. Gebruik alleen een expliciete heap-override op supervisorniveau nadat je de aanhoudende werkbelasting hebt bevestigd en voldoende ruimte voor native geheugen hebt vrijgehouden.
- `Memory pressure: critical/rss_growth` → het geheugengebruik groeide snel binnen één meetvenster. Controleer de nieuwste logs op een grote import, uit de hand gelopen tooluitvoer, herhaalde nieuwe pogingen of een reeks in de wachtrij geplaatste agenttaken.
- Kritieke geheugendruk verschijnt in de logs, maar er bestaat geen bundel → leg na de gebeurtenis `openclaw gateway diagnostics export` vast voor het beschikbare operationele bewijs.

De stabiliteitsbundel bevat geen payloads. Deze bevat operationeel geheugenbewijs en geredigeerde relatieve bestandspaden, maar geen berichttekst, webhook-bodies, inloggegevens, tokens, cookies of onbewerkte sessie-id's. Voeg de diagnostische export toe aan bugrapporten in plaats van onbewerkte logs te kopiëren.

Gerelateerd:

- [Gateway-status](/nl/gateway/health)
- [Diagnostische export](/nl/gateway/diagnostics)
- [Sessies](/nl/cli/sessions)

## Gateway heeft ongeldige configuratie geweigerd

Gebruik dit wanneer het opstarten van de Gateway mislukt met `Invalid config` of wanneer hot-reloadlogs melden dat een ongeldige bewerking is overgeslagen.

```bash
openclaw logs --follow
openclaw config file
openclaw config validate
openclaw doctor
```

Let op:

- `Invalid config at ...`
- `config reload skipped (invalid config): ...`
- `Config write rejected: ...`
- Een van een tijdstempel voorzien `openclaw.json.rejected.*`-bestand naast de actieve configuratie.
- Een van een tijdstempel voorzien `openclaw.json.clobbered.*`-bestand als `doctor --fix` een defecte rechtstreekse bewerking heeft hersteld.
- OpenClaw bewaart voor elk configuratiepad de nieuwste 32 `.clobbered.*`-bestanden en roteert oudere bestanden.

<AccordionGroup>
  <Accordion title="Wat er is gebeurd">
    - De configuratie is tijdens het opstarten, hot reload of een door OpenClaw beheerde schrijfbewerking niet gevalideerd.
    - Het opstarten van de Gateway mislukt veilig in plaats van `openclaw.json` te herschrijven.
    - Hot reload slaat ongeldige externe bewerkingen over en houdt de huidige runtimeconfiguratie actief.
    - Door OpenClaw beheerde schrijfbewerkingen weigeren ongeldige/destructieve payloads vóór de commit en slaan `.rejected.*` op.
    - `openclaw doctor --fix` beheert het herstel. Het kan niet-JSON-prefixen verwijderen of de laatst bekende werkende kopie herstellen, terwijl de geweigerde payload als `.clobbered.*` behouden blijft.
    - Wanneer voor één configuratiepad veel reparaties plaatsvinden, roteert OpenClaw oudere `.clobbered.*`-bestanden, zodat de nieuwste herstelde payload beschikbaar blijft.

  </Accordion>
  <Accordion title="Inspecteren en repareren">
    ```bash
    CONFIG="$(openclaw config file)"
    ls -lt "$CONFIG".clobbered.* "$CONFIG".rejected.* 2>/dev/null | head
    diff -u "$CONFIG" "$(ls -t "$CONFIG".clobbered.* 2>/dev/null | head -n 1)"
    openclaw config validate
    openclaw doctor
    ```
  </Accordion>
  <Accordion title="Veelvoorkomende signalen">
    - `.clobbered.*` bestaat → doctor heeft een defecte externe bewerking behouden tijdens het repareren van de actieve configuratie.
    - `.rejected.*` bestaat → een door OpenClaw beheerde configuratieschrijfactie heeft vóór het vastleggen niet aan de schema- of overschrijvingscontroles voldaan.
    - `Config write rejected:` → de schrijfactie probeerde de vereiste structuur te verwijderen, het bestand sterk te verkleinen of een ongeldige configuratie op te slaan.
    - `config reload skipped (invalid config):` → een rechtstreekse bewerking heeft de validatie niet doorstaan en is door de actieve Gateway genegeerd.
    - `Invalid config at ...` → het opstarten is mislukt voordat de Gateway-services waren gestart.
    - `missing-meta-vs-last-good`, `gateway-mode-missing-vs-last-good` of `size-drop-vs-last-good:*` → een door OpenClaw beheerde schrijfactie is geweigerd omdat er velden of bestandsgrootte ontbraken ten opzichte van de laatst bekende goede back-up.
    - `Config last-known-good promotion skipped` → de kandidaat bevatte geredigeerde placeholders voor geheimen, zoals `***`.

  </Accordion>
  <Accordion title="Oplossingsopties">
    1. Voer `openclaw doctor --fix` uit om doctor de configuratie met voorvoegsels of overschrijvingen te laten repareren of de laatst bekende goede versie te herstellen.
    2. Kopieer alleen de bedoelde sleutels uit `.clobbered.*` of `.rejected.*` en pas ze vervolgens toe met `openclaw config set` of `config.patch`.
    3. Voer `openclaw config validate` uit voordat je opnieuw opstart.
    4. Als je handmatig bewerkt, behoud dan de volledige JSON5-configuratie, niet alleen het gedeeltelijke object dat je wilde wijzigen.
  </Accordion>
</AccordionGroup>

Gerelateerd:

- [Configuratie](/nl/cli/config)
- [Configuratie: dynamisch herladen](/nl/gateway/configuration#config-hot-reload)
- [Configuratie: strikte validatie](/nl/gateway/configuration#strict-validation)
- [Doctor](/nl/gateway/doctor)

## Waarschuwingen van de Gateway-probe

Gebruik dit wanneer `openclaw gateway probe` iets bereikt, maar nog steeds een waarschuwingsblok weergeeft.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Let op:

- `warnings[].code` en `primaryTargetId` in de JSON-uitvoer.
- Of de waarschuwing gaat over SSH-terugval, meerdere gateways, ontbrekende bereiken of niet-opgeloste authenticatieverwijzingen.

Veelvoorkomende signalen:

- `SSH tunnel failed to start; falling back to direct probes.` → de SSH-configuratie is mislukt, maar de opdracht heeft nog steeds rechtstreeks geconfigureerde of loopbackdoelen geprobeerd.
- `multiple reachable gateway identities detected` → verschillende gateways hebben geantwoord, of OpenClaw kon niet bewijzen dat de bereikbare doelen dezelfde Gateway zijn. Een SSH-tunnel, proxy-URL of geconfigureerde externe URL naar dezelfde Gateway wordt behandeld als één Gateway met meerdere transportmethoden, zelfs wanneer de transportpoorten verschillen.
- `Read-probe diagnostics are limited by gateway scopes (missing operator.read)` → de verbinding werkte, maar de gedetailleerde RPC is beperkt door het bereik; koppel de apparaatidentiteit of gebruik aanmeldgegevens met `operator.read`.
- `Gateway accepted the WebSocket connection, but follow-up read diagnostics failed` → de verbinding werkte, maar de volledige set diagnostische RPC's heeft een time-out bereikt of is mislukt. Behandel dit als een bereikbare Gateway met beperkte diagnostiek; vergelijk `connect.ok` en `connect.rpcOk` in de uitvoer van `--json`.
- `Capability: pairing-pending` of `gateway closed (1008): pairing required` → de Gateway heeft geantwoord, maar deze client moet nog worden gekoppeld of goedgekeurd voordat normale beheerderstoegang mogelijk is.
- Niet-opgeloste waarschuwingstekst voor `gateway.auth.*` / `gateway.remote.*` SecretRef → authenticatiemateriaal was in dit opdrachtpad niet beschikbaar voor het mislukte doel.

Gerelateerd:

- [Gateway](/nl/cli/gateway)
- [Meerdere gateways op dezelfde host](/nl/gateway#multiple-gateways-same-host)
- [Externe toegang](/nl/gateway/remote)

## Kanaal verbonden, berichten worden niet doorgegeven

Als de kanaalstatus verbonden is maar de berichtenstroom stilligt, richt je dan op beleid, machtigingen en kanaalspecifieke leveringsregels.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Let op:

- DM-beleid (`pairing`, `allowlist`, `open`, `disabled`).
- Toestaanlijst voor groepen en vereisten voor vermeldingen.
- Ontbrekende API-machtigingen of bereiken voor het kanaal.

Veelvoorkomende signalen:

- `mention required` → bericht genegeerd door het beleid voor groepsvermeldingen.
- `pairing` / sporen van een wachtende goedkeuring → de afzender is niet goedgekeurd.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → probleem met kanaalauthenticatie of -machtigingen.

Gerelateerd:

- [Problemen met kanalen oplossen](/nl/channels/troubleshooting)
- [Discord](/nl/channels/discord)
- [Telegram](/nl/channels/telegram)
- [WhatsApp](/nl/channels/whatsapp)

## Levering via Cron en Heartbeat

Als Cron of Heartbeat niet is uitgevoerd of niets heeft geleverd, controleer dan eerst de plannerstatus en vervolgens het leveringsdoel.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Let op:

- Cron is ingeschakeld en de volgende activering is aanwezig.
- Status van de uitvoeringsgeschiedenis van de taak (`ok`, `skipped`, `error`).
- Redenen voor het overslaan van Heartbeat (`quiet-hours`, `requests-in-flight`, `cron-in-progress`, `lanes-busy`, `alerts-disabled`, `empty-heartbeat-file`).

<AccordionGroup>
  <Accordion title="Veelvoorkomende signalen">
    - `cron: scheduler disabled; jobs will not run automatically` → Cron is uitgeschakeld.
    - `cron: timer tick failed` → de plannercyclus is mislukt; controleer bestands-, logboek- en runtimefouten.
    - `heartbeat skipped` met `reason=quiet-hours` → buiten het venster met actieve uren.
    - `heartbeat skipped` met `reason=empty-heartbeat-file` → het kladblok van de Heartbeat-monitor bevat alleen lege regels, opmerkingen, koppen, fences of een lege checkliststructuur, waardoor OpenClaw de modelaanroep overslaat.
    - `heartbeat: unknown accountId` → ongeldige account-id voor het Heartbeat-leveringsdoel.
    - `heartbeat skipped` met `reason=dm-blocked` → het Heartbeat-doel is omgezet naar een DM-achtige bestemming terwijl `agents.defaults.heartbeat.directPolicy` (of de overschrijving per agent) is ingesteld op `block`.

  </Accordion>
</AccordionGroup>

Gerelateerd:

- [Heartbeat](/nl/gateway/heartbeat)
- [Geplande taken](/nl/automation/cron-jobs)
- [Geplande taken: problemen oplossen](/nl/automation/cron-jobs#troubleshooting)

## Node gekoppeld, tool mislukt

Als een Node is gekoppeld maar tools mislukken, isoleer dan de status van de voorgrond, machtigingen en goedkeuringen.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Let op:

- Node is online met de verwachte mogelijkheden.
- OS-machtigingen voor camera, microfoon, locatie en scherm.
- Status van uitvoeringsgoedkeuringen en de toestaanlijst.

Veelvoorkomende signalen:

- `NODE_BACKGROUND_UNAVAILABLE` → de Node-app moet op de voorgrond staan.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → ontbrekende OS-machtiging.
- `SYSTEM_RUN_DENIED: approval required` → uitvoeringsgoedkeuring in behandeling.
- `SYSTEM_RUN_DENIED: allowlist miss` → opdracht geblokkeerd door de toestaanlijst.

Gerelateerd:

- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals)
- [Problemen met Nodes oplossen](/nl/nodes/troubleshooting)
- [Nodes](/nl/nodes/index)

## Browsertool mislukt

Gebruik dit wanneer acties van de browsertool mislukken, hoewel de Gateway zelf goed werkt.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Let op:

- Of `plugins.allow` is ingesteld en `browser` bevat.
- Geldig pad naar het uitvoerbare browserbestand.
- Bereikbaarheid van het CDP-profiel.
- Beschikbaarheid van lokale Chrome voor profielen van `existing-session` / `user`.

<AccordionGroup>
  <Accordion title="Signalen van Plugin / uitvoerbaar bestand">
    - `unknown command "browser"` of `unknown command 'browser'` → de meegeleverde browser-Plugin wordt uitgesloten door `plugins.allow`.
    - Browsertool ontbreekt / is niet beschikbaar terwijl `browser.enabled=true` → `plugins.allow` sluit `browser` uit, waardoor de Plugin nooit is geladen.
    - `Failed to start Chrome CDP on port` → het browserproces kon niet worden gestart.
    - `browser.executablePath not found` → het geconfigureerde pad is ongeldig.
    - `browser.cdpUrl must be http(s) or ws(s)` → de geconfigureerde CDP-URL gebruikt een niet-ondersteund schema, zoals `file:` of `ftp:`.
    - `browser.cdpUrl has invalid port` → de geconfigureerde CDP-URL heeft een onjuiste poort of een poort buiten het geldige bereik.
    - `Playwright is not available in this gateway build; '<feature>' is unsupported.` → de huidige Gateway-installatie mist de runtimeafhankelijkheid voor de kernbrowser; installeer OpenClaw opnieuw of werk het bij en start vervolgens de Gateway opnieuw. ARIA-snapshots en eenvoudige schermafbeeldingen van pagina's kunnen nog steeds werken, maar navigatie, AI-snapshots, schermafbeeldingen van elementen met CSS-selectors en PDF-export blijven niet beschikbaar.

  </Accordion>
  <Accordion title="Signalen van Chrome MCP / bestaande sessie">
    - `Could not find DevToolsActivePort for chrome` → de bestaande sessie van Chrome MCP kon nog geen verbinding maken met de geselecteerde browsergegevensmap. Open de inspectiepagina van de browser, schakel foutopsporing op afstand in, houd de browser geopend, keur de eerste verbindingsprompt goed en probeer het opnieuw. Als een aangemelde status niet vereist is, gebruik dan bij voorkeur het beheerde profiel `openclaw`.
    - `No browser tabs found for profile="user"` → het verbindingsprofiel van Chrome MCP heeft geen geopende lokale Chrome-tabbladen.
    - `Remote CDP for profile "<name>" is not reachable` → het geconfigureerde externe CDP-eindpunt is niet bereikbaar vanaf de Gateway-host.
    - `Browser attachOnly is enabled ... not reachable` of `Browser attachOnly is enabled and CDP websocket ... is not reachable` → het profiel dat alleen verbinding maakt heeft geen bereikbaar doel, of het HTTP-eindpunt heeft geantwoord maar de CDP-WebSocket kon nog steeds niet worden geopend.

  </Accordion>
  <Accordion title="Signalen van elementen / schermafbeeldingen / uploads">
    - `fullPage is not supported for element screenshots` → het verzoek om een schermafbeelding combineerde `--full-page` met `--ref` of `--element`.
    - `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → aanroepen voor schermafbeeldingen via Chrome MCP / `existing-session` moeten paginaopname of een snapshot-`--ref` gebruiken, niet CSS-`--element`.
    - `existing-session file uploads do not support element selectors; use ref/inputRef.` → uploadhooks van Chrome MCP hebben snapshotverwijzingen nodig, geen CSS-selectors.
    - `existing-session file uploads currently support one file at a time.` → verstuur bij Chrome MCP-profielen één upload per aanroep.
    - `existing-session dialog handling does not support timeoutMs.` → dialooghooks bij Chrome MCP-profielen ondersteunen geen overschrijvingen van de time-out.
    - `existing-session type does not support timeoutMs overrides.` → laat `timeoutMs` weg voor `act:type` bij `profile="user"` / bestaande-sessieprofielen van Chrome MCP, of gebruik een beheerd/CDP-browserprofiel wanneer een aangepaste time-out vereist is.
    - `response body is not supported for existing-session profiles yet.` → `responsebody` vereist nog steeds een beheerde browser of een onbewerkt CDP-profiel.
    - Verouderde overschrijvingen voor viewport, donkere modus, landinstelling of offlinemodus bij profielen die alleen verbinding maken of externe CDP-profielen → voer `openclaw browser stop --browser-profile <name>` uit om de actieve besturingssessie te sluiten en de Playwright-/CDP-emulatiestatus vrij te geven zonder de volledige Gateway opnieuw te starten.

  </Accordion>
</AccordionGroup>

Gerelateerd:

- [Browser (beheerd door OpenClaw)](/nl/tools/browser)
- [Browserproblemen oplossen](/nl/tools/browser-linux-troubleshooting)

## Als je een upgrade hebt uitgevoerd en er plotseling iets niet meer werkt

De meeste problemen na een upgrade worden veroorzaakt door configuratieafwijkingen of doordat strengere standaardinstellingen nu worden afgedwongen.

<AccordionGroup>
  <Accordion title="1. Het gedrag van authenticatie- en URL-overschrijvingen is gewijzigd">
    ```bash
    openclaw gateway status
    openclaw config get gateway.mode
    openclaw config get gateway.remote.url
    openclaw config get gateway.auth.mode
    ```

    Wat je moet controleren:

    - Als `gateway.mode=remote`, zijn CLI-aanroepen mogelijk gericht op een externe service terwijl je lokale service correct werkt.
    - Expliciete `--url`-aanroepen vallen niet terug op opgeslagen aanmeldgegevens.

    Veelvoorkomende signalen:

    - `gateway connect failed:` → onjuist URL-doel.
    - `unauthorized` → eindpunt bereikbaar, maar onjuiste authenticatie.

  </Accordion>
  <Accordion title="2. Beveiligingsmaatregelen voor binding en authenticatie zijn strenger">
    ```bash
    openclaw config get gateway.bind
    openclaw config get gateway.auth.mode
    openclaw config get gateway.auth.token
    openclaw gateway status
    openclaw logs --follow
    ```

    Wat je moet controleren:

    - Niet-loopbackbindingen (`lan`, `tailnet`, `custom`) vereisen een geldig authenticatiepad voor de Gateway: authenticatie met een gedeeld token/wachtwoord, of een correct geconfigureerde niet-loopback-implementatie van `trusted-proxy`.
    - Oude sleutels zoals `gateway.token` vervangen `gateway.auth.token` niet.

    Veelvoorkomende signalen:

    - `refusing to bind gateway ... without auth` → niet-loopbackbinding zonder een geldig authenticatiepad voor de Gateway.
    - `Connectivity probe: failed` terwijl de runtime actief is → Gateway is actief, maar niet toegankelijk met de huidige authenticatie/URL.

  </Accordion>
  <Accordion title="3. De status van koppeling en apparaatidentiteit is gewijzigd">
    ```bash
    openclaw devices list
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    openclaw doctor
    ```

    Wat je moet controleren:

    - Openstaande apparaatgoedkeuringen voor dashboard/nodes.
    - Openstaande goedkeuringen voor DM-koppelingen na wijzigingen in beleid of identiteit.

    Veelvoorkomende signalen:

    - `device identity required` → niet voldaan aan apparaatauthenticatie.
    - `pairing required` → afzender/apparaat moet worden goedgekeurd.

  </Accordion>
</AccordionGroup>

Als de serviceconfiguratie en runtime na de controles nog steeds niet overeenkomen, installeer je de servicemetadata opnieuw vanuit dezelfde profiel-/statusmap:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Gerelateerd:

- [Authenticatie](/nl/gateway/authentication)
- [Uitvoering op de achtergrond en procestool](/nl/gateway/background-process)
- [Node-koppeling](/nl/gateway/pairing)

## Gerelateerd

- [Doctor](/nl/gateway/doctor)
- [Veelgestelde vragen](/nl/help/faq)
- [Gateway-draaiboek](/nl/gateway)
