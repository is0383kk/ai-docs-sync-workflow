---
read_when:
    - Je komt van Claude Code of Claude Desktop en wilt instructies, MCP-servers en skills behouden
    - Je moet begrijpen wat OpenClaw automatisch importeert en wat uitsluitend in het archief blijft
summary: Verplaats de lokale status van Claude Code en Claude Desktop naar OpenClaw met een importvoorbeeld
title: Migreren van Claude
x-i18n:
    generated_at: "2026-07-27T05:37:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d5a5e63727e1583fc3fa27ac45215c72df9074b21d7c5f6b33800bec916769b
    source_path: install/migrating-claude.md
    workflow: 16
---

OpenClaw importeert lokale Claude-status via de gebundelde Claude-migratieprovider. De provider toont een voorbeeld van elk item voordat de status wordt gewijzigd en maskeert geheimen in plannen en rapporten. Zelfstandig `openclaw migrate` maakt een geverifieerde back-up; het nieuwe onboardingpad bereidt de import voor en publiceert deze pas nadat de verificatie is geslaagd.

<Note>
Voor onboardingimports is een nieuwe OpenClaw-configuratie vereist. Als je al lokale OpenClaw-status hebt, stel dan eerst de configuratie, referenties, sessies en werkruimte opnieuw in, of gebruik `openclaw migrate` rechtstreeks met `--overwrite` nadat je het plan hebt gecontroleerd.
</Note>

## Twee manieren om te importeren

<Tabs>
  <Tab title="Onboardingwizard">
    De wizard biedt Claude aan wanneer lokale Claude-status wordt gedetecteerd.

    ```bash
    openclaw onboard --flow import
    ```

    Of verwijs naar een specifieke bron:

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

  </Tab>
  <Tab title="CLI">
    Gebruik `openclaw migrate` voor gescripte of herhaalbare uitvoeringen. Zie [`openclaw migrate`](/nl/cli/migrate) voor de volledige referentie.

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    Voeg `--from <path>` toe om een specifieke Claude Code-homemap of projecthoofdmap te importeren.

  </Tab>
</Tabs>

## Wat wordt geïmporteerd

<AccordionGroup>
  <Accordion title="Instructies en geheugen">
    - Inhoud van project-`CLAUDE.md` en `.claude/CLAUDE.md` wordt gekopieerd of toegevoegd aan `AGENTS.md` in de OpenClaw-agentwerkruimte.
    - Inhoud van gebruikers-`~/.claude/CLAUDE.md` wordt toegevoegd aan `USER.md` in de werkruimte.

  </Accordion>
  <Accordion title="MCP-servers">
    MCP-serverdefinities worden, indien aanwezig, geïmporteerd uit project-`.mcp.json`, Claude Code-`~/.claude.json` en Claude Desktop-`claude_desktop_config.json`.
  </Accordion>
  <Accordion title="Skills en opdrachten">
    - Claude-skills met een `SKILL.md`-bestand worden gekopieerd naar de map met skills in de OpenClaw-werkruimte.
    - Markdown-bestanden met Claude-opdrachten onder `.claude/commands/` of `~/.claude/commands/` worden geconverteerd naar OpenClaw-skills met `disable-model-invocation: true`.

  </Accordion>
</AccordionGroup>

## Wat alleen in het archief blijft

De provider kopieert het volgende naar het migratierapport voor handmatige controle, maar laadt het **niet** in de actieve OpenClaw-configuratie:

- Claude-hooks
- Claude-machtigingen en brede lijsten met toegestane tools
- Standaardwaarden voor de Claude-omgeving
- `CLAUDE.local.md`
- `.claude/rules/`
- Claude-subagents onder `.claude/agents/` of `~/.claude/agents/`
- Cache-, plan- en projectgeschiedenismappen van Claude Code
- Claude Desktop-extensies en in het besturingssysteem opgeslagen referenties

OpenClaw weigert hooks uit te voeren, lijsten met vertrouwde machtigingen te vertrouwen of ondoorzichtige OAuth- en Desktop-referentiestatus automatisch te decoderen. Verplaats wat je nodig hebt handmatig nadat je het archief hebt gecontroleerd.

## Bronselectie

Zonder `--from` inspecteert OpenClaw de standaardhomemap van Claude Code op `~/.claude`, het bemonsterde Claude Code-statusbestand `~/.claude.json` en de MCP-configuratie van Claude Desktop op macOS.

Wanneer `--from` naar een projecthoofdmap verwijst, importeert OpenClaw alleen de Claude-bestanden van dat project, zoals `CLAUDE.md`, `.claude/settings.json`, `.claude/commands/`, `.claude/skills/` en `.mcp.json`. Tijdens een import vanuit een projecthoofdmap wordt je globale Claude-homemap niet gelezen.

## Aanbevolen werkwijze

<Steps>
  <Step title="Bekijk een voorbeeld van het plan">
    ```bash
    openclaw migrate claude --dry-run
    ```

    Het plan vermeldt alles wat wordt gewijzigd, waaronder conflicten, overgeslagen items en gevoelige waarden die zijn gemaskeerd in geneste MCP-velden `env` of `headers`.

  </Step>
  <Step title="Pas toe met back-up">
    ```bash
    openclaw migrate apply claude --yes
    ```

    OpenClaw maakt en verifieert een back-up voordat de wijzigingen worden toegepast.

  </Step>
  <Step title="Voer Doctor uit">
    ```bash
    openclaw doctor
    ```

    [Doctor](/nl/gateway/doctor) controleert na de import op problemen met de configuratie of status.

  </Step>
  <Step title="Start opnieuw en verifieer">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Controleer of de Gateway in orde is en je geïmporteerde instructies, MCP-servers en skills zijn geladen.

  </Step>
</Steps>

## Conflictafhandeling

Het toepassen wordt geweigerd wanneer het plan conflicten meldt (er bestaat al een bestand of configuratiewaarde op het doel).

<Warning>
Voer de opdracht alleen opnieuw uit met `--overwrite` wanneer je het bestaande doel bewust wilt vervangen. Providers kunnen nog steeds back-ups per item maken voor overschreven bestanden in de map met migratierapporten.
</Warning>

Bij een nieuwe OpenClaw-installatie zijn conflicten ongebruikelijk. Ze treden doorgaans op wanneer je de import opnieuw uitvoert op een configuratie die al gebruikerswijzigingen bevat.

## JSON-uitvoer voor automatisering

```bash
openclaw migrate claude --dry-run --json
openclaw migrate apply claude --json --yes
```

`--yes` is vereist voor `migrate apply` buiten een interactieve terminal; zonder deze optie geeft OpenClaw een foutmelding in plaats van de wijzigingen toe te passen. Scripts en CI moeten daarom `--yes` expliciet doorgeven. Bekijk eerst een voorbeeld met `--dry-run --json` en pas de wijzigingen vervolgens toe met `--json --yes` zodra het plan er correct uitziet.

## Probleemoplossing

<AccordionGroup>
  <Accordion title="Claude-status bevindt zich buiten ~/.claude">
    Geef `--from /actual/path` (CLI) of `--import-source /actual/path` (onboarding) door.
  </Accordion>
  <Accordion title="Onboarding weigert te importeren in een bestaande configuratie">
    Voor onboardingimports is een nieuwe configuratie vereist. Stel de status opnieuw in en voer de onboarding opnieuw uit, of gebruik `openclaw migrate apply claude` rechtstreeks, dat `--overwrite` en expliciet back-upbeheer ondersteunt.
  </Accordion>
  <Accordion title="MCP-servers van Claude Desktop zijn niet geïmporteerd">
    Claude Desktop leest `claude_desktop_config.json` vanaf een platformspecifiek pad. Laat `--from` naar de map van dat bestand verwijzen als OpenClaw deze niet automatisch heeft gedetecteerd.
  </Accordion>
  <Accordion title="Claude-opdrachten zijn skills geworden waarvoor modelaanroepen zijn uitgeschakeld">
    Dit is zo ontworpen. Claude-opdrachten worden door gebruikers geactiveerd, dus OpenClaw importeert ze als skills met `disable-model-invocation: true`. Bewerk de frontmatter van elke skill als je wilt dat de agent ze automatisch aanroept.
  </Accordion>
</AccordionGroup>

## Gerelateerd

- [`openclaw migrate`](/nl/cli/migrate): volledige CLI-referentie, Plugin-contract en JSON-structuren.
- [Migratiehandleiding](/nl/install/migrating): alle migratiepaden.
- [Migreren vanuit Hermes](/nl/install/migrating-hermes): het andere importschema tussen systemen.
- [Onboarding](/nl/cli/onboard): wizardproces en niet-interactieve vlaggen.
- [Doctor](/nl/gateway/doctor): statuscontrole na de migratie.
- [Agentwerkruimte](/nl/concepts/agent-workspace): waar `AGENTS.md`, `USER.md` en skills zich bevinden.
