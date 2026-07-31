---
read_when:
    - Je wilt het GitHub Copilot SDK-harnas voor een agent gebruiken
    - Je hebt configuratievoorbeelden nodig voor de `copilot`-runtime
    - Je koppelt een agent aan Copilot met een abonnement (github / openclaw / copilot) en wilt deze via de Copilot CLI uitvoeren.
summary: Voer beurten van de ingebouwde OpenClaw-agent uit via de externe GitHub Copilot SDK-harnaslaag
title: Copilot SDK-harnas
x-i18n:
    generated_at: "2026-07-27T06:24:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

De externe `@openclaw/copilot`-plugin voert ingebedde Copilot-agentbeurten met een abonnement uit
via de GitHub Copilot CLI (`@github/copilot-sdk`) in plaats van via
de ingebouwde harness van OpenClaw. De Copilot CLI-sessie beheert de onderliggende
agentlus: native tooluitvoering, native Compaction (`infiniteSessions`) en
door de CLI beheerde threadstatus onder `copilotHome`. OpenClaw blijft verantwoordelijk voor chatkanalen,
sessiebestanden, modelselectie, dynamische tools (overbrugd), goedkeuringen,
medialevering, de zichtbare transcriptspiegel, `/btw`-nevenvragen (zie
[Nevenvragen (`/btw`)](#side-questions-btw)) en `openclaw doctor`.

Begin voor de bredere scheiding tussen model/provider/runtime bij
[Agentruntimes](/nl/concepts/agent-runtimes).

## Vereisten

- OpenClaw met de `@openclaw/copilot`-plugin geïnstalleerd.
- Als je configuratie `plugins.allow` gebruikt, neem dan `copilot` op (de manifest-id die de
  plugin declareert). Een allowlist-vermelding voor de naam van het npm-pakket
  `@openclaw/copilot` komt niet overeen en laat de plugin geblokkeerd, zelfs als
  `agentRuntime.id: "copilot"` is ingesteld.
- Een GitHub Copilot-abonnement dat de Copilot CLI kan aansturen, of een
  `gitHubToken`-omgevingsvariabele/auth-profielvermelding voor headless- of Cron-uitvoeringen.
- Een beschrijfbare map `copilotHome`. Standaard is dit `<agentDir>/copilot` wanneer
  OpenClaw een agentmap opgeeft, en anders
  `~/.openclaw/agents/<agentId>/copilot`.

`openclaw doctor` voert het [doctor-contract](#doctor) van de plugin uit voor
eigenaarschap van sessiestatus en toekomstige configuratiemigraties. Het controleert de
Copilot CLI-omgeving niet.

## Installatie

De Copilot-runtime wordt als externe plugin geleverd, zodat het kernpakket `openclaw`
`@github/copilot-sdk` of het platformspecifieke
`@github/copilot-<platform>-<arch>` CLI-binaire bestand (samen ongeveer 260 MB) niet bevat.
Installeer deze alleen voor agents die voor deze runtime kiezen:

```bash
openclaw plugins install @openclaw/copilot
```

De installatiewizard installeert de plugin automatisch wanneer je voor het eerst
een `github-copilot/*`-model selecteert **en** je configuratie dat model (of de
provider ervan) via `agentRuntime: { id: "copilot" }` naar de Copilot-runtime routeert; zie
[Snelle start](#quickstart). Zonder die opt-in gebruikt OpenClaw de ingebouwde
GitHub Copilot-provider en installeert het deze plugin nooit.

De runtime zoekt de SDK in deze volgorde:

1. `import("@github/copilot-sdk")` uit het geïnstalleerde `@openclaw/copilot`-pakket.
2. De terugvalmap `~/.openclaw/npm-runtime/copilot/` (verouderd installatiedoel
   op aanvraag).

Een ontbrekende SDK levert één fout met code `COPILOT_SDK_MISSING` en de
bovenstaande herinstallatieopdracht op.

## Snelle start

Koppel één model (of één provider) aan de harness:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

Stel `agentRuntime.id` in bij één modelvermelding om alleen dat model via
de harness te routeren, of bij een provider om elk model van die provider te routeren.

`github-copilot/auto` is het overdraagbare startpunt. Benoemde Copilot-modellen zijn
afhankelijk van account- en organisatiebeleid; controleer of je geauthenticeerde
Copilot CLI een model daadwerkelijk beschikbaar stelt voordat je het vastlegt.

## Ondersteunde providers

De harness ondersteunt de canonieke provider `github-copilot` (beheerd door
`extensions/github-copilot`), plus aangepaste `models.providers`-vermeldingen wanneer het
model een niet-lege `baseUrl` en een van deze `api`-vormen heeft:

- `anthropic-messages`
- `azure-openai-responses`
- `ollama` (OpenAI-compatibele completions)
- `openai-completions`
- `openai-responses`

Native provider-id's (`openai`, `anthropic`, `google`, `ollama`) blijven onder beheer van
hun native runtimes. Gebruik in plaats daarvan een afzonderlijke aangepaste provider-id om een endpoint
via Copilot BYOK te routeren.

Copilot BYOK-endpoints moeten openbare HTTPS-URL's zijn. De harness geeft de
Copilot SDK per poging een loopbackproxy en stuurt providerverkeer vervolgens
door via het beveiligde fetch-pad van OpenClaw, zodat DNS-pinning en SSRF-beleid
onder beheer van OpenClaw blijven. Gebruik de native OpenClaw-runtime voor lokale Ollama-, LM
Studio- of LAN-modelservers.

## BYOK

Copilot BYOK gebruikt het contract voor aangepaste providers op sessieniveau van de SDK. OpenClaw
geeft het opgeloste modeleindpunt, de API-sleutel, bearer-tokenmodus, headers, model-
id en context-/uitvoerlimieten door; de providertransportlogica blijft in de SDK en niet
in de kern.

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

BYOK-sessies krijgen afzonderlijke sleutels ten opzichte van abonnementssessies en andere
BYOK-endpoints of referenties. Door de sleutel, headers, het model of endpoint te
roteren, wordt een nieuwe Copilot SDK-sessie gestart in plaats van incompatibele status te hervatten.

## Authenticatie

Prioriteitsvolgorde, per agent toegepast tijdens `runCopilotAttempt`:

1. **Expliciete `useLoggedInUser: true`** in de invoer van de poging — gebruikt de
   aangemelde gebruiker van de Copilot CLI onder de `copilotHome` van de agent.
2. **Expliciete `gitHubToken`** in de invoer van de poging (vereist `profileId` +
   `profileVersion`). Voor directe CLI-aanroepen en tests die
   het oplossen van auth-profielen moeten omzeilen.
3. **Via het contract opgeloste `resolvedApiKey` + `authProfileId`** — het primaire
   productiepad. De kern lost het geconfigureerde auth-
   profiel (`src/infra/provider-usage.auth.ts:resolveProviderAuths`) van de agent voor `github-copilot` op voordat
   de harness wordt aangeroepen, zodat een `github-copilot:<profile>`-auth-profiel
   end-to-end werkt voor headless-, Cron- of multiprofielconfiguraties zonder omgevingsvariabelen.
4. **Terugval op omgevingsvariabelen**, gecontroleerd in deze volgorde (de eerste niet-lege waarde wint;
   lege tekenreeksen gelden als afwezig; weerspiegelt de prioriteitsvolgorde van de geleverde provider `github-copilot`
   in `extensions/github-copilot/auth.ts`):
   1. `OPENCLAW_GITHUB_TOKEN` — harness-specifieke overschrijving; hiermee kun je een
      token voor de OpenClaw-harness vastleggen zonder de systeembrede `gh`-/
      Copilot CLI-configuratie te verstoren.
   2. `COPILOT_GITHUB_TOKEN` — standaardomgevingsvariabele van de Copilot SDK/CLI.
   3. `GH_TOKEN` — standaardomgevingsvariabele van de `gh` CLI.
   4. `GITHUB_TOKEN` — generieke terugval op een GitHub-token.

   De samengestelde poolprofiel-id is `env:<NAME>`; de profielversie is een
   niet-omkeerbare sha256-vingerafdruk van het token, zodat het roteren van de omgevingswaarde
   de clientpool correct ongeldig maakt.

5. **Standaard-`useLoggedInUser`** wanneer er geen tokensignaal beschikbaar is.

Elke agent krijgt een eigen `copilotHome`, zodat Copilot CLI-tokens, sessies en
configuratie nooit uitlekken tussen agents op dezelfde machine. Standaard:
`<agentDir>/copilot` (houdt SDK-status buiten dezelfde map als
`models.json` / `auth-profiles.json` van OpenClaw), of
`~/.openclaw/agents/<agentId>/copilot` wanneer geen agentmap wordt opgegeven.
Overschrijf dit met `copilotHome: <path>` in de invoer van de poging voor een aangepaste
locatie (bijvoorbeeld een gedeelde mount voor migratie).

Live harness-tests gebruiken `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` voor een rechtstreeks
token. De gedeelde live-testconfiguratie wist `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`
en `GITHUB_TOKEN` nadat echte auth-profielen in de geïsoleerde test-
homemap zijn geplaatst, zodat een `gh auth token`-waarde die via de specifieke variabele wordt doorgegeven,
onterechte skips voorkomt zonder naar niet-gerelateerde suites te lekken.

## Configuratieoppervlak

De harness leest configuratie uit de invoer per poging (`runCopilotAttempt({...})`)
plus een kleine set standaardwaarden uit omgevingsvariabelen binnen `extensions/copilot/src/`:

| Veld                     | Doel                                                                                                                                                                                                                                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | CLI-statusmap per agent (standaardwaarden hierboven).                                                                                                                                                                                                                                           |
| `model`                  | Tekenreeks of `{ provider, id, api?, baseUrl?, headers?, authHeader? }`. Laat weg om de normale modelselectie van de agent te gebruiken; de harness controleert of de opgeloste provider wordt ondersteund.                                                                                                          |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`. Wordt toegewezen vanuit de oplossing van `ThinkLevel` / `ReasoningLevel` van OpenClaw in `auto-reply/thinking.ts`.                                                                                                                                                          |
| `infiniteSessionConfig`  | Optionele overschrijving voor het SDK-`infiniteSessions`-blok dat door `harness.compact` wordt aangestuurd. Kan veilig ongewijzigd blijven.                                                                                                                                                      |
| `hooksConfig`            | Optionele native Copilot SDK-`SessionHooks`-configuratie voor tool-/MCP-, gebruikersprompt-, sessie- en foutcallbacks. Staat los van de overdraagbare lifecycle-hooks van OpenClaw.                                                                                                             |
| `permissionPolicy`       | Optionele overschrijving voor de `onPermissionRequest`-handler van de SDK voor ingebouwde SDK-tooltypen (`shell`, `write`, `read`, `url`, `mcp`, `memory`, `hook`). Standaard `rejectAllPolicy` als vangnet; zie [Machtigingen en ask_user](#permissions-and-ask_user) voor waarom deze nooit daadwerkelijk wordt geactiveerd. |
| `enableSessionTelemetry` | Optionele telemetrievlag voor de SDK-sessie.                                                                                                                                                                                                                                                    |

OpenClaw-pluginhooks vereisen geen Copilot-specifieke pogingconfiguratie. De
harness voert `before_prompt_build`, `llm_input`, `llm_output` en `agent_end` uit via de
standaard harness-helpers. Succesvolle SDK-compactions voeren ook
`before_compaction` en `after_compaction` uit. Overbrugde OpenClaw-tools voeren
`before_tool_call` uit en rapporteren `after_tool_call`; `hooksConfig` blijft beschikbaar voor
native callbacks die alleen in de SDK bestaan en geen overdraagbaar equivalent hebben.

Niets anders in OpenClaw hoeft van deze velden te weten. Andere plugins,
kanalen en kerncode zien alleen de standaardvorm `AgentHarnessAttemptParams` /
`AgentHarnessAttemptResult`.

## Compaction

Wanneer `harness.compact` wordt uitgevoerd, doet de Copilot SDK-harness het volgende:

1. Hervat de gevolgde SDK-sessie zonder werk in behandeling voort te zetten.
2. Roept de RPC voor geschiedeniscompaction op sessieniveau van de SDK aan.
3. Retourneert het SDK-compactionresultaat zonder compatibiliteitsmarkeringsbestanden
   onder de werkruimte te schrijven.

De transcriptspiegel aan OpenClaw-zijde (hieronder) blijft berichten na de compaction
ontvangen, zodat de voor de gebruiker zichtbare chatgeschiedenis consistent blijft.

## Transcriptspiegeling

`runCopilotAttempt` schrijft de spiegelbare berichten van elke beurt dubbel naar het
OpenClaw-audittranscript via
`extensions/copilot/src/dual-write-transcripts.ts`. De spiegel is begrensd per
sessie (`copilot:${sessionId}`) en heeft een sleutel per bericht
(`${role}:${sha256_16(role,content)}`), zodat opnieuw uitgezonden vermeldingen uit eerdere beurten
botsen met bestaande sleutels op schijf in plaats van te worden gedupliceerd.

Twee lagen foutisolatie omhullen de spiegel, zodat een schrijffout in het transcript
de poging nooit laat mislukken: een interne best-effort-wrapper, plus een
gelaagde beveiliging met `.catch(...)` op pogingsniveau. Fouten worden gelogd, niet
getoond.

## Nevenvragen (`/btw`)

`/btw` is **niet** ingebouwd in deze harness. `createCopilotAgentHarness()`
laat `harness.runSideQuestion` bewust ongedefinieerd
(bevestigd in `extensions/copilot/harness.test.ts`, `describe("runSideQuestion")`),
zodat OpenClaws `/btw`-dispatcher (`src/agents/btw.ts`) terugvalt op
hetzelfde pad dat voor elke niet-Codex-runtime wordt gebruikt: de geconfigureerde modelprovider
wordt rechtstreeks aangeroepen met een korte prompt voor een nevenvraag en het antwoord wordt teruggestreamd via
`streamSimple` (geen CLI-sessie, geen extra poolslot).

Hierdoor blijven Copilot CLI-sessies gereserveerd voor de hoofdbeurtlus van de agent en
blijft het gedrag van `/btw` identiek aan dat van andere niet-Codex-runtimes.

## Doctor

`extensions/copilot/doctor-contract-api.ts` wordt automatisch geladen door
`src/plugins/doctor-contract-registry.ts`. Deze draagt het volgende bij:

- Een lege `legacyConfigRules` (nog geen uitgefaseerde velden).
- Een `normalizeCompatibilityConfig` zonder bewerking (behouden zodat toekomstige uitfaseringen van velden
  een stabiele plek in de bronstructuur hebben).
- Eén `sessionRouteStateOwners`-vermelding: provider `github-copilot`, runtime
  `copilot`, CLI-sessiesleutel `copilot`, voorvoegsel van het authenticatieprofiel `github-copilot:`.

## Beperkingen

- De harness claimt `github-copilot` plus niet-beheerde aangepaste BYOK-provider-id's.
  Ingebouwde provider-id's waarvan het manifest eigenaar is, blijven bij hun eigen runtime, zelfs wanneer
  `agentRuntime.id` wordt geforceerd naar `copilot`.
- Geen TUI-oppervlak; de TUI van PI blijft de terugvaloptie voor runtimes zonder een vergelijkbaar
  oppervlak.
- De sessiestatus van PI wordt niet gemigreerd wanneer een agent overschakelt naar `copilot`.
  De selectie geldt per poging; bestaande PI-sessies blijven geldig.
- `ask_user` gebruikt de providerneutrale vraagruntime van de Gateway. De Control
  UI toont dezelfde vraagkaart als voor andere OpenClaw-vragen, ondersteunde
  kanalen geven keuzeknoppen weer en het volgende gewone tekstbericht in de wachtrij
  handelt die Gateway-record af voordat de SDK-aanvraag terugkeert.

## Machtigingen en ask_user

De handhaving van machtigingen voor gekoppelde OpenClaw-tools vindt **binnen de toolwrapper**
plaats, niet via de callback `onPermissionRequest` van de SDK. Dezelfde
`wrapToolWithBeforeToolCallHook` die PI gebruikt
(`src/agents/agent-tools.before-tool-call.ts`) wordt door
`createOpenClawCodingTools` toegepast op elke codeertool: lusdetectie, beleid voor vertrouwde
plugins, hooks vóór toolaanroepen en tweefasige plugingoedkeuringen via
de Gateway (`plugin.approval.request`) lopen allemaal via exact hetzelfde codepad
als ingebouwde PI-pogingen.

Elke SDK-tool die door de Copilot-toolbridge wordt geretourneerd, wordt gemarkeerd met:

- `overridesBuiltInTool: true` — vervangt de ingebouwde tool van de Copilot CLI met
  dezelfde naam (edit, read, write, bash, ...), zodat elke toolaanroep terug naar
  OpenClaw wordt geleid.
- `skipPermission: true` — instrueert de SDK om
  `onPermissionRequest({kind: "custom-tool"})` niet te activeren voordat de tool wordt aangeroepen. De
  omhulde `execute()` voert de uitgebreidere OpenClaw-beleidscontrole al uit; een
  prompt op SDK-niveau zou de handhaving van OpenClaw omzeilen
  (alles toestaan) of elke toolaanroep blokkeren (alles afwijzen) — geen van beide komt overeen met
  PI-pariteit.

De Codex-harness in de bronstructuur gebruikt dezelfde scheiding: gekoppelde OpenClaw-tools worden
omhuld (`extensions/codex/src/app-server/dynamic-tools.ts`) en de
eigen ingebouwde goedkeuringstypen van codex-app-server
(`item/commandExecution/requestApproval`, `item/fileChange/requestApproval`,
`item/permissions/requestApproval`) worden via `plugin.approval.request` geleid
(`extensions/codex/src/app-server/approval-bridge.ts`). Het equivalent in de Copilot SDK
— bij fouten standaard weigeren met `rejectAllPolicy` voor elk niet-`custom-tool`-type
dat ooit `onPermissionRequest` bereikt — is hetzelfde vangnet en wordt
in de praktijk nooit geactiveerd omdat `overridesBuiltInTool: true` elk
ingebouwd onderdeel vervangt.

Om de laag met omhulde tools beleidsbeslissingen te laten nemen die gelijkwaardig zijn aan die van PI,
stuurt de harness de volledige PI-context voor pogingstools door naar
`createOpenClawCodingTools`: identiteit (`senderIsOwner`, `memberRoleIds`,
`ownerOnlyToolAllowlist`, ...), kanaal/routering (`groupId`,
`currentChannelId`, `replyToMode`, schakelaars voor berichttools), authenticatie
(`authProfileStore`), uitvoeringsidentiteit (`sessionKey` / `runSessionKey` afgeleid
van `sandboxSessionKey`, `runId`), modelcontext (`modelApi`,
`modelContextWindowTokens`, `modelCompat`, `modelHasVision`) en uitvoeringshooks
(`onToolOutcome`, `onYield`). Zonder deze velden weigeren allowlists die alleen voor eigenaren gelden
standaard geruisloos, kan beleid voor pluginvertrouwen niet naar het juiste
bereik worden omgezet en wordt `session_status: "current"` omgezet naar een verouderde sandboxsleutel. De
bridgebouwer is `extensions/copilot/src/tool-bridge.ts`, die de gezaghebbende PI-aanroep
in `src/agents/embedded-agent-runner/run/attempt.ts:1262` weerspiegelt.
`runAttempt` bepaalt de sandboxcontext via de gedeelde
`resolveSandboxContext`-koppeling, geeft een effectieve werkmap door aan de SDK
en stuurt `sandbox` plus de werkruimte voor het starten van subagents door naar de toolbridge.
De bridge stuurt ook de begrensde besturingselementen voor toolconstructie door die
aan de SDK-grens kunnen worden gehandhaafd: `includeCoreTools`, de allowlist voor runtimetools
en `toolConstructionPlan`.

De bridge gebruikt voor PI-pariteit ook de gedeelde helper voor het tooloppervlak van de harness uit
`openclaw/plugin-sdk/agent-harness-tool-runtime`. Wanneer
toolzoeken is ingeschakeld, ziet de SDK compacte besturingstools plus een verborgen
catalogusuitvoerder in plaats van elk OpenClaw-toolschema. Wanneer de codemodus is
ingeschakeld, bouwt de helper hetzelfde besturingsoppervlak voor de codemodus en dezelfde cataloguslevenscyclus
als bij andere agentharnesses. Efficiënte standaardwaarden voor lokale modellen,
runtimecompatibele schemafiltering, maphydratatie en catalogusopschoning
blijven allemaal in de gedeelde helper, zodat Copilot en aan Codex verwante
harnesses niet uiteenlopen.

### GitHub-token op sessieniveau

Het contract van de Copilot SDK maakt onderscheid tussen het GitHub-token op **clientniveau**
(`CopilotClientOptions.gitHubToken`, authenticeert het CLI-proces zelf)
en het token op **sessieniveau** (`SessionConfig.gitHubToken`, bepaalt
inhoudsuitsluiting, modelroutering en quota voor die sessie; wordt zowel bij
`createSession` als `resumeSession` gehonoreerd). De harness bepaalt de authenticatie eenmaal via
`resolveCopilotAuth` en stelt beide velden in wanneer de authenticatiemodus `gitHubToken` is
(een expliciete `auth.gitHubToken` of een volgens het contract bepaalde `resolvedApiKey` uit
een geconfigureerd `github-copilot`-authenticatieprofiel). Wanneer de bepaalde modus
`useLoggedInUser` is, wordt het veld op sessieniveau weggelaten, zodat de SDK
de identiteit blijft afleiden van de aangemelde identiteit.

`ask_user` gebruikt `SessionConfig.onUserInputRequest`. De bridge registreert SDK-keuzes
of vrije-tekstprompts zonder opties als Gateway-vragen, accepteert keuze-indexen
of labels voor aanvragen met vaste keuzes en accepteert vrije antwoorden
wanneer de SDK-aanvraag die toestaat. Als de OpenClaw-poging wordt afgebroken, wordt de
Gateway-record geannuleerd en een leeg SDK-antwoord geretourneerd.

## Gerelateerd

- [Agentruntimes](/nl/concepts/agent-runtimes)
- [Codex-harness](/nl/plugins/codex-harness)
- [Plugins voor agentharnesses (SDK-referentie)](/nl/plugins/sdk-agent-harness)
