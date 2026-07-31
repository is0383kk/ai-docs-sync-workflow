---
read_when:
    - Je wilt code_execution inschakelen of configureren
    - Je wilt analyse op afstand zonder toegang tot de lokale shell
    - Je wilt x_search of web_search combineren met Python-analyse op afstand
summary: 'code_execution: voer Python-analyse op afstand uit in een sandbox met xAI'
title: Code-uitvoering
x-i18n:
    generated_at: "2026-07-27T05:17:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution` voert externe Python-analyse in een sandbox uit via xAI's Responses API
(`https://api.x.ai/v1/responses`, hetzelfde eindpunt dat `x_search` gebruikt). Het wordt
door de meegeleverde `xai`-plugin geregistreerd onder het `tools`-contract.

<Warning>
  `code_execution` wordt uitgevoerd op de servers van xAI. xAI brengt $5 per 1.000 toolaanroepen
  in rekening, plus de invoer- en uitvoertokens van het model.
</Warning>

| Eigenschap         | Waarde                                                                            |
| ------------------ | --------------------------------------------------------------------------------- |
| Toolnaam           | `code_execution`                                                                  |
| Providerplugin     | `xai` (meegeleverd, `enabledByDefault: true`)                                    |
| Authenticatie      | xAI-authenticatieprofiel, `XAI_API_KEY` of `plugins.entries.xai.config.webSearch.apiKey` |
| Standaardmodel     | `grok-4.3`                                                                        |
| Standaardtime-out  | 30 seconden                                                                       |
| Standaardwaarde voor `maxTurns` | niet ingesteld (xAI past zijn eigen interne limiet toe)                           |

Gebruik het voor berekeningen, tabellen, snelle statistieken en analyses in
grafiekvorm, waaronder van gegevens die door `x_search` of `web_search` worden
geretourneerd. Het heeft geen toegang tot lokale bestanden, je shell, je repository
of gekoppelde apparaten en bewaart geen status tussen aanroepen. Beschouw elke
aanroep daarom als tijdelijke analyse en niet als een notebooksessie. Voer voor
actuele X-gegevens eerst [`x_search`](/nl/tools/web#x_search) uit en leid het resultaat door.

Gebruik voor lokale uitvoering in plaats daarvan [`exec`](/nl/tools/exec).

## Instellen

<Steps>
  <Step title="Geef xAI-inloggegevens op">
    OAuth vereist een geschikt SuperGrok- of X Premium-abonnement
    (verificatie met apparaatcode, zodat dit vanaf externe hosts werkt zonder
    callback naar localhost):

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Tijdens een nieuwe installatie is dezelfde keuze beschikbaar in de onboarding:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    Of een API-sleutel:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    Of via de configuratie:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    Elk van deze drie voorziet ook `x_search` en Grok `web_search` van toegang.

  </Step>

  <Step title="Schakel code_execution in en stel het af">
    Als `enabled` is weggelaten, wordt `code_execution` alleen beschikbaar gesteld
    wanneer de provider van het actieve model `xai` is en de xAI-inloggegevens
    kunnen worden gevonden. Stel voor een actief model met een bekende niet-xAI-provider
    `plugins.entries.xai.config.codeExecution.enabled` in op `true` om gebruik tussen providers
    in te schakelen. Als de provider van het actieve model ontbreekt of niet kan worden
    bepaald, blijft de tool verborgen. Stel `enabled` in op `false` om de
    tool voor elke provider uit te schakelen. xAI-inloggegevens zijn altijd vereist.

    Gebruik hetzelfde blok om het model, de limiet voor beurten of de time-out te overschrijven:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // vereist voor een bekende niet-xAI-modelprovider
                model: "grok-4.3", // overschrijft het standaardmodel van xAI voor code-uitvoering
                maxTurns: 2,            // optionele limiet voor interne toolbeurten
                timeoutSeconds: 30,     // time-out van de aanvraag (standaard: 30)
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Start de Gateway opnieuw">
    ```bash
    openclaw gateway restart
    ```

    `code_execution` verschijnt in de toollijst van de agent zodra de xAI-plugin
    zich opnieuw registreert en de bovenstaande controles voor provider, inschakeling
    en authenticatie slagen.

  </Step>
</Steps>

## Gebruik

Maak het doel van de analyse expliciet. De tool gebruikt één parameter,
`task`, dus verstuur het volledige verzoek en eventuele inlinegegevens in één prompt:

```text
Gebruik code_execution om het voortschrijdende gemiddelde over 7 dagen voor deze getallen te berekenen: ...
```

```text
Gebruik x_search om berichten te vinden waarin OpenClaw deze week wordt genoemd en gebruik vervolgens code_execution om ze per dag te tellen.
```

```text
Gebruik web_search om de nieuwste cijfers van AI-benchmarks te verzamelen en gebruik vervolgens code_execution om procentuele veranderingen te vergelijken.
```

## Fouten

Zonder authenticatie retourneert de tool een gestructureerde JSON-fout (geen
gegenereerde uitzondering), zodat de agent zichzelf kan corrigeren:

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution heeft xAI-inloggegevens nodig. Voer `openclaw onboard --auth-choice xai-oauth` uit om je aan te melden met Grok, voer `openclaw onboard --auth-choice xai-api-key` uit, stel `XAI_API_KEY` in de Gateway-omgeving in of configureer `plugins.entries.xai.config.webSearch.apiKey`.",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Exec-tool" href="/nl/tools/exec" icon="terminal">
    Lokale shelluitvoering op je machine of gekoppelde Node.
  </Card>
  <Card title="Exec-goedkeuringen" href="/nl/tools/exec-approvals" icon="shield">
    Toestaan/weigeren-beleid voor shelluitvoering.
  </Card>
  <Card title="Webtools" href="/nl/tools/web" icon="globe">
    `web_search`, `x_search` en `web_fetch`.
  </Card>
  <Card title="xAI-provider" href="/nl/providers/xai" icon="microchip">
    Grok-modellen, zoeken op internet/X en configuratie voor code-uitvoering.
  </Card>
</CardGroup>
