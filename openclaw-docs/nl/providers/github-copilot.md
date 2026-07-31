---
read_when:
    - Je wilt GitHub Copilot als modelprovider gebruiken
    - Je hebt de `openclaw models auth login-github-copilot`-flow nodig
    - Je kiest tussen de ingebouwde Copilot-provider, de Copilot SDK-harness en Copilot Proxy
summary: Meld je vanuit OpenClaw aan bij GitHub Copilot via de apparaatflow of niet-interactieve tokenimport
title: GitHub Copilot
x-i18n:
    generated_at: "2026-07-27T05:46:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e839e6c72e7e7cb106a2f98c62c4994b4f3d6f34a2e76b549f2f6ccfdac91fe6
    source_path: providers/github-copilot.md
    workflow: 16
---

GitHub Copilot is de AI-codeerassistent van GitHub. Deze biedt toegang tot Copilot-
modellen voor je GitHub-account en -abonnement. OpenClaw kan Copilot op drie verschillende
manieren gebruiken als modelprovider of agentruntime.

## Drie manieren om Copilot in OpenClaw te gebruiken

<Tabs>
  <Tab title="Ingebouwde provider (github-copilot)">
    Gebruik de ingebouwde aanmeldingsflow voor apparaten om een GitHub-token te verkrijgen en wissel dit vervolgens in voor
    Copilot-API-tokens wanneer OpenClaw wordt uitgevoerd. Dit is het **standaardpad** en het eenvoudigste pad,
    omdat VS Code niet vereist is.

    <Steps>
      <Step title="Voer de aanmeldingsopdracht uit">
        ```bash
        openclaw models auth login-github-copilot
        ```

        Je wordt gevraagd een URL te bezoeken en een eenmalige code in te voeren. Houd de
        terminal open totdat het proces is voltooid.
      </Step>
      <Step title="Stel een standaardmodel in">
        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        Of in de configuratie:

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Copilot SDK-harnasplugin (copilot)">
    Installeer de externe `@openclaw/copilot`-plugin wanneer je wilt dat de Copilot CLI
    en SDK van GitHub de agentlus op laag niveau beheren voor geselecteerde
    `github-copilot/*`-modellen.

    ```bash
    openclaw plugins install @openclaw/copilot
    ```

    Laat vervolgens een model of provider de runtime gebruiken:

    ```json5
    {
      agents: {
        defaults: {
          model: "github-copilot/gpt-5.5",
          models: {
            "github-copilot/gpt-5.5": {
              agentRuntime: { id: "copilot" },
            },
          },
        },
      },
    }
    ```

    Kies dit wanneer je systeemeigen Copilot CLI-sessies, door de SDK beheerde threadstatus
    en door Copilot beheerde Compaction voor die agentbeurten wilt. Zonder de
    expliciete `agentRuntime`-inschakeling blijven `github-copilot/*`-modellen de
    ingebouwde provider gebruiken. Zie [Copilot SDK-harnas](/nl/plugins/copilot) voor het volledige
    runtimecontract.

  </Tab>

  <Tab title="Copilot Proxy-plugin (copilot-proxy)">
    Gebruik de VS Code-extensie **Copilot Proxy** als lokale brug. OpenClaw communiceert met
    het `/v1`-eindpunt van de proxy (standaard `http://localhost:3000/v1`) en gebruikt de
    door jou geconfigureerde modellenlijst.

    De `copilot-proxy`-plugin wordt met OpenClaw meegeleverd en is standaard ingeschakeld.
    Configureer de basis-URL en model-id's met:

    ```bash
    openclaw models auth login --provider copilot-proxy --set-default
    ```

    <Note>
    Kies dit wanneer je Copilot Proxy al in VS Code uitvoert of verkeer
    erdoorheen moet routeren. De VS Code-extensie moet actief blijven.
    </Note>

  </Tab>
</Tabs>

## GitHub Enterprise (gegevensresidentie)

Als je organisatie een GitHub Enterprise-tenant met gegevensresidentie gebruikt (een
`*.ghe.com`-host zoals `your-org.ghe.com`), bevindt Copilot zich op tenantlokale
eindpunten in plaats van op de openbare `github.com`. OpenClaw biedt dit aan als een
volwaardige authenticatiekeuze, zodat je URL's niet handmatig hoeft te bewerken.

<Steps>
  <Step title="Kies de Enterprise-authenticatieoptie">
    Kies tijdens de onboarding of in `openclaw models auth`
    **GitHub Copilot (Enterprise / data residency)**. Je wordt gevraagd naar
    je Enterprise-domein (bijvoorbeeld `your-org.ghe.com`), waarna de apparaataanmelding
    voor die tenant wordt uitgevoerd.

    Voer alleen de tenantroot in (`your-org.ghe.com`). Afgeleide servicehosts zoals
    `api.your-org.ghe.com` of `copilot-api.your-org.ghe.com` worden niet geaccepteerd;
    OpenClaw leidt die eindpunten automatisch af van de tenantroot.

    ```bash
    openclaw models auth login --provider github-copilot --method device-enterprise
    ```

  </Step>
  <Step title="Het domein wordt opgeslagen in de configuratie">
    De gekozen host wordt opgeslagen onder de providerparameters, zodat latere tokenvernieuwingen
    en voltooiingen automatisch op de tenant worden gericht:

    ```json5
    {
      models: {
        providers: {
          "github-copilot": { params: { githubDomain: "your-org.ghe.com" } },
        },
      },
    }
    ```

  </Step>
</Steps>

De apparaatflow, tokenuitwisseling en voltooiingen worden respectievelijk omgezet naar
`https://your-org.ghe.com/login/device/code`,
`https://api.your-org.ghe.com/copilot_internal/v2/token` en
`https://copilot-api.your-org.ghe.com`. Tokens voor gegevensresidentie bevatten
een tenantstempel en geen proxyhint, waardoor de basis-URL voor voltooiingen terugvalt op de
Copilot-host van de tenant in plaats van op het openbare eindpunt.

<Note>
Bij het wisselen van domein wordt de apparaataanmelding altijd opnieuw uitgevoerd. Als je al een opgeslagen
Copilot-token hebt en een ander domein kiest (openbare `github.com` ↔ een `*.ghe.com`-
tenant, of van de ene tenant naar een andere), gebruikt OpenClaw het bestaande token niet opnieuw —
het dwingt een nieuwe aanmelding af, zodat het token is beperkt tot het domein dat naar de
configuratie wordt geschreven. Als je je opnieuw aanmeldt voor *hetzelfde* domein, wordt nog steeds aangeboden het huidige
token opnieuw te gebruiken. Als je terugschakelt naar de openbare `github.com`, wordt de opgeslagen
`githubDomain` gewist, zodat de configuratie terugkeert naar de standaardinstelling.
</Note>

<Note>
De omgevingsvariabele `COPILOT_GITHUB_DOMAIN` overschrijft het omgezette domein
voor elk Copilot-pad dat dit omzet — de Enterprise-apparaataanmelding
(`--method device-enterprise`), de zelfstandige
`openclaw models auth login-github-copilot`-snelkoppeling, tokenvernieuwing, embeddings
en voltooiingen. Stel deze voor volledig headless- of CI-
configuraties in op je `*.ghe.com`-host. Laat deze oningesteld (en laat de configuratieparameter weg) om de openbare `github.com` te gebruiken.
Aanmeldingen slaan het domein op waarvoor ze het token hebben aangemaakt (en wissen het bij een aanmelding
bij de openbare `github.com`), zodat de routering correct blijft, zelfs nadat de
omgevingsvariabele is verwijderd.
</Note>

## Optionele vlaggen

| Opdracht                                                                | Vlag            | Beschrijving                                          |
| ---------------------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| `openclaw models auth login-github-copilot`                            | `--yes`         | Overschrijf een bestaand authenticatieprofiel zonder bevestiging te vragen |
| `openclaw models auth login --provider github-copilot --method device` | `--set-default` | Pas ook het aanbevolen standaardmodel van de provider toe  |

```bash
# Sla de bevestiging voor opnieuw aanmelden over
openclaw models auth login-github-copilot --yes

# Meld je aan en stel het standaardmodel in één stap in
openclaw models auth login --provider github-copilot --method device --set-default
```

## Niet-interactieve onboarding

De apparaataanmeldingsflow vereist een interactieve TTY. Importeer voor een headless configuratie
een bestaand GitHub OAuth-toegangstoken met `openclaw onboard --non-interactive`:

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

Je kunt `--auth-choice` ook weglaten; door `--github-copilot-token` door te geven, wordt de
GitHub Copilot-provider als authenticatiekeuze afgeleid. Als de vlag wordt weggelaten, valt onboarding
terug op `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` en vervolgens `GITHUB_TOKEN`. Gebruik
`--secret-input-mode ref` terwijl `COPILOT_GITHUB_TOKEN` is ingesteld om een door een omgevingsvariabele ondersteunde
`tokenRef` op te slaan in plaats van platte tekst in `auth-profiles.json`.

<AccordionGroup>
  <Accordion title="Interactieve TTY vereist">
    De apparaataanmeldingsflow vereist een interactieve TTY. Voer deze rechtstreeks uit in een
    terminal, niet in een niet-interactief script of een CI-pijplijn.
  </Accordion>

  <Accordion title="De beschikbaarheid van modellen hangt af van je abonnement">
    De beschikbaarheid van Copilot-modellen hangt af van je GitHub-abonnement. Als een model wordt
    geweigerd, probeer dan een andere ID (bijvoorbeeld `github-copilot/gpt-5.5`). Zie
    de [ondersteunde modellen per Copilot-abonnement](https://docs.github.com/en/copilot/reference/ai-models/supported-models#supported-ai-models-per-copilot-plan) van GitHub
    voor de actuele modellenlijst.
  </Accordion>

  <Accordion title="Live catalogusvernieuwing vanuit de Copilot API">
    Zodra het authenticatiepad via apparaataanmelding (of omgevingsvariabele) een GitHub-token heeft verkregen,
    vernieuwt OpenClaw de modelcatalogus op verzoek vanuit `${baseUrl}/models`
    (hetzelfde eindpunt dat VS Code Copilot gebruikt), zodat de runtime
    de rechten per account en nauwkeurige contextvensters volgt zonder
    wijzigingen aan het manifest. Nieuw gepubliceerde Copilot-modellen worden zichtbaar zonder een OpenClaw-
    upgrade en contextvensters weerspiegelen de werkelijke limieten per model
    (bijvoorbeeld 400k voor de gpt-5.x-serie, 1M voor de interne
    `claude-opus-*-1m`-varianten).

    De meegeleverde statische catalogus blijft de zichtbare terugvaloptie wanneer detectie
    is uitgeschakeld, de gebruiker geen GitHub-authenticatieprofiel heeft, de tokenuitwisseling
    mislukt of de HTTPS-aanroep naar `/models` een fout oplevert. Om dit uit te schakelen en volledig
    te vertrouwen op de statische manifestcatalogus (offline-/air-gappedscenario's):

    ```json5
    {
      plugins: {
        entries: {
          "github-copilot": {
            config: { discovery: { enabled: false } },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Transportselectie">
    Claude-model-id's gebruiken automatisch het Anthropic Messages-transport.
    Gemini-modellen gebruiken het OpenAI Chat Completions-transport; GPT- en o-serie-
    modellen blijven het OpenAI Responses-transport gebruiken. OpenClaw selecteert het juiste
    transport op basis van de modelverwijzing.
  </Accordion>

  <Accordion title="Compatibiliteit van aanvragen">
    OpenClaw verzendt aanvraagheaders in Copilot IDE-stijl via Copilot-transporten
    (versies van de VS Code-editor/plugin en de `vscode-chat`-integratie-id),
    markeert vervolgbeurten met toolresultaten als door de agent geïnitieerd en stelt de Copilot-
    vision-header in wanneer een beurt afbeeldingsinvoer bevat.
  </Accordion>

  <Accordion title="Volgorde voor het omzetten van omgevingsvariabelen">
    OpenClaw haalt Copilot-authenticatie uit omgevingsvariabelen op in de volgende
    prioriteitsvolgorde:

    | Prioriteit | Variabele              | Opmerkingen                            |
    | -------- | --------------------- | -------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN` | Hoogste prioriteit, specifiek voor Copilot |
    | 2        | `GH_TOKEN`            | GitHub CLI-token (terugvaloptie)      |
    | 3        | `GITHUB_TOKEN`        | Standaard GitHub-token (laagste prioriteit)   |

    Wanneer meerdere variabelen zijn ingesteld, gebruikt OpenClaw de variabele met de hoogste prioriteit.
    De apparaataanmeldingsflow (`openclaw models auth login-github-copilot`) slaat
    het token op in de authenticatieprofielopslag en krijgt voorrang op alle omgevingsvariabelen.

  </Accordion>

  <Accordion title="Tokenopslag">
    De aanmelding slaat een GitHub-token op in de authenticatieprofielopslag (profiel-id
    `github-copilot:github`) en wisselt dit in voor een kortlevend Copilot API-
    token wanneer OpenClaw wordt uitgevoerd. Je hoeft het token niet handmatig te beheren.
  </Accordion>
</AccordionGroup>

## Embeddings voor geheugenzoekopdrachten

GitHub Copilot kan ook fungeren als embeddingprovider voor
[geheugenzoekopdrachten](/nl/concepts/memory-search). Als je een Copilot-abonnement hebt en
bent aangemeld, kan OpenClaw dit zonder afzonderlijke API-sleutel gebruiken voor embeddings.

### Configuratie

Stel `memory.search.provider` expliciet in om GitHub Copilot-embeddings te gebruiken. Als een
GitHub-token beschikbaar is, detecteert OpenClaw beschikbare embeddingmodellen via
de Copilot API en kiest het automatisch het beste model.

```json5
{
  memory: {
    search: {
      provider: "github-copilot",
      // Optioneel: overschrijf het automatisch gedetecteerde model
      model: "text-embedding-3-small",
    },
  },
}
```

### Hoe het werkt

1. OpenClaw haalt je GitHub-token op (uit omgevingsvariabelen of het authenticatieprofiel).
2. Wisselt het in voor een kortlevend Copilot API-token.
3. Vraagt het Copilot-eindpunt `/models` op om beschikbare embeddingmodellen te detecteren.
4. Kiest het beste model (voorkeursvolgorde: `text-embedding-3-small`,
   `text-embedding-3-large`, `text-embedding-ada-002`).
5. Verzendt embeddingaanvragen naar het Copilot-eindpunt `/embeddings`.

De beschikbaarheid van modellen hangt af van je GitHub-abonnement. Als er geen embeddingmodellen
beschikbaar zijn, slaat OpenClaw Copilot over en probeert het de volgende provider.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="OAuth en authenticatie" href="/nl/gateway/authentication" icon="key">
    Authenticatiedetails en regels voor het hergebruik van aanmeldgegevens.
  </Card>
</CardGroup>
