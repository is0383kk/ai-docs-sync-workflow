---
read_when:
    - Je stapt over van Hermes en wilt je modelconfiguratie, prompts, geheugen en Skills behouden
    - Je wilt weten wat OpenClaw automatisch importeert en wat alleen in het archief blijft
    - Je hebt een schoon, gescript migratiepad nodig (CI, nieuwe laptop, automatisering)
summary: Stap over van Hermes naar OpenClaw met een vooraf bekeken, omkeerbare import
title: Migreren vanaf Hermes
x-i18n:
    generated_at: "2026-07-27T05:50:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

De meegeleverde Hermes-migratieprovider volgt `HERMES_HOME` en het actieve Hermes-profiel, met een terugval op `~/.hermes` op macOS/Linux of `%LOCALAPPDATA%\hermes` op Windows. Deze toont een voorbeeld van elke wijziging voordat die wordt toegepast en maskeert geheimen in plannen en rapporten. Zelfstandig `openclaw migrate` schrijft een geverifieerde back-up; het nieuwe onboardingpad zet configuratie, referenties en bestanden klaar en publiceert ze pas nadat de geïmporteerde inferentie is geverifieerd. Een expliciet `--from`-pad heeft altijd voorrang.

<Note>
Voor imports is een nieuwe OpenClaw-installatie vereist. Als je al lokale OpenClaw-status hebt, stel dan eerst de configuratie, referenties, sessies en de werkruimte opnieuw in, of gebruik `openclaw migrate apply hermes` rechtstreeks met `--overwrite` nadat je het plan hebt gecontroleerd.
</Note>

## Twee manieren om te importeren

<Tabs>
  <Tab title="Onboardingwizard">
    Detecteert de actieve Hermes-thuismap/het actieve Hermes-profiel en toont een voorbeeld voordat wijzigingen worden toegepast.

    ```bash
    openclaw onboard --flow import
    ```

    Of verwijs naar een specifieke bron:

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    Gebruik `openclaw migrate` voor gescripte of herhaalbare uitvoeringen. Zie [`openclaw migrate`](/nl/cli/migrate) voor de volledige referentie.

    ```bash
    openclaw migrate hermes --dry-run    # alleen voorbeeld
    openclaw migrate apply hermes --yes  # toepassen zonder bevestiging
    ```

    Voeg `--from <path>` toe om de detectie van de Hermes-thuismap/het Hermes-profiel te overschrijven.

  </Tab>
</Tabs>

## Wat wordt geïmporteerd

<AccordionGroup>
  <Accordion title="Modelconfiguratie">
    - Standaardmodelselectie uit Hermes `config.yaml`.
    - Geconfigureerde modelproviders en aangepaste eindpunten uit `model`, `providers` en `custom_providers`, inclusief de huidige Hermes-transports voor Chat Completions, Codex Responses en Anthropic Messages.

  </Accordion>
  <Accordion title="MCP-servers">
    MCP-serverdefinities uit `mcp_servers` of `mcp.servers`, inclusief uitgeschakelde status, time-outs, ondersteuning voor parallelle tools, OAuth-bereik, compatibele TLS-velden en beleid voor native tools, resources en prompts. Voor letterlijke omgevingsvariabelen en headers is toestemming voor het importeren van referenties vereist. Instellingen die alleen in Hermes bestaan voor levenscyclus, sampling, elicitation, preflight, keepalive, CA-bundels, met een wachtwoord beveiligde clientsleutels en vooraf geregistreerde OAuth-clients worden items voor handmatige controle in plaats van ongeldige OpenClaw-configuratie.
  </Accordion>
  <Accordion title="Werkruimtebestanden">
    - `SOUL.md` en `AGENTS.md` worden naar de OpenClaw-agentwerkruimte gekopieerd.
    - `memories/MEMORY.md` en `memories/USER.md` worden **toegevoegd** aan de overeenkomende OpenClaw-geheugenbestanden in plaats van deze te overschrijven.
    - Oppervlakken die uitsluitend voor geheugen zijn bedoeld, gedragen zich anders: de geheugenspagina van onboarding en de importpagina voor Memory in de Control UI kopiëren deze twee bestanden onder `memory/imports/hermes/` voor geïndexeerd terughalen en laten bestaand werkruimtegeheugen onaangetast.

  </Accordion>
  <Accordion title="Geheugenconfiguratie">
    Standaardwaarden voor de geheugenconfiguratie van OpenClaw-bestandsgeheugen. Externe geheugenproviders zoals Honcho worden vastgelegd als archiefitems of items voor handmatige controle, zodat je ze doelbewust kunt verplaatsen.
  </Accordion>
  <Accordion title="Skills">
    Skills met ergens onder `skills/` een `SKILL.md`-bestand worden recursief gedetecteerd, samengevoegd in de map voor Skills van de OpenClaw-werkruimte en samen met hun ondersteunende bestanden gekopieerd. Configuratiewaarden per Skill uit `skills.config` blijven behouden.
  </Accordion>
  <Accordion title="Authenticatiereferenties">
    Interactief `openclaw migrate` vraagt voordat authenticatiereferenties worden geïmporteerd, waarbij ja standaard is geselecteerd. Geaccepteerde imports omvatten huidige Hermes OpenAI Codex OAuth-vermeldingen, OpenCode OpenAI OAuth- en GitHub Copilot-vermeldingen en de [ondersteunde Hermes-`.env`-sleutels](/nl/cli/migrate#supported-env-keys). Gebruik `--include-secrets` voor niet-interactieve import, `--no-auth-credentials` om referenties over te slaan of de vlag `--import-secrets` van onboarding. Laat Hermes en OpenClaw na het importeren van Hermes OAuth niet dezelfde vernieuwingstoekenning gebruiken; authenticeer één kant opnieuw voordat je beide uitvoert.
  </Accordion>
</AccordionGroup>

## Wat alleen in het archief blijft

De provider kopieert het volgende naar de map met migratierapporten voor handmatige controle, maar laadt het **niet** in actieve OpenClaw-configuratie of -referenties:

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`, `workspace/`, `skins/` en `kanban/`
- `pairing/`- en `platforms/`-opslag, plus routerings-/processtatus van de Gateway
- `state.db`, `hermes_state.db`, `projects.db`, `response_store.db`, `memory_store.db`, `verification_evidence.db`, `kanban.db` en `retaindb_queue.db`

OpenClaw weigert deze status automatisch uit te voeren of te vertrouwen, omdat indelingen en vertrouwensaannames tussen systemen kunnen afwijken. Verplaats na controle van het archief handmatig wat je nodig hebt.

## Aanbevolen werkwijze

<Steps>
  <Step title="Bekijk een voorbeeld van het plan">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    Het plan vermeldt alles wat verandert, inclusief conflicten, overgeslagen items en gevoelige items. Geneste sleutels die op geheimen lijken, worden in de uitvoer gemaskeerd.

  </Step>
  <Step title="Pas toe met back-up">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw maakt en verifieert een back-up voordat wijzigingen worden toegepast. Dit niet-interactieve voorbeeld importeert alleen niet-geheime status. Voer het uit zonder `--yes` om de vraag over referenties interactief te beantwoorden, of voeg `--include-secrets` toe om ondersteunde referenties op te nemen in een uitvoering zonder toezicht.

  </Step>
  <Step title="Voer doctor uit">
    ```bash
    openclaw doctor
    ```

    [Doctor](/nl/gateway/doctor) past eventuele openstaande configuratiemigraties opnieuw toe en controleert op problemen die tijdens de import zijn ontstaan.

  </Step>
  <Step title="Start opnieuw en verifieer">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Controleer of de Gateway correct werkt en je geïmporteerde model, geheugen en Skills zijn geladen.

  </Step>
</Steps>

## Conflictafhandeling

Apply weigert door te gaan wanneer het plan conflicten meldt (een bestand of configuratiewaarde bestaat al op het doel).

<Warning>
Voer opnieuw uit met `--overwrite` alleen wanneer het bestaande doel opzettelijk moet worden vervangen. Providers kunnen nog steeds back-ups per item schrijven voor overschreven bestanden in de map met migratierapporten.
</Warning>

Conflicten zijn ongebruikelijk bij een nieuwe installatie. Ze verschijnen meestal wanneer je de import opnieuw uitvoert voor een installatie die al gebruikerswijzigingen bevat.

Als tijdens het toepassen een conflict optreedt (bijvoorbeeld een onverwachte racecondition bij een configuratiebestand), wordt dat item als conflict gemeld terwijl onafhankelijke bestanden, Skills, referenties, archieven en configuratievermeldingen doorgaan. Los het conflicterende item op en voer de import opnieuw uit; identieke geheugenimports zijn idempotent.

## Geheimen

Interactief `openclaw migrate` vraagt of gedetecteerde authenticatiereferenties moeten worden geïmporteerd, waarbij ja standaard is geselecteerd.

- Bij acceptatie worden huidige Hermes OpenAI Codex OAuth-vermeldingen, OpenCode OpenAI OAuth- en GitHub Copilot-vermeldingen en de [ondersteunde `.env`-sleutels](/nl/cli/migrate#supported-env-keys) geïmporteerd.
- Gebruik `--no-auth-credentials`, of antwoord nee op de vraag, om alleen niet-geheime status te importeren.
- Gebruik `--include-secrets` om referenties te importeren in een uitvoering van `--yes` zonder toezicht.
- Gebruik de vlag `--import-secrets` van de onboardingwizard om referenties vanuit de wizard te importeren.

## JSON-uitvoer voor automatisering

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

Met `--json` en zonder `--yes` drukt Apply het plan af en wijzigt het geen status — de veiligste modus voor CI en gedeelde scripts.

## Probleemoplossing

<AccordionGroup>
  <Accordion title="Apply weigert vanwege conflicten">
    Controleer de planuitvoer. Elk conflict vermeldt het bronpad en het bestaande doel. Bepaal per item of je het wilt overslaan, het doel wilt bewerken of opnieuw wilt uitvoeren met `--overwrite`.
  </Accordion>
  <Accordion title="Hermes bevindt zich buiten ~/.hermes">
    Geef `--from /actual/path` (CLI) of `--import-source /actual/path` (onboarding) door.
  </Accordion>
  <Accordion title="Onboarding weigert te importeren in een bestaande installatie">
    Voor onboardingimports is een nieuwe installatie vereist. Stel de status opnieuw in en doorloop onboarding opnieuw, of gebruik `openclaw migrate apply hermes` rechtstreeks, dat `--overwrite` en expliciet back-upbeheer ondersteunt.
  </Accordion>
  <Accordion title="API-sleutels zijn niet geïmporteerd">
    Interactief `openclaw migrate` importeert API-sleutels alleen wanneer je de vraag over referenties accepteert. Niet-interactieve uitvoeringen van `--yes` hebben `--include-secrets` nodig; onboardingimports hebben `--import-secrets` nodig. Alleen de [ondersteunde `.env`-sleutels](/nl/cli/migrate#supported-env-keys) worden herkend — andere `.env`-variabelen worden genegeerd.
  </Accordion>
</AccordionGroup>

## Gerelateerd

- [`openclaw migrate`](/nl/cli/migrate): volledige CLI-referentie, Plugin-contract en JSON-structuren.
- [Onboarding](/nl/cli/onboard): wizardwerkwijze en niet-interactieve vlaggen.
- [Migreren](/nl/install/migrating): verplaats een OpenClaw-installatie tussen machines.
- [Doctor](/nl/gateway/doctor): statuscontrole na migratie.
- [Agentwerkruimte](/nl/concepts/agent-workspace): waar `SOUL.md`, `AGENTS.md` en geheugenbestanden zich bevinden.
