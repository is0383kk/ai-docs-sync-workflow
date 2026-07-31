---
read_when:
    - Je wilt Cloudflare AI Gateway gebruiken met OpenClaw
    - Je hebt de account-ID, gateway-ID of omgevingsvariabele voor de API-sleutel nodig
summary: Cloudflare AI Gateway instellen (authenticatie + modelselectie)
title: Cloudflare AI-gateway
x-i18n:
    generated_at: "2026-07-27T06:30:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 02c7785616e7aee645bb3fc41ef6a3585e1f2f9d886fab1a06231e497effd045
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 16
---

[Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) bevindt zich vóór de provider-API's en voegt analyses, caching en beheermogelijkheden toe. Voor Anthropic gebruikt OpenClaw de Anthropic Messages API via je Gateway-eindpunt.

| Eigenschap     | Waarde                                                                                    |
| -------------- | ----------------------------------------------------------------------------------------- |
| Provider       | `cloudflare-ai-gateway`                                                                        |
| Plugin         | officieel extern pakket (`@openclaw/cloudflare-ai-gateway-provider`)                                               |
| Basis-URL      | `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`                                                                        |
| Standaardmodel | `cloudflare-ai-gateway/claude-sonnet-4-6`                                                                        |
| API-sleutel    | `CLOUDFLARE_AI_GATEWAY_API_KEY` (je provider-API-sleutel voor aanvragen via de Gateway)                 |

<Note>
Gebruik voor Anthropic-modellen die via Cloudflare AI Gateway worden gerouteerd je **Anthropic API-sleutel** als providersleutel.
</Note>

Wanneer thinking is ingeschakeld voor Anthropic Messages-modellen, verwijdert OpenClaw afsluitende
assistant-prefillbeurten voordat de payload via Cloudflare AI Gateway wordt verzonden.
Anthropic weigert het vooraf invullen van antwoorden bij extended thinking, terwijl gewone
prefill zonder thinking beschikbaar blijft.

## Plugin installeren

Installeer de officiële Plugin en start daarna de Gateway opnieuw:

```bash
openclaw plugins install @openclaw/cloudflare-ai-gateway-provider
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="De provider-API-sleutel en Gateway-gegevens instellen">
    Voer de onboarding uit en kies de authenticatieoptie voor Cloudflare AI Gateway:

    ```bash
    openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
    ```

    Je wordt gevraagd om je account-ID, gateway-ID en API-sleutel.

  </Step>
  <Step title="Een standaardmodel instellen">
    Voeg het model toe aan je OpenClaw-configuratie:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-6" },
        },
      },
    }
    ```

  </Step>
  <Step title="Controleren of het model beschikbaar is">
    ```bash
    openclaw models list --provider cloudflare-ai-gateway
    ```
  </Step>
</Steps>

## Niet-interactief voorbeeld

Geef voor gescripte configuraties of CI-configuraties alle waarden op via de opdrachtregel:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Geverifieerde gateways">
    Als je Gateway-authenticatie in Cloudflare hebt ingeschakeld, voeg je de header `cf-aig-authorization` toe. Dit komt **naast** je provider-API-sleutel.

    ```json5
    {
      models: {
        providers: {
          "cloudflare-ai-gateway": {
            headers: {
              "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
            },
          },
        },
      },
    }
    ```

    <Tip>
    De header `cf-aig-authorization` verifieert je bij de Cloudflare Gateway zelf, terwijl de provider-API-sleutel (bijvoorbeeld je Anthropic-sleutel) je bij de upstreamprovider verifieert.
    </Tip>

  </Accordion>

  <Accordion title="Opmerking over de omgeving">
    Als de Gateway als daemon (launchd/systemd) wordt uitgevoerd, moet `CLOUDFLARE_AI_GATEWAY_API_KEY` beschikbaar zijn voor dat proces.

    <Warning>
    Een sleutel die alleen in een interactieve shell is geëxporteerd, helpt een launchd/systemd-daemon niet, tenzij die omgeving daar ook wordt geïmporteerd. Stel de sleutel in via `~/.openclaw/.env` of `env.shellEnv`, zodat het gatewayproces deze kan lezen.
    </Warning>

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Algemene probleemoplossing en veelgestelde vragen.
  </Card>
</CardGroup>
