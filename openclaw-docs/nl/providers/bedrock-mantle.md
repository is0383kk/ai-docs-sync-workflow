---
read_when:
    - Je wilt door Bedrock Mantle gehoste OSS-modellen gebruiken met OpenClaw
    - Je hebt het OpenAI-compatibele Mantle-eindpunt nodig voor GPT-OSS, Qwen, Kimi of GLM
    - Je wilt Claude Opus 5, Sonnet 5 of Mythos 5 gebruiken via Amazon Bedrock Mantle
summary: Gebruik OpenAI-compatibele en Claude Messages-modellen van Amazon Bedrock Mantle met OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-07-27T05:12:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d2b49120560c4466aff217c3101fab057dd87c1c501f1b8eb94d74f62bd1037
    source_path: providers/bedrock-mantle.md
    workflow: 16
---

OpenClaw bevat een gebundelde **Amazon Bedrock Mantle**-provider die verbinding maakt met
het OpenAI-compatibele Mantle-eindpunt. Mantle host opensource- en
modellen van derden (GPT-OSS, Qwen, Kimi, GLM en vergelijkbare modellen) via een standaard
`/v1/chat/completions`-oppervlak dat wordt ondersteund door de Bedrock-infrastructuur. Mantle stelt ook
Anthropic Claude-modellen beschikbaar via een Anthropic Messages-route.

| Eigenschap       | Waarde                                                                                  |
| -------------- | -------------------------------------------------------------------------------------- |
| Provider-ID    | `amazon-bedrock-mantle`                                                                |
| API            | `openai-completions` voor ontdekte OSS-modellen, `anthropic-messages` voor Claude-modellen |
| Authenticatie           | Expliciete `AWS_BEARER_TOKEN_BEDROCK` of genereren van een bearer-token via de IAM-referentieketen    |
| Standaardregio | `us-east-1` (overschrijven met `AWS_REGION` of `AWS_DEFAULT_REGION`)                       |

## Aan de slag

Kies de gewenste authenticatiemethode en volg de configuratiestappen.

<Tabs>
  <Tab title="Expliciet bearer-token">
    **Meest geschikt voor:** omgevingen waarin je al een Mantle-bearer-token hebt.

    <Steps>
      <Step title="Stel het bearer-token in op de Gateway-host">
        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        Stel desgewenst een regio in (standaard `us-east-1`):

        ```bash
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Controleer of modellen worden ontdekt">
        ```bash
        openclaw models list
        ```

        Ontdekte modellen worden weergegeven onder de provider `amazon-bedrock-mantle`. Er is geen
        aanvullende configuratie vereist, tenzij je de standaardwaarden wilt overschrijven.
      </Step>
    </Steps>

  </Tab>

  <Tab title="IAM-referenties">
    **Meest geschikt voor:** het gebruik van AWS SDK-compatibele referenties (gedeelde configuratie, SSO, webidentiteit, instantie- of taakrollen).

    <Steps>
      <Step title="Configureer AWS-referenties op de Gateway-host">
        Elke AWS SDK-compatibele authenticatiebron werkt:

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Controleer of modellen worden ontdekt">
        ```bash
        openclaw models list
        ```

        OpenClaw genereert automatisch een Mantle-bearer-token uit de referentieketen.
      </Step>
    </Steps>

    <Tip>
    Wanneer `AWS_BEARER_TOKEN_BEDROCK` niet is ingesteld, maakt OpenClaw het bearer-token voor je aan vanuit de standaard AWS-referentieketen, waaronder gedeelde referenties/configuratieprofielen, SSO, webidentiteit en instantie- of taakrollen.
    </Tip>

  </Tab>
</Tabs>

## Automatische modeldetectie

Wanneer `AWS_BEARER_TOKEN_BEDROCK` is ingesteld, gebruikt OpenClaw deze rechtstreeks. Anders
probeert OpenClaw een Mantle-bearer-token te genereren vanuit de standaard
AWS-referentieketen. Vervolgens ontdekt het beschikbare Mantle-modellen door het
`/v1/models`-eindpunt van de regio te bevragen.

| Gedrag          | Details                                                                               |
| ----------------- | ------------------------------------------------------------------------------------ |
| Detectiecache   | Resultaten worden per regio 1 uur gecachet; bij een ophaalfout wordt het laatst gecachete resultaat geretourneerd |
| Vernieuwing van IAM-token | Elke 2 uur, per regio gecachet                                                     |

Als je de Mantle-plugin ingeschakeld wilt houden maar automatische detectie en het
genereren van IAM-bearer-tokens wilt onderdrukken, schakel je de detectieschakelaar van de plugin uit:

```bash
openclaw config set plugins.entries.amazon-bedrock-mantle.config.discovery.enabled false
```

<Note>
Het bearer-token is dezelfde `AWS_BEARER_TOKEN_BEDROCK` die door de standaardprovider [Amazon Bedrock](/nl/providers/bedrock) wordt gebruikt.
</Note>

### Ondersteunde regio's

`us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Handmatige configuratie

Als je expliciete configuratie verkiest boven automatische detectie:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

Een expliciete, niet-lege `models`-lijst is bepalend en vervangt elke
ontdekte rij, inclusief de Claude-rijen hieronder. Laat `models` weg om de
automatische Mantle-catalogus te behouden, of neem de volledige Claude-modelvermeldingen op die je
wilt gebruiken.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Ondersteuning voor redeneren">
    Ondersteuning voor redeneren wordt afgeleid uit model-ID's die patronen bevatten zoals
    `thinking`, `reasoner`, `reasoning`, `deepseek.r`, `gpt-oss-120b` of
    `gpt-oss-safeguard-120b`. OpenClaw stelt `reasoning: true` automatisch in voor
    overeenkomende modellen tijdens de detectie.
  </Accordion>

  <Accordion title="Onbeschikbaarheid van het eindpunt">
    Als het Mantle-eindpunt niet beschikbaar is, geen modellen retourneert of het
    ophalen van het bearer-token mislukt, retourneert de detectie een leeg resultaat en wordt de impliciete
    provider overgeslagen. OpenClaw geeft geen foutmelding; andere geconfigureerde providers
    blijven normaal werken.
  </Accordion>

  <Accordion title="Claude via de Anthropic Messages-route">
    Wanneer automatische detectie de modellenlijst beheert, voegt OpenClaw na een
    geslaagde zoekactie vijf Claude-modellen toe, ongeacht wat `/v1/models` retourneert:
    `amazon-bedrock-mantle/anthropic.claude-opus-5` (Claude Opus 5),
    `amazon-bedrock-mantle/anthropic.claude-sonnet-5` (Claude Sonnet 5),
    `amazon-bedrock-mantle/anthropic.claude-opus-4-7` (Claude Opus 4.7) en
    `amazon-bedrock-mantle/anthropic.claude-mythos-5` (Claude Mythos 5), plus
    `amazon-bedrock-mantle/anthropic.claude-mythos-preview` (Claude Mythos
    Preview). Ze gebruiken het `anthropic-messages`-API-oppervlak en streamen via
    hetzelfde met een bearer-token geauthenticeerde, Anthropic-compatibele eindpunt
    (`<mantle-base>/anthropic`), zodat het AWS-bearer-token niet als een
    Anthropic-API-sleutel wordt behandeld.

    Claude Opus 5 publiceert een contextvenster van 1.000.000 tokens, een
    uitvoerlimiet van 128.000 tokens, afbeeldingsinvoer en `$5/$25`-prijzen voor invoer/uitvoer. Adaptief
    denken gebruikt standaard `high`; `/think off` schakelt denken uit en
    `/think xhigh|max` gebruikt de ingebouwde inspanningsniveaus van het model. OpenClaw laat
    door de aanroeper geselecteerde samplingparameters weg.

    Claude Sonnet 5 gebruikt altijd adaptief denken en standaard een
    inspanningsniveau van `high`. `/think off` en `/think minimal` worden toegewezen aan `low`, omdat de Mantle-
    route denken niet kan uitschakelen. OpenClaw laat ook een aangepaste temperatuur weg bij
    Sonnet 5-verzoeken.

    Claude Mythos 5 heeft beperkte toegang. Het publiceert een contextvenster van
    1.000.000 tokens en een uitvoerlimiet van 128.000 tokens, gebruikt altijd adaptief denken, wijst
    `/think off` en `/think minimal` toe aan `low` en laat door de aanroeper geselecteerde
    samplingparameters weg.

    Claude Mythos Preview vraagt altijd om redeneren, met standaard een inspanningsniveau van
    `high` wanneer er geen `/think`-niveau is ingesteld (toegewezen van `xhigh`/`max` omlaag naar
    `high` en `minimal` omhoog naar `low`). Opus 4.7 streamt op Mantle zonder
    door het model aangeleverde redenering en OpenClaw laat de parameter `temperature`
    weg, omdat Opus 4.7 op deze route geen samplingoverschrijvingen accepteert; Mythos
    Preview accepteert normaal een `temperature`-overschrijving.

    Een niet-lege expliciete `models.providers["amazon-bedrock-mantle"].models`-
    lijst vervangt de volledige ontdekte catalogus. Laat die lijst weg als je
    deze ingebouwde Claude-rijen wilt.

  </Accordion>

  <Accordion title="Relatie tot de Amazon Bedrock-provider">
    Bedrock Mantle is een afzonderlijke provider van de standaardprovider
    [Amazon Bedrock](/nl/providers/bedrock). Mantle gebruikt een
    OpenAI-compatibel `/v1`-oppervlak voor zijn OSS-catalogus, terwijl de standaard-
    Bedrock-provider de systeemeigen Bedrock Converse-API gebruikt.

    Beide providers delen dezelfde `AWS_BEARER_TOKEN_BEDROCK`-referentie wanneer
    die aanwezig is.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Amazon Bedrock" href="/nl/providers/bedrock" icon="cloud">
    Systeemeigen Bedrock-provider voor Anthropic Claude, Titan en andere modellen.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="OAuth en authenticatie" href="/nl/gateway/authentication" icon="key">
    Authenticatiedetails en regels voor hergebruik van referenties.
  </Card>
  <Card title="Probleemoplossing" href="/nl/help/troubleshooting" icon="wrench">
    Veelvoorkomende problemen en hoe je ze oplost.
  </Card>
</CardGroup>
