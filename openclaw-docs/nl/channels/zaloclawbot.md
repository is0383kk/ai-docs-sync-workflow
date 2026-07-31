---
read_when:
    - Je wilt een persoonlijke Zalo-assistentbot met inloggen via een QR-code
    - Je installeert de kanaalplugin openclaw-zaloclawbot of lost problemen ermee op
summary: Installatie van het Zalo ClawBot-kanaal via de externe openclaw-zaloclawbot-plugin
title: Zalo ClawBot
x-i18n:
    generated_at: "2026-07-27T05:44:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76c9f79d114856b86026a5e4b98a43f451b0d3f16dd41a67e9226da4f8b37b33
    source_path: channels/zaloclawbot.md
    workflow: 16
---

OpenClaw maakt verbinding met Zalo ClawBot via de externe `@zalo-platforms/openclaw-zaloclawbot`-plugin die in de catalogus staat. Voor het aanmelden wordt een QR-code van een Zalo Mini App gebruikt; de plugin-id in de configuratie is `openclaw-zaloclawbot`.

## Compatibiliteit

| Pluginversie | OpenClaw-versie | npm-dist-tag | Status        |
| ------------- | ---------------- | ------------ | ------------- |
| 0.1.4         | >=2026.4.10      | `latest`     | Actief / Bèta |

## Vereisten

- Node.js >= 22
- [OpenClaw](https://docs.openclaw.ai/install) geïnstalleerd (`openclaw`-CLI beschikbaar)
- Een Zalo-account op een mobiel apparaat om de QR-code voor het aanmelden te scannen

## Installeren met onboard (aanbevolen)

```bash
openclaw onboard
```

Kies **Zalo ClawBot** in het kanaalmenu. De wizard installeert de plugin vanuit de officiële catalogus (met integriteitsverificatie), geeft de QR-code voor het aanmelden weer in de terminal en voltooit de configuratie van het kanaal zodra je deze met de Zalo-app scant.

## Handmatige installatie

Zo voeg je het kanaal toe aan een Gateway waarvoor onboard al is uitgevoerd:

### 1. Installeer de plugin

```bash
openclaw plugins install "@zalo-platforms/openclaw-zaloclawbot@0.1.4"
```

Gebruik exact de vastgezette versie, zodat OpenClaw het pakket tijdens de installatie kan verifiëren aan de hand van de integriteitshash in de catalogus.

### 2. Schakel de plugin in de configuratie in

```bash
openclaw config set plugins.entries.openclaw-zaloclawbot.enabled true
```

### 3. Genereer een QR-code en meld je aan

```bash
openclaw channels login --channel openclaw-zaloclawbot
```

Scan de in de terminal weergegeven QR-code met de mobiele Zalo-app, accepteer de gebruiksvoorwaarden in de Zalo Mini App en autoriseer de sessie.

### 4. Start de Gateway opnieuw

```bash
openclaw gateway restart
```

## Hoe het werkt

Anders dan bij het standaard Zalo-kanaal, waarvoor je jouw eigen Zalo Official Account (OA) moet registreren en statische ontwikkelaarsreferenties moet configureren, is Zalo ClawBot een **persoonsgebonden persoonlijke assistent** op gedeelde officiële infrastructuur:

1. **Onboarding:** de QR-code verwijst naar een Zalo Mini App die een nieuw aangemaakte privébot onder een gedeeld officieel OA rechtstreeks aan jouw Zalo-gebruikers-ID koppelt.
2. **Persoonsgebonden privacy:** de bot communiceert alleen met de eigenaar. Berichten van andere gebruikers worden op platformniveau geweigerd.
3. **Officiële API-route:** de plugin gebruikt API's van het Zalo Bot Platform en geen automatisering via een browser of websessie.

## Onder de motorkap

De plugin communiceert met Zalo via een permanente long-pollinglus (`getUpdates`). Webhooks zijn standaard uitgeschakeld wanneer de Gateway lokaal op een desktop of in een terminal wordt uitgevoerd. Berichten worden aan de clientzijde verwerkt en aan de runtime van je lokale agent gekoppeld.

De plugin beheert botreferenties in de OpenClaw-statusmap. Behandel die map als gevoelig en pas er hetzelfde toegangsbeheer- en back-upbeleid op toe als op de rest van de OpenClaw-status.

De runtime van deze plugin bevindt zich volledig in het externe `@zalo-platforms/openclaw-zaloclawbot`-pakket; de onderstaande gedragsdetails die verder gaan dan installatie en configuratie zijn afkomstig van de beheerders van de plugin en zijn niet geverifieerd aan de hand van de broncode van OpenClaw-core.

## Problemen oplossen

- **Time-out bij aanmelden via QR-code:** het aanmeldingstoken (`zbsk`) verloopt om veiligheidsredenen na 5 minuten. Als de QR-code verloopt voordat je deze scant, voer je de aanmeldingsopdracht opnieuw uit om een nieuwe te genereren.
- **Gateway kan niet worden geladen:** controleer of de versie van je OpenClaw-host `2026.4.10` of hoger is. Oudere versies ondersteunen het register voor installaties van externe npm-plugins dat voor deze ID vereist is niet.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Zalo](/nl/channels/zalo) - het gebundelde kanaal voor Zalo Bot Creator / Marketplace
- [Koppelen](/nl/channels/pairing) - DM-authenticatie en koppelingsflow
- [Plugins](/nl/tools/plugin) - plugins installeren en beheren
