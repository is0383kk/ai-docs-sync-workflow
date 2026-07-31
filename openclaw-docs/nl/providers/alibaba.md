---
read_when:
    - Je wilt Alibaba Wan-videogeneratie gebruiken in OpenClaw
    - Je moet een API-sleutel voor Model Studio of DashScope instellen om video's te genereren
summary: Alibaba Model Studio Wan-videogeneratie in OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-07-27T05:28:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb74e2361500ccfbc5d3c4f2d08c3b62aacba8c79c704570952e2181abacf9fb
    source_path: providers/alibaba.md
    workflow: 16
---

De gebundelde `alibaba`-plugin registreert een provider voor videogeneratie voor Wan-modellen op Alibaba Model Studio (de internationale naam voor DashScope). Deze is standaard ingeschakeld; alleen een API-sleutel is vereist.

| Eigenschap       | Waarde                                                                          |
| ---------------- | ------------------------------------------------------------------------------- |
| Provider-id      | `alibaba`                                                              |
| Plugin           | gebundeld, `enabledByDefault: true`                                                   |
| Auth-omgevingsvariabelen | `MODELSTUDIO_API_KEY` → `DASHSCOPE_API_KEY` → `QWEN_API_KEY` (eerste overeenkomst wint) |
| Onboarding-vlag  | `--auth-choice alibaba-model-studio-api-key`                                                              |
| Directe CLI-vlag | `--alibaba-model-studio-api-key <key>`                                                              |
| Standaardmodel   | `alibaba/wan2.6-t2v`                                                              |
| Standaardbasis-URL | `https://dashscope-intl.aliyuncs.com`                                                            |

## Aan de slag

<Steps>
  <Step title="Stel een API-sleutel in">
    Sla de sleutel via onboarding op voor de provider `alibaba`:

    ```bash
    openclaw onboard --auth-choice alibaba-model-studio-api-key
    ```

    Of geef de sleutel rechtstreeks door:

    ```bash
    openclaw onboard --alibaba-model-studio-api-key <your-key>
    ```

    Of exporteer een van de geaccepteerde omgevingsvariabelen voordat je de Gateway start:

    ```bash
    export MODELSTUDIO_API_KEY=sk-...
    # of DASHSCOPE_API_KEY=...
    # of QWEN_API_KEY=...
    ```

  </Step>
  <Step title="Stel een standaardvideomodel in">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "alibaba/wan2.6-t2v",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Controleer of de provider is geconfigureerd">
    ```bash
    openclaw models list --provider alibaba
    ```

    De lijst bevat alle vijf gebundelde Wan-modellen. Als `MODELSTUDIO_API_KEY` niet kan worden gevonden, meldt `openclaw models status --json` de ontbrekende referentie onder `auth.unusableProfiles`.

  </Step>
</Steps>

<Note>
  De Alibaba-plugin en de [Qwen-plugin](/nl/providers/qwen) authenticeren beide bij DashScope en accepteren overlappende omgevingsvariabelen. Gebruik `alibaba/...`-model-id's voor de speciale Wan-video-interface; gebruik `qwen/...`-id's voor Qwen-chat, embeddings of mediabegrip.
</Note>

## Ingebouwde Wan-modellen

| Modelreferentie            | Modus                     |
| -------------------------- | ------------------------- |
| `alibaba/wan2.6-t2v`         | Tekst-naar-video (standaard) |
| `alibaba/wan2.6-i2v`         | Afbeelding-naar-video     |
| `alibaba/wan2.6-r2v`         | Referentie-naar-video     |
| `alibaba/wan2.6-r2v-flash`         | Referentie-naar-video (snel) |
| `alibaba/wan2.7-r2v`         | Referentie-naar-video     |

## Mogelijkheden en limieten

Alle drie de modi hebben dezelfde limiet voor het aantal video's en de duur per aanvraag; alleen de invoervorm verschilt.

| Modus                    | Max. uitvoervideo's | Max. invoerafbeeldingen | Max. invoervideo's | Max. duur | Ondersteunde instellingen                                  |
| ------------------------ | ------------------- | ----------------------- | ------------------- | --------- | ---------------------------------------------------------- |
| Tekst-naar-video         | 1                   | n.v.t.                  | n.v.t.              | 10 s      | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Afbeelding-naar-video    | 1                   | 1                       | n.v.t.              | 10 s      | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Referentie-naar-video    | 1                   | n.v.t.                  | 4                    | 10 s      | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |

Een aanvraag zonder `durationSeconds` krijgt de door DashScope geaccepteerde standaardwaarde van **5 seconden**. Stel `durationSeconds` expliciet in voor de [tool voor videogeneratie](/nl/tools/video-generation) om de duur te verlengen tot maximaal 10 s.

<Warning>
  Referentieafbeeldingen en -video's moeten externe `http(s)`-URL's zijn; de referentiemodi van DashScope weigeren lokale bestandspaden. Upload ze eerst naar objectopslag of gebruik de workflow van de [mediatool](/nl/tools/media-overview), die al een openbare URL produceert.
</Warning>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Overschrijf de basis-URL van DashScope">
    De provider gebruikt standaard het internationale DashScope-eindpunt. Zo gebruik je het eindpunt voor de Chinese regio:

    ```json5
    {
      models: {
        providers: {
          alibaba: {
            baseUrl: "https://dashscope.aliyuncs.com",
          },
        },
      },
    }
    ```

    De provider verwijdert afsluitende schuine strepen voordat AIGC-taak-URL's worden samengesteld.

  </Accordion>

  <Accordion title="Prioriteit van auth-omgevingsvariabelen">
    OpenClaw haalt de Alibaba-API-sleutel in deze volgorde op uit omgevingsvariabelen en gebruikt de eerste niet-lege waarde:

    1. `MODELSTUDIO_API_KEY`
    2. `DASHSCOPE_API_KEY`
    3. `QWEN_API_KEY`

    Geconfigureerde `auth.profiles`-vermeldingen (ingesteld via `openclaw models auth login`) hebben voorrang op het ophalen uit omgevingsvariabelen. Zie [Auth-profielen in de veelgestelde vragen over modellen](/nl/help/faq-models#auth-profiles-what-they-are-and-how-to-manage-them) voor profielrotatie, afkoelperioden en mechanismen voor overschrijving.

  </Accordion>

  <Accordion title="Relatie met de Qwen-plugin">
    Beide gebundelde plugins communiceren met DashScope en accepteren overlappende API-sleutels. Gebruik:

    - `alibaba/wan*.*`-id's voor de speciale Wan-videoprovider die op deze pagina wordt beschreven.
    - `qwen/*`-id's voor Qwen-chat, embeddings en mediabegrip (zie [Qwen](/nl/providers/qwen)).

    Als je `MODELSTUDIO_API_KEY` eenmaal instelt, worden beide plugins geauthenticeerd, omdat de lijsten met auth-omgevingsvariabelen opzettelijk overlappen; afzonderlijke onboarding voor elke plugin is niet vereist.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor de videotool en providerselectie.
  </Card>
  <Card title="Qwen" href="/nl/providers/qwen" icon="microchip">
    Configuratie van Qwen-chat, embeddings en mediabegrip met dezelfde DashScope-authenticatie.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults" icon="gear">
    Standaardinstellingen voor agents en modelconfiguratie.
  </Card>
  <Card title="Veelgestelde vragen over modellen" href="/nl/help/faq-models" icon="circle-question">
    Auth-profielen, wisselen tussen modellen en fouten met "geen profiel" oplossen.
  </Card>
</CardGroup>
