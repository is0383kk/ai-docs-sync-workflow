---
read_when:
    - Synology Chat instellen met OpenClaw
    - Probleemoplossing voor Synology Chat Webhook-routering
summary: Instellen van de Synology Chat-webhook en OpenClaw-configuratie
title: Synology Chat
x-i18n:
    generated_at: "2026-07-27T05:25:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat maakt verbinding met OpenClaw via een Webhook-paar: een uitgaande Webhook van Synology Chat plaatst inkomende directe berichten bij de Gateway en antwoorden worden teruggestuurd via een inkomende Webhook van Synology Chat.

Status: officiële Plugin, afzonderlijk geïnstalleerd. Alleen directe berichten; tekstberichten en bestandsverzendingen via URL's worden ondersteund.

## Installatie

```bash
openclaw plugins install @openclaw/synology-chat
```

Lokale checkout (bij uitvoering vanuit een git-repo):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

Details: [Plugins](/nl/tools/plugin)

## Snelle installatie

1. Installeer de Plugin (hierboven).
2. In de integraties van Synology Chat:
   - Maak een inkomende Webhook en kopieer de URL ervan.
   - Maak een uitgaande Webhook met je geheime token.
3. Laat de URL van de uitgaande Webhook naar je OpenClaw Gateway verwijzen:
   - `https://gateway-host/webhook/synology` standaard.
   - Of je aangepaste `channels.synology-chat.webhookPath`.
4. Voltooi de installatie in OpenClaw. Synology Chat verschijnt in beide flows in dezelfde lijst voor kanaalinstallatie:
   - Begeleid: `openclaw onboard` of `openclaw channels add`
   - Direct: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Start de Gateway opnieuw en stuur een direct bericht naar de Synology Chat-bot.

Details over Webhook-authenticatie:

- OpenClaw accepteert het token van de uitgaande Webhook uit `body.token`, vervolgens
  `?token=...` en daarna headers.
- Geaccepteerde headerindelingen:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- Lege of ontbrekende tokens worden standaard geweigerd.
- Payloads mogen `application/x-www-form-urlencoded` of `application/json` zijn; `token`, `user_id` en `text` zijn vereist.

## Duurzaamheid van inkomende berichten

Nadat de controles op token, afzenderbeleid en snelheidslimiet zijn geslaagd, verwijdert OpenClaw het Webhook-token uit de opgeslagen envelop en plaatst het de gebeurtenis duurzaam in de wachtrij voordat deze wordt bevestigd. De route retourneert pas `204` nadat het toevoegen is geslaagd; bij een opslagfout wordt `503` geretourneerd, zodat Synology Chat het opnieuw kan proberen in plaats van het bericht ongemerkt te verliezen.

Openstaande gebeurtenissen en gebeurtenissen die opnieuw kunnen worden geprobeerd, blijven behouden na een herstart van de Gateway. De stabiele `post_id` van Synology voorkomt dubbele wachtrij-items zolang de bijbehorende actieve of bewaarde voltooiingsrecord bestaat. De aflevering blijft ten minste één keer plaatsvinden bij de overdracht van de wachtrij naar de agent, waardoor een crash op die grens een beurt nog steeds opnieuw kan afspelen.

Minimale configuratie:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## Omgevingsvariabelen

Voor het standaardaccount kun je omgevingsvariabelen gebruiken:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (door komma's gescheiden)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

Configuratiewaarden overschrijven omgevingsvariabelen.

`SYNOLOGY_CHAT_INCOMING_URL` en `SYNOLOGY_NAS_HOST` kunnen niet worden ingesteld vanuit een `.env` van een werkruimte; zie [`.env`-bestanden van werkruimten](/nl/gateway/security#workspace-env-files).

## Beleid en toegangsbeheer voor directe berichten

- Ondersteunde waarden voor `dmPolicy`: `allowlist` (standaard), `open` en `disabled`. Synology Chat heeft geen koppelingsflow; keur afzenders goed door hun numerieke Synology-gebruikers-ID's toe te voegen aan `allowedUserIds`.
- `allowedUserIds` accepteert een lijst (of een door komma's gescheiden tekenreeks) met Synology-gebruikers-ID's.
- In de modus `allowlist` wordt een lege lijst voor `allowedUserIds` als een onjuiste configuratie beschouwd en wordt de Webhook-route niet gestart.
- `dmPolicy: "open"` staat openbare directe berichten alleen toe wanneer `allowedUserIds` `"*"` bevat; met beperkende items kunnen alleen overeenkomende gebruikers chatten. `open` met een lege lijst voor `allowedUserIds` weigert de route eveneens te starten.
- `dmPolicy: "disabled"` blokkeert directe berichten.
- De koppeling van de ontvanger van antwoorden blijft standaard gebaseerd op de stabiele numerieke `user_id`. `channels.synology-chat.dangerouslyAllowNameMatching: true` is een compatibiliteitsmodus voor noodgevallen die het opzoeken van veranderlijke gebruikersnamen/bijnamen voor het afleveren van antwoorden opnieuw inschakelt.

## Uitgaande aflevering

Gebruik numerieke Synology Chat-gebruikers-ID's als doelen. De voorvoegsels `synology-chat:`, `synology_chat:` en `synology:` worden geaccepteerd.

Voorbeelden:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hallo van OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Nogmaals hallo"
openclaw message send --channel synology-chat --target synology:123456 --message "Kort voorvoegsel"
```

Uitgaande tekst wordt opgesplitst in stukken van 2000 tekens. Mediaverzendingen worden ondersteund via bestandsaflevering op basis van URL's: de NAS downloadt het bestand en voegt het als bijlage toe (maximaal 32 MB). URL's voor uitgaande bestanden moeten `http` of `https` gebruiken, en particuliere of anderszins geblokkeerde netwerkdoelen worden geweigerd voordat OpenClaw de URL doorstuurt naar de NAS-Webhook.

## Meerdere accounts

Meerdere Synology Chat-accounts worden ondersteund onder `channels.synology-chat.accounts`.
Elk account kan het token, de inkomende URL, het Webhook-pad, het beleid voor directe berichten en de limieten overschrijven.
Sessies voor directe berichten zijn per account en gebruiker geïsoleerd, zodat dezelfde numerieke `user_id`
op twee verschillende Synology-accounts geen transcriptstatus deelt.
Geef elk ingeschakeld account een afzonderlijke `webhookPath`. OpenClaw weigert exact dubbele paden
en weigert benoemde accounts te starten die bij een configuratie met meerdere accounts alleen een gedeeld Webhook-pad overnemen.
Als je bewust verouderde overerving voor een benoemd account nodig hebt, stel je
`dangerouslyAllowInheritedWebhookPath: true` in voor dat account of bij `channels.synology-chat`,
maar exact dubbele paden worden nog steeds standaard geweigerd. Geef de voorkeur aan expliciete paden per account.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## Beveiligingsopmerkingen

- Houd `token` geheim en roteer het als het is uitgelekt.
- Behoud `allowInsecureSsl: false`, tenzij je expliciet een zelfondertekend lokaal NAS-certificaat vertrouwt.
- Inkomende Webhook-verzoeken worden per afzender op token geverifieerd en in snelheid beperkt (`rateLimitPerMinute`, standaard 30).
- Controles op ongeldige tokens gebruiken een geheimvergelijking met constante uitvoeringstijd en weigeren standaard; herhaalde pogingen met ongeldige tokens blokkeren het bron-IP tijdelijk.
- Tekst van inkomende berichten wordt opgeschoond tegen bekende patronen voor promptinjectie en afgekapt op 4000 tekens.
- Geef voor productie de voorkeur aan `dmPolicy: "allowlist"`.
- Laat `dangerouslyAllowNameMatching` uitgeschakeld, tenzij je expliciet verouderde aflevering van antwoorden op basis van gebruikersnamen nodig hebt.
- Laat `dangerouslyAllowInheritedWebhookPath` uitgeschakeld, tenzij je expliciet het risico van routering via gedeelde paden in een configuratie met meerdere accounts accepteert.

## Problemen oplossen

- `Missing required fields (token, user_id, text)`:
  - in de payload van de uitgaande Webhook ontbreekt een van de vereiste velden
  - als Synology het token in headers verzendt, zorg er dan voor dat de Gateway/proxy die headers behoudt
- `Invalid token`:
  - het geheim van de uitgaande Webhook komt niet overeen met `channels.synology-chat.token`
  - het verzoek bereikt het verkeerde account/Webhook-pad
  - een reverse proxy heeft de tokenheader verwijderd voordat het verzoek OpenClaw bereikte
- `Rate limit exceeded`:
  - te veel pogingen met ongeldige tokens vanaf dezelfde bron kunnen die bron tijdelijk blokkeren
  - geverifieerde afzenders hebben ook een afzonderlijke snelheidslimiet voor berichten per gebruiker
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` is ingeschakeld, maar er zijn geen gebruikers geconfigureerd
- `User not authorized`:
  - de numerieke `user_id` van de afzender staat niet in `allowedUserIds`

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Groepen](/nl/channels/groups) — gedrag van groepschats en beperking op basis van vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiliging
