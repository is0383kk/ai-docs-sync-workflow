---
read_when:
    - Je hebt een Plugin-hook of -tool nodig om toestemming te vragen voordat een neveneffect wordt uitgevoerd
    - Je moet configureren waar goedkeuringsverzoeken voor plugins worden afgeleverd
    - Je maakt een keuze tussen optionele tools, uitvoeringsgoedkeuringen en plugingoedkeuringen
sidebarTitle: Permission requests
summary: Vraag gebruikers om toolaanroepen van plugins en door plugins beheerde toestemmingsverzoeken goed te keuren
title: Verzoeken om Plugin-machtigingen
x-i18n:
    generated_at: "2026-07-27T05:13:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 675534212e70cc7b2e7bdc801955929c6a8156b08d620483edf0133afc3bfdaa
    source_path: plugins/plugin-permission-requests.md
    workflow: 16
---

Pluginmachtigingsverzoeken laten Plugincode een toolaanroep of een door een Plugin beheerde
bewerking onderbreken totdat een gebruiker deze goedkeurt of weigert. Ze gebruiken de Gateway-
`plugin.approval.*`-flow en dezelfde goedkeuringsinterfaces die goedkeuringsknoppen in chats
en `/approve`-opdrachten verwerken.

Gebruik Pluginmachtigingsverzoeken voor machtigingen van Plugins/apps. Ze vervangen
goedkeuringen voor uitvoering op de host, optionele toestemmingslijsten voor tools of de systeemeigen
machtigingscontrole van Codex niet.

## Kies de juiste poort

Kies de poort die past bij het beslismoment dat je nodig hebt:

| Poort                            | Gebruik deze wanneer                                                       | Wat deze beheert                                                                                                          |
| -------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Optionele tools                  | Een tool pas zichtbaar voor het model mag zijn nadat de gebruiker instemt.  | Beschikbaarstelling van tools via `tools.allow`.                                                                     |
| Pluginmachtigingsverzoeken       | Een Pluginhook of door een Plugin beheerde bewerking vóór één actie toestemming moet vragen. | Goedkeuring tijdens runtime via `plugin.approval.*`.                                                        |
| Uitvoeringsgoedkeuringen         | Een hostopdracht of shellachtige tool goedkeuring van de beheerder vereist. | Uitvoeringsbeleid van de host en permanente toestemmingslijsten voor uitvoering.                                          |
| Systeemeigen machtigingsverzoeken van Codex | Codex toestemming vraagt vóór systeemeigen shell-, bestands-, MCP- of app-serveracties. | Goedkeuringsafhandeling door de Codex-app-server of systeemeigen hook, doorgestuurd via Plugingoedkeuringen wanneer OpenClaw de prompt beheert. |
| MCP-goedkeuringsverzoeken        | Een Codex MCP-server goedkeuring voor een toolaanroep vraagt.               | MCP-goedkeuringsreacties die via OpenClaw-Plugingoedkeuringen worden doorgegeven.                                          |

Optionele tools vormen een poort tijdens de detectiefase. Pluginmachtigingsverzoeken vormen een
poort per aanroep. Gebruik beide wanneer een gevoelige tool expliciete instemming moet vereisen
voordat het model deze kan zien, plus goedkeuring voordat de actie wordt uitgevoerd.

## Vraag goedkeuring vóór een toolaanroep

De meeste door Plugins opgestelde prompts moeten beginnen in een `before_tool_call`-hook. De hook
wordt uitgevoerd nadat het model een tool selecteert en voordat OpenClaw deze uitvoert:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "deploy-policy",
  name: "Implementatiebeleid",
  register(api) {
    api.on("before_tool_call", async (event) => {
      if (event.toolName !== "deploy_service") {
        return;
      }

      const environment =
        typeof event.params.environment === "string" ? event.params.environment : "onbekend";

      return {
        requireApproval: {
          title: "Service implementeren",
          description: `Service implementeren in ${environment}.`,
          severity: environment === "production" ? "critical" : "warning",
          allowedDecisions:
            environment === "production"
              ? ["allow-once", "deny"]
              : ["allow-once", "allow-always", "deny"],
          timeoutMs: 120_000,
          onResolution(decision) {
            console.log(`implementatiegoedkeuring afgehandeld: ${decision}`);
          },
        },
      };
    });
  },
});
```

Schrijf de prompttekst voor degene die de actie zal goedkeuren:

- Houd `title` kort en actiegericht; de Gateway beperkt deze tot 80 tekens.
- Houd `description` specifiek en afgebakend; de Gateway beperkt deze tot 512
  tekens.
- Vermeld de actie, het doel en het risico. Neem geen geheimen, tokens of
  privépayloads op die niet in goedkeuringsinterfaces voor chats mogen verschijnen.
- `severity` gebruikt standaard `"warning"` wanneer deze is weggelaten. Gebruik `"critical"` alleen voor
  acties waarbij een verkeerde beslissing productieschade of gegevensverlies kan veroorzaken.
- `allowedDecisions` gebruikt standaard `["allow-once", "allow-always", "deny"]` wanneer
  deze is weggelaten. Geef `["allow-once", "deny"]` door wanneer permanent vertrouwen onveilig is voor
  die actie.
- `timeoutMs` is standaard 120000 (2 minuten) en wordt beperkt tot 600000 (10
  minuten), ongeacht de aangevraagde waarde.

## Beslissingsgedrag

OpenClaw maakt een wachtende goedkeuring met een `plugin:`-ID, levert deze aan de
beschikbare goedkeuringsinterfaces en wacht op een beslissing.

| Beslissing        | Resultaat                                                                 |
| ----------------- | ------------------------------------------------------------------------- |
| `allow-once`      | De huidige aanroep gaat door.                                             |
| `allow-always`    | De huidige aanroep gaat door en de beslissing wordt aan de Plugin doorgegeven. |
| `deny`            | De aanroep wordt geblokkeerd met een geweigerd toolresultaat.             |
| Time-out          | De aanroep wordt geblokkeerd.                                             |
| Annulering        | De aanroep wordt geblokkeerd wanneer de uitvoering wordt afgebroken.      |
| Geen goedkeuringsroute | De aanroep wordt geblokkeerd omdat geen verbonden goedkeuringsinterface deze kan afhandelen. |

Alleen de exacte door het verzoek toegestane beslissingen `allow-once` en `allow-always`
staan uitvoering toe. Onbekende, ongeldige, niet-overeenkomende, ontbrekende en verlopen
beslissingen worden standaard geweigerd. Het verouderde veld `timeoutBehavior` blijft geaccepteerd voor
Plugincompatibiliteit, maar is afgeschaft en wordt genegeerd; stel het niet in bij nieuwe hooks.

`allow-always` is alleen permanent wanneer de aanvragende Plugin of runtime
deze persistentie implementeert. Voor gewone `before_tool_call.requireApproval`-hooks
behandelt OpenClaw `allow-once` en `allow-always` als goedkeuringsbeslissingen voor de
huidige aanroep en geeft het de afgehandelde waarde door aan `onResolution`. Als je Plugin
`allow-always` aanbiedt, documenteer en implementeer dan exact welke toekomstige aanroepen
worden vertrouwd.

Als de hook ook `params` retourneert, past OpenClaw die parameterwijzigingen pas toe
nadat de goedkeuring is geslaagd. Een hook met lagere prioriteit kan nog steeds blokkeren nadat een
hook met hogere prioriteit om goedkeuring heeft gevraagd.

`allowedDecisions` beperkt de knoppen en opdrachten die aan de gebruiker worden getoond. De
Gateway weigert een afhandelingspoging voor elke beslissing die niet door het verzoek werd aangeboden.

## Stuur goedkeuringsprompts door

Goedkeuringsprompts kunnen worden afgehandeld in lokale gebruikersinterfaces of in chatkanalen die
goedkeuringsafhandeling ondersteunen. Configureer `approvals.plugin` om Plugingoedkeuringsprompts
door te sturen naar expliciete chatdoelen:

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [{ channel: "slack", to: "U12345678" }],
    },
  },
}
```

`approvals.plugin` staat los van `approvals.exec`. Het inschakelen van doorsturen van
uitvoeringsgoedkeuringen stuurt geen Plugingoedkeuringsprompts door, en het inschakelen van
doorsturen van Plugingoedkeuringen wijzigt het uitvoeringsbeleid van de host niet.

Wanneer een prompt handmatige goedkeuringstekst bevat, handel je deze af met een van de aangeboden
beslissingen:

```text
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

Zie [Geavanceerde uitvoeringsgoedkeuringen](/nl/tools/exec-approvals-advanced#plugin-approval-forwarding)
voor het volledige doorstuurmodel, goedkeuringsgedrag binnen dezelfde chat, systeemeigen
kanaallevering en kanaalspecifieke regels voor goedkeurders.

## Systeemeigen Codex-machtigingen

Systeemeigen machtigingsprompts van Codex kunnen ook via Plugingoedkeuringen worden verzonden, maar
ze hebben een andere eigenaar dan door Plugins opgestelde hooks.

- Goedkeuringsverzoeken van de Codex-app-server worden na controle door Codex via OpenClaw doorgestuurd.
- De relay van de systeemeigen hook `permission_request` kan via
  `plugin.approval.request` toestemming vragen wanneer die relay is ingeschakeld.
- Goedkeuringsverzoeken voor MCP-tools worden via Plugingoedkeuringen doorgestuurd wanneer Codex
  `_meta.codex_approval_kind` markeert als `"mcp_tool_call"`.

Zie [Codex-harnasruntime](/nl/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
voor het Codex-specifieke gedrag en de terugvalregels.

## Problemen oplossen

**De tool meldt dat Plugingoedkeuringen niet beschikbaar zijn.** Geen goedkeuringsinterface of
geconfigureerde goedkeuringsroute heeft het verzoek geaccepteerd. Verbind een client die goedkeuringen
ondersteunt, gebruik een kanaal dat `/approve` binnen dezelfde chat ondersteunt, of configureer
`approvals.plugin`.

**`allow-always` verschijnt, maar de volgende aanroep vraagt opnieuw om toestemming.** De algemene
Plugingoedkeuringsflow slaat vertrouwen voor willekeurige hooks niet automatisch permanent op. Sla
door de Plugin beheerd vertrouwen in je Plugin op na `onResolution("allow-always")`, of
bied alleen `allow-once` en `deny` aan.

**`/approve` weigert de beslissing.** Het verzoek heeft
`allowedDecisions` beperkt. Gebruik een van de beslissingen die in de prompt worden weergegeven.

**Een prompt van Discord, Matrix, Slack of Telegram wordt anders doorgestuurd dan
uitvoeringsgoedkeuringen.** Plugingoedkeuringen en uitvoeringsgoedkeuringen gebruiken afzonderlijke
configuratie en kunnen verschillende autorisatiecontroles gebruiken. Controleer `approvals.plugin`
en de ondersteuning voor Plugingoedkeuringen van het kanaal in plaats van alleen `approvals.exec`
te controleren.

## Gerelateerd

- [Pluginhooks](/nl/plugins/hooks#tool-call-policy)
- [Plugins bouwen](/nl/plugins/building-plugins#registering-tools)
- [Geavanceerde uitvoeringsgoedkeuringen](/nl/tools/exec-approvals-advanced#plugin-approval-forwarding)
- [Gateway-protocol](/nl/gateway/protocol)
- [Codex-harnasruntime](/nl/plugins/codex-harness-runtime#native-permissions-and-mcp-elicitations)
