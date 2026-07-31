---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Waarom een tool wordt geblokkeerd: sandboxruntime, beleid voor het toestaan/weigeren van tools en poorten voor uitvoering met verhoogde rechten'
title: Sandbox versus toolbeleid versus verhoogde bevoegdheden
x-i18n:
    generated_at: "2026-07-27T05:13:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw heeft drie gerelateerde maar verschillende besturingselementen:

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`) bepaalt **waar tools worden uitgevoerd** (sandboxbackend versus host).
2. **Toolbeleid** (`tools.*`, `tools.sandbox.tools.*`, `agents.entries.*.tools.*`) bepaalt **welke tools beschikbaar/toegestaan zijn**.
3. **Verhoogd** (`tools.elevated.*`, `agents.entries.*.tools.elevated.*`) is een **uitsluitend voor exec bedoeld ontsnappingsmechanisme** om buiten de sandbox te werken wanneer je in een sandbox werkt (standaard `gateway`, of `node` wanneer het exec-doel is geconfigureerd als `node`).

## Snel debuggen

Gebruik de inspectietool om te zien wat OpenClaw _daadwerkelijk_ doet:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Deze toont:

- effectieve sandboxmodus/-scope/-werkruimtetoegang
- of de sessie momenteel in een sandbox werkt (hoofd- versus niet-hoofdsessie)
- effectief toestaan/weigeren van sandboxtools (en of dit afkomstig is van agent/globaal/standaard)
- verhoogde toegangspoorten en sleutelpaden voor oplossingen

## Sandbox: waar tools worden uitgevoerd

Sandboxing wordt bestuurd door `agents.defaults.sandbox.mode`:

- `"off"`: alles wordt op de host uitgevoerd.
- `"non-main"`: alleen niet-hoofdsessies werken in een sandbox (een veelvoorkomende 'verrassing' voor groepen/kanalen).
- `"all"`: alles werkt in een sandbox.

`agents.defaults.sandbox.workspaceAccess` bepaalt wat de sandbox kan zien: `"none"`, `"ro"` of `"rw"`.

Zie [Sandboxing](/nl/gateway/sandboxing) voor de volledige matrix (scope, werkruimtekoppelingen, images).

### Bind-mounts (snelle beveiligingscontrole)

- `docker.binds` _doorbreekt_ het sandboxbestandssysteem: alles wat je koppelt, is binnen de container zichtbaar in de ingestelde modus (`:ro` of `:rw`).
- Als je de modus weglaat, is de standaard lezen-schrijven; geef voor bronbestanden/geheimen de voorkeur aan `:ro`.
- `scope: "shared"` negeert bind-mounts per agent (alleen globale bind-mounts zijn van toepassing).
- OpenClaw valideert bind-bronnen tweemaal: eerst op het genormaliseerde bronpad en daarna opnieuw na omzetting via de diepste bestaande voorouder. Ontsnappingen via bovenliggende symlinks omzeilen controles op geblokkeerde paden of toegestane hoofdmappen niet.
- Niet-bestaande eindpaden worden nog steeds veilig gecontroleerd. Als `/workspace/alias-out/new-file` via een bovenliggende symlink wordt omgezet naar een geblokkeerd pad of een locatie buiten de geconfigureerde toegestane hoofdmappen, wordt de bind-mount geweigerd.
- Door `/var/run/docker.sock` te koppelen, geef je de sandbox feitelijk controle over de host; doe dit alleen bewust.
- Werkruimtetoegang (`workspaceAccess`) staat los van bind-modi.

Zie [Meerdere mappen voor één agent](/nl/gateway/sandboxing#multiple-folders-for-one-agent) voor een configuratie per agent met meerdere hostmappen, toegangsmodi en de veiligheidsopt-in voor externe bronnen.

## Toolbeleid: welke tools bestaan/kunnen worden aangeroepen

Twee lagen zijn van belang:

- **Toolprofiel**: `tools.profile` en `agents.entries.*.tools.profile` (basislijst met toegestane tools)
- **Toolprofiel van provider**: `tools.byProvider[provider].profile` en `agents.entries.*.tools.byProvider[provider].profile`
- **Globaal toolbeleid/toolbeleid per agent**: `tools.allow`/`tools.deny` en `agents.entries.*.tools.allow`/`agents.entries.*.tools.deny`
- **Toolbeleid van provider**: `tools.byProvider[provider].allow/deny` en `agents.entries.*.tools.byProvider[provider].allow/deny`
- **Toolbeleid van sandbox** (alleen van toepassing binnen een sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` en `agents.entries.*.tools.sandbox.tools.*`

Vuistregels:

- `deny` heeft altijd voorrang.
- Als `allow` niet leeg is, wordt al het overige als geblokkeerd beschouwd.
- Het toolbeleid is de harde grens: `/exec` kan een geweigerde `exec`-tool niet overschrijven.
- Het toolbeleid filtert de beschikbaarheid van tools op naam; het inspecteert geen neveneffecten binnen `exec`. Als `exec` is toegestaan, maakt het weigeren van `write`, `edit` of `apply_patch` shellopdrachten niet alleen-lezen.
- `/exec` wijzigt alleen de sessiestandaarden voor geautoriseerde afzenders; het verleent geen toegang tot tools.
- Toolsleutels van providers accepteren `provider` (bijvoorbeeld `google-antigravity`) of `provider/model` (bijvoorbeeld `openai/gpt-5.4`).
- Gateway-logboeken bevatten `agents/tool-policy`-auditvermeldingen wanneer een stap in het toolbeleid tools verwijdert of een toolbeleid van een sandbox een aanroep blokkeert. Gebruik `openclaw logs` om het regellabel, de configuratiesleutel en de betrokken toolnamen te bekijken.

### Toolgroepen (verkorte notaties)

Toolbeleid (globaal, agent, sandbox) ondersteunt `group:*`-vermeldingen die worden uitgebreid naar meerdere tools:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Beschikbare groepen:

| Groep              | Tools                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` wordt geaccepteerd als alias voor `exec`)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | de meeste ingebouwde OpenClaw-tools (uitgezonderd de bestandssysteem- en runtimeprimitieven `read`/`write`/`edit`/`apply_patch`/`exec`/`process`, `canvas` en providerplugins)                                                                                             |
| `group:plugins`    | alle geladen tools die eigendom zijn van plugins, waaronder geconfigureerde MCP-servers die via `bundle-mcp` beschikbaar worden gesteld                                                                                                                                                           |

Weiger voor alleen-lezen-agents naast tools die het bestandssysteem wijzigen ook `group:runtime`, tenzij het bestandssysteembeleid van de sandbox of een afzonderlijke hostgrens de beperking tot alleen-lezen afdwingt.

Voor MCP-servers in een sandbox vormt het toolbeleid van de sandbox een tweede toegangspoort. Als `mcp.servers` is geconfigureerd maar beurten in een sandbox alleen ingebouwde tools tonen, voeg je `bundle-mcp`, `group:plugins` of een MCP-toolnaam/glob met servervoorvoegsel, zoals `outlook__send_mail` of `outlook__*`, toe aan `tools.sandbox.tools.alsoAllow`. Start/herlaad vervolgens de Gateway en leg de toollijst opnieuw vast. Serverglobs gebruiken het voor providers veilige MCP-servervoorvoegsel: niet-`[A-Za-z0-9_-]`-tekens worden `-`, namen die niet met een letter beginnen krijgen het voorvoegsel `mcp-`, en lange of dubbele voorvoegsels kunnen worden afgekapt of van een achtervoegsel worden voorzien.

`openclaw doctor` controleert deze vorm momenteel voor door OpenClaw beheerde servers in `mcp.servers`. MCP-servers die worden geladen vanuit gebundelde pluginmanifesten of Claude `.mcp.json` gebruiken dezelfde sandboxpoort, maar deze diagnose inventariseert die bronnen nog niet; gebruik dezelfde vermeldingen in de lijst met toegestane tools als hun tools verdwijnen tijdens beurten in een sandbox.

## Verhoogd: uitsluitend voor exec 'uitvoeren op host'

Verhoogd verleent **geen** extra tools; het heeft alleen invloed op `exec`.

- Als je in een sandbox werkt, wordt `/elevated on` (of `exec` met `elevated: true`) buiten de sandbox uitgevoerd (goedkeuringen kunnen nog steeds van toepassing zijn).
- Gebruik `/elevated full` om exec-goedkeuringen voor de sessie over te slaan.
- Als je al rechtstreeks werkt, heeft verhoogde toegang feitelijk geen effect (de toegangspoorten blijven van toepassing).
- Verhoogde toegang is **niet** beperkt tot Skills en overschrijft het toestaan/weigeren van tools **niet**.
- Verhoogde toegang verleent geen willekeurige hostoverschrijdende overrides vanuit `host=auto`; deze volgt de normale regels voor exec-doelen en behoudt `node` alleen wanneer het geconfigureerde/sessiedoel al `node` is.
- `/exec` staat los van verhoogde toegang. Het past alleen de exec-standaarden per sessie aan voor geautoriseerde afzenders.

Toegangspoorten:

- Inschakeling: `tools.elevated.enabled` (en optioneel `agents.entries.*.tools.elevated.enabled`)
- Lijsten met toegestane afzenders: `tools.elevated.allowFrom.<provider>` (en optioneel `agents.entries.*.tools.elevated.allowFrom.<provider>`)

Zie [Verhoogde modus](/nl/tools/elevated).

## Veelvoorkomende oplossingen voor een 'sandboxgevangenis'

### 'Tool X geblokkeerd door het toolbeleid van de sandbox'

Sleutels voor oplossingen (kies er één):

- Sandbox uitschakelen: `agents.defaults.sandbox.mode=off` (of per agent `agents.entries.*.sandbox.mode=off`)
- De tool binnen de sandbox toestaan:
  - verwijder deze uit `tools.sandbox.tools.deny` (of per agent `agents.entries.*.tools.sandbox.tools.deny`)
  - of voeg deze toe aan `tools.sandbox.tools.allow` (of per agent toestaan)
- Controleer `openclaw logs` op de vermelding `agents/tool-policy`. Daarin worden de sandboxmodus en of de toestaan- of weigerenregel de tool heeft geblokkeerd vastgelegd.

### "Ik dacht dat dit de hoofdsessie was; waarom draait deze in een sandbox?"

In de modus `"non-main"` zijn groeps-/kanaalsleutels _niet_ de hoofdsessie. Gebruik de sleutel van de hoofdsessie (weergegeven door `sandbox explain`) of schakel over naar de modus `"off"`.

## Gerelateerd

- [Sandboxing](/nl/gateway/sandboxing) -- volledige sandboxreferentie (modi, bereiken, backends, images)
- [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) -- overschrijvingen per agent en voorrangsregels
- [Verhoogde modus](/nl/tools/elevated)
