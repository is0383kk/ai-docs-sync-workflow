---
read_when:
    - Je wilt OpenClaw met LINE verbinden
    - Je moet de LINE-webhook en aanmeldgegevens instellen
    - Je wilt LINE-specifieke berichtopties
summary: Installatie, configuratie en gebruik van de LINE Messaging API-plugin
title: LINE
x-i18n:
    generated_at: "2026-07-27T06:03:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa160970278e0899637307136139f7d2fc83bf57defc30771d77649060f77274
    source_path: channels/line.md
    workflow: 16
---

LINE maakt via de LINE Messaging API verbinding met OpenClaw. De plugin draait als Webhook-
ontvanger op de Gateway en gebruikt je kanaaltoegangstoken + kanaalgeheim voor
authenticatie.

Status: officiële plugin, afzonderlijk geïnstalleerd. Directe berichten, groepschats, media,
locaties, Flex-berichten, sjabloonberichten en snelle antwoorden worden ondersteund.
Reacties en threads worden niet ondersteund.

## Installeren

Installeer LINE voordat je het kanaal configureert:

```bash
openclaw plugins install @openclaw/line
```

Lokale checkout (bij uitvoering vanuit een git-repo):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## Instellen

1. Maak een LINE Developers-account en open de Console:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. Maak (of kies) een Provider en voeg een **Messaging API**-kanaal toe.
3. Kopieer de **Channel access token** en **Channel secret** uit de kanaalinstellingen.
4. Schakel **Use webhook** in de Messaging API-instellingen in.
5. Stel de Webhook-URL in op je Gateway-eindpunt (HTTPS vereist):

```text
https://gateway-host/line/webhook
```

De Gateway beantwoordt de Webhook-verificatie van LINE (GET). Voor ondertekende inkomende gebeurtenissen
(POST) schrijft deze elke gebeurtenis naar de duurzame wachtrij voor inkomend verkeer voordat `200` wordt geretourneerd;
de agentverwerking gaat asynchroon verder. Een mislukte bezorging wordt opnieuw geprobeerd vanuit de
wachtrij, ook na een herstart van de Gateway, en problematische gebeurtenissen worden na een begrensd aantal
pogingen als mislukte wachtrijrecords gemarkeerd. Als duurzame opslag mislukt, retourneert het verzoek
`500` in plaats van een gebeurtenis te bevestigen die verloren zou kunnen gaan.
Bezorging vindt minstens één keer plaats over de grens tussen wachtrij en agent: als de Gateway wordt afgesloten of
crasht tijdens een actieve bezorging, kan de beurt opnieuw worden afgespeeld. Berichtgebeurtenissen worden ontdubbeld op
LINE-bericht-ID; andere gebeurtenistypen gebruiken `webhookEventId`. Bewaarde voltooiingsrecords
onderdrukken gewone dubbele Webhooks, maar handlers die externe neveneffecten uitvoeren
moeten nog steeds idempotent zijn.
Als je een aangepast pad nodig hebt, stel je `channels.line.webhookPath` of
`channels.line.accounts.<id>.webhookPath` in en werk je de URL overeenkomstig bij.

Beveiligingsopmerkingen:

- De handtekeningverificatie van LINE is afhankelijk van de body (HMAC over de onbewerkte body), dus OpenClaw past vóór authenticatie een strikte limiet voor de body (64 KB) en een leestime-out toe.
- OpenClaw verwerkt Webhook-gebeurtenissen uit de geverifieerde onbewerkte bytes van het verzoek. Door upstreammiddleware getransformeerde `req.body`-waarden worden genegeerd om de integriteit van de handtekening te beschermen.

## Configureren

Minimale configuratie:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

Configuratie voor openbare directe berichten:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "open",
      allowFrom: ["*"],
    },
  },
}
```

Omgevingsvariabelen (alleen standaardaccount):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

Token-/geheimbestanden:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` en `secretFile` moeten naar gewone bestanden verwijzen. Symbolische koppelingen worden geweigerd.
Inline configuratiewaarden hebben voorrang op bestanden; omgevingsvariabelen zijn de laatste terugvaloptie voor het standaardaccount.

Meerdere accounts:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## Toegangsbeheer

Directe berichten gebruiken standaard koppeling. Onbekende afzenders krijgen een koppelingscode en hun
berichten worden genegeerd totdat ze zijn goedgekeurd:

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

Toestaanlijsten en beleidsregels:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled` (standaard `pairing`)
- `channels.line.allowFrom`: toegestane LINE-gebruikers-ID's voor directe berichten; `dmPolicy: "open"` vereist `["*"]`
- `channels.line.groupPolicy`: `allowlist | open | disabled` (standaard `allowlist`)
- `channels.line.groupAllowFrom`: toegestane LINE-gebruikers-ID's voor groepen; vermeldingen voor directe berichten in `allowFrom` laten geen groepsafzenders toe
- Overschrijvingen per groep: `channels.line.groups.<groupId>.allowFrom` (plus `enabled`, `requireMention`, `systemPrompt`, `skills`). Stel bij
  `groupPolicy: "allowlist"` `groupAllowFrom` of de groepsspecifieke `allowFrom` in; een lege groepstoestaanlijst blokkeert groepsberichten, zelfs wanneer directe berichten openstaan.
- Er kan vanuit `allowFrom`, `groupAllowFrom` en de groepsspecifieke `allowFrom` met `accessGroup:<name>` naar statische toegangsgroepen voor afzenders worden verwezen; zie [Toegangsgroepen](/nl/channels/access-groups).
- Runtime-opmerking: als `channels.line` volledig ontbreekt, valt de runtime voor groepscontroles terug op `groupPolicy="allowlist"` (zelfs als `channels.defaults.groupPolicy` is ingesteld).

LINE-ID's zijn hoofdlettergevoelig. Geldige ID's zien er als volgt uit:

- Gebruiker: `U` + 32 hexadecimale tekens
- Groep: `C` + 32 hexadecimale tekens
- Ruimte: `R` + 32 hexadecimale tekens

## Berichtgedrag

- Tekst wordt opgesplitst bij 5000 tekens.
- Markdown-opmaak wordt verwijderd; codeblokken en tabellen worden waar mogelijk omgezet in Flex-
  kaarten.
- Streamingreacties worden gebufferd; LINE ontvangt volledige segmenten met een laadanimatie
  terwijl de agent werkt.
- Mediadownloads worden beperkt door `channels.line.mediaMaxMb` (standaard 10).
- Inkomende media worden opgeslagen onder `~/.openclaw/media/inbound/` voordat ze aan
  de agent worden doorgegeven, overeenkomstig de gedeelde mediaopslag die door andere kanaalplugins wordt gebruikt.

## Kanaalgegevens (uitgebreide berichten)

Gebruik `channelData.line` om snelle antwoorden, locaties, Flex-kaarten of sjabloonberichten
te verzenden.

```json5
{
  text: "Alsjeblieft",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Kantoor",
        address: "Hoofdstraat 123",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Statuskaart",
        contents: {/* Flex-payload */},
      },
      templateMessage: {
        type: "confirm",
        text: "Doorgaan?",
        confirmLabel: "Ja",
        confirmData: "yes",
        cancelLabel: "Nee",
        cancelData: "no",
      },
    },
  },
}
```

De LINE-plugin levert ook een `/card`-opdracht voor vooraf ingestelde Flex-berichten:

```text
/card info "Welkom" "Bedankt dat je meedoet!"
```

## ACP-ondersteuning

LINE ondersteunt gesprekskoppelingen voor ACP (Agent Communication Protocol):

- `/acp spawn <agent> --bind here` koppelt de huidige LINE-chat aan een ACP-sessie zonder een onderliggende thread te maken.
- Geconfigureerde ACP-koppelingen en actieve gespreksgebonden ACP-sessies werken op LINE zoals bij andere gesprekskanalen.

Zie [ACP-agenten](/nl/tools/acp-agents) voor details.

## Uitgaande media

De LINE-plugin verzendt afbeeldingen, video's en audio via het berichtgereedschap van de agent:

- **Afbeeldingen**: verzonden als LINE-afbeeldingsberichten; de voorbeeldafbeelding gebruikt standaard de media-URL.
- **Video's**: vereisen een voorbeeldafbeelding; stel `channelData.line.previewImageUrl` in op een afbeeldings-URL.
- **Audio**: verzonden als LINE-audioberichten; de duur is standaard 60 seconden, tenzij `channelData.line.durationMs` is ingesteld.

Het mediatype wordt overgenomen uit `channelData.line.mediaKind` wanneer dit is ingesteld, en anders afgeleid
uit de overige LINE-opties of het bestandsachtervoegsel van de URL, met afbeelding als terugvaloptie.

URL's voor uitgaande media moeten openbare HTTPS-URL's van maximaal 2000 tekens zijn. OpenClaw
valideert de doelhostnaam voordat de URL aan LINE wordt doorgegeven en weigert loopback-,
link-local- en privénetwerkdoelen.

Algemene mediaverzendingen zonder LINE-specifieke opties gebruiken de afbeeldingsroute.

## Problemen oplossen

- **Webhook-verificatie mislukt:** zorg dat de Webhook-URL HTTPS gebruikt en dat
  `channelSecret` overeenkomt met de LINE Console.
- **Geen inkomende gebeurtenissen:** controleer of het Webhook-pad overeenkomt met `channels.line.webhookPath`
  en of de Gateway bereikbaar is vanuit LINE.
- **Fouten bij het downloaden van media:** verhoog `channels.line.mediaMaxMb` als media de
  standaardlimiet overschrijden.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Koppeling](/nl/channels/pairing) — authenticatie voor directe berichten en koppelingsflow
- [Groepen](/nl/channels/groups) — gedrag van groepschats en beperkingen op vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiliging
