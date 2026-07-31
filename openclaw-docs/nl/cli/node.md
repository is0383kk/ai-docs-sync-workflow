---
read_when:
    - De headless Node-host uitvoeren
    - Een niet-macOS-Node koppelen voor system.run
summary: CLI-referentie voor `openclaw node` (headless Node-host)
title: Node
x-i18n:
    generated_at: "2026-07-27T04:54:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 341539d05545ddcbf6175c34af7dca49332ba55906283b9933b9c9b1732c0e4d
    source_path: cli/node.md
    workflow: 16
---

# `openclaw node`

Voer een **headless node-host** uit die verbinding maakt met de Gateway-WebSocket en
`system.run` / `system.which` op deze machine beschikbaar stelt.

Op macOS bevat de menubalk-app deze node-hostruntime al in zijn eigen
nodeverbinding en voegt deze native Mac-mogelijkheden toe. Gebruik `openclaw node run` op een
Mac alleen als je bewust een headless node zonder de app wilt. Als je
beide uitvoert, ontstaan er twee node-identiteiten voor dezelfde machine.

## Waarom een node-host gebruiken?

Gebruik een node-host als je agents **opdrachten op andere machines** in je
netwerk wilt laten uitvoeren zonder daar een volledige bijbehorende macOS-app te installeren.

Veelvoorkomende toepassingen:

- Voer opdrachten uit op externe Linux-/Windows-systemen (buildservers, labmachines, NAS).
- Houd exec **gesandboxed** op de Gateway, maar delegeer goedgekeurde uitvoeringen aan andere hosts.
- Bied een lichtgewicht, headless uitvoeringsdoel voor automatisering of CI-nodes.

De uitvoering wordt nog steeds beschermd door **exec-goedkeuringen** en allowlists per agent op de
node-host, zodat je opdrachttoegang beperkt en expliciet kunt houden.

`openclaw node run` kan na het verbinden door plugins of MCP ondersteunde tools publiceren.
De Gateway vertrouwt standaard descriptors van de gekoppelde node, maar vereist
dat de opdracht van elke descriptor binnen het goedgekeurde opdrachtoppervlak van de node blijft. De
agent ziet elke geaccepteerde descriptor als een normale plugintool, maar de uitvoering verloopt nog steeds
via `node.invoke`, zodat het verbreken van de nodeverbinding de tool uit nieuwe
agentuitvoeringen verwijdert. Gateway-operators kunnen publicatie uitschakelen met
`gateway.nodes.pluginTools.enabled: false`.

Voeg voor declaratieve MCP-tools de normale MCP-serverstructuur toe onder
`nodeHost.mcp.servers` in `openclaw.json` op de nodemachine en herstart vervolgens de
node-host. De node declareert de door goedkeuring beschermde opdrachtfamilie `mcp.tools.call.v1`
en publiceert de vermelde tools na het verbinden; als je de serverlijst
later wijzigt, hoef je niet opnieuw te koppelen. Zie
[Door nodes gehoste MCP-servers](/nl/nodes#node-hosted-mcp-servers).

## Browserproxy (zonder configuratie)

Node-hosts maken automatisch een browserproxy bekend als `browser.enabled` niet
op de node is uitgeschakeld. Hierdoor kan de agent browserautomatisering op die node gebruiken
zonder extra configuratie.

Standaard stelt de proxy het normale browserprofieloppervlak van de node beschikbaar. Als je
`nodeHost.browserProxy.allowProfiles` instelt, wordt de proxy restrictief:
het benaderen van profielen die niet op de allowlist staan, wordt geweigerd en routes voor het
maken/verwijderen van permanente profielen worden via de proxy geblokkeerd.

Schakel dit indien nodig uit op de node:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## Uitvoeren (voorgrond)

```bash
openclaw node run --host <gateway-host> --port 18789
```

Opties:

- `--host <host>`: Gateway-WebSockethost (standaard: `127.0.0.1`)
- `--port <port>`: Gateway-WebSocketpoort (standaard: `18789`)
- `--context-path <path>`: Gateway-WebSocketcontextpad (bijv. `/openclaw-gw`). Wordt aan de WebSocket-URL toegevoegd.
- `--tls`: Gebruik TLS voor de Gateway-verbinding
- `--no-tls`: Forceer een Gateway-verbinding met platte tekst, zelfs wanneer TLS in de lokale Gateway-configuratie is ingeschakeld
- `--tls-fingerprint <sha256>`: Verwachte vingerafdruk van het TLS-certificaat (sha256)
- `--node-id <id>`: Overschrijf de clientinstantie-ID die in de gedeelde SQLite-status is opgeslagen (stelt de koppeling niet opnieuw in)
- `--display-name <name>`: Overschrijf de weergavenaam van de node

## Gateway-authenticatie voor de node-host

`openclaw node run` en `openclaw node install` bepalen Gateway-authenticatie uit configuratie/omgeving (geen vlaggen `--token`/`--password` voor nodeopdrachten):

- `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` worden eerst gecontroleerd.
- Daarna volgt de lokale configuratie als terugval: `gateway.auth.token` / `gateway.auth.password`.
- In lokale modus neemt de node-host bewust `gateway.remote.token` / `gateway.remote.password` niet over.
- Als `gateway.auth.token` / `gateway.auth.password` expliciet via SecretRef is geconfigureerd en niet kan worden opgelost, wordt de node-authenticatie fail-closed beëindigd (zonder maskering door een externe terugval).
- In `gateway.mode=remote` komen externe clientvelden (`gateway.remote.token` / `gateway.remote.password`) ook in aanmerking volgens de externe prioriteitsregels.
- De authenticatiebepaling van de node-host respecteert alleen omgevingsvariabelen van `OPENCLAW_GATEWAY_*`.

Voor een node die verbinding maakt met een Gateway met platte tekst via `ws://`, worden loopback, letterlijke
privé-IP-adressen, `.local` en Tailnet-hosts van `*.ts.net` geaccepteerd. Stel voor andere
vertrouwde privé-DNS-namen `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` in; zonder
deze instelling wordt het starten van de node fail-closed beëindigd en word je gevraagd `wss://`, een SSH-tunnel of
Tailscale te gebruiken. Dit is een opt-in via de procesomgeving, geen configuratiesleutel
voor `openclaw.json`.
`openclaw node install` slaat deze instelling op in de bewaakte nodeservice wanneer deze
aanwezig is in de omgeving van de installatieopdracht.

## Service (achtergrond)

Installeer een headless node-host als gebruikersservice (launchd op macOS, systemd op
Linux, Windows Task Scheduler op Windows).

```bash
openclaw node install --host <gateway-host> --port 18789
```

Opties:

- `--host <host>`: Gateway-WebSockethost (standaard: `127.0.0.1`)
- `--port <port>`: Gateway-WebSocketpoort (standaard: `18789`)
- `--context-path <path>`: Gateway-WebSocketcontextpad (bijv. `/openclaw-gw`). Wordt aan de WebSocket-URL toegevoegd.
- `--tls`: Gebruik TLS voor de Gateway-verbinding
- `--tls-fingerprint <sha256>`: Verwachte vingerafdruk van het TLS-certificaat (sha256)
- `--node-id <id>`: Overschrijf de clientinstantie-ID die in de gedeelde SQLite-status is opgeslagen (stelt de koppeling niet opnieuw in)
- `--display-name <name>`: Overschrijf de weergavenaam van de node
- `--runtime <runtime>`: Serviceruntime (`node`)
- `--force`: Installeer opnieuw/overschrijf indien al geïnstalleerd

Beheer de service:

```bash
openclaw node status
openclaw node start
openclaw node stop
openclaw node restart
openclaw node uninstall
```

Gebruik `openclaw node run` voor een node-host op de voorgrond (geen service).

Serviceopdrachten accepteren `--json` voor machineleesbare uitvoer.

De node-host probeert Gateway-herstarts en netwerkverbrekingen opnieuw binnen hetzelfde proces. Als de
Gateway een definitieve authenticatiepauze vanwege token/wachtwoord/bootstrap meldt, registreert de node-host
de details van de verbroken verbinding en sluit deze af met een niet-nulstatus, zodat launchd/systemd/Task Scheduler de host kan
herstarten met actuele configuratie en referenties. Pauzes waarvoor koppeling vereist is, blijven in
de voorgrondstroom, zodat het openstaande verzoek kan worden goedgekeurd.

## Koppelen

Bij de eerste verbinding wordt een openstaand verzoek voor apparaatkoppeling (`role: node`) op de Gateway gemaakt.

Wanneer de Gateway-host zonder interactie via SSH verbinding kan maken met de node-host (dezelfde gebruiker,
vertrouwde hostsleutel), wordt het openstaande verzoek automatisch goedgekeurd: de Gateway
voert `openclaw node identity --json` via SSH uit op de node-host en keurt het verzoek goed bij
een exacte overeenkomst van de apparaatsleutel. Dit is standaard ingeschakeld; zie
[SSH-geverifieerde automatische goedkeuring van apparaten](/nl/gateway/pairing#ssh-verified-device-auto-approval-default)
voor de vereisten en hoe je dit uitschakelt (`gateway.nodes.pairing.sshVerify: false`).

Keur het verzoek anders handmatig goed via:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Bekijk de lokale node-identiteit waarmee de Gateway vergelijkt:

```bash
openclaw node identity --json
```

Deze opdracht toont de apparaat-ID en openbare sleutel uit de rij `primary` in
`state/openclaw.sqlite` en maakt nooit de database of een nieuwe identiteit aan.

In streng beheerde nodenetwerken kan de Gateway-operator expliciet kiezen voor
automatische goedkeuring van de eerste nodekoppeling vanuit vertrouwde CIDR's:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Dit is standaard uitgeschakeld (`autoApproveCidrs` is niet ingesteld). Het geldt alleen voor
nieuwe `role: node`-koppelingen zonder aangevraagde bereiken, vanaf een client-IP dat de
Gateway vertrouwt. Operator-/browserclients, Control UI, WebChat en upgrades van rol,
bereik, metadata of openbare sleutel moeten nog steeds handmatig worden goedgekeurd.

Als de node de koppeling opnieuw probeert met gewijzigde authenticatiegegevens (rol/bereiken/openbare sleutel),
wordt het vorige openstaande verzoek vervangen en wordt een nieuwe `requestId` gemaakt.
Voer `openclaw devices list` opnieuw uit voordat je het verzoek goedkeurt.

### Identiteits- en koppelingsstatus

De headless node houdt zijn clientinstantie-ID gescheiden van de ondertekende apparaatidentiteit
die de Gateway gebruikt voor koppeling en routering. Deze status bevindt zich in de
OpenClaw-statusmap (standaard `~/.openclaw`, of `$OPENCLAW_STATE_DIR`
wanneer ingesteld):

| Status                                                   | Doel                                                                                                                             |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `state/openclaw.sqlite` (`node_host_config`)             | Clientinstantie-ID, weergavenaam en metagegevens van de Gateway-verbinding. De client verzendt deze ID als `instanceId`.         |
| `state/openclaw.sqlite` (`device_identities`, `primary`) | Ondertekend Ed25519-sleutelpaar en afgeleide apparaat-ID. Voor ondertekende verbindingen is deze apparaat-ID de gerouteerde node-ID en koppelingsidentiteit. |
| `state/openclaw.sqlite` (`device_auth_tokens`)           | Tokens van gekoppelde apparaten, geïndexeerd op cryptografische apparaat-ID en rol.                                              |

`--node-id` wijzigt alleen de clientinstantie-ID in de gedeelde SQLite-status. Het
wijzigt de cryptografische apparaat-ID niet en wist de koppelingsauthenticatie niet. Het migreren van een buiten gebruik gestelde
`node.json` met `openclaw doctor --fix` stelt de koppeling evenmin opnieuw in. Een
node intrekken en opnieuw koppelen:

1. Voer op de Gateway `openclaw nodes remove --node <id|name|ip>` uit.
2. Herstart op de node de geïnstalleerde service met `openclaw node restart`, of
   stop en voer de voorgrondopdracht `openclaw node run` opnieuw uit. Hiermee start de
   stroom voor apparaatkoppeling. Als `openclaw devices list` geen verzoek toont
   en de node `AUTH_DEVICE_TOKEN_MISMATCH` meldt, herstart of voer je deze nog
   eenmaal uit. De geweigerde poging wist het nu ingetrokken lokale token; de volgende
   poging kan om koppeling verzoeken.
3. Voer op de Gateway `openclaw devices list` en vervolgens
   `openclaw devices approve <deviceRequestId>` uit.
4. Herstart of voer de node opnieuw uit. Een client die voor koppeling is gepauzeerd, wordt na
   goedkeuring niet automatisch hervat; deze nieuwe verbinding maakt het afzonderlijke
   verzoek voor het opdrachtoppervlak.
5. Voer op de Gateway `openclaw nodes pending` en vervolgens
   `openclaw nodes approve <nodeRequestId>` uit.

De twee verzoek-ID's zijn verschillend. Een toepasselijk beleid voor vertrouwde CIDR's kan
de eerste stap voor apparaatkoppeling automatisch goedkeuren; goedkeuring van het opdrachtoppervlak blijft
een afzonderlijke controle.

Oudere OpenClaw-releases sloegen de node-hoststatus op in `node.json`, de ondertekende
identiteit in `identity/device.json` en gekoppelde authenticatie in
`identity/device-auth.json`. Stop de node-host en voer
`openclaw doctor --fix` eenmaal uit; Doctor claimt elke buiten gebruik gestelde bron, valideert deze,
importeert en verifieert de canonieke SQLite-rij en verwijdert vervolgens het oude bestand. Normale
nodeopdrachten worden fail-closed beëindigd met deze reparatie-instructie zolang een van de buiten gebruik gestelde bestanden
of een onderbroken Doctor-claim aanwezig blijft. Houd `state/openclaw.sqlite` privé;
dit bevat het apparaatsleutelpaar en de authenticatietokens.

## Exec-goedkeuringen

`system.run` wordt beschermd door lokale exec-goedkeuringen:

- `$OPENCLAW_STATE_DIR/exec-approvals.json`, of
  `~/.openclaw/exec-approvals.json` wanneer de variabele niet is ingesteld
- [Exec-goedkeuringen](/nl/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (bewerken vanaf de Gateway)

Voor goedgekeurde asynchrone node-exec bereidt OpenClaw vóór de vraag om goedkeuring een canonieke `systemRunPlan`
voor. De later goedgekeurde doorsturing van `system.run` hergebruikt dat opgeslagen
plan, zodat wijzigingen in opdracht-/cwd-/sessievelden nadat het goedkeuringsverzoek is
gemaakt, worden geweigerd in plaats van te wijzigen wat de node uitvoert.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Nodes](/nl/nodes)
