---
read_when:
    - Je wilt de configuratie niet-interactief lezen of bewerken
sidebarTitle: Config
summary: CLI-referentie voor `openclaw config` (get/set/patch/unset/file/schema/validate)
title: Configuratie
x-i18n:
    generated_at: "2026-07-27T05:40:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c4f8edb19737070e421c9107f7da8886e5617d9a043d8647666505c7ac9638d
    source_path: cli/config.md
    workflow: 16
---

Niet-interactieve helpers voor `openclaw.json`: een waarde per pad ophalen/instellen/patchen/verwijderen, het schema weergeven, valideren of het actieve bestandspad weergeven. Voer `openclaw config` zonder subopdracht uit om dezelfde begeleide wizard te openen als `openclaw configure`.

<Note>
Wanneer `OPENCLAW_NIX_MODE=1`, behandelt OpenClaw `openclaw.json` als onveranderlijk. Alleen-lezenopdrachten (`config get`, `config file`, `config schema`, `config validate`) werken nog steeds; configuratieschrijvers weigeren. Bewerk in plaats daarvan de Nix-bron voor de installatie; gebruik voor de eigen nix-openclaw-distributie de [snelstart voor nix-openclaw](https://github.com/openclaw/nix-openclaw#quick-start) en stel waarden in onder `programs.openclaw.config` of `instances.<name>.config`.
</Note>

## Hoofdopties

<ParamField path="--section <section>" type="string">
  Herhaalbaar sectiefilter voor begeleide configuratie wanneer je `openclaw config` zonder subopdracht uitvoert.
</ParamField>

Begeleide secties: `workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`.

## Voorbeelden

```bash
openclaw config file
openclaw config --section model
openclaw config --section gateway --section daemon
openclaw config schema
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set secrets.providers.vaultfile --provider-source file --provider-path /etc/openclaw/secrets.json --provider-mode json
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config validate
openclaw config validate --json
```

### Paden

Punt- of haakjesnotatie. Zet haakjespaden tussen aanhalingstekens in shellvoorbeelden, zodat zsh `[0]` niet als glob uitbreidt:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.entries.main
openclaw config get agents.entries
openclaw config set 'agents.entries.work.tools.exec.node' "node-id-or-name"
```

### `config get`

Leest een waarde uit de geredigeerde configuratiesnapshot (geheimen worden nooit weergegeven). `--json` geeft de onbewerkte waarde als JSON weer; anders worden tekenreeksen/getallen/booleans zonder opmaak weergegeven en objecten/arrays als opgemaakte JSON.

Wanneer het pad ontbreekt, schrijft `--json` `{ "error": "Config path not found: <path>" }` naar stdout en wordt afgesloten met status 1. Zonder `--json` blijft de diagnose op stderr.

```bash
openclaw config get browser.executablePath
openclaw config get agents.defaults.model --json
```

### `config file`

Geeft het actieve configuratiebestandspad weer, herleid uit `OPENCLAW_CONFIG_PATH` of de standaardlocatie. Het pad verwijst naar een regulier bestand, niet naar een symbolische koppeling; zie [Schrijfveiligheid](#write-safety).

### `config schema`

Geeft het gegenereerde JSON-schema voor `openclaw.json` weer op stdout.

<AccordionGroup>
  <Accordion title="Wat het bevat">
    - Het huidige hoofdconfiguratieschema, plus een `$schema`-tekenreeksveld op hoofdniveau voor editorhulpmiddelen.
    - Documentatiemetadata van velden `title` / `description` die door de Control UI wordt gebruikt.
    - Geneste object-, jokerteken- (`*`) en array-itemknooppunten (`[]`) nemen dezelfde `title`- / `description`-metadata over wanneer bijpassende velddocumentatie bestaat.
    - `anyOf`- / `oneOf`- / `allOf`-vertakkingen nemen ook dezelfde documentatiemetadata over.
    - Naar beste vermogen actuele schema-metadata van plugins en kanalen wanneer runtimemanifesten kunnen worden geladen.
    - Een schoon terugvalschema, zelfs wanneer de huidige configuratie ongeldig is.

  </Accordion>
  <Accordion title="Gerelateerde runtime-RPC">
    `config.schema.lookup` retourneert één genormaliseerd configuratiepad met een oppervlakkig schemaknooppunt (`title`, `description`, `type`, `enum`, `const`, algemene grenzen), overeenkomende metadata voor UI-hints en samenvattingen van directe onderliggende elementen. Gebruik dit voor padgerichte verdieping in de Control UI of aangepaste clients.
  </Accordion>
</AccordionGroup>

```bash
openclaw config schema
openclaw config schema > openclaw.schema.json
```

### `config validate`

Valideert de huidige configuratie aan de hand van het actieve schema zonder de Gateway te starten.

```bash
openclaw config validate
openclaw config validate --json
```

<Note>
Als de validatie al mislukt, begin dan met `openclaw configure` of `openclaw doctor --fix`. `openclaw chat` omzeilt de blokkering voor ongeldige configuratie niet.
</Note>

## Waarden

Waarden worden waar mogelijk als JSON5 geparseerd; anders worden ze als onbewerkte tekenreeksen behandeld. Gebruik `--strict-json` om standaard-JSON zonder terugval naar een tekenreeks te vereisen (alleen-JSON5-syntaxis zoals opmerkingen, afsluitende komma's of sleutels zonder aanhalingstekens wordt dan geweigerd). `--json` is een verouderde alias voor `--strict-json` op `config set`.

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

`config get <path> --json` geeft de onbewerkte waarde als JSON weer in plaats van als voor de terminal opgemaakte tekst.

Wanneer een schrijfbewerking `agents.defaults.model` of een `agents.entries.*.model` per agent wijzigt, herleidt OpenClaw vóór het schrijven elke gewijzigde primaire optie of terugvaloptie via de geconfigureerde providercatalogi. Onbekende modelverwijzingen worden geweigerd zonder de actieve configuratie te wijzigen; voer `openclaw models list` uit om beschikbare modellen te bekijken.

<Note>
Objecttoewijzing vervangt standaard het doelpad. Beveiligde paden die vaak door gebruikers toegevoegde vermeldingen bevatten, weigeren vervangingen waardoor bestaande vermeldingen zouden worden verwijderd, tenzij je `--replace` doorgeeft: `agents.defaults.models`, `agents.entries`, `models.providers`, `models.providers.<id>`, `models.providers.<id>.models`, `plugins.entries` en `auth.profiles`.
</Note>

Gebruik `--merge` wanneer je vermeldingen aan die toewijzingen toevoegt:

```bash
openclaw config set agents.defaults.models '{"openai/gpt-5.4":{}}' --strict-json --merge
openclaw config set models.providers.ollama.models '[{"id":"llama3.2","name":"Llama 3.2"}]' --strict-json --merge
```

Gebruik `--replace` alleen wanneer de opgegeven waarde opzettelijk de volledige doelwaarde moet worden.

## `config set`-modi

<Tabs>
  <Tab title="Waardemodus">
    ```bash
    openclaw config set <path> <value>
    ```
  </Tab>
  <Tab title="SecretRef-opbouwmodus">
    ```bash
    openclaw config set channels.discord.token \
      --ref-provider default \
      --ref-source env \
      --ref-id DISCORD_BOT_TOKEN
    ```
  </Tab>
  <Tab title="Provideropbouwmodus">
    Alleen voor `secrets.providers.<alias>`-paden:

    ```bash
    openclaw config set secrets.providers.vault \
      --provider-source exec \
      --provider-command /usr/local/bin/openclaw-vault \
      --provider-arg read \
      --provider-arg openai/api-key \
      --provider-timeout-ms 5000
    ```

  </Tab>
  <Tab title="Batchmodus">
    ```bash
    openclaw config set --batch-json '[
      {
        "path": "secrets.providers.default",
        "provider": { "source": "env" }
      },
      {
        "path": "channels.discord.token",
        "ref": { "source": "env", "provider": "default", "id": "DISCORD_BOT_TOKEN" }
      }
    ]'
    ```

    ```bash
    openclaw config set --batch-file ./config-set.batch.json --dry-run
    ```

    Batchbestanden zijn beperkt tot 8 MiB.

  </Tab>
</Tabs>

<Warning>
SecretRef-toewijzingen worden geweigerd op niet-ondersteunde, tijdens runtime wijzigbare oppervlakken (bijvoorbeeld `hooks.token`, `commands.ownerDisplaySecret`, webhooktokens voor Discord-threadbinding en WhatsApp-referentiegegevens in JSON). Zie [SecretRef-referentiegegevensoppervlak](/nl/reference/secretref-credential-surface).
</Warning>

Bij batchparsing wordt altijd de batchpayload (`--batch-json`/`--batch-file`) als bron van waarheid gebruikt; `--strict-json` / `--json` wijzigen het batchparsegedrag niet.

De JSON-pad/waardemodus werkt ook rechtstreeks voor SecretRefs en providers:

```bash
openclaw config set channels.discord.token \
  '{"source":"env","provider":"default","id":"DISCORD_BOT_TOKEN"}' \
  --strict-json

openclaw config set secrets.providers.vaultfile \
  '{"source":"file","path":"/etc/openclaw/secrets.json","mode":"json"}' \
  --strict-json
```

### Vlaggen voor provideropbouw

Doelen voor provideropbouw moeten `secrets.providers.<alias>` als pad gebruiken.

<AccordionGroup>
  <Accordion title="Algemene vlaggen">
    - `--provider-source <env|file|exec>`
    - `--provider-timeout-ms <ms>` (`file`, `exec`)

  </Accordion>
  <Accordion title="Omgevingsprovider (--provider-source env)">
    - `--provider-allowlist <ENV_VAR>` (herhaalbaar)

  </Accordion>
  <Accordion title="Bestandsprovider (--provider-source file)">
    - `--provider-path <path>` (vereist)
    - `--provider-mode <singleValue|json>`
    - `--provider-max-bytes <bytes>`
    - `--provider-allow-insecure-path`

  </Accordion>
  <Accordion title="Uitvoerprovider (--provider-source exec)">
    - `--provider-command <path>` (vereist)
    - `--provider-arg <arg>` (herhaalbaar)
    - `--provider-no-output-timeout-ms <ms>`
    - `--provider-max-output-bytes <bytes>`
    - `--provider-json-only`
    - `--provider-env <KEY=VALUE>` (herhaalbaar)
    - `--provider-pass-env <ENV_VAR>` (herhaalbaar)
    - `--provider-trusted-dir <path>` (herhaalbaar)
    - `--provider-allow-insecure-path`
    - `--provider-allow-symlink-command`

  </Accordion>
</AccordionGroup>

Voorbeeld van een geharde uitvoerprovider:

```bash
openclaw config set secrets.providers.vault \
  --provider-source exec \
  --provider-command /usr/local/bin/openclaw-vault \
  --provider-arg read \
  --provider-arg openai/api-key \
  --provider-json-only \
  --provider-pass-env VAULT_TOKEN \
  --provider-trusted-dir /usr/local/bin \
  --provider-timeout-ms 5000
```

## `config patch`

Plak of pipe een configuratievormige JSON5-patch in plaats van veel padgebaseerde `config set`-opdrachten uit te voeren. Objecten worden recursief samengevoegd; arrays en scalaire waarden vervangen het doel; `null` verwijdert het doelpad.

```bash
openclaw config patch --file ./openclaw.patch.json5 --dry-run
openclaw config patch --file ./openclaw.patch.json5
```

Patchbestanden zijn beperkt tot 8 MiB. Gepipete `--stdin`-patches zijn beperkt tot 1 MiB.

Pipe voor externe configuratiescripts een patch via stdin:

```bash
ssh user@gateway-host 'openclaw config patch --stdin --dry-run' < ./openclaw.patch.json5
ssh user@gateway-host 'openclaw config patch --stdin' < ./openclaw.patch.json5
```

Voorbeeldpatch:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

Gebruik `--replace-path <path>` wanneer één object of array exact de opgegeven waarde moet worden in plaats van recursief te worden gepatcht:

```bash
openclaw config patch --file ./discord.patch.json5 --replace-path 'channels.discord.guilds["123"].channels'
```

`--dry-run` voert controles op het schema en de oplosbaarheid van SecretRefs uit zonder te schrijven. Door exec aangestuurde SecretRefs worden tijdens een dry-run standaard overgeslagen; voeg `--allow-exec` toe wanneer je de dry-run bewust provideropdrachten wilt laten uitvoeren.

## Dry-run

`--dry-run` valideert wijzigingen zonder `openclaw.json` te schrijven. Beschikbaar voor `config set`, `config patch` en `config unset`.

```bash
openclaw config set channels.discord.token \
  --ref-provider default \
  --ref-source env \
  --ref-id DISCORD_BOT_TOKEN \
  --dry-run \
  --json

openclaw config set channels.discord.token \
  --ref-provider vault \
  --ref-source exec \
  --ref-id discord/token \
  --dry-run \
  --allow-exec
```

<AccordionGroup>
  <Accordion title="Gedrag van de dry-run">
    - Builder-modus: voert controles op de oplosbaarheid van SecretRefs uit voor gewijzigde refs/providers.
    - JSON-modus (`--strict-json`, `--json` of batchmodus): voert schemavalidatie en controles op de oplosbaarheid van SecretRefs uit.
    - Beleidsvalidatie wordt uitgevoerd op de volledige configuratie na de wijziging, zodat schrijfbewerkingen van bovenliggende objecten (bijvoorbeeld `hooks` als object instellen) de validatie van niet-ondersteunde oppervlakken niet kunnen omzeilen.
    - Controles van exec-SecretRefs worden standaard overgeslagen om neveneffecten van opdrachten te voorkomen; geef `--allow-exec` door om dit in te schakelen (hierdoor kunnen provideropdrachten worden uitgevoerd). `--allow-exec` is alleen voor dry-runs en geeft een fout zonder `--dry-run`.

  </Accordion>
  <Accordion title="Velden van --dry-run --json">
    - `ok`: of de dry-run is geslaagd
    - `operations`: aantal geëvalueerde toewijzingen
    - `checks`: of controles op schema/oplosbaarheid zijn uitgevoerd
    - `checks.resolvabilityComplete`: of de oplosbaarheidscontroles volledig zijn uitgevoerd (onwaar wanneer exec-refs worden overgeslagen)
    - `refsChecked`: aantal refs dat tijdens de dry-run daadwerkelijk is opgelost
    - `skippedExecRefs`: aantal exec-refs dat is overgeslagen omdat `--allow-exec` niet was ingesteld
    - `errors`: gestructureerde fouten voor ontbrekende paden, schema's of oplosbaarheid wanneer `ok=false`

  </Accordion>
</AccordionGroup>

### Structuur van de JSON-uitvoer

```json5
{
  ok: boolean,
  operations: number,
  configPath: string,
  inputModes: ["value" | "json" | "builder" | "unset", ...],
  checks: {
    schema: boolean,
    resolvability: boolean,
    resolvabilityComplete: boolean,
  },
  refsChecked: number,
  skippedExecRefs: number,
  errors?: [
    {
      kind: "missing-path" | "schema" | "resolvability" | "model",
      message: string,
      ref?: string, // aanwezig voor oplosbaarheidsfouten
    },
  ],
}
```

<Tabs>
  <Tab title="Voorbeeld van succes">
    ```json
    {
      "ok": true,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0
    }
    ```
  </Tab>
  <Tab title="Voorbeeld van een fout">
    ```json
    {
      "ok": false,
      "operations": 1,
      "configPath": "~/.openclaw/openclaw.json",
      "inputModes": ["builder"],
      "checks": {
        "schema": false,
        "resolvability": true,
        "resolvabilityComplete": true
      },
      "refsChecked": 1,
      "skippedExecRefs": 0,
      "errors": [
        {
          "kind": "resolvability",
          "message": "Fout: Omgevingsvariabele \"MISSING_TEST_SECRET\" is niet ingesteld.",
          "ref": "env:default:MISSING_TEST_SECRET"
        }
      ]
    }
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Als de dry-run mislukt">
    - `config schema validation failed`: de structuur van je configuratie na de wijziging is ongeldig; herstel het pad/de waarde of de structuur van het provider-/ref-object.
    - `Config policy validation failed: unsupported SecretRef usage`: zet die referentie terug naar invoer als platte tekst/tekenreeks; gebruik SecretRefs uitsluitend op ondersteunde oppervlakken.
    - `SecretRef assignment(s) could not be resolved`: de provider/ref waarnaar wordt verwezen, kan momenteel niet worden opgelost (ontbrekende omgevingsvariabele, ongeldige bestandsverwijzing, fout van exec-provider of een niet-overeenkomende provider/bron).
    - `model reference validation failed`: een gewijzigd primair tekstmodel of terugvalmodel is onbekend; voer `openclaw models list` uit en kies een beschikbaar model.
    - `Dry run note: skipped <n> exec SecretRef resolvability check(s)`: voer de opdracht opnieuw uit met `--allow-exec` als je de oplosbaarheid van exec wilt valideren.
    - Herstel in de batchmodus de mislukte vermeldingen en voer `--dry-run` opnieuw uit voordat je schrijft.

  </Accordion>
</AccordionGroup>

## Wijzigingen toepassen

Na elke geslaagde `config set` / `config patch` / `config unset` drukt de CLI een van drie aanwijzingen af, zodat je weet of de Gateway opnieuw moet worden gestart:

| Aanwijzing                                          | Betekenis                                      |
| --------------------------------------------------- | ---------------------------------------------- |
| `Restart the gateway to apply.`                                  | Het gewijzigde pad vereist een volledige herstart. |
| `Change will apply without restarting the gateway.`                                  | Hot reload neemt de wijziging automatisch over. |
| `No gateway restart needed.`                                  | Er is niets gewijzigd dat relevant is voor de runtime. |

Schrijfbewerkingen naar `plugins.entries` (of een onderliggend pad) vereisen altijd een herstart, omdat de CLI niet kan bewijzen dat de herlaadmetadata van elke Plugin is geladen.

## Veilig schrijven

`openclaw config set` en andere configuratieschrijvers van OpenClaw valideren de volledige configuratie na de wijziging voordat deze naar schijf wordt geschreven. Als de nieuwe inhoud niet door de schemavalidatie komt of op destructief overschrijven lijkt, blijft de actieve configuratie ongewijzigd en wordt de geweigerde inhoud ernaast opgeslagen als `openclaw.json.rejected.*`.

Schrijfbewerkingen van OpenClaw serialiseren JSON5 opnieuw als standaard-JSON. Wanneer de bron opmerkingen bevat, waarschuwt de schrijver direct voordat deze worden verwijderd; gebruik een teksteditor als het behouden van opmerkingen belangrijk is.

<Warning>
Het actieve configuratiepad moet een normaal bestand zijn. Indelingen met een symbolische koppeling naar `openclaw.json` worden niet ondersteund voor schrijfbewerkingen; gebruik in plaats daarvan `OPENCLAW_CONFIG_PATH` om rechtstreeks naar het werkelijke bestand te verwijzen.
</Warning>

Geef voor kleine bewerkingen de voorkeur aan schrijfbewerkingen via de CLI:

```bash
openclaw config set gateway.reload.mode hybrid --dry-run
openclaw config set gateway.reload.mode hybrid
openclaw config validate
```

Als een schrijfbewerking wordt geweigerd, inspecteer je de opgeslagen inhoud en herstel je de volledige configuratiestructuur:

```bash
CONFIG="$(openclaw config file)"
ls -lt "$CONFIG".rejected.* 2>/dev/null | head
openclaw config validate
```

Rechtstreeks schrijven met een teksteditor is nog steeds toegestaan, maar de actieve Gateway behandelt die wijzigingen als onvertrouwd totdat ze zijn gevalideerd. Ongeldige rechtstreekse bewerkingen verhinderen het opstarten of worden bij hot reload overgeslagen; Gateway herschrijft `openclaw.json` niet. Voer `openclaw doctor --fix` uit om configuraties met een voorvoegsel of overschreven configuraties te herstellen, of om de laatst bekende geldige kopie terug te zetten. Zie [Problemen met Gateway oplossen](/nl/gateway/troubleshooting#gateway-rejected-invalid-config).

Herstel van het volledige bestand is voorbehouden aan reparatie door doctor. Wijzigingen in het schema van een Plugin of afwijkingen in `minHostVersion` blijven duidelijk zichtbaar in plaats van niet-gerelateerde gebruikersinstellingen terug te draaien, zoals de configuratie van modellen, providers, authenticatieprofielen, kanalen, Gateway-blootstelling, tools, geheugen, browser of Cron.

## Reparatielus

Nadat `openclaw config validate` is geslaagd, gebruik je de lokale TUI om een ingebouwde agent de actieve configuratie met de documentatie te laten vergelijken, terwijl je elke wijziging vanuit dezelfde terminal valideert:

```bash
openclaw chat
```

Binnen de TUI voert een voorafgaande `!` een letterlijke lokale shellopdracht uit (na een eenmalige bevestigingsvraag per sessie):

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

<Steps>
  <Step title="Vergelijken met de documentatie">
    Vraag de agent om je huidige configuratie met de relevante documentatiepagina te vergelijken en de kleinst mogelijke oplossing voor te stellen.
  </Step>
  <Step title="Gerichte bewerkingen toepassen">
    Pas gerichte bewerkingen toe met `openclaw config set` of `openclaw configure`.
  </Step>
  <Step title="Opnieuw valideren">
    Voer `openclaw config validate` na elke wijziging opnieuw uit.
  </Step>
  <Step title="Doctor gebruiken voor runtimeproblemen">
    Als de validatie slaagt maar de runtime nog steeds niet goed werkt, voer je `openclaw doctor` of `openclaw doctor --fix` uit voor hulp bij migratie en reparatie.
  </Step>
</Steps>

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Configuratie](/nl/gateway/configuration)
