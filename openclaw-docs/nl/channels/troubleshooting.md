---
read_when:
    - Kanaaltransport meldt dat er verbinding is, maar antwoorden mislukken
    - Je hebt kanaalspecifieke controles nodig voordat je diepgaande providerdocumentatie raadpleegt
summary: Snelle probleemoplossing op kanaalniveau met foutkenmerken en oplossingen per kanaal
title: Problemen met kanalen oplossen
x-i18n:
    generated_at: "2026-07-27T06:05:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

Gebruik deze pagina wanneer een kanaal verbinding maakt, maar het gedrag niet correct is.

## Commandovolgorde

Voer eerst deze opdrachten in deze volgorde uit:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Gezonde uitgangssituatie:

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`, `write-capable` of `admin-capable`
- Kanaalcontrole toont dat het transport verbonden is en, waar ondersteund, `works` of `audit ok`

## Na een update

Gebruik dit wanneer Telegram, iMessage, configuraties uit het BlueBubbles-tijdperk of een ander Plugin-kanaal verdwijnt
na een update.

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

Zoek naar `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` in `openclaw
status --all`. Dit betekent dat het kanaal is geconfigureerd, maar dat tijdens het instellen of laden van de Plugin een beschadigde
afhankelijkheidsstructuur is aangetroffen in plaats van dat het kanaal is geregistreerd. `openclaw doctor --fix` verwijdert verouderde
symbolische afhankelijkheidskoppelingen van de Plugin-runtime en verouderde authenticatieschaduwen, waarna `openclaw gateway restart` een
schone status opnieuw laadt.

## WhatsApp

### WhatsApp-foutpatronen

| Symptoom                             | Snelste controle                                       | Oplossing                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Verbonden, maar geen antwoorden in DM's         | `openclaw pairing list whatsapp`                    | Keur de afzender goed of wijzig het DM-beleid/de toelatingslijst.                                                                                    |
| Groepsberichten worden genegeerd              | Controleer `requireMention` en vermeldingspatronen in de configuratie | Vermeld de bot of versoepel het vermeldingsbeleid voor die groep.                                                                          |
| QR-aanmelding verloopt met 408         | Controleer Gateway-`HTTPS_PROXY` / `HTTP_PROXY`-omgeving      | Stel een bereikbare proxy in; gebruik `NO_PROXY` alleen voor omzeilingen.                                                                         |
| Willekeurige cycli van verbreken/opnieuw aanmelden     | `openclaw channels status --probe` en logboeken           | Recente nieuwe verbindingen worden gemarkeerd, zelfs wanneer er momenteel verbinding is; bekijk de logboeken, herstart de Gateway en koppel vervolgens opnieuw als de verbinding instabiel blijft. |
| `status=408 Request Time-out`-cyclus  | Controle, logboeken, doctor en vervolgens Gateway-status            | Los eerst problemen met de hostverbinding/timing op; maak een back-up van de authenticatie en koppel het account opnieuw als de cyclus aanhoudt.                                   |
| Antwoorden komen seconden/minuten te laat aan | `openclaw doctor --fix`                             | Doctor stopt geverifieerde verouderde lokale TUI-clients wanneer deze de eventloop van de Gateway verslechteren.                                    |

Volledige probleemoplossing: [Probleemoplossing voor WhatsApp](/nl/channels/whatsapp#troubleshooting)

## Telegram

### Telegram-foutpatronen

| Symptoom                              | Snelste controle                                    | Oplossing                                                                                                                    |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start`, maar geen bruikbare antwoordstroom    | `openclaw pairing list telegram`                 | Keur de koppeling goed of wijzig het DM-beleid.                                                                                   |
| Bot is online, maar de groep blijft stil    | Controleer de vermeldingsvereiste en privacymodus van de bot  | Schakel de privacymodus uit voor zichtbaarheid in de groep of vermeld de bot.                                                              |
| Verzendfouten met netwerkfouten    | Controleer de logboeken op mislukte Telegram-API-aanroepen      | Herstel DNS-/IPv6-/proxyroutering naar `api.telegram.org`.                                                                      |
| Bij het opstarten wordt `getMe returned 401` gemeld | Controleer de geconfigureerde tokenbron                    | Kopieer het BotFather-token opnieuw of genereer het opnieuw en werk `botToken`, `tokenFile` of `TELEGRAM_BOT_TOKEN` van het standaardaccount bij. |
| Pollen loopt vast of maakt langzaam opnieuw verbinding  | `openclaw logs --follow` voor polldiagnostiek | Voer een upgrade uit; aanhoudende blokkeringen wijzen meestal op proxy/DNS/IPv6.                                                            |
| `setMyCommands` wordt bij het opstarten geweigerd  | Controleer de logboeken op `BOT_COMMANDS_TOO_MUCH`         | Verminder het aantal aangepaste Telegram-opdrachten of opdrachten van Plugins/Skills, of schakel native menu's uit.                                                  |
| Na een upgrade word je door de toelatingslijst geblokkeerd    | `openclaw security audit` en toelatingslijsten in de configuratie  | Voer `openclaw doctor --fix` uit of vervang `@username` door numerieke afzender-ID's.                                            |

Volledige probleemoplossing: [Probleemoplossing voor Telegram](/nl/channels/telegram#troubleshooting)

## Discord

### Discord-foutpatronen

| Symptoom                                   | Snelste controle                                                                                                                | Oplossing                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bot is online, maar antwoordt niet in de server           | `openclaw channels status --probe`                                                                                           | Sta de server/het kanaal toe en controleer de intentie voor berichtinhoud.                                                                                                                                                                                                                |
| Groepsberichten worden genegeerd                    | Controleer de logboeken op weigeringen door de vermeldingsvoorwaarde                                                                                          | Vermeld de bot of stel `requireMention: false` voor de server/het kanaal in.                                                                                                                                                                                                             |
| Typen/tokengebruik, maar geen Discord-bericht | Controleer of dit een omgevingsgebeurtenis in een ruimte is of een ruimte met ingeschakelde `message_tool` waarin het model `message(action=send)` heeft gemist | Controleer het uitgebreide Gateway-logboek op metagegevens van onderdrukte definitieve payloads, verifieer `messages.groupChat.unmentionedInbound`, lees [Omgevingsgebeurtenissen in ruimtes](/nl/channels/ambient-room-events) of behoud `messages.groupChat.visibleReplies: "automatic"` voor normale groepsverzoeken. |
| Antwoorden in DM's ontbreken                        | `openclaw pairing list discord`                                                                                              | Keur de DM-koppeling goed of pas het DM-beleid aan.                                                                                                                                                                                                                               |

Volledige probleemoplossing: [Probleemoplossing voor Discord](/nl/channels/discord#troubleshooting)

## Slack

### Slack-foutpatronen

| Symptoom                                | Snelste controle                             | Oplossing                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Socketmodus is verbonden, maar geen antwoorden | `openclaw channels status --probe`        | Controleer het app-token, het bot-token en de vereiste bereiken; let bij configuraties op basis van SecretRef op `botTokenStatus` / `appTokenStatus = configured_unavailable`. |
| DM's worden geblokkeerd                            | `openclaw pairing list slack`             | Keur de koppeling goed of versoepel het DM-beleid.                                                                                                                  |
| Kanaalbericht wordt genegeerd                | Controleer `groupPolicy` en de kanaaltoelatingslijst | Sta het kanaal toe of wijzig het beleid in `open`.                                                                                                        |

Volledige probleemoplossing: [Probleemoplossing voor Slack](/nl/channels/slack#troubleshooting)

## iMessage

### iMessage-foutpatronen

| Symptoom                              | Snelste controle                                           | Oplossing                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| `imsg` ontbreekt of mislukt op andere systemen dan macOS | `openclaw channels status --probe --channel imessage`   | Voer OpenClaw uit op de Mac met Messages of gebruik een SSH-wrapper voor `cliPath`. |
| Kan verzenden, maar niet ontvangen op macOS     | Controleer de macOS-privacyrechten voor automatisering van Messages | Verleen de TCC-rechten opnieuw en herstart het kanaalproces.                 |
| DM-afzender wordt geblokkeerd                    | `openclaw pairing list imessage`                        | Keur de koppeling goed of werk de toelatingslijst bij.                                  |

Volledige probleemoplossing: [Probleemoplossing voor iMessage](/nl/channels/imessage#troubleshooting)

## Signal

### Signal-foutpatronen

| Symptoom                         | Snelste controle                              | Oplossing                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| Daemon is bereikbaar, maar bot blijft stil | `openclaw channels status --probe`         | Controleer de daemon-URL/het account voor `signal-cli` en de ontvangstmodus. |
| DM wordt geblokkeerd                      | `openclaw pairing list signal`             | Keur de afzender goed of pas het DM-beleid aan.                      |
| Groepsantwoorden worden niet geactiveerd    | Controleer de groepstoelatingslijst en vermeldingspatronen | Voeg de afzender/groep toe of versoepel de toegangsbeperking.                       |

Volledige probleemoplossing: [Probleemoplossing voor Signal](/nl/channels/signal#troubleshooting)

## QQ Bot

### QQ Bot-foutpatronen

| Symptoom                         | Snelste controle                               | Oplossing                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| Bot antwoordt "naar Mars vertrokken"      | Controleer `appId` en `clientSecret` in de configuratie | Stel inloggegevens in of herstart de Gateway.                         |
| Geen inkomende berichten             | `openclaw channels status --probe`          | Controleer de inloggegevens op het QQ Open Platform.                     |
| Spraak wordt niet getranscribeerd           | Controleer de configuratie van de STT-provider                   | Configureer `channels.qqbot.stt` of `tools.media.audio`.          |
| Proactieve berichten komen niet aan | Controleer de interactievereisten van het QQ-platform  | QQ kan door de bot geïnitieerde berichten blokkeren wanneer er geen recente interactie is geweest. |

Volledige probleemoplossing: [Probleemoplossing voor QQ Bot](/nl/channels/qqbot#troubleshooting)

## Matrix

### Matrix-foutpatronen

| Symptoom                             | Snelste controle                          | Oplossing                                                                       |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| Aangemeld, maar negeert berichten in ruimtes | `openclaw channels status --probe`     | Controleer `groupPolicy`, de toelatingslijst voor ruimtes en de vermeldingsfilter.                  |
| Privéberichten worden niet verwerkt                  | `openclaw pairing list matrix`         | Keur de afzender goed of pas het beleid voor privéberichten aan.                                       |
| Versleutelde ruimtes werken niet                | `openclaw matrix verify status`        | Verifieer het apparaat opnieuw en controleer vervolgens `openclaw matrix verify backup status`.  |
| Herstel van back-up is in behandeling of werkt niet    | `openclaw matrix verify backup status` | Voer `openclaw matrix verify backup restore` uit of probeer het opnieuw met een herstelsleutel. |
| Cross-signing/bootstrap lijkt onjuist | `openclaw matrix verify bootstrap`     | Herstel de opslag van geheimen, cross-signing en de back-upstatus in één keer.       |

Volledige installatie en configuratie: [Matrix](/nl/channels/matrix)

## Gerelateerd

- [Koppelen](/nl/channels/pairing)
- [Kanaalroutering](/nl/channels/channel-routing)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
