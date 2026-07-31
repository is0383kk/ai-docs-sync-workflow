---
read_when:
    - Je wilt migreren van Hermes of een ander agentsysteem naar OpenClaw
    - Je voegt een migratieprovider toe die eigendom is van een Plugin
summary: CLI-referentie voor `openclaw migrate` (status importeren uit een ander agentsysteem)
title: Migreren
x-i18n:
    generated_at: "2026-07-27T05:46:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

Importeer status uit een ander agentsysteem via een migratieprovider die eigendom is van een plugin. Meegeleverde providers ondersteunen Claude, Codex CLI en [Hermes](/nl/install/migrating-hermes); plugins kunnen aanvullende providers registreren.

<Tip>
Zie voor gebruikersgerichte stappenplannen [Migreren vanuit Claude](/nl/install/migrating-claude) en [Migreren vanuit Hermes](/nl/install/migrating-hermes). De [migratiehub](/nl/install/migrating) vermeldt alle paden.
</Tip>

## Opdrachten

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

Als je `openclaw migrate <provider>` zonder andere vlaggen uitvoert, wordt de migratie gepland en weergegeven en wordt er (in een TTY) om bevestiging gevraagd voordat deze wordt toegepast. `openclaw migrate plan <provider>` en `openclaw migrate apply <provider>` splitsen de voorvertoning en toepassing op in afzonderlijke subopdrachten met dezelfde vlaggen.

<ParamField path="<provider>" type="string">
  Naam van een geregistreerde migratieprovider, bijvoorbeeld `hermes`. Voer `openclaw migrate list` uit om de geïnstalleerde providers te bekijken.
</ParamField>
<ParamField path="--dry-run" type="boolean">
  Stel het plan op en sluit af zonder de status te wijzigen.
</ParamField>
<ParamField path="--from <path>" type="string">
  Overschrijf de bronmap voor de status. Hermes volgt `$HERMES_HOME` en het actieve profiel en gebruikt vervolgens de platformstandaard (`~/.hermes` of `%LOCALAPPDATA%\hermes`). Codex gebruikt standaard `~/.codex` (of `$CODEX_HOME`), Claude gebruikt standaard `~/.claude`.
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  Importeer ondersteunde aanmeldgegevens zonder om bevestiging te vragen. Bij interactieve toepassing wordt vóór het importeren van gedetecteerde authenticatiegegevens om bevestiging gevraagd, waarbij ja standaard is geselecteerd; niet-interactieve `--yes` vereist `--include-secrets` om deze te importeren.
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  Sla de import van authenticatiegegevens over, inclusief de interactieve bevestigingsvraag.
</ParamField>
<ParamField path="--overwrite" type="boolean">
  Sta toe dat bij toepassing bestaande doelen worden vervangen wanneer het plan conflicten meldt.
</ParamField>
<ParamField path="--yes" type="boolean">
  Sla de bevestigingsvraag over. Vereist in niet-interactieve modus.
</ParamField>
<ParamField path="--skill <name>" type="string">
  Selecteer één te kopiëren skillitem op skillnaam of item-id. Herhaal de vlag om meerdere skills te migreren. Wanneer deze wordt weggelaten, tonen interactieve Codex-migraties een selectievakjeskiezer en behouden niet-interactieve migraties alle geplande skills.
</ParamField>
<ParamField path="--plugin <name>" type="string">
  Selecteer één installatie-item voor een Codex-plugin op pluginnaam of item-id. Herhaal de vlag om meerdere Codex-plugins te migreren. Wanneer deze wordt weggelaten, tonen interactieve Codex-migraties een systeemeigen selectievakjeskiezer voor Codex-plugins en behouden niet-interactieve migraties alle geplande plugins. Geldt alleen voor vanuit de bron geïnstalleerde `openai-curated` Codex-plugins die door de inventaris van de Codex-appserver zijn gevonden.
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  Alleen voor Codex. Dwingt vóór het plannen van systeemeigen pluginactivering een nieuwe `app/list`-doorloop van de Codex-appserver van de bron af. Standaard uitgeschakeld om de migratieplanning snel te houden.
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  Pad of map voor het back-uparchief vóór de migratie. Wordt doorgegeven aan `openclaw backup create`.
</ParamField>
<ParamField path="--no-backup" type="boolean">
  Sla de back-up vóór toepassing over. Vereist `--force` wanneer lokale OpenClaw-status bestaat.
</ParamField>
<ParamField path="--force" type="boolean">
  Vereist naast `--no-backup` wanneer de toepassing anders zou weigeren de back-up over te slaan.
</ParamField>
<ParamField path="--json" type="boolean">
  Druk het plan of toepassingsresultaat af als JSON. Met `--json` en zonder `--yes` drukt de toepassing het plan af en wordt de status niet gewijzigd.
</ParamField>

## Veiligheidsmodel

`openclaw migrate` toont eerst een voorvertoning.

<AccordionGroup>
  <Accordion title="Voorvertoning vóór toepassing">
    De provider retourneert een gespecificeerd plan voordat er iets verandert, inclusief conflicten, overgeslagen items en gevoelige items. JSON-plannen, toepassingsuitvoer en migratierapporten verbergen geneste sleutels die op geheimen lijken, zoals API-sleutels, tokens, autorisatieheaders, cookies en wachtwoorden.

    `openclaw migrate apply <provider>` toont een voorvertoning van het plan en vraagt om bevestiging voordat de status wordt gewijzigd, tenzij `--yes` is ingesteld. In niet-interactieve modus vereist de toepassing `--yes`.

  </Accordion>
  <Accordion title="Back-ups">
    De toepassing maakt en verifieert een OpenClaw-back-up voordat de migratie wordt toegepast. Als er nog geen lokale OpenClaw-status bestaat, wordt de back-upstap overgeslagen en gaat de migratie verder. Geef zowel `--no-backup` als `--force` op om een back-up over te slaan wanneer er status bestaat.
  </Accordion>
  <Accordion title="Conflicten">
    De toepassing weigert door te gaan wanneer het plan conflicten bevat. Controleer het plan en voer het vervolgens opnieuw uit met `--overwrite` als het vervangen van bestaande doelen de bedoeling is. Providers kunnen in de migratierapportmap nog steeds back-ups op itemniveau schrijven voor overschreven bestanden.
  </Accordion>
  <Accordion title="Geheimen">
    Bij interactieve toepassing wordt gevraagd of gedetecteerde authenticatiegegevens moeten worden geïmporteerd, waarbij ja standaard is geselecteerd. Gebruik `--no-auth-credentials` om deze over te slaan, of `--include-secrets` met `--yes` voor onbeheerde import van aanmeldgegevens.
  </Accordion>
</AccordionGroup>

## Claude-provider

De meegeleverde Claude-provider detecteert Claude Code-status standaard in `~/.claude`. Gebruik `--from <path>` om een specifieke Claude Code-homemap of projecthoofdmap te importeren.

<Tip>
Zie voor een gebruikersgericht stappenplan [Migreren vanuit Claude](/nl/install/migrating-claude).
</Tip>

### Wat Claude importeert

- Markdown voor het automatische geheugen van Claude Code uit `~/.claude/projects/*/memory` en een
  door de gebruiker geconfigureerde `autoMemoryDirectory`, gekopieerd naar
  `memory/imports/claude-code/` voor geïndexeerd terughalen.
- Project-`CLAUDE.md` en `.claude/CLAUDE.md` naar de OpenClaw-agentwerkruimte (`AGENTS.md`).
- Gebruikers-`~/.claude/CLAUDE.md` toegevoegd aan `USER.md` in de werkruimte.
- MCP-serverdefinities uit project-`.mcp.json`, Claude Code-`~/.claude.json` (inclusief de vermeldingen per project) en Claude Desktop-`claude_desktop_config.json`.
- Claude-skillmappen die `SKILL.md` bevatten (gebruikers-`~/.claude/skills` en project-`.claude/skills`).
- Markdown-bestanden voor Claude-opdrachten (gebruikers-`~/.claude/commands` en project-`.claude/commands`) geconverteerd naar OpenClaw-skills die alleen handmatig kunnen worden aangeroepen.

### Gearchiveerde status en status voor handmatige controle

Claude-hooks, machtigingen, standaardomgevingswaarden, project-`CLAUDE.local.md`, `.claude/rules`, gebruikers- en projectmappen `agents/` en projectgeschiedenis (`projects`, `cache`, `plans` onder `~/.claude`) worden bewaard in het migratierapport of gemeld als items voor handmatige controle. OpenClaw voert hooks niet uit, kopieert geen brede toelatingslijsten en importeert de status van OAuth-/Desktop-aanmeldgegevens niet automatisch.

## Codex-provider

De meegeleverde Codex-provider detecteert Codex CLI-status standaard in `~/.codex`, of in `CODEX_HOME` wanneer die omgevingsvariabele is ingesteld. Gebruik `--from <path>` om een specifieke Codex-homemap te inventariseren.

Gebruik deze provider wanneer je overstapt op de OpenClaw Codex-harness en nuttige persoonlijke Codex CLI-middelen doelbewust wilt overzetten. Lokale starts van de Codex-appserver gebruiken een `CODEX_HOME` per agent, zodat ze je persoonlijke `~/.codex` niet standaard lezen. De normale proces-`HOME` wordt nog steeds overgenomen, zodat Codex gedeelde `$HOME/.agents/*`-skills/pluginmarktplaatsvermeldingen kan zien en subprocessen configuratie en tokens in de gebruikershomemap kunnen vinden.

Als je `openclaw migrate codex` in een interactieve terminal uitvoert, wordt eerst een voorvertoning van het volledige plan weergegeven en worden vervolgens selectievakjeskiezers geopend vóór de laatste toepassingsbevestiging. Eerst wordt om de te kopiëren skillitems gevraagd. Gebruik `Toggle all on` of `Toggle all off` voor bulkselectie. Druk op de spatiebalk om rijen in of uit te schakelen, of op Enter om de gemarkeerde rij te activeren en door te gaan. Geplande skills zijn aanvankelijk aangevinkt, skills met conflicten zijn aanvankelijk niet aangevinkt en `Skip for now` slaat het kopiëren van skills voor deze uitvoering over terwijl de pluginselectie wel doorgaat. Wanneer vanuit de bron geïnstalleerde, beheerde Codex-plugins kunnen worden gemigreerd en `--plugin` niet is opgegeven, wordt vervolgens gevraagd om systeemeigen activering van Codex-plugins op pluginnaam. Pluginitems zijn aanvankelijk aangevinkt, tenzij de doelconfiguratie voor OpenClaw Codex-plugins die plugin al bevat. Bestaande doelplugins zijn aanvankelijk niet aangevinkt en tonen een conflictaanwijzing zoals `conflict: plugin exists`; kies `Toggle all off` om tijdens die uitvoering geen systeemeigen Codex-plugins te migreren, of `Skip for now` om te stoppen vóór toepassing.

Selecteer voor gescripte of exacte uitvoeringen expliciet een of meer skills of plugins:

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### Wat Codex importeert

- Geconsolideerde Codex-`MEMORY.md` en `memory_summary.md` uit
  `$CODEX_HOME/memories`, gekopieerd naar `memory/imports/codex/` voor geïndexeerd
  terughalen. Ruw uitrolgeheugen wordt niet geïmporteerd.
- Codex CLI-skillmappen onder `$CODEX_HOME/skills`, met uitzondering van de `.system`-cache van Codex.
- Persoonlijke AgentSkills onder `$HOME/.agents/skills`, gekopieerd naar de huidige OpenClaw-agentwerkruimte voor eigendom per agent.
- Vanuit de bron geïnstalleerde `openai-curated` Codex-plugins die via `plugin/list` van de Codex-appserver zijn gevonden. De planning leest `plugin/read` voor elke ingeschakelde, geïnstalleerde plugin.

Voor app-gebaseerde pluginmigratie gelden aanvullende voorwaarden:

- Voor app-gebaseerde plugins moet het account van de Codex-appserver van de bron een ChatGPT-abonnementsaccount zijn. Reacties voor niet-ChatGPT-accounts of ontbrekende accounts worden overgeslagen met `codex_subscription_required`.
- Standaard roept de migratie `app/list` van de bron niet aan, zodat app-gebaseerde plugins die aan de accountvoorwaarde voldoen worden gepland zonder verificatie van de toegankelijkheid van de bronapp, en transportfouten bij het opzoeken van het account worden overgeslagen met `codex_account_unavailable`.
- Geef `--verify-plugin-apps` op om een nieuwe `app/list`-momentopname van de bron af te dwingen en te vereisen dat elke app in eigendom aanwezig, ingeschakeld en toegankelijk is voordat systeemeigen activering wordt gepland. In die modus wordt bij transportfouten tijdens het opzoeken van het account alsnog geprobeerd de app-inventaris van de bron te verifiëren. De momentopname wordt alleen voor het huidige proces in het geheugen bewaard; deze wordt nooit naar migratie-uitvoer of doelconfiguratie geschreven.

Uitgeschakelde plugins, onleesbare plugingegevens, bronaccounts waarvoor een abonnement vereist is en (wanneer `--verify-plugin-apps` is ingesteld) ontbrekende, uitgeschakelde of ontoegankelijke apps worden handmatig te controleren, overgeslagen items met getypeerde redenen in plaats van vermeldingen in de doelconfiguratie. De toepassing roept `plugin/install` van de appserver aan voor elke geselecteerde, geschikte plugin, zelfs als de doelappserver al meldt dat die plugin is geïnstalleerd en ingeschakeld. Gemigreerde Codex-plugins zijn alleen bruikbaar in sessies waarin de systeemeigen Codex-harness is geselecteerd; ze worden niet beschikbaar gesteld aan OpenClaw-provideruitvoeringen, ACP-gesprekskoppelingen of andere harnesses.

### Handmatig te controleren Codex-status

Codex `config.toml`, native `hooks/hooks.json`, niet-gecureerde marketplaces, gecachte pluginbundels die geen vanuit de bron geïnstalleerde gecureerde plugins zijn, en vanuit de bron geïnstalleerde plugins die niet door de bronabonnementspoort komen, worden niet automatisch geactiveerd. Wanneer `--verify-plugin-apps` is ingesteld, worden plugins die niet door de app-inventarispoort van de bron komen ook overgeslagen. Al deze items worden gekopieerd naar of gerapporteerd in het migratierapport voor handmatige beoordeling.

Pas voor gemigreerde, vanuit de bron geïnstalleerde gecureerde plugins de volgende schrijfbewerkingen toe:

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- één expliciete pluginvermelding met `marketplaceName: "openai-curated"` en `pluginName` voor elke geselecteerde plugin

Migratie schrijft nooit `plugins["*"]` en slaat nooit lokale cachepaden van marketplaces op.

Overgeslagen plugins worden niet naar de doelconfiguratie geschreven. Abonnementsfouten aan de bronzijde worden bij handmatige items gerapporteerd met getypeerde redenen: `codex_subscription_required`, `codex_account_unavailable`, `plugin_disabled` of `plugin_read_unavailable`. Met `--verify-plugin-apps` kunnen fouten in de app-inventaris van de bron ook worden weergegeven als `app_inaccessible`, `app_disabled`, `app_missing` of `app_inventory_unavailable`. Installaties aan de doelzijde waarvoor authenticatie vereist is, worden bij het betreffende pluginitem gerapporteerd met `status: "skipped"`, `reason: "auth_required"` en opgeschoonde app-identificatoren; hun expliciete configuratievermeldingen worden uitgeschakeld geschreven totdat je ze opnieuw autoriseert en inschakelt. Andere installatiefouten zijn tot het item beperkte `error`-resultaten.

Als de plugininventaris van de Codex-appserver tijdens de planning niet beschikbaar is, valt de migratie terug op adviesitems uit gecachte bundels in plaats van de volledige migratie te laten mislukken.

## Hermes-provider

De gebundelde Hermes-provider volgt `$HERMES_HOME` en het actieve profiel en gebruikt vervolgens de standaard van het platform (`~/.hermes` of `%LOCALAPPDATA%\hermes`). Gebruik `--from <path>` om detectie te overschrijven.

### Wat Hermes importeert

- Standaardmodelconfiguratie uit `config.yaml`.
- Geconfigureerde modelproviders en aangepaste OpenAI-compatibele eindpunten uit `model`, `providers` en `custom_providers`.
- MCP-serverdefinities uit `mcp_servers` of `mcp.servers`. Exacte OpenClaw-toewijzingen omvatten standaardroutering via Streamable HTTP, OAuth-bereik, booleaanse TLS-verificatie, afzonderlijke paden voor het clientcertificaat en de clientsleutel en het native/resource/prompt-toolbeleid van Hermes. Niet-ondersteunde runtime- of referentievelden die uitsluitend voor Hermes gelden, worden gerapporteerd voor handmatige beoordeling.
- `SOUL.md` en `AGENTS.md` naar de OpenClaw-agentwerkruimte.
- `memories/MEMORY.md` en `memories/USER.md` toegevoegd aan werkruimtegeheugenbestanden.
  Oppervlakken die uitsluitend voor geheugen zijn bedoeld (de geheugenspagina voor onboarding en de
  Memory-importpagina van de Control UI) kopiëren deze bestanden in plaats daarvan onder `memory/imports/hermes/` voor
  geïndexeerd terughalen zonder bestaand werkruimtegeheugen te wijzigen.
- Standaardwaarden voor geheugenconfiguratie voor het OpenClaw-bestandsgeheugen, plus archief- of handmatige-beoordelingsitems voor externe geheugenproviders zoals Honcho.
- Skills die ergens onder `skills/` een bestand `SKILL.md` bevatten; geneste Skills worden afgevlakt naar de Skills-map van de werkruimte.
- Configuratiewaarden per Skill uit `skills.config`.
- Huidige Hermes OpenAI Codex OAuth-referenties en OpenCode OpenAI OAuth-referenties wanneer interactieve referentiemigratie wordt geaccepteerd of wanneer `--include-secrets` is ingesteld. Laat Hermes en OpenClaw niet dezelfde geïmporteerde vernieuwingsautorisatie gebruiken.
- Ondersteunde API-sleutels en tokens uit Hermes `.env` en OpenCode `auth.json` wanneer interactieve referentiemigratie wordt geaccepteerd of wanneer `--include-secrets` is ingesteld.

### Ondersteunde `.env`-sleutels

`AI_GATEWAY_API_KEY`, `ALIBABA_API_KEY`, `ANTHROPIC_API_KEY`, `ARCEEAI_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `FIREWORKS_API_KEY`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GLM_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `KIMI_CODING_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODING_API_KEY`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MOONSHOT_API_KEY`, `NVIDIA_API_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_GO_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `TOGETHER_API_KEY`, `VENICE_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `ZAI_API_KEY`, `Z_AI_API_KEY`.

### Alleen-archiefstatus

Hermes-status die OpenClaw niet veilig kan interpreteren, wordt voor handmatige beoordeling naar het migratierapport gekopieerd, maar wordt niet in de actieve OpenClaw-configuratie of -referenties geladen. Dit omvat `plugins/`, `sessions/`, `logs/`, `cron/`, `mcp-tokens/`, `plans/`, `workspace/`, `skins/`, `kanban/`, koppelings-/platformstatus, Gateway-routerings-/processtatus en de gedetecteerde Hermes SQLite-databases.

### Na toepassing

```bash
openclaw doctor
```

## Plugincontract

Migratiebronnen zijn plugins. Een plugin declareert zijn provider-id's in `openclaw.plugin.json`:

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

Tijdens runtime roept de plugin `api.registerMigrationProvider(...)` aan. De provider implementeert `detect`, `plan` en `apply`. Core beheert CLI-orkestratie, back-upbeleid, prompts, JSON-uitvoer en de conflictcontrole vooraf. Core geeft het beoordeelde plan door aan `apply(ctx, plan)`, en providers mogen het plan alleen opnieuw opbouwen wanneer dat argument om compatibiliteitsredenen ontbreekt. Migratie-items kunnen `applyPhase: "after-promotion"` instellen voor externe activeringseffecten die onboarding moet uitstellen totdat gefaseerde lokale gegevens duurzaam zijn gepubliceerd. Die providers moeten `deferredApply: { retrySafe: true }` declareren en elk uitgesteld effect veilig herhaalbaar maken na een onderbroken proces; onboarding weigert niet-gedeclareerde uitgestelde effecten. Een idempotente no-op moet een niet-wijzigend item met `deferredCompletion: true` retourneren, zodat herstel dit als voltooid kan vastleggen. Zelfstandig `openclaw migrate` past het volledige plan nog steeds toe via de normale, door back-ups ondersteunde flow.

Providerplugins kunnen `openclaw/plugin-sdk/migration` gebruiken voor het opbouwen van items en samenvattingstellingen, plus `openclaw/plugin-sdk/migration-runtime` voor conflictbewuste bestandskopieën, alleen-archiefrapportkopieën, gecachte configuratie-runtimewrappers en migratierapporten.

## Onboarding-integratie

Onboarding kan migratie aanbieden wanneer een provider een bekende bron detecteert. Zowel `openclaw onboard --flow import` als `openclaw setup --wizard --import-from hermes` gebruiken dezelfde pluginmigratieprovider en tonen nog steeds een voorbeeld voordat de migratie wordt toegepast. In tegenstelling tot zelfstandige migratie zet het onboardingpad voor een nieuw doel lokale artefacten en geïmporteerde referenties klaar, verifieert of herstelt het geïmporteerde inferentie binnen de faseringsomgeving en promoveert het vervolgens de werkruimte- en agentstatus voordat de configuratie wordt vastgelegd. Met een promotielogboek in modus `0600` kan de volgende uitvoering een onderbroken publicatie voltooien of terugdraaien, inclusief eventuele uitgestelde externe activering, zonder geïmporteerde lokale gegevens opnieuw af te spelen.

<Note>
Voor onboardingimport is een nieuwe OpenClaw-installatie vereist. Stel eerst de configuratie, referenties, sessies en werkruimte opnieuw in als je al lokale status hebt. Import via back-up plus overschrijven of samenvoegen is voor bestaande installaties achter een featuregate geplaatst.
</Note>

## Gerelateerd

- [Migreren vanuit Hermes](/nl/install/migrating-hermes): gebruikersgerichte handleiding.
- [Migreren vanuit Claude](/nl/install/migrating-claude): gebruikersgerichte handleiding.
- [Migreren](/nl/install/migrating): verplaats OpenClaw naar een nieuwe machine.
- [Doctor](/nl/gateway/doctor): statuscontrole na het toepassen van een migratie.
- [Plugins](/nl/tools/plugin): installatie en registratie van plugins.
