---
read_when:
    - Werken aan functies voor het Google Chat-kanaal
summary: Ondersteuningsstatus, mogelijkheden en configuratie van de Google Chat-app
title: Google Chat
x-i18n:
    generated_at: "2026-07-27T06:03:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat wordt uitgevoerd als de officiële `@openclaw/googlechat`-plugin: privéberichten en ruimtes via Google Chat API-webhooks (alleen HTTP-eindpunt, geen Pub/Sub).

## Installeren

```bash
openclaw plugins install @openclaw/googlechat
```

Lokale checkout (bij uitvoering vanuit een git-repository):

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## Snelle installatie (beginners)

1. Maak een Google Cloud-project en schakel de **Google Chat API** in.
   - Ga naar: [Google Chat API Credentials](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - Schakel de API in als deze nog niet is ingeschakeld.
2. Maak een **Service Account**:
   - Klik op **Create Credentials** > **Service Account**.
   - Geef het een willekeurige naam (bijvoorbeeld `openclaw-chat`).
   - Laat machtigingen en principals leeg (**Continue** en daarna **Done**).
3. Maak en download de **JSON-sleutel**:
   - Klik op het nieuwe serviceaccount > tabblad **Keys** > **Add Key** > **Create new key** > **JSON** > **Create**.
4. Sla het gedownloade JSON-bestand op de host van je Gateway op (bijvoorbeeld `~/.openclaw/googlechat-service-account.json`).
5. Maak een Google Chat-app in de [Google Cloud Console Chat Configuration](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat):
   - Vul **Application info** in (appnaam, avatar-URL, beschrijving).
   - Schakel **Interactive features** in.
   - Vink onder **Functionality** de optie **Join spaces and group conversations** aan.
   - Selecteer onder **Connection settings** de optie **HTTP endpoint URL**.
   - Selecteer onder **Triggers** de optie **Use a common HTTP endpoint URL for all triggers** en stel deze in op de openbare URL van je Gateway, gevolgd door `/googlechat` (zie [Openbare URL](#public-url-webhook-only)).
   - Vink onder **Visibility** de optie **Make this Chat app available to specific people and groups in `<Your Domain>`** aan en voer je e-mailadres in.
   - Klik op **Save**.
6. Schakel de appstatus in: vernieuw de pagina, zoek **App status**, stel deze in op **Live - available to users** en klik opnieuw op **Save**.
7. Configureer OpenClaw met het serviceaccount en de webhookdoelgroep (moet overeenkomen met de configuratie van de Chat-app):
   - Omgevingsvariabele: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json` (alleen standaardaccount), of
   - Configuratie: zie [Belangrijkste configuratie-instellingen](#config-highlights). `openclaw channels add --channel googlechat` accepteert ook `--audience-type`, `--audience`, `--webhook-path` en `--webhook-url`.
8. Start de Gateway. Google Chat verstuurt POST-verzoeken naar je webhookpad (standaard `/googlechat`).

## Toevoegen aan Google Chat

Zodra de Gateway actief is en je e-mailadres op de zichtbaarheidslijst staat:

1. Ga naar [Google Chat](https://chat.google.com/).
2. Klik op het pictogram **+** (plus) naast **Direct Messages**.
3. Zoek naar de **App name** die je in de Google Cloud Console hebt geconfigureerd.
   - De bot verschijnt _niet_ in de bladerlijst van Marketplace omdat het een privé-app is; zoek de bot op naam.
4. Selecteer de bot, klik op **Add** of **Chat** en stuur een bericht.

## Openbare URL (alleen Webhook)

Google Chat-webhooks vereisen een openbaar HTTPS-eindpunt. Stel voor de veiligheid **alleen het pad `/googlechat`** beschikbaar op internet en houd het OpenClaw-dashboard en andere eindpunten privé.

### Optie A: Tailscale Funnel (aanbevolen)

Gebruik Tailscale Serve voor het privédashboard en Funnel voor het openbare webhookpad.

1. Controleer aan welk adres je Gateway is gebonden:

   ```bash
   ss -tlnp | grep 18789
   ```

   Noteer het IP-adres (bijvoorbeeld `127.0.0.1`, `0.0.0.0` of een Tailscale-`100.x.x.x`-adres).

2. Stel het dashboard alleen beschikbaar voor het tailnet (poort 8443):

   ```bash
   # Indien gebonden aan localhost (127.0.0.1 of 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # Indien alleen gebonden aan een Tailscale-IP:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. Stel alleen het webhookpad openbaar beschikbaar:

   ```bash
   # Indien gebonden aan localhost (127.0.0.1 of 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # Indien alleen gebonden aan een Tailscale-IP:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. Ga desgevraagd naar de autorisatie-URL die in de uitvoer wordt weergegeven om Funnel voor deze Node in te schakelen.

5. Verifieer:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

Je openbare webhook-URL is `https://<node-name>.<tailnet>.ts.net/googlechat`; het dashboard blijft alleen via het tailnet beschikbaar op `https://<node-name>.<tailnet>.ts.net:8443/`. Gebruik de openbare URL (zonder `:8443`) in de configuratie van de Google Chat-app.

> Opmerking: deze configuratie blijft na opnieuw opstarten behouden. Verwijder deze later met `tailscale funnel reset` en `tailscale serve reset`.

### Optie B: Reverse proxy (Caddy)

Proxy alleen het webhookpad:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

Verzoeken naar `your-domain.com/` worden genegeerd of krijgen een 404-respons, terwijl `your-domain.com/googlechat` naar OpenClaw wordt gerouteerd.

### Optie C: Cloudflare Tunnel

Configureer de ingressregels van de tunnel om alleen het webhookpad te routeren:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## Werking

1. Google Chat verstuurt JSON via POST naar het webhookpad van de Gateway (alleen POST, JSON-inhoudstype vereist, snelheidsbeperking per IP).
2. OpenClaw verifieert elk verzoek vóór verwerking:
   - Chat-appgebeurtenissen bevatten `Authorization: Bearer <token>`; het token wordt geverifieerd voordat de volledige body wordt geparseerd.
   - Google Workspace Add-on-gebeurtenissen bevatten het token in de body (`authorizationEventObject.systemIdToken`) en worden vóór verificatie verwerkt binnen een strikter pre-auth-budget (16 KB, 3 s).
3. Het token wordt gecontroleerd aan de hand van `audienceType` + `audience`:
   - `audienceType: "app-url"` → de doelgroep is je HTTPS-webhook-URL.
   - `audienceType: "project-number"` → de doelgroep is het Cloud-projectnummer.
   - Voor add-ontokens onder `app-url` moet bovendien `appPrincipal` zijn ingesteld op de numerieke OAuth 2.0-client-ID van de app (21 cijfers, geen e-mailadres); anders mislukt de verificatie en wordt een waarschuwing gelogd.
4. Berichten worden per ruimte gerouteerd:
   - Ruimtes krijgen sessies per ruimte `agent:<agentId>:googlechat:group:<spaceId>`; antwoorden worden in de berichtenthread geplaatst.
   - Privéberichten worden standaard samengevoegd in de hoofdsessie van de agent; stel `session.dmScope` in voor privéberichtsessies per gesprekspartner (zie [Sessie](/nl/concepts/session)).
5. Toegang tot privéberichten verloopt standaard via koppeling. Onbekende afzenders ontvangen een koppelingscode; keur deze goed met:
   - `openclaw pairing approve googlechat <code>`
6. Groepsruimtes vereisen standaard een @-vermelding. Vermeldingen worden gedetecteerd via Chat-`USER_MENTION`-annotaties die op de app zijn gericht; stel `botUser` in (bijvoorbeeld `users/1234567890`) als voor detectie de naam van de gebruikersresource van de app nodig is.
7. Wanneer een uitvoerings- of plugingoedkeuring vanuit Google Chat wordt gestart en een stabiele `users/<id>`-goedkeurder is geconfigureerd, plaatst OpenClaw een systeemeigen goedkeuringskaart (`cardsV2`) in de oorspronkelijke ruimte of thread. Kaartknoppen bevatten ondoorzichtige callbacktokens; de handmatige `/approve <id> <decision>`-prompt verschijnt alleen wanneer systeemeigen levering niet beschikbaar is.

### Duurzaamheid van inkomende gebeurtenissen

Na authenticatie van het verzoek verwijdert OpenClaw het autorisatieobject van de add-on uit de opslag en plaatst het Google Chat-`MESSAGE`-gebeurtenissen duurzaam in de wachtrij voordat `200` wordt geretourneerd. Een persistentiefout retourneert `503`, zodat Google Chat het opnieuw kan proberen in plaats van een gebeurtenis te bevestigen die verloren kan gaan.

Wachtende berichten en berichten die opnieuw kunnen worden geprobeerd, blijven behouden na een herstart van de Gateway, worden per ruimte serieel verwerkt en gebruiken de resourcenaam van het Google Chat-bericht om dubbele wachtrij-items te onderdrukken zolang de actieve of bewaarde voltooiingsrecord bestaat. Acties die geen berichten zijn, behouden hun bestaande losgekoppelde webhookpad en krijgen deze garantie voor een duurzame wachtrij niet. Levering over de grens van wachtrij naar agent blijft ten minste eenmaal plaatsvinden, zodat een crash tijdens de overdracht een beurt opnieuw kan afspelen.

## Doelen

Gebruik deze identificatoren voor levering en toelatingslijsten:

- Privéberichten: `users/<userId>` (aanbevolen).
- Ruimtes: `spaces/<spaceId>`.
- Een onbewerkt e-mailadres `name@example.com` is veranderlijk en wordt alleen gebruikt voor vergelijking met de toelatingslijst wanneer `channels.googlechat.dangerouslyAllowNameMatching: true`.
- Verouderd: `users/<email>` wordt behandeld als een gebruikers-ID, niet als een e-mailadres in de toelatingslijst.
- De voorvoegsels `googlechat:`, `google-chat:` en `gchat:` worden geaccepteerd en verwijderd.

## Belangrijkste configuratie-instellingen

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // of serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // alleen add-onverificatie; numerieke OAuth-client-ID
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // optioneel; helpt bij detectie van vermeldingen
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "Alleen korte antwoorden.",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

Opmerkingen:

- Referenties van het serviceaccount: `serviceAccountFile` (pad), `serviceAccount` (inline JSON-tekenreeks of -object) of `serviceAccountRef` (SecretRef voor omgevingsvariabele/bestand). De omgevingsvariabelen `GOOGLE_CHAT_SERVICE_ACCOUNT` (inline JSON) en `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (pad) gelden alleen voor het standaardaccount. Configuraties met meerdere accounts gebruiken `channels.googlechat.accounts.<id>` met dezelfde sleutels, waaronder `serviceAccountRef` per account.
- Het standaardwebhookpad is `/googlechat` wanneer `webhookPath` niet is ingesteld; `webhookUrl` kan in plaats daarvan het pad leveren.
- Groepssleutels moeten stabiele ruimte-ID's zijn (`spaces/<spaceId>`). Sleutels met weergavenamen zijn verouderd en worden als zodanig gelogd.
- `dangerouslyAllowNameMatching` schakelt vergelijking van veranderlijke e-mailprincipals voor toelatingslijsten opnieuw in (compatibiliteitsmodus voor noodgevallen); doctor waarschuwt voor e-mailvermeldingen.
- Reactieacties van Google Chat worden niet beschikbaar gesteld. De Plugin gebruikt serviceaccountauthenticatie, terwijl reactie-eindpunten van Google Chat gebruikersauthenticatie vereisen. Bestaande `actions.reactions`-configuratie wordt voor compatibiliteit geaccepteerd, maar heeft geen effect.
- Systeemeigen goedkeuringskaarten gebruiken Google Chat-`cardsV2`-knopklikken, geen reactiegebeurtenissen. Goedkeurders zijn afkomstig uit `allowFrom` of `defaultTo` en moeten stabiele numerieke `users/<id>`-waarden zijn.
- Berichtacties maken alleen tekst-`send` beschikbaar. Voor het uploaden van bijlagen in Google Chat is gebruikersauthenticatie vereist, terwijl deze Plugin serviceaccountauthenticatie gebruikt; daarom is het uploaden van uitgaande bestanden niet beschikbaar.
- `typingIndicator`: `message` (standaard) plaatst een tijdelijke aanduiding `_<Bot> is typing..._` en bewerkt deze tot het eerste antwoord; `none` schakelt dit uit; `reaction` vereist gebruikers-OAuth en valt bij serviceaccountauthenticatie momenteel terug op `message`, waarbij een fout wordt gelogd.
- Inkomende bijlagen (de eerste bijlage per bericht) worden via de Chat API naar de mediapijplijn gedownload, met een limiet van `mediaMaxMb` (standaard 20).
- Door bots geschreven berichten worden standaard genegeerd. Met `allowBots: true` gebruiken geaccepteerde botberichten gedeelde [bescherming tegen botlussen](/nl/channels/bot-loop-protection): configureer `channels.defaults.botLoopProtection` en overschrijf dit vervolgens met `channels.googlechat.botLoopProtection` of `channels.googlechat.groups.<space>.botLoopProtection`.

Details over geheimen: [Geheimenbeheer](/nl/gateway/secrets).

## Probleemoplossing

### 405 Method Not Allowed

Als Google Cloud Logs Explorer fouten zoals deze toont:

```text
statuscode: 405, reden: HTTP-foutrespons: HTTP/1.1 405 Method Not Allowed
```

De Webhook-handler is niet geregistreerd. Veelvoorkomende oorzaken:

1. **Kanaal niet geconfigureerd**: de sectie `channels.googlechat` ontbreekt. Controleer dit met:

   ```bash
   openclaw config get channels.googlechat
   ```

   Als dit "Config path not found" retourneert, voeg je de configuratie toe (zie [Belangrijkste configuratie-instellingen](#config-highlights)).

2. **Plugin niet ingeschakeld**: controleer de Plugin-status:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   Als "disabled" wordt weergegeven, voeg je `plugins.entries.googlechat.enabled: true` toe aan je configuratie.

3. **Gateway niet opnieuw gestart** na configuratiewijzigingen:

   ```bash
   openclaw gateway restart
   ```

Controleer of het kanaal actief is:

```bash
openclaw channels status
# Moet het volgende tonen: Google Chat default: enabled, configured, ...
```

### Andere problemen

- `openclaw channels status --probe` toont authenticatiefouten en een ontbrekende doelgroepconfiguratie (`audience` en `audienceType` zijn beide vereist).
- Als er geen berichten binnenkomen, controleer je de Webhook-URL en triggerconfiguratie van de Chat-app.
- Als de vermeldingsfilter antwoorden blokkeert, stel je `botUser` in op de naam van de gebruikersresource van de app en controleer je `requireMention`.
- `openclaw logs --follow` tijdens het verzenden van een testbericht laat zien of verzoeken de Gateway bereiken.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Gateway-configuratie](/nl/gateway/configuration)
- [Groepen](/nl/channels/groups) — gedrag van groepschats en vermeldingsfilter
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsflow
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiligingsversterking
