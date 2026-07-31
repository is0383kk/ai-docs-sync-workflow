---
read_when:
    - Je wilt Together AI gebruiken met OpenClaw
    - Je hebt de omgevingsvariabele voor de API-sleutel of de CLI-authenticatiekeuze nodig
summary: Together AI instellen (authenticatie + modelselectie)
title: Together AI
x-i18n:
    generated_at: "2026-07-27T06:07:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9b08cae93c1ea7df46e1d2fbe78692f73bb3e56809122f70a56eec8b3dc5d8a4
    source_path: providers/together.md
    workflow: 16
---

[Together AI](https://together.ai) biedt via één uniforme API toegang tot toonaangevende opensourcemodellen, waaronder Llama, DeepSeek, Kimi en meer.
OpenClaw levert deze mee als de provider `together`.

| Eigenschap | Waarde                        |
| ---------- | ----------------------------- |
| Provider   | `together`            |
| Authenticatie | `TOGETHER_API_KEY`         |
| API        | OpenAI-compatibel             |
| Basis-URL  | `https://api.together.xyz/v1`            |

## Aan de slag

<Steps>
  <Step title="Een API-sleutel verkrijgen">
    Maak een API-sleutel aan op
    [api.together.ai/settings/api-keys](https://api.together.ai/settings/api-keys).
  </Step>
  <Step title="De onboarding uitvoeren">
    ```bash
    openclaw onboard --auth-choice together-api-key
    ```
  </Step>
  <Step title="Een standaardmodel instellen">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "together/meta-llama/Llama-3.3-70B-Instruct-Turbo",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

### Niet-interactief voorbeeld

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

<Note>
Tijdens de onboarding wordt `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` ingesteld als het
standaardmodel.
</Note>

## Ingebouwde catalogus

De kosten zijn in USD per miljoen tokens.

| Modelreferentie                                    | Naam                         | Invoer      | Context | Maximale uitvoer | Kosten (in/uit) | Opmerkingen              |
| -------------------------------------------------- | ---------------------------- | ----------- | ------- | ---------------- | --------------- | ------------------------ |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` | Llama 3.3 70B Instruct Turbo | tekst       | 131,072 | 8,192            | 0.88 / 0.88     | Standaardmodel           |
| `together/moonshotai/Kimi-K2.6`                    | Kimi K2.6 FP4                | tekst, afbeelding | 262,144 | 32,768     | 1.20 / 4.50     | Redeneermodel            |
| `together/deepseek-ai/DeepSeek-V4-Pro`             | DeepSeek V4 Pro              | tekst       | 512,000 | 8,192            | 2.10 / 4.40     | Redeneermodel            |
| `together/Qwen/Qwen2.5-7B-Instruct-Turbo`          | Qwen2.5 7B Instruct Turbo    | tekst       | 32,768  | 8,192            | 0.30 / 0.30     | Snel, niet-redenerend    |
| `together/zai-org/GLM-5.1`                         | GLM 5.1 FP4                  | tekst       | 202,752 | 8,192            | 1.40 / 4.40     | Redeneermodel            |

## Videogeneratie

De meegeleverde Plugin `together` registreert ook videogeneratie via de
gedeelde tool `video_generate`.

| Eigenschap              | Waarde                                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| Standaardvideomodel     | `Wan-AI/Wan2.2-T2V-A14B`                                                                        |
| Andere modellen         | `Wan-AI/Wan2.2-I2V-A14B`, `minimax/hailuo-02`, `kwaivgI/kling-2.1-master`                                |
| Modi                    | tekst-naar-video; afbeelding-naar-video alleen met `Wan-AI/Wan2.2-I2V-A14B` (één referentieafbeelding) |
| Duur                    | 1-10 seconden                                                                             |
| Ondersteunde parameters | `size` (geparseerd als `<width>x<height>`); `aspectRatio`/`resolution` worden niet gelezen |

Together als standaardprovider voor video gebruiken:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

<Tip>
Zie [Videogeneratie](/nl/tools/video-generation) voor de gedeelde toolparameters,
providerselectie en het failovergedrag.
</Tip>

<AccordionGroup>
  <Accordion title="Opmerking over de omgeving">
    Als de Gateway als daemon wordt uitgevoerd (launchd/systemd), zorg er dan voor dat
    `TOGETHER_API_KEY` beschikbaar is voor dat proces (bijvoorbeeld in
    `~/.openclaw/.env` of via `env.shellEnv`).

    <Warning>
    Sleutels die alleen in je interactieve shell zijn ingesteld, zijn niet zichtbaar voor
    gatewayprocessen die door een daemon worden beheerd. Gebruik de configuratie
    `~/.openclaw/.env` of `env.shellEnv` voor permanente beschikbaarheid.
    </Warning>

  </Accordion>

  <Accordion title="Probleemoplossing">
    - Controleer of je sleutel werkt: `openclaw models list --provider together`
    - Als modellen niet verschijnen, controleer dan of de API-sleutel in de juiste
      omgeving voor je Gateway-proces is ingesteld.
    - Modelreferenties gebruiken de vorm `together/<model-id>`.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providerregels, modelreferenties en failovergedrag.
  </Card>
  <Card title="Videogeneratie" href="/nl/tools/video-generation" icon="video">
    Gedeelde toolparameters voor videogeneratie en providerselectie.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledig configuratieschema, inclusief providerinstellingen.
  </Card>
  <Card title="Together AI" href="https://together.ai" icon="arrow-up-right-from-square">
    Together AI-dashboard, API-documentatie en prijzen.
  </Card>
</CardGroup>
