---
read_when:
    - Je wilt Qwen met OpenClaw gebruiken
    - Je hebt een Alibaba Cloud Token Plan-abonnement
summary: Gebruik Qwen Cloud via de bijbehorende OpenClaw-plugin
title: Qwen
x-i18n:
    generated_at: "2026-07-27T06:07:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74f94a35631dcdf8c9afc12e86d7a9d6b51a359411ba36f8820f8b1e7c03a27a
    source_path: providers/qwen.md
    workflow: 16
---

Qwen Cloud is een officiële externe providerplugin voor OpenClaw met de canonieke id `qwen`. De plugin is gericht op de Standard- en Coding Plan-eindpunten van Qwen Cloud / Alibaba DashScope, stelt Token Plan beschikbaar als `qwen-token-plan`, behoudt `modelstudio` als compatibiliteitsalias en beheert onafhankelijk de door Alibaba gedocumenteerde aangepaste provider-id `bailian-token-plan`.

| Eigenschap                    | Waarde                                     |
| ----------------------------- | ------------------------------------------ |
| Provider                      | `qwen`                         |
| Token Plan-provider           | `qwen-token-plan`                         |
| Voorkeursomgevingsvariabele   | `QWEN_API_KEY`                         |
| Token Plan-omgevingsvariabele | `QWEN_TOKEN_PLAN_API_KEY`                         |
| Ook geaccepteerd (compatibel) | `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`     |
| API-stijl                     | OpenAI-compatibel                          |

<Tip>
`qwen3.7-plus` en `qwen3.6-plus` werken met Coding Plan- en Standard-eindpunten.
Gebruik voor `qwen3.7-max` of `qwen3.6-flash` een **Standard-eindpunt (betalen naar gebruik)**.
</Tip>

## Plugin installeren

`qwen` wordt geleverd als een officiële externe plugin en is niet gebundeld met de kern. Installeer de plugin en start de Gateway opnieuw:

```bash
openclaw plugins install @openclaw/qwen-provider
openclaw gateway restart
```

## Aan de slag

Kies je plantype en volg de configuratiestappen.

<Tabs>
  <Tab title="Coding Plan (abonnement)">
    **Meest geschikt voor:** toegang op abonnementsbasis via het Qwen Coding Plan.

    <Steps>
      <Step title="Je API-sleutel ophalen">
        Maak of kopieer een API-sleutel op [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys).
      </Step>
      <Step title="Onboarding uitvoeren">
        Voor het **Global**-eindpunt:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        Voor het **China**-eindpunt:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```
      </Step>
      <Step title="Een standaardmodel instellen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="Controleren of het model beschikbaar is">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    Verouderde `modelstudio-*`-auth-choice-id's en `modelstudio/...`-modelverwijzingen werken nog
    als compatibiliteitsaliassen, maar nieuwe configuratieflows moeten bij voorkeur de canonieke
    `qwen-*`-auth-choice-id's en `qwen/...`-modelverwijzingen gebruiken. Als je een exacte
    aangepaste `models.providers.modelstudio`-vermelding met een andere `api`-waarde definieert, beheert die
    aangepaste provider `modelstudio/...`-verwijzingen in plaats van de Qwen-compatibiliteitsalias.
    </Note>

  </Tab>

  <Tab title="Standard (betalen naar gebruik)">
    **Meest geschikt voor:** toegang met betaling naar gebruik via het Standard Model Studio-eindpunt, waaronder `qwen3.7-max` en `qwen3.6-flash`, die niet beschikbaar zijn in het Coding Plan.

    <Steps>
      <Step title="Je API-sleutel ophalen">
        Maak of kopieer een API-sleutel op [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys).
      </Step>
      <Step title="Onboarding uitvoeren">
        Voor het **Global**-eindpunt:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        Voor het **China**-eindpunt:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```
      </Step>
      <Step title="Een standaardmodel instellen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="Controleren of het model beschikbaar is">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    Verouderde `modelstudio-*`-auth-choice-id's en `modelstudio/...`-modelverwijzingen werken nog
    als compatibiliteitsaliassen, maar nieuwe configuratieflows moeten bij voorkeur de canonieke
    `qwen-*`-auth-choice-id's en `qwen/...`-modelverwijzingen gebruiken. Als je een exacte
    aangepaste `models.providers.modelstudio`-vermelding met een andere `api`-waarde definieert, beheert die
    aangepaste provider `modelstudio/...`-verwijzingen in plaats van de Qwen-compatibiliteitsalias.
    </Note>

  </Tab>

  <Tab title="Token Plan (Team Edition)">
    **Meest geschikt voor:** toegang voor teams met een abonnement op basis van tegoed tot Qwen en ondersteunde modellen van derden via Alibaba Cloud Model Studio.

    <Steps>
      <Step title="Je toegewezen sleutel ophalen">
        Wijs een Token Plan-licentie toe en maak de bijbehorende toegewezen `sk-sp-...`-sleutel. Sleutels voor Token Plan, Coding Plan en betalen naar gebruik zijn niet onderling uitwisselbaar. Zie het [overzicht van het Global Token Plan](https://www.alibabacloud.com/help/en/model-studio/token-plan-overview) of het [overzicht van het China Token Plan](https://help.aliyun.com/zh/model-studio/token-plan-overview).
      </Step>
      <Step title="Onboarding uitvoeren">
        Voor het **Global / International**-eindpunt in Singapore:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan
        ```

        Voor het **China**-eindpunt in Beijing:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan-cn
        ```
      </Step>
      <Step title="De provider controleren">
        ```bash
        openclaw models list --provider qwen-token-plan
        openclaw agent --model qwen-token-plan/qwen3.7-plus --message "Antwoord met: tokenplan gereed"
        ```
      </Step>
    </Steps>

    <Note>
    De OpenClaw-handleiding van Alibaba gebruikt `bailian-token-plan` voor een handmatige aangepaste
    provider. De plugin registreert die id als compatibiliteitseigenaar, maar nieuwe
    configuraties moeten `qwen-token-plan` gebruiken. Een exacte aangepaste
    `models.providers.bailian-token-plan`-vermelding behoudt het beheer over het geconfigureerde
    transport en de catalogus; deze wordt nooit samengevoegd met de canonieke OpenAI-catalogus.
    </Note>

    <Warning>
    Gebruik Token Plan alleen voor interactieve OpenClaw-sessies. Selecteer het niet voor
    Cron-taken, onbeheerde scripts of applicatiebackends. Alibaba vermeldt dat
    niet-interactief gebruik het abonnement kan opschorten of de API-sleutel ervan kan intrekken.
    </Warning>

  </Tab>

</Tabs>

## Plantypen en eindpunten

| Plan                       | Regio  | Auth-keuze                  | Eindpunt                                                         |
| -------------------------- | ------ | --------------------------- | ---------------------------------------------------------------- |
| Coding Plan (abonnement)   | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`                                               |
| Coding Plan (abonnement)   | Global | `qwen-api-key`          | `coding-intl.dashscope.aliyuncs.com/v1`                                               |
| Standard (betalen naar gebruik) | China  | `qwen-standard-api-key-cn`     | `dashscope.aliyuncs.com/compatible-mode/v1`                                               |
| Standard (betalen naar gebruik) | Global | `qwen-standard-api-key`     | `dashscope-intl.aliyuncs.com/compatible-mode/v1`                                               |
| Token Plan (Team Edition)  | China  | `qwen-token-plan-cn`          | `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`                                               |
| Token Plan (Team Edition)  | Global | `qwen-token-plan`          | `token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`                                               |

De provider selecteert het eindpunt automatisch op basis van je auth-keuze. Canonieke
keuzes gebruiken de `qwen-*`-familie; `modelstudio-*` blijft uitsluitend voor compatibiliteit.
Overschrijf dit met een aangepaste `baseUrl` in de configuratie.

<Tip>
**Sleutels beheren:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**Documentatie:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)
</Tip>

## Ingebouwde catalogus

OpenClaw wordt geleverd met deze statische Qwen-catalogus. De catalogus houdt rekening met het eindpunt: Coding
Plan-configuraties laten modellen weg die alleen met het Standard-eindpunt werken.

| Modelverwijzing             | Invoer      | Context   | Opmerkingen                 |
| --------------------------- | ----------- | --------- | --------------------------- |
| `qwen/qwen3.5-plus`          | tekst, afbeelding | 1,000,000 | Standaardmodel          |
| `qwen/qwen3.6-flash`          | tekst, afbeelding | 1,000,000 | Alleen Standard-eindpunten |
| `qwen/qwen3.6-plus`          | tekst, afbeelding | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3.7-max`          | tekst       | 1,000,000 | Alleen Standard-eindpunten  |
| `qwen/qwen3.7-plus`          | tekst, afbeelding | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3-max-2026-01-23`          | tekst       | 262,144   | Qwen Max-reeks              |
| `qwen/qwen3-coder-next`          | tekst       | 262,144   | Codering                    |
| `qwen/qwen3-coder-plus`          | tekst       | 1,000,000 | Codering                    |
| `qwen/MiniMax-M2.5`          | tekst       | 1,000,000 | Redeneren ingeschakeld      |
| `qwen/glm-5`          | tekst       | 202,752   | GLM                         |
| `qwen/glm-4.7`          | tekst       | 202,752   | GLM                         |
| `qwen/kimi-k2.5`          | tekst, afbeelding | 262,144 | Moonshot AI via Alibaba |

<Note>
De beschikbaarheid kan nog steeds per eindpunt en factureringsplan verschillen, zelfs wanneer een model
in de statische catalogus staat.
</Note>

### Token Plan-catalogus

Token Plan gebruikt een afzonderlijke acceptatielijst met exacte tekenreeksen. Planmodellen die
uitsluitend afbeeldingen genereren, zijn hier niet opgenomen omdat ze andere API's gebruiken.

| Modelverwijzing                     | Invoer            | Context   |
| ----------------------------------- | ----------------- | --------- |
| `qwen-token-plan/qwen3.7-max`                  | tekst             | 1,000,000 |
| `qwen-token-plan/qwen3.7-plus`                  | tekst, afbeelding | 1,000,000 |
| `qwen-token-plan/qwen3.6-plus`                  | tekst, afbeelding | 1,000,000 |
| `qwen-token-plan/qwen3.6-flash`                  | tekst, afbeelding | 1,000,000 |
| `qwen-token-plan/deepseek-v4-pro`                  | tekst             | 1,000,000 |
| `qwen-token-plan/deepseek-v4-flash`                  | tekst             | 1,000,000 |
| `qwen-token-plan/deepseek-v3.2`                  | tekst             | 131,072   |
| `qwen-token-plan/kimi-k2.7-code`                  | tekst, afbeelding | 262,144   |
| `qwen-token-plan/kimi-k2.6`                  | tekst, afbeelding | 262,144   |
| `qwen-token-plan/kimi-k2.5`                  | tekst, afbeelding | 262,144   |
| `qwen-token-plan/glm-5.2`                  | tekst             | 1,000,000 |
| `qwen-token-plan/glm-5.1`                  | tekst             | 202,752   |
| `qwen-token-plan/glm-5`                  | tekst             | 202,752   |
| `qwen-token-plan/MiniMax-M2.5`                  | tekst             | 196,608   |

## Denkbesturing

`qwen3.7-max`, `qwen3.7-plus`, `qwen3.6-flash` en `qwen3.6-plus` zijn
geschikt voor redeneren in de ingebouwde catalogus. Voor redeneermodellen in de `qwen`-
familie koppelt de provider de denkniveaus van OpenClaw aan de `enable_thinking`-aanvraagvlag
op het hoogste niveau van DashScope: bij uitgeschakeld denken wordt `enable_thinking: false` verzonden,
bij elk ander niveau wordt `enable_thinking: true` verzonden. Aangepaste modellen kunnen een
alternatieve denkpayload voor chatsjablonen gebruiken door
`compat.thinkingFormat: "qwen-chat-template"` in te stellen bij de modelvermelding.

Token Plan-modellen zijn eveneens gemarkeerd als geschikt voor redeneren. `kimi-k2.7-code` en
`MiniMax-M2.5` ondersteunen uitsluitend denken, dus OpenClaw houdt denken ingeschakeld, zelfs wanneer
de sessie om `/think off` vraagt. DeepSeek V4 koppelt `minimal` tot en met `high` aan
de `high`-inspanning van de service en koppelt `xhigh` of `max` aan `max`. GLM 5.2 accepteert
het volledige bereik van `minimal` tot en met `max`; GLM 5.1 en GLM 5 accepteren tot en met
`xhigh`, en alle drie gebruiken standaard `high`. Andere hybride modellen volgen de
aangevraagde aan/uit-status.

## Multimodale uitbreidingen

De plugin `qwen` stelt multimodale mogelijkheden uitsluitend beschikbaar op de **Standard**-eindpunten
van DashScope, niet op de Coding Plan-eindpunten:

- **Begrip van afbeeldingen en video's** via `qwen3.6-plus`
- **Wan-videogeneratie** via `wan2.6-t2v` (standaard), `wan2.6-i2v`, `wan2.6-r2v`, `wan2.6-r2v-flash`, `wan2.7-r2v`

Mediabegrip wordt automatisch afgeleid uit de geconfigureerde Qwen-authenticatie; er is geen extra
configuratie nodig. Zorg dat je een Standard-eindpunt (betalen naar gebruik) gebruikt om
mediabegrip te laten werken.

Qwen instellen als de standaardvideoprovider:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

Limieten voor videogeneratie: 1 uitvoervideo per aanvraag, maximaal 1 invoerafbeelding
(afbeelding-naar-video), maximaal 4 invoervideo's (video-naar-video), maximale duur van 10 seconden.
Ondersteunt `size`, `aspectRatio`, `resolution`, `audio` en
`watermark`. Invoer van referentieafbeeldingen/-video's vereist externe http(s)-URL's; lokale
bestandspaden worden vooraf geweigerd, omdat het DashScope-video-eindpunt geen geüploade lokale buffers
voor deze referenties accepteert.

<Note>
Zie [Videogeneratie](/nl/tools/video-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Beschikbaarheid van Qwen 3.6 en 3.7">
    `qwen3.7-plus` en `qwen3.6-plus` zijn beschikbaar via Coding Plan- en Standard-eindpunten. `qwen3.7-max` en `qwen3.6-flash` zijn alleen beschikbaar via Standard. De Standard-eindpunten (betalen naar gebruik) zijn:

    - China: `dashscope.aliyuncs.com/compatible-mode/v1`
    - Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    OpenClaw laat `qwen3.7-max` en `qwen3.6-flash` weg uit Coding Plan-catalogi.
    Als een Coding Plan-eindpunt voor een van beide de foutmelding "unsupported model" retourneert,
    schakel je over naar het bijbehorende Standard-eindpunt en de bijbehorende sleutel.

  </Accordion>

  <Accordion title="Regionale routering voor videogeneratie">
    OpenClaw koppelt de geconfigureerde Qwen-regio aan de bijbehorende DashScope AIGC-host
    voordat een videotaak wordt ingediend:

    - Global/Intl: `https://dashscope-intl.aliyuncs.com`
    - China: `https://dashscope.aliyuncs.com`

    Een normale `models.providers.qwen.baseUrl` die naar de Coding Plan-
    of Standard Qwen-hosts verwijst, routeert videogeneratie nog steeds naar het bijbehorende
    regionale DashScope-video-eindpunt.

  </Accordion>

  <Accordion title="Compatibiliteit met streaminggebruik">
    Native Qwen-eindpunten geven compatibiliteit met streaminggebruik aan via het gedeelde
    `openai-completions`-transport, zodat aangepaste, met DashScope compatibele provider-id's
    die op dezelfde native hosts zijn gericht, hetzelfde gedrag overnemen zonder dat specifiek
    de ingebouwde provider-id `qwen` vereist is. Dit geldt voor Coding Plan-,
    Standard- en Token Plan-eindpunten:

    - `https://coding.dashscope.aliyuncs.com/v1`
    - `https://coding-intl.dashscope.aliyuncs.com/v1`
    - `https://dashscope.aliyuncs.com/compatible-mode/v1`
    - `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

  </Accordion>

  <Accordion title="Plan voor mogelijkheden">
    De Plugin `qwen` wordt gepositioneerd als de centrale locatie van de leverancier voor het volledige Qwen
    Cloud-aanbod, en niet alleen voor codeer-/tekstmodellen.

    - **Tekst-/chatmodellen:** beschikbaar via de Plugin
    - **Toolaanroepen, gestructureerde uitvoer, denkproces:** overgenomen van het OpenAI-compatibele transport
    - **Afbeeldingsgeneratie:** gepland op de provider-Plugin-laag
    - **Begrip van afbeeldingen/video's:** beschikbaar via de Plugin op het Standard-eindpunt
    - **Spraak/audio:** gepland op de provider-Plugin-laag
    - **Geheugenembeddings/herrangschikking:** gepland via het oppervlak van de embeddingadapter
    - **Videogeneratie:** beschikbaar via de Plugin door middel van de gedeelde mogelijkheid voor videogeneratie

  </Accordion>

  <Accordion title="Omgevings- en daemonconfiguratie">
    Als de Gateway als daemon (launchd/systemd) wordt uitgevoerd, zorg je ervoor dat `QWEN_API_KEY`
    of `QWEN_TOKEN_PLAN_API_KEY` beschikbaar is voor dat proces (bijvoorbeeld in
    `~/.openclaw/.env` of via `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor de videotool en providerselectie.
  </Card>
  <Card title="Alibaba Model Studio" href="/nl/providers/alibaba" icon="cloud">
    Meegeleverde provider voor Wan-videogeneratie op hetzelfde DashScope-platform.
  </Card>
  <Card title="Probleemoplossing" href="/nl/help/troubleshooting" icon="wrench">
    Algemene probleemoplossing en veelgestelde vragen.
  </Card>
</CardGroup>
