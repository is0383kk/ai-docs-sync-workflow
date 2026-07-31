---
read_when:
    - Je wilt OpenClaw OAuth van begin tot eind begrijpen
    - Je ondervindt problemen met het ongeldig worden van tokens / uitloggen
    - Je wilt Claude CLI- of OAuth-authenticatiestromen
    - Je wilt meerdere accounts of profielroutering
summary: 'OAuth in OpenClaw: tokenuitwisseling, opslag en patronen voor meerdere accounts'
title: OAuth
x-i18n:
    generated_at: "2026-07-27T06:12:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ef94af0601b7d57bb7e2d53c3d8231708b401251eca7dc1bb1e7e4fc09b46da
    source_path: concepts/oauth.md
    workflow: 16
---

OpenClaw ondersteunt OAuth ("abonnementsauthenticatie") voor providers die dit aanbieden,
met name **OpenAI Codex (ChatGPT OAuth)** en **hergebruik van Anthropic Claude CLI**.
Voor Anthropic is de praktische verdeling:

- **Anthropic API-sleutel**: normale facturering voor de Anthropic API.
- **Anthropic Claude CLI / abonnementsauthenticatie binnen OpenClaw**: medewerkers van Anthropic
  hebben ons verteld dat dit gebruik weer is toegestaan, dus beschouwt OpenClaw hergebruik van Claude CLI en
  gebruik van `claude -p` als toegestaan voor deze integratie, tenzij Anthropic
  nieuw beleid publiceert. Voor Anthropic in productie blijft authenticatie met een API-sleutel
  de veiligere aanbevolen methode.

OpenClaw slaat zowel authenticatie met een OpenAI API-sleutel als ChatGPT/Codex OAuth op onder de
canonieke provider-id `openai`. Oudere `openai-codex:*`-profiel-id's en
`auth.order.openai-codex`-vermeldingen zijn verouderde status die wordt hersteld door
`openclaw doctor --fix`; gebruik `openai:*`-profiel-id's en `auth.order.openai` voor
nieuwe configuratie.

Deze pagina behandelt:

- hoe de OAuth-**tokenuitwisseling** werkt (PKCE)
- waar tokens worden **opgeslagen** (en waarom)
- hoe je **meerdere accounts** verwerkt (profielen + overschrijvingen per sessie)

Providerplugins die hun eigen OAuth- of API-sleutelprocedure leveren, gebruiken
hetzelfde toegangspunt:

```bash
openclaw models auth login --provider <id>
```

## De tokenopvang (waarom deze bestaat)

OAuth-providers genereren doorgaans bij elke aanmelding/vernieuwing een nieuwe vernieuwingstoken.
Sommige providers maken de vorige vernieuwingstoken ongeldig wanneer voor dezelfde
gebruiker/app een nieuwe wordt uitgegeven. Praktisch gevolg: je meldt je aan via OpenClaw _en_
via Claude Code / Codex CLI, waarna een van beide later willekeurig wordt afgemeld.

Om dit te beperken, behandelt OpenClaw de opslag voor authenticatieprofielen als een **tokenopvang**:

- de runtime leest referenties voor elke agent vanaf één locatie
- meerdere profielen kunnen naast elkaar bestaan en deterministisch worden gerouteerd
- hergebruik van een externe CLI is providerspecifiek: zodra OpenClaw een lokaal OAuth-
  profiel voor een provider beheert, is de lokale vernieuwingstoken canoniek. Als die lokale
  vernieuwingstoken wordt geweigerd, meldt OpenClaw dat het profiel opnieuw moet worden
  geauthenticeerd, in plaats van terug te vallen op tokenmateriaal van de externe CLI.
  Het opstarten via Codex CLI is nog beperkter: het kan alleen een leeg profiel in
  `openai:default`-stijl initialiseren voordat OpenClaw OAuth voor die
  provider beheert; daarna blijven door OpenClaw beheerde vernieuwingen canoniek
- status- en opstartpaden beperken de detectie van externe CLI's tot de providerset
  die al is geconfigureerd, zodat voor een configuratie met één provider niet wordt gezocht
  in de aanmeldingsopslag van een niet-gerelateerde CLI

## Opslag (waar tokens zich bevinden)

Geheimen worden per agent opgeslagen, met de logische naam `auth-profiles.json` als sleutel (de
onderliggende opslag is de SQLite-database van de agent; de JSON-naam blijft behouden voor
compatibiliteit en weergave in hulpmiddelen):

- Authenticatieprofielen (OAuth + API-sleutels + optionele verwijzingen op waardeniveau):
  `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Verouderd compatibiliteitsbestand: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (statische `api_key`-vermeldingen worden bij ontdekking verwijderd)

Verouderd bestand dat alleen voor import wordt gebruikt (nog steeds ondersteund, maar niet de hoofdopslag):

- `~/.openclaw/credentials/oauth.json` (bij het eerste gebruik geïmporteerd in de opslag voor authenticatieprofielen)

Al het bovenstaande respecteert ook `$OPENCLAW_STATE_DIR` (overschrijving van de statusmap). Volledige referentie: [/gateway/configuration-reference#auth-storage](/nl/gateway/configuration-reference#auth-storage)

Zie [Geheimenbeheer](/nl/gateway/secrets) voor statische geheimverwijzingen en het activeringsgedrag van runtime-snapshots.

Wanneer een secundaire agent geen lokaal authenticatieprofiel heeft, gebruikt OpenClaw doorlezende
overerving vanuit de opslag van de standaard-/hoofd-agent; de opslag van de hoofd-agent wordt
bij het lezen niet gekloond. Vooral OAuth-vernieuwingstokens zijn gevoelig: normale
kopieerprocedures slaan ze standaard over, omdat sommige providers vernieuwingstokens na gebruik
rouleren of ongeldig maken. Configureer een afzonderlijke OAuth-aanmelding voor een agent wanneer
deze een onafhankelijk account nodig heeft.

## Hergebruik van Anthropic Claude CLI

OpenClaw ondersteunt hergebruik van Anthropic Claude CLI en `claude -p` als een toegestane
authenticatiemethode. Als er op de host al een lokale Claude-aanmelding bestaat,
kan onboarding/configuratie deze rechtstreeks hergebruiken. De Anthropic-installatietoken blijft
beschikbaar als ondersteunde methode voor tokenauthenticatie, maar OpenClaw geeft de voorkeur aan
hergebruik van Claude CLI wanneer dit beschikbaar is.

<Warning>
In de openbare documentatie van Anthropic voor Claude Code staat dat rechtstreeks gebruik van Claude Code binnen
de limieten van Claude-abonnementen blijft, en medewerkers van Anthropic hebben ons verteld dat Claude
CLI-gebruik in de stijl van OpenClaw weer is toegestaan. OpenClaw beschouwt daarom hergebruik van Claude CLI en
gebruik van `claude -p` als toegestaan voor deze integratie, tenzij Anthropic
nieuw beleid publiceert.

Zie voor de huidige documentatie van Anthropic over abonnementen voor rechtstreeks Claude Code-gebruik [Claude Code
gebruiken met je Pro- of Max-
abonnement](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
en [Claude Code gebruiken met je Team- of Enterprise-
abonnement](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/).

Zie [OpenAI
Codex](/nl/providers/openai), [Qwen Cloud Coding
Plan](/nl/providers/qwen), [MiniMax Coding Plan](/nl/providers/minimax)
en [Z.AI / GLM Coding Plan](/nl/providers/zai) als je andere opties in abonnementsstijl in OpenClaw wilt.
</Warning>

## OAuth-uitwisseling (hoe de aanmelding werkt)

De interactieve aanmeldingsprocedures van OpenClaw zijn geïmplementeerd in `openclaw/plugin-sdk/llm.ts` en gekoppeld aan de wizards/opdrachten.

### Anthropic-installatietoken

Vorm van de procedure:

1. maak de token aan door `claude setup-token` uit te voeren op een willekeurige machine met Claude Code en start daarna Anthropic-installatietoken of plak-token vanuit OpenClaw
2. OpenClaw slaat de resulterende Anthropic-referentie op in een authenticatieprofiel
3. de modelselectie blijft op `anthropic/...`
4. bestaande Anthropic-authenticatieprofielen blijven beschikbaar voor terugdraaien/volgordebeheer

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth wordt expliciet ondersteund voor gebruik buiten de Codex CLI, waaronder OpenClaw-workflows.

De aanmeldingsopdracht gebruikt de canonieke OpenAI-provider-id:

```bash
openclaw models auth login --provider openai
```

Gebruik `--profile-id openai:<name>` voor meerdere ChatGPT/Codex OAuth-accounts in
één agent. Gebruik `openai-codex:<name>` niet voor nieuwe profielen. Doctor migreert
dat oudere voorvoegsel naar een botsingsvrije `openai:*`-profiel-id; voer
`openclaw models auth list --provider openai` na het herstel uit voordat je
profiel-id's naar `auth.order` of `/model ...@<profileId>` kopieert.

Vorm van de procedure (PKCE):

1. genereer een PKCE-verificatiecode/-challenge en een willekeurige `state`
2. open `https://auth.openai.com/oauth/authorize?...` (bereik
   `openid profile email offline_access`)
3. probeer de callback op `http://localhost:1455/auth/callback` op te vangen (de
   callbackhost is standaard `localhost` en accepteert alleen loopbackhosts;
   overschrijf dit met `OPENCLAW_OAUTH_CALLBACK_HOST`)
4. als je een code kunt plakken voordat de callback binnenkomt (of je werkt
   extern/headless en de callback kan niet worden gebonden), plak dan in plaats daarvan de omleidings-URL/code
   \- handmatig plakken wedijvert met de browsercallback en wat als eerste is
   voltooid, wint
5. wissel de code uit bij `https://auth.openai.com/oauth/token`
6. extraheer `accountId` uit de toegangstoken en sla `{ access, refresh, expires, accountId }` op

Het wizardpad is `openclaw onboard` → authenticatiekeuze `openai`.

## Vernieuwing + vervaldatum

Profielen slaan een `expires`-tijdstempel op. Tijdens runtime:

- als `expires` in de toekomst ligt, gebruik je de opgeslagen toegangstoken
- als deze is verlopen, vernieuw je deze (onder een bestandsvergrendeling) en overschrijf je de opgeslagen referenties
- als een secundaire agent een overgeërfd OAuth-profiel van de hoofd-agent leest,
  schrijft de vernieuwing terug naar de opslag van de hoofd-agent in plaats van de vernieuwingstoken
  naar de opslag van de secundaire agent te kopiëren
- extern beheerde CLI-referenties (Claude CLI, beperkte initialisatie via Codex CLI;
  zie [De tokenopvang](#the-token-sink-why-it-exists)) worden opnieuw gelezen in plaats van
  een gekopieerde vernieuwingstoken te gebruiken. Als een beheerde vernieuwing mislukt, meldt OpenClaw
  dat het betreffende profiel opnieuw moet worden geauthenticeerd, in plaats van
  tokenmateriaal van de externe CLI te retourneren.

De vernieuwingsprocedure verloopt automatisch; doorgaans hoef je tokens niet handmatig te beheren.

## Meerdere accounts (profielen) + routering

Twee patronen:

### 1) Aanbevolen: afzonderlijke agents

Als je wilt dat "persoonlijk" en "werk" nooit met elkaar interageren, gebruik je geïsoleerde agents (afzonderlijke sessies + referenties + werkruimte):

```bash
openclaw agents add work
openclaw agents add personal
```

Configureer daarna de authenticatie per agent (wizard) en routeer chats naar de juiste agent.

### 2) Geavanceerd: meerdere profielen in één agent

De opslag voor authenticatieprofielen ondersteunt meerdere profiel-id's voor dezelfde provider.
Kies welk profiel wordt gebruikt:

- globaal via de configuratievolgorde (`auth.order`)
- per sessie via `/model ...@<profileId>`

Voorbeeld (sessieoverschrijving):

- `/model Opus@anthropic:work`

Geef bestaande profiel-id's weer met:

```bash
openclaw models auth list --provider <id>
```

Gerelateerde documentatie:

- [Model-failover](/nl/concepts/model-failover) (regels voor roulatie + afkoelperiode)
- [Slash-opdrachten](/nl/tools/slash-commands) (opdrachtinterface)

## Gerelateerd

- [Authenticatie](/nl/gateway/authentication) - overzicht van authenticatie bij modelproviders
- [Geheimen](/nl/gateway/secrets) - opslag van referenties en SecretRef
- [Configuratiereferentie](/nl/gateway/configuration-reference#auth-storage) - configuratiesleutels voor authenticatie
