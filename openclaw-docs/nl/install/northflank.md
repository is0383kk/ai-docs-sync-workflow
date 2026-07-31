---
read_when:
    - OpenClaw implementeren op Northflank
    - Je wilt met één klik implementeren in de cloud, met een browsergebaseerde Control UI
summary: Implementeer OpenClaw op Northflank met een sjabloon voor implementatie met één klik
title: Northflank
x-i18n:
    generated_at: "2026-07-27T05:09:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 16bb96fdf470999e15e163b6227d228ce8b60b9a172eb74cadc87bddd3955957
    source_path: install/northflank.mdx
    workflow: 16
---

Implementeer OpenClaw op Northflank met een éénklikssjabloon en krijg er toegang toe via de web-Control UI. Dit is de eenvoudigste manier zonder terminal op de server: Northflank voert de Gateway voor je uit.

## Aan de slag

1. Klik op [OpenClaw implementeren](https://northflank.com/stacks/deploy-openclaw) om het sjabloon te openen.
2. Maak een [account bij Northflank](https://app.northflank.com/signup) als je er nog geen hebt.
3. Klik op **Deploy OpenClaw now**.
4. Stel de vereiste omgevingsvariabele in: `OPENCLAW_GATEWAY_TOKEN` (gebruik een sterke willekeurige waarde).
5. Klik op **Deploy stack** om het OpenClaw-sjabloon te bouwen en uit te voeren.
6. Wacht tot de implementatie is voltooid en klik vervolgens op **View resources**.
7. Open de OpenClaw-service.
8. Open de openbare OpenClaw-URL op `/openclaw` en maak verbinding met het geconfigureerde gedeelde geheim. Dit sjabloon gebruikt standaard `OPENCLAW_GATEWAY_TOKEN`; als je dit vervangt door wachtwoordauthenticatie, gebruik je in plaats daarvan dat wachtwoord.

## Wat je krijgt

- Gehoste OpenClaw Gateway + Control UI
- Permanente opslag via een Northflank-volume (`/data`), zodat `openclaw.json`, `auth-profiles.json` per agent, kanaal-/providerstatus, sessies en de werkruimte behouden blijven bij nieuwe implementaties

## Een kanaal verbinden

Gebruik de Control UI op `/openclaw` of voer `openclaw onboard` via SSH uit voor instructies om kanalen in te stellen:

- [Telegram](/nl/channels/telegram) (het snelst, alleen een bottoken)
- [Discord](/nl/channels/discord)
- [Alle kanalen](/nl/channels)

## Volgende stappen

- Stel berichtenkanalen in: [Kanalen](/nl/channels)
- Configureer de Gateway: [Gateway-configuratie](/nl/gateway/configuration)
- Houd OpenClaw up-to-date: [Bijwerken](/nl/install/updating)
