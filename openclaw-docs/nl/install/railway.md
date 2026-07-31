---
read_when:
    - OpenClaw implementeren op Railway
    - Je wilt met één klik in de cloud implementeren met een browsergebaseerde Control UI
summary: Implementeer OpenClaw op Railway met een éénklikssjabloon
title: Railway
x-i18n:
    generated_at: "2026-07-27T05:51:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbef00b8de61545e9971b18164472c2f47fe607f69ec36f83a27a11b65ea863f
    source_path: install/railway.mdx
    workflow: 16
---

Implementeer OpenClaw op Railway met een sjabloon voor implementatie met één klik en open het via de webgebaseerde Control UI. Dit is de eenvoudigste route „zonder terminal op de server”: Railway voert de Gateway voor je uit.

## Implementatie met één klik

<a href="https://railway.com/deploy/clawdbot-railway-template" target="_blank" rel="noreferrer">
  Deploy on Railway
</a>

<Steps>
  <Step title="Het sjabloon implementeren">
    Klik hierboven op **Deploy on Railway**.
  </Step>

<Step title="Een volume toevoegen">
  Koppel een volume dat is gekoppeld op `/data` (vereist voor permanente status).
</Step>

  <Step title="Variabelen instellen">
    Stel de vereiste **Variables** in voor de service:

    - `OPENCLAW_GATEWAY_PORT=8080` (vereist -- moet overeenkomen met de poort in Public Networking)
    - `OPENCLAW_GATEWAY_TOKEN` (vereist; behandel dit als een beheerdersgeheim)
    - `OPENCLAW_STATE_DIR=/data/.openclaw` (aanbevolen)
    - `OPENCLAW_WORKSPACE_DIR=/data/workspace` (aanbevolen)

  </Step>

<Step title="Openbare netwerktoegang inschakelen">
  Schakel onder **Public Networking** **HTTP Proxy** in voor de service op poort `8080`.
</Step>

  <Step title="Verbinding maken">
    Zoek je openbare URL onder **Railway -> your service -> Settings -> Domains** -- dit is een gegenereerd domein (vaak `https://<something>.up.railway.app`) of je gekoppelde aangepaste domein.

    Open `https://<your-railway-domain>/openclaw` en maak verbinding met het geconfigureerde gedeelde geheim. Het sjabloon gebruikt standaard `OPENCLAW_GATEWAY_TOKEN`; als je dit vervangt door wachtwoordauthenticatie, gebruik je in plaats daarvan dat wachtwoord.

  </Step>
</Steps>

## Wat je krijgt

- Gehoste OpenClaw Gateway + Control UI
- Permanente opslag via het Railway-volume (`/data`), zodat `openclaw.json`, `auth-profiles.json` per agent, kanaal-/providerstatus, sessies en de werkruimte behouden blijven bij nieuwe implementaties

## Een kanaal verbinden

Gebruik de Control UI op `/openclaw` of voer `openclaw onboard` uit via de shell van Railway voor instructies voor het instellen van kanalen:

- [Discord](/nl/channels/discord)
- [Telegram](/nl/channels/telegram) (het snelst -- alleen een bottoken)
- [Alle kanalen](/nl/channels)

## Back-ups en migratie

Exporteer je status, configuratie, authenticatieprofielen en werkruimte:

```bash
openclaw backup create
```

Hiermee wordt een draagbaar back-uparchief gemaakt met de OpenClaw-status en elke geconfigureerde werkruimte. Zie [Back-up](/nl/cli/backup) voor meer informatie.

## Volgende stappen

- Stel berichtkanalen in: [Kanalen](/nl/channels)
- Configureer de Gateway: [Gatewayconfiguratie](/nl/gateway/configuration)
- Houd OpenClaw up-to-date: [Bijwerken](/nl/install/updating)
