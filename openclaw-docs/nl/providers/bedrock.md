---
read_when:
    - Je wilt Amazon Bedrock-modellen gebruiken met OpenClaw
    - Je moet AWS-referenties en een regio instellen voor modelaanroepen
summary: Gebruik Amazon Bedrock-modellen (Converse API) met OpenClaw
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-07-27T05:18:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9cbc9534c0d06e0d5642b8d167c633c16880908812b97adbbf9c6bd6c5511603
    source_path: providers/bedrock.md
    workflow: 16
---

OpenClaw kan **Amazon Bedrock**-modellen gebruiken via de streamingprovider **Bedrock Converse**. Bedrock-authenticatie gebruikt de **standaardreferentieketen van de AWS SDK**,
niet een API-sleutel.

| Eigenschap | Waarde                                                       |
| -------- | ----------------------------------------------------------- |
| Provider | `amazon-bedrock`                                            |
| API      | `bedrock-converse-stream`                                   |
| Authenticatie     | AWS-referenties (omgevingsvariabelen, gedeelde configuratie of instantirol) |
| Regio   | `AWS_REGION` of `AWS_DEFAULT_REGION` (standaard: `us-east-1`) |

## Aan de slag

Kies de gewenste authenticatiemethode en volg de configuratiestappen.

<Tabs>
  <Tab title="Toegangssleutels / omgevingsvariabelen">
    **Meest geschikt voor:** ontwikkelaarsmachines, CI of hosts waarop je AWS-referenties rechtstreeks beheert.

    <Steps>
      <Step title="Stel AWS-referenties in op de Gateway-host">
        ```bash
        export AWS_ACCESS_KEY_ID="EXAMPLE_AWS_ACCESS_KEY_ID"
        export AWS_SECRET_ACCESS_KEY="..."
        export AWS_REGION="us-east-1"
        # Optioneel:
        export AWS_SESSION_TOKEN="..."
        export AWS_PROFILE="your-profile"
        # Optioneel (Bedrock API-sleutel/bearertoken):
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```
      </Step>
      <Step title="Voeg een Bedrock-provider en -model toe aan je configuratie">
        Er is geen `apiKey` vereist. Configureer de provider met `auth: "aws-sdk"`:

        ```json5
        {
          models: {
            providers: {
              "amazon-bedrock": {
                baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
                api: "bedrock-converse-stream",
                auth: "aws-sdk",
                models: [
                  {
                    id: "us.anthropic.claude-opus-4-6-v1",
                    name: "Claude Opus 4.6 (Bedrock)",
                    reasoning: true,
                    input: ["text", "image"],
                    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                    contextWindow: 200000,
                    maxTokens: 8192,
                  },
                ],
              },
            },
          },
          agents: {
            defaults: {
              model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1" },
            },
          },
        }
        ```
      </Step>
      <Step title="Controleer of modellen beschikbaar zijn">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Tip>
    Met authenticatie via omgevingsmarkeringen (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE` of `AWS_BEARER_TOKEN_BEDROCK`) schakelt OpenClaw automatisch de impliciete Bedrock-provider in voor modeldetectie, zonder extra configuratie.
    </Tip>

  </Tab>

  <Tab title="EC2-instantirollen (IMDS)">
    **Meest geschikt voor:** EC2-instanties waaraan een IAM-rol is gekoppeld en die de metadataservice voor instanties gebruiken voor authenticatie.

    <Steps>
      <Step title="Schakel detectie expliciet in">
        Bij gebruik van IMDS kan OpenClaw AWS-authenticatie niet uitsluitend aan de hand van omgevingsmarkeringen detecteren. Je moet dit daarom expliciet inschakelen:

        ```bash
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
        openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1
        ```
      </Step>
      <Step title="Voeg desgewenst een omgevingsmarkering toe voor de automatische modus">
        Als je ook wilt dat het automatische detectiepad via omgevingsmarkeringen werkt (bijvoorbeeld voor `openclaw status`-oppervlakken):

        ```bash
        export AWS_PROFILE=default
        export AWS_REGION=us-east-1
        ```

        Je hebt **geen** fictieve API-sleutel nodig.
      </Step>
      <Step title="Controleer of modellen zijn gedetecteerd">
        ```bash
        openclaw models list
        ```
      </Step>
    </Steps>

    <Warning>
    De IAM-rol die aan je EC2-instantie is gekoppeld, moet de volgende machtigingen hebben:

    - `bedrock:InvokeModel`
    - `bedrock:InvokeModelWithResponseStream`
    - `bedrock:ListFoundationModels` (voor automatische detectie)
    - `bedrock:ListInferenceProfiles` (voor detectie van inferentieprofielen)

    Of koppel het beheerde beleid `AmazonBedrockFullAccess`.
    </Warning>

    <Note>
    Je hebt `AWS_PROFILE=default` alleen nodig als je specifiek een omgevingsmarkering wilt voor de automatische modus of statusoppervlakken. Het daadwerkelijke authenticatiepad van de Bedrock-runtime gebruikt de standaardketen van de AWS SDK. Authenticatie via een IMDS-instantirol werkt dus ook zonder omgevingsmarkeringen.
    </Note>

  </Tab>
</Tabs>

## Automatische modeldetectie

OpenClaw kan automatisch Bedrock-modellen detecteren die **streaming**
en **tekstuitvoer** ondersteunen. De detectie gebruikt `bedrock:ListFoundationModels` en
`bedrock:ListInferenceProfiles`, en de resultaten worden in de cache opgeslagen (standaard: 1 uur).

Zo wordt de impliciete provider ingeschakeld:

- Als `plugins.entries.amazon-bedrock.config.discovery.enabled` `true` is,
  probeert OpenClaw detectie uit te voeren, zelfs als er geen AWS-omgevingsmarkering aanwezig is.
- Als `plugins.entries.amazon-bedrock.config.discovery.enabled` niet is ingesteld,
  voegt OpenClaw de impliciete Bedrock-provider alleen automatisch toe
  wanneer een van deze AWS-authenticatiemarkeringen wordt aangetroffen:
  `AWS_BEARER_TOKEN_BEDROCK`, `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY` of `AWS_PROFILE`.
- Het daadwerkelijke authenticatiepad van de Bedrock-runtime gebruikt nog steeds de standaardketen van de AWS SDK. Daardoor kunnen gedeelde configuratie, SSO en authenticatie via een IMDS-instantirol werken, zelfs wanneer voor detectie `enabled: true` nodig was om deze in te schakelen.

<Note>
Voor expliciete `models.providers["amazon-bedrock"]`-vermeldingen kan OpenClaw Bedrock-authenticatie via omgevingsmarkeringen al vroeg herleiden uit AWS-omgevingsmarkeringen zoals `AWS_BEARER_TOKEN_BEDROCK`, zonder het volledig laden van runtime-authenticatie af te dwingen. Het daadwerkelijke authenticatiepad voor modelaanroepen gebruikt nog steeds de standaardketen van de AWS SDK.
</Note>

<AccordionGroup>
  <Accordion title="Configuratieopties voor detectie">
    Configuratieopties bevinden zich onder `plugins.entries.amazon-bedrock.config.discovery`:

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              discovery: {
                enabled: true,
                region: "us-east-1",
                providerFilter: ["anthropic", "amazon"],
                refreshInterval: 3600,
                defaultContextWindow: 32000,
                defaultMaxTokens: 4096,
              },
            },
          },
        },
      },
    }
    ```

    | Optie | Standaard | Beschrijving |
    | ------ | ------- | ----------- |
    | `enabled` | automatisch | In de automatische modus schakelt OpenClaw de impliciete Bedrock-provider alleen in wanneer een ondersteunde AWS-omgevingsmarkering wordt aangetroffen. Stel `true` in om detectie af te dwingen. |
    | `region` | `AWS_REGION` / `AWS_DEFAULT_REGION` / `us-east-1` | AWS-regio die wordt gebruikt voor API-aanroepen voor detectie. |
    | `providerFilter` | (alle) | Komt overeen met namen van Bedrock-providers (bijvoorbeeld `anthropic`, `amazon`). |
    | `refreshInterval` | `3600` | Cacheduur in seconden. Stel in op `0` om caching uit te schakelen. |
    | `defaultContextWindow` | `32000` | Contextvenster dat wordt gebruikt voor gedetecteerde modellen zonder bekende tokenlimieten (overschrijf dit als je de limieten van je model kent). |
    | `defaultMaxTokens` | `4096` | Maximaal aantal uitvoertokens dat wordt gebruikt voor gedetecteerde modellen zonder bekende tokenlimieten (overschrijf dit als je de limieten van je model kent). |

  </Accordion>

  <Accordion title="Contextvenster en limieten voor het maximale aantal tokens">
    De Bedrock-API's `ListFoundationModels` en `GetFoundationModel` retourneren geen
    metagegevens over tokenlimieten, maar alleen de model-ID, naam, modaliteiten en levenscyclusstatus.
    OpenClaw wordt geleverd met een opzoektabel van bekende contextvensters en uitvoerlimieten
    voor populaire Bedrock-modellen (Claude, Nova, Llama, Mistral, DeepSeek
    en andere), zodat sessiebeheer, Compaction-drempels en
    detectie van contextoverschrijding correct werken voor deze modellen.

    Gedetecteerde modellen die niet in de tabel staan, vallen terug op `defaultContextWindow`
    en `defaultMaxTokens`. Als voor een model dat je gebruikt nauwkeurige limieten ontbreken,
    overschrijf je deze met een expliciete
    `models.providers["amazon-bedrock"].models`-vermelding.

  </Accordion>
</AccordionGroup>

## Snelle configuratie (AWS-pad)

Deze stapsgewijze uitleg maakt een IAM-rol, koppelt Bedrock-machtigingen, associeert
het instantieprofiel en schakelt OpenClaw-detectie in op de EC2-host.

```bash
# 1. Maak een IAM-rol en instantieprofiel
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. Koppel aan je EC2-instantie
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. Schakel detectie expliciet in op de EC2-instantie
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. Optioneel: voeg een omgevingsmarkering toe als je de automatische modus zonder expliciete inschakeling wilt
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. Controleer of modellen zijn gedetecteerd
openclaw models list
```

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Inferentieprofielen">
    OpenClaw detecteert **regionale en globale inferentieprofielen** naast
    basismodellen. Wanneer een profiel naar een bekend basismodel verwijst, neemt
    het profiel de mogelijkheden van dat model over (contextvenster, maximaal aantal tokens,
    redeneren, beeldverwerking) en wordt automatisch de juiste Bedrock-aanvraagregio
    ingevoegd. Dit betekent dat regioverschrijdende Claude-profielen zonder handmatige
    provideroverschrijvingen werken. Globale regioverschrijdende profielen (`global.*`) worden
    als eerste vermeld in `openclaw models list`, omdat ze doorgaans betere capaciteit
    en automatische failover bieden.

    ID's van inferentieprofielen zien eruit als `us.anthropic.claude-opus-4-6-v1` (regionaal)
    of `anthropic.claude-opus-4-6-v1` (globaal). Als het onderliggende model al
    in de detectieresultaten staat, neemt het profiel de volledige reeks mogelijkheden ervan over;
    anders gelden veilige standaardwaarden.

    Er is geen extra configuratie nodig. Zolang detectie is ingeschakeld en de IAM-principal
    `bedrock:ListInferenceProfiles` heeft, verschijnen profielen naast
    basismodellen in `openclaw models list`.

  </Accordion>

  <Accordion title="Serviceniveau">
    Sommige Bedrock-modellen ondersteunen een parameter `service_tier` om kosten
    of latentie te optimaliseren. De volgende niveaus zijn beschikbaar:

    | Niveau | Beschrijving |
    |------|-------------|
    | `default` | Standaardniveau van Bedrock |
    | `flex` | Verwerking met korting voor workloads die een langere latentie kunnen verdragen |
    | `priority` | Verwerking met prioriteit voor latentiegevoelige workloads |
    | `reserved` | Gereserveerde capaciteit voor workloads met een constante belasting |

    Stel `serviceTier` (of `service_tier`) via `agents.defaults.params` in voor
    Bedrock-modelaanvragen, of per model in
    `agents.defaults.models["<model-key>"].params`:

    ```json5
    {
      agents: {
        defaults: {
          params: {
            serviceTier: "flex", // geldt voor alle modellen
          },
          models: {
            "amazon-bedrock/mistral.mistral-large-3-675b-instruct": {
              params: {
                serviceTier: "priority", // overschrijving per model
              },
            },
          },
        },
      },
    }
    ```

    Geldige waarden zijn `default`, `flex`, `priority` en `reserved`. Claude
    Fable 5, Opus 5 en Sonnet 5 ondersteunen alleen het niveau `default`; OpenClaw waarschuwt en
    negeert `flex`, `priority` of `reserved` wanneer deze voor die modellen worden aangevraagd. Voor
    andere modellen geldt dat niet elk model elk niveau ondersteunt -- een niet-ondersteund niveau
    retourneert een Bedrock-validatiefout en de foutmelding kan
    misleidend zijn (bijvoorbeeld "The provided model identifier is invalid"
    in plaats van het niveau als probleem te benoemen). Controleer bij deze fout
    of het model het aangevraagde niveau ondersteunt.

  </Accordion>

  <Accordion title="Temperatuur voor Claude Opus 5, 4.8 en 4.7">
    Bedrock weigert de parameter `temperature` voor Claude Opus 5, Opus 4.8
    en Opus 4.7. OpenClaw laat `temperature` automatisch weg voor elke overeenkomende Bedrock-
    ref, waaronder foundationmodel-id's, benoemde inferentieprofielen, applicatie-
    inferentieprofielen waarvan het onderliggende model via
    `bedrock:GetInferenceProfile` wordt herleid tot Opus 5/4.8/4.7, en varianten met punten van
    `opus-4.7`/`opus-4.8` met optionele regiovoorvoegsels (`us.`, `eu.`, `ap.`, `apac.`, `au.`, `jp.`,
    `global.`). Er is geen configuratieoptie vereist en de weglating geldt zowel voor
    het object met aanvraagopties als voor het payloadveld `inferenceConfig`.
  </Accordion>

  <Accordion title="Claude Opus 5">
    Gebruik `amazon-bedrock/anthropic.claude-opus-5` op het Bedrock-eindpunt van de Messages-API,
    of een regionaal/globaal inferentieprofiel zoals
    `global.anthropic.claude-opus-5` wanneer dit in Bedrock-detectie verschijnt.
    OpenClaw past het contextvenster van 1,000,000 tokens, de uitvoerlimiet van 128,000 tokens,
    afbeeldingsinvoer, promptcaching, veilig streamen bij weigeringen en native
    inspanningsniveaus `xhigh`/`max` toe.

    Adaptief denken is standaard ingesteld op `high`. `/think off` schakelt denken uit, terwijl
    `/think xhigh|max` adaptief denken ingeschakeld houdt. OpenClaw laat aangepaste
    samplingparameters en niet-ondersteunde niet-standaard serviceniveaus weg.

  </Accordion>

  <Accordion title="Claude Fable 5">
    Gebruik `amazon-bedrock/anthropic.claude-fable-5` in `us-east-1`, of de
    regionale inferentie-id's zoals `us.anthropic.claude-fable-5`.
    OpenClaw past Fable's contextvenster van 1M, uitvoerlimiet van 128K, permanent
    ingeschakeld adaptief denken en ondersteunde inspanningstoewijzing toe. `/think off` en
    `/think minimal` worden toegewezen aan `low`; temperatuur en besturingselementen voor gedwongen toolkeuze
    worden weggelaten, in overeenstemming met de Opus 4.7/4.8-route. Gestreamde uitvoer wordt vastgehouden
    totdat Bedrock een eindstatus retourneert, zodat weigeringen tijdens het streamen geen
    gedeeltelijke tekst blootleggen.

    AWS vereist expliciete aanmelding voor gegevensbewaring via `provider_data_share` voordat
    Fable beschikbaar is. Prompts en voltooiingen worden met Anthropic gedeeld en
    maximaal 30 dagen bewaard voor vertrouwen en veiligheid. Controleer en configureer
    [gegevensbewaring van Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html)
    voordat je het model inschakelt.

  </Accordion>

  <Accordion title="Claude Mythos 5">
    Claude Mythos 5 is via Bedrock alleen beschikbaar voor accounts met de
    vereiste goedkeuring voor beperkte toegang. OpenClaw herkent het foundationmodel
    `anthropic.claude-mythos-5` en regionale of globale inferentieprofielen zoals
    `us.anthropic.claude-mythos-5`.

    OpenClaw past het contextvenster van 1,000,000 tokens, de uitvoerlimiet van 128,000 tokens,
    afbeeldingsinvoer, promptcaching, veilig streamen bij weigeringen en native
    inspanningsniveaus toe. Adaptief denken is altijd ingeschakeld: `/think off` en
    `/think minimal` worden toegewezen aan `low`, terwijl `xhigh` en `max` beschikbaar blijven.
    Aangepaste samplingwaarden en waarden voor gedwongen toolkeuze worden weggelaten.

  </Accordion>

  <Accordion title="Claude Sonnet 5">
    AWS documenteert Sonnet 5 voor zowel de
    [eindpunten `bedrock-runtime` en `bedrock-mantle`](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html).
    OpenClaw herkent het Bedrock-foundationmodel
    `anthropic.claude-sonnet-5` en regionale of globale inferentieprofielen zoals
    `us.anthropic.claude-sonnet-5`. Het past het contextvenster van 1,000,000 tokens,
    de uitvoerlimiet van 128,000 tokens, afbeeldingsinvoer, native inspanningsniveaus,
    promptcaching en veilig streamen bij weigeringen toe.

    Bedrock houdt adaptief denken ingeschakeld voor Sonnet 5. OpenClaw gebruikt standaard
    `high`; `/think off` en `/think minimal` worden toegewezen aan `low`, omdat deze route
    denken niet kan uitschakelen. Aangepaste temperatuurwaarden en waarden voor gedwongen toolkeuze
    worden weggelaten zolang adaptief denken actief is.

  </Accordion>

  <Accordion title="Beveiligingsregels">
    Je kunt [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
    toepassen op alle aanroepen van Bedrock-modellen door een object `guardrail` toe te voegen aan de
    Plugin-configuratie `amazon-bedrock`. Met beveiligingsregels kun je inhoudsfiltering,
    het weigeren van onderwerpen, woordfilters, filters voor gevoelige informatie en controles voor
    contextuele onderbouwing afdwingen.

    ```json5
    {
      plugins: {
        entries: {
          "amazon-bedrock": {
            config: {
              guardrail: {
                guardrailIdentifier: "abc123", // ID van beveiligingsregel of volledige ARN
                guardrailVersion: "1", // versienummer of "DRAFT"
                streamProcessingMode: "sync", // optioneel: "sync" of "async"
                trace: "enabled", // optioneel: "enabled", "disabled" of "enabled_full"
              },
            },
          },
        },
      },
    }
    ```

    `guardrailIdentifier` en `guardrailVersion` zijn vereist.

    | Optie | Beschrijving |
    | ------ | ----------- |
    | `guardrailIdentifier` | ID van beveiligingsregel (bijv. `abc123`) of volledige ARN (bijv. `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`). |
    | `guardrailVersion` | Gepubliceerd versienummer, of `"DRAFT"` voor het werkconcept. |
    | `streamProcessingMode` | `"sync"` of `"async"` voor evaluatie van beveiligingsregels tijdens het streamen. Als dit wordt weggelaten, gebruikt Bedrock de standaardwaarde. |
    | `trace` | `"enabled"` of `"enabled_full"` voor foutopsporing; laat dit weg of stel `"disabled"` in voor productie. |

    <Warning>
    De IAM-principal die door de Gateway wordt gebruikt, moet naast de standaardaanroepmachtigingen ook over de machtiging `bedrock:ApplyGuardrail` beschikken.
    </Warning>

  </Accordion>

  <Accordion title="Embeddings voor geheugenzoekopdrachten">
    Bedrock kan ook dienen als embeddingprovider voor
    [geheugenzoekopdrachten](/nl/concepts/memory-search). Dit wordt afzonderlijk van de
    inferentieprovider geconfigureerd -- stel `memory.search.provider` in op `"bedrock"`:

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0", // standaard
        },
      },
    }
    ```

    Bedrock-embeddings gebruiken dezelfde AWS SDK-referentieketen als inferentie (instantie-
    rollen, SSO, toegangssleutels, gedeelde configuratie en webidentiteit). Er is geen API-sleutel
    nodig.

    Ondersteunde embeddingmodellen zijn onder andere Amazon Titan Embed (v1, v2), Amazon Nova
    Embed, Cohere Embed (v3, v4) en TwelveLabs Marengo. Zie
    [Referentie voor geheugenconfiguratie -- Bedrock](/nl/reference/memory-config#bedrock-embedding-config)
    voor de volledige modellenlijst en dimensieopties.

  </Accordion>

  <Accordion title="Opmerkingen en aandachtspunten">
    - Voor Bedrock moet **modeltoegang** zijn ingeschakeld in je AWS-account/regio.
    - Automatische detectie vereist de machtigingen `bedrock:ListFoundationModels` en
      `bedrock:ListInferenceProfiles`.
    - Als je de automatische modus gebruikt, stel je een van de ondersteunde AWS-omgevingsmarkeringen voor authenticatie in op de
      Gateway-host. Als je IMDS-/gedeelde-configuratie-authenticatie zonder omgevingsmarkeringen verkiest, stel je
      `plugins.entries.amazon-bedrock.config.discovery.enabled: true` in.
    - OpenClaw toont de referentiebron in deze volgorde: `AWS_BEARER_TOKEN_BEDROCK`,
      vervolgens `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, daarna `AWS_PROFILE` en vervolgens de
      standaard AWS SDK-keten.
    - Ondersteuning voor redeneren is afhankelijk van het model; raadpleeg de Bedrock-modelkaart voor
      de huidige mogelijkheden.
    - Als je de voorkeur geeft aan een beheerde sleutelprocedure, kun je ook een OpenAI-compatibele
      proxy vóór Bedrock plaatsen en deze in plaats daarvan configureren als OpenAI-provider.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelrefs en failovergedrag kiezen.
  </Card>
  <Card title="Geheugenzoekopdrachten" href="/nl/concepts/memory-search" icon="magnifying-glass">
    Bedrock-embeddings voor de configuratie van geheugenzoekopdrachten.
  </Card>
  <Card title="Referentie voor geheugenconfiguratie" href="/nl/reference/memory-config#bedrock-embedding-config" icon="database">
    Volledige lijst van Bedrock-embeddingmodellen en dimensieopties.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Algemene probleemoplossing en veelgestelde vragen.
  </Card>
</CardGroup>
