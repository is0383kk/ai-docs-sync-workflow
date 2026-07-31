---
read_when:
    - Mattermost instellen
    - Mattermost-routering debuggen
sidebarTitle: Mattermost
summary: Mattermost-bot instellen en OpenClaw-configuratie
title: Mattermost
x-i18n:
    generated_at: "2026-07-27T05:43:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea41fb9a7e4e9ea6bd8d04a4f2c6d2d7f2e43cf71830e445f1e28e2e8737f3cb
    source_path: channels/mattermost.md
    workflow: 16
---

Status: downloadbare plugin (bottoken + WebSocket-gebeurtenissen). Kanalen, privékanalen, groeps-DM's en DM's worden ondersteund. Mattermost is een zelf te hosten platform voor teamberichten ([mattermost.com](https://mattermost.com)).

## Installeren

<Tabs>
  <Tab title="npm-register">
    ```bash
    openclaw plugins install @openclaw/mattermost
    ```
  </Tab>
  <Tab title="Lokale checkout">
    ```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  </Tab>
</Tabs>

Details: [Plugins](/nl/tools/plugin)

## Snelle configuratie

<Steps>
  <Step title="Zorg dat de plugin beschikbaar is">
    Installeer `@openclaw/mattermost` met de bovenstaande opdracht en start daarna de Gateway opnieuw als deze al actief is.
  </Step>
  <Step title="Maak een Mattermost-bot">
    Maak een Mattermost-botaccount, kopieer het **bottoken** en voeg de bot toe aan de teams en kanalen die deze moet kunnen lezen.
  </Step>
  <Step title="Kopieer de basis-URL">
    Kopieer de **basis-URL** van Mattermost (bijvoorbeeld `https://chat.example.com`). Een afsluitende `/api/v4` wordt automatisch verwijderd.
  </Step>
  <Step title="Configureer OpenClaw en start de Gateway">
    Minimale configuratie:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

    Niet-interactief alternatief:

    ```bash
    openclaw channels add --channel mattermost --bot-token <token> --http-url https://chat.example.com
    ```

  </Step>
</Steps>

<Note>
Zelfgehoste Mattermost op een privé-, LAN- of tailnetadres: uitgaande Mattermost-API-verzoeken gaan door een SSRF-beveiliging die privé- en interne IP-adressen standaard blokkeert. Schakel dit in met `channels.mattermost.network.dangerouslyAllowPrivateNetwork: true` (per account: `channels.mattermost.accounts.<id>.network.dangerouslyAllowPrivateNetwork`).
</Note>

## Systeemeigen slash-opdrachten

Systeemeigen slash-opdrachten zijn optioneel. Wanneer ze zijn ingeschakeld, registreert OpenClaw `oc_*` slash-opdrachten voor elk team waarvan de bot lid is en ontvangt het callback-POST-verzoeken op de HTTP-server van de Gateway.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Gebruik dit wanneer Mattermost de Gateway niet rechtstreeks kan bereiken (reverse proxy/openbare URL).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

Geregistreerde opdrachten: `/oc_status`, `/oc_model`, `/oc_models`, `/oc_new`, `/oc_help`, `/oc_think`, `/oc_reasoning`, `/oc_verbose`, `/oc_queue`. Met `nativeSkills: true` worden Skills-opdrachten ook geregistreerd als `/oc_<skill>`.

<AccordionGroup>
  <Accordion title="Opmerkingen over gedrag">
    - `native` en `nativeSkills` zijn standaard ingesteld op `"auto"`, wat voor Mattermost wordt geïnterpreteerd als uitgeschakeld. Stel ze expliciet in op `true`.
    - `callbackPath` is standaard ingesteld op `/api/channels/mattermost/command`.
    - Als `callbackUrl` is weggelaten, leidt OpenClaw `http://<gateway.customBindHost or localhost>:<gateway.port, default 18789><callbackPath>` af. Bindhosts met jokertekens (`0.0.0.0`, `::`) vallen terug op `localhost`.
    - Voor configuraties met meerdere accounts kan `commands` op het hoogste niveau of onder `channels.mattermost.accounts.<id>.commands` worden ingesteld (accountwaarden overschrijven velden op het hoogste niveau).
    - Bestaande slash-opdrachten met dezelfde trigger die door andere integraties zijn gemaakt, blijven ongewijzigd (de registratie slaat ze over); opdrachten die door de bot zijn gemaakt, worden bijgewerkt of opnieuw gemaakt wanneer de callback-URL afwijkt.
    - Callbacks van opdrachten worden gevalideerd met de tokens per opdracht die Mattermost retourneert wanneer OpenClaw `oc_*` opdrachten registreert.
    - OpenClaw vernieuwt de huidige registratie van Mattermost-opdrachten voordat elke callback wordt geaccepteerd, zodat verouderde tokens van verwijderde of opnieuw gegenereerde slash-opdrachten niet meer worden geaccepteerd zonder de Gateway opnieuw te starten.
    - Callbackvalidatie mislukt gesloten als de Mattermost-API niet kan bevestigen dat de opdracht nog actueel is; mislukte validaties worden kort in de cache bewaard, gelijktijdige zoekacties worden samengevoegd en nieuwe zoekacties worden per opdracht in frequentie beperkt om de druk van replay-aanvallen te begrenzen.
    - Slash-callbacks mislukken gesloten wanneer de registratie is mislukt, het opstarten gedeeltelijk was of het callbacktoken niet overeenkomt met het geregistreerde token van de gevonden opdracht (een token dat geldig is voor één opdracht kan de bovenliggende validatie voor een andere opdracht niet bereiken).
    - Geaccepteerde callbacks worden bevestigd met een tijdelijk antwoord "Bezig met verwerken..."; het werkelijke antwoord volgt als een normaal bericht.

  </Accordion>
  <Accordion title="Bereikbaarheidsvereiste">
    Het callback-eindpunt moet bereikbaar zijn vanaf de Mattermost-server.

    - Stel `callbackUrl` niet in op `localhost`, tenzij Mattermost op dezelfde host of in dezelfde netwerknaamruimte als OpenClaw draait.
    - Stel `callbackUrl` niet in op de basis-URL van Mattermost, tenzij die URL `/api/channels/mattermost/command` via een reverse proxy naar OpenClaw doorstuurt.
    - Een snelle controle is `curl https://<gateway-host>/api/channels/mattermost/command`; een GET-verzoek moet `405 Method Not Allowed` van OpenClaw retourneren, niet `404`.

  </Accordion>
  <Accordion title="Mattermost-toelatingslijst voor uitgaand verkeer">
    Als je callback is gericht op privé-, tailnet- of interne adressen, stel je `ServiceSettings.AllowedUntrustedInternalConnections` van Mattermost zo in dat de callbackhost of het callbackdomein wordt opgenomen.

    Gebruik host- of domeinvermeldingen, geen volledige URL's.

    - Goed: `gateway.tailnet-name.ts.net`
    - Fout: `https://gateway.tailnet-name.ts.net`

  </Accordion>
</AccordionGroup>

## Omgevingsvariabelen (standaardaccount)

Stel deze in op de Gateway-host als je de voorkeur geeft aan omgevingsvariabelen:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

<Note>
Omgevingsvariabelen gelden alleen voor het **standaardaccount** (`default`). Andere accounts moeten configuratiewaarden gebruiken.

`MATTERMOST_URL` kan niet vanuit een `.env` van een werkruimte worden ingesteld; zie [.env-bestanden van werkruimten](/nl/gateway/security).
</Note>

## Chatmodi

Mattermost reageert automatisch op DM's. Het gedrag in kanalen wordt bepaald door `chatmode`:

<Tabs>
  <Tab title="oncall (standaard)">
    Reageer in kanalen alleen bij een @vermelding.
  </Tab>
  <Tab title="onmessage">
    Reageer op elk kanaalbericht.
  </Tab>
  <Tab title="onchar">
    Reageer wanneer een bericht begint met een triggerprefix.
  </Tab>
</Tabs>

Configuratievoorbeeld:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"], // standaard
    },
  },
}
```

Opmerkingen:

- `onchar` reageert nog steeds op expliciete @vermeldingen.
- `channels.mattermost.requireMention` wordt nog steeds gehonoreerd, maar `chatmode` heeft de voorkeur. Instellingen van `groups.<channelId>.requireMention` per kanaal hebben voorrang op beide.
- Nadat de bot een zichtbaar antwoord in een kanaalthread heeft verzonden, worden latere berichten in diezelfde thread beantwoord zonder een nieuwe @vermelding of `onchar`-prefix, zodat gesprekken met meerdere beurten in de thread blijven doorlopen. Deelname wordt 7 dagen onthouden nadat de bot voor het laatst in die thread heeft geantwoord en blijft behouden nadat de Gateway opnieuw is gestart. Threads die de bot alleen heeft waargenomen, worden niet beïnvloed; begin een nieuw bericht op het hoogste niveau om opnieuw een expliciete vermelding te vereisen.
- Stel `channels.mattermost.implicitMentions.threadParticipation: false` in om te voorkomen dat vervolgberichten in threads waaraan is deelgenomen de vermeldingsbeperking omzeilen. Accountoverschrijvingen gebruiken `channels.mattermost.accounts.<id>.implicitMentions`. Mattermost levert momenteel geen `replyToBot`- of `quotedBot`-feiten, waardoor deze vlaggen hier geen effect hebben.

## Threads en sessies

Gebruik `channels.mattermost.replyToMode` om te bepalen of antwoorden in kanalen en groepen in het hoofdkanaal blijven of een thread onder het activerende bericht starten.

- `off` (standaard): antwoord alleen in een thread wanneer het inkomende bericht al in een thread staat.
- `first`: start voor kanaal- of groepsberichten op het hoogste niveau een thread onder dat bericht en leid het gesprek naar een sessie die aan die thread is gekoppeld.
- `all` en `batched`: momenteel hetzelfde gedrag als `first` voor Mattermost, omdat vervolgsegmenten en media in dezelfde thread doorgaan zodra Mattermost een threadhoofdbericht heeft.
- Directe berichten gebruiken standaard `off`, zelfs wanneer `replyToMode` is ingesteld.

Gebruik `channels.mattermost.replyToModeByChatType` om de modus te overschrijven voor `direct`-, `group`- of `channel`-chats. Stel `direct` in om threading voor directe berichten in te schakelen:

- `off` (standaard): directe berichten blijven zonder threads in één doorlopende sessie.
- `first`, `all` of `batched`: elk direct bericht op het hoogste niveau start een Mattermost-thread die wordt ondersteund door een nieuwe, onafhankelijke sessie.

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
      replyToModeByChatType: {
        direct: "first",
      },
    },
  },
}
```

Opmerkingen:

- Sessies die aan een thread zijn gekoppeld, gebruiken de id van het activerende bericht als threadhoofdbericht.
- `first` en `all` zijn momenteel gelijkwaardig, omdat vervolgsegmenten en media in dezelfde thread doorgaan zodra Mattermost een threadhoofdbericht heeft.
- Overschrijvingen per chattype hebben voorrang op `replyToMode`. Zonder een overschrijving van `direct` behouden bestaande implementaties vlakke DM's zonder threads.

## Toegangsbeheer (DM's)

- Standaard: `channels.mattermost.dmPolicy = "pairing"` (onbekende afzenders krijgen een koppelingscode). Andere waarden: `allowlist`, `open`, `disabled`.
- Goedkeuren via:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- Openbare DM's: `channels.mattermost.dmPolicy="open"` plus `channels.mattermost.allowFrom=["*"]` (het configuratieschema dwingt het jokerteken af).
- `channels.mattermost.allowFrom` accepteert gebruikers-id's (aanbevolen) en `accessGroup:<name>`-vermeldingen. Zie [Toegangsgroepen](/nl/channels/access-groups).

## Kanalen (groepen)

- Standaard: `channels.mattermost.groupPolicy = "allowlist"` (alleen met vermelding).
- Sta afzenders toe met `channels.mattermost.groupAllowFrom` (gebruikers-id's aanbevolen).
- `channels.mattermost.groupAllowFrom` accepteert `accessGroup:<name>`-vermeldingen. Zie [Toegangsgroepen](/nl/channels/access-groups).
- Overschrijvingen voor vermeldingen per kanaal staan onder `channels.mattermost.groups.<channelId>.requireMention`, of onder `channels.mattermost.groups["*"].requireMention` voor een standaardwaarde.
- Overeenkomsten met `@username` zijn veranderlijk en alleen ingeschakeld wanneer `channels.mattermost.dangerouslyAllowNameMatching: true`.
- Open kanalen: `channels.mattermost.groupPolicy="open"` (alleen met vermelding).
- Volgorde van omzetting: `channels.mattermost.groupPolicy`, vervolgens `channels.defaults.groupPolicy` en daarna `"allowlist"`.
- Runtime-opmerking: als de sectie `channels.mattermost` volledig ontbreekt, mislukt de runtime gesloten naar `groupPolicy="allowlist"` voor groepscontroles (zelfs als `channels.defaults.groupPolicy` is ingesteld) en wordt een eenmalige waarschuwing gelogd.

Voorbeeld:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## Doelen voor uitgaande levering

Gebruik deze doelindelingen met `openclaw message send` of Cron/Webhooks:

| Doel                                | Levert aan                                                        |
| ----------------------------------- | ----------------------------------------------------------------- |
| `channel:<id>`                      | Kanaal op id                                                      |
| `channel:<name>` of `#channel-name` | Kanaal op naam, gezocht in alle teams waarvan de bot lid is       |
| `user:<id>` of `mattermost:<id>`    | DM met die gebruiker                                              |
| `@username`                         | DM (gebruikersnaam wordt via de Mattermost-API omgezet)           |

Uitgaande verzendingen ondersteunen maximaal één bijlage per bericht; splits meerdere bestanden op in afzonderlijke verzendingen.

<Warning>
Losse ondoorzichtige id's (zoals `64ifufp...`) zijn **dubbelzinnig** in Mattermost (gebruikers-id tegenover kanaal-id).

OpenClaw zet ze **eerst als gebruiker** om:

- Als de ID als gebruiker bestaat (`GET /api/v4/users/<id>` slaagt), stuurt OpenClaw een **DM** door het directe kanaal via `/api/v4/channels/direct` op te zoeken.
- Anders wordt de ID behandeld als een **kanaal-ID**.

Als je deterministisch gedrag nodig hebt, gebruik dan altijd de expliciete voorvoegsels (`user:<id>` / `channel:<id>`).
</Warning>

## Opnieuw proberen van DM-kanaal

Wanneer OpenClaw naar een Mattermost-DM-doel verzendt en eerst het directe kanaal moet opzoeken, probeert het tijdelijke fouten bij het aanmaken van het directe kanaal standaard opnieuw.

Gebruik `channels.mattermost.dmChannelRetry` om dat gedrag globaal voor de Mattermost-plugin af te stemmen, of `channels.mattermost.accounts.<id>.dmChannelRetry` voor één account. Standaardwaarden:

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

Opmerkingen:

- Dit geldt alleen voor het aanmaken van DM-kanalen (`/api/v4/channels/direct`), niet voor elke Mattermost-API-aanroep.
- Nieuwe pogingen gebruiken exponentiële back-off met jitter en zijn van toepassing op tijdelijke fouten, zoals snelheidslimieten, 5xx-antwoorden en netwerk- of time-outfouten.
- 4xx-clientfouten anders dan `429` worden als permanent behandeld en niet opnieuw geprobeerd.

## Previewstreaming

Mattermost streamt denkwerk, toolactiviteit en gedeeltelijke antwoordtekst naar een **concept-previewbericht** dat ter plaatse wordt afgerond wanneer het definitieve antwoord veilig kan worden verzonden. In de modus `partial` wordt de preview bijgewerkt met dezelfde bericht-ID, in plaats van het kanaal te overspoelen met berichten per fragment. In de modus `block` wisselt de preview tussen voltooide tekst- en toolactiviteitsblokken, zodat eerdere blokken als afzonderlijke berichten zichtbaar blijven in plaats van door het volgende blok te worden overschreven. Definitieve media- of foutberichten annuleren openstaande previewbewerkingen en gebruiken normale bezorging in plaats van een overbodig previewbericht te publiceren.

Previewstreaming staat **standaard aan** in de modus `partial`. Configureer dit via `channels.mattermost.streaming.mode` (verouderde scalaire/booleaanse waarden van `streaming` worden door `openclaw doctor --fix` gemigreerd):

```json5
{
  channels: {
    mattermost: {
      streaming: { mode: "partial" }, // uit | gedeeltelijk | blok | voortgang
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Streamingmodi">
    - `partial` (standaard): één previewbericht dat wordt bewerkt terwijl het antwoord groeit en daarna wordt afgerond met het volledige antwoord.
    - `block` wisselt de preview tussen voltooide tekst- en toolactiviteitsblokken, zodat elk blok als afzonderlijk bericht zichtbaar blijft in plaats van ter plaatse te worden overschreven. Parallelle en opeenvolgende toolupdates delen het huidige toolactiviteitsbericht.
    - `progress` toont tijdens het genereren een statuspreview en plaatst het definitieve antwoord pas na voltooiing.
    - `off` schakelt previewstreaming uit. Met `streaming.block.enabled: true` worden voltooide assistentblokken nog steeds als normale blokantwoorden (afzonderlijke berichten) bezorgd, in plaats van als één samengevoegd definitief bericht.

  </Accordion>
  <Accordion title="Opmerkingen over streaminggedrag">
    - Als de stream niet ter plaatse kan worden afgerond (bijvoorbeeld omdat het bericht tijdens het streamen is verwijderd), valt OpenClaw terug op het verzenden van een nieuw definitief bericht, zodat het antwoord nooit verloren gaat.
    - Payloads met alleen denkwerk worden onderdrukt in kanaalberichten, inclusief tekst die als een `> Thinking`-blokcitaat binnenkomt. Stel `/reasoning on` in om denkwerk op andere oppervlakken te zien; het definitieve Mattermost-bericht bevat alleen het antwoord.
    - Zie [Streaming](/nl/concepts/streaming#preview-streaming-modes) voor de kanaaltoewijzingsmatrix.

  </Accordion>
</AccordionGroup>

## Reacties (berichtentool)

- Gebruik `message action=react` met `channel=mattermost`.
- `messageId` is de Mattermost-bericht-ID.
- `emoji` accepteert namen zoals `thumbsup` of `:+1:` (dubbele punten zijn optioneel).
- Stel `remove=true` (booleaans) in om een reactie te verwijderen.
- Gebeurtenissen voor het toevoegen/verwijderen van reacties worden als systeemgebeurtenissen doorgestuurd naar de gerouteerde agentsessie, onderworpen aan dezelfde DM-/groepsbeleidscontroles als berichten.

Voorbeelden:

```text
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

Configuratie:

- `channels.mattermost.actions.reactions`: reactieacties in-/uitschakelen (standaard true).
- Overschrijving per account: `channels.mattermost.accounts.<id>.actions.reactions`.

## Interactieve knoppen (berichtentool)

Verzend berichten met aanklikbare knoppen. Wanneer een gebruiker op een knop klikt, ontvangt de agent de selectie en kan deze reageren.

Knoppen zijn afkomstig uit de semantische `presentation`-payload (in normale agentantwoorden en in `message action=send`). OpenClaw geeft waardeknoppen weer als interactieve Mattermost-knoppen, houdt URL-knoppen zichtbaar in de berichttekst en degradeert keuzemenu's naar leesbare tekst.

```text
message action=send channel=mattermost target=channel:<channelId> presentation={"blocks":[{"type":"buttons","buttons":[{"label":"Ja","value":"yes"},{"label":"Nee","value":"no"}]}]}
```

Velden voor presentatieknoppen:

<ParamField path="label" type="string" required>
  Weergavelabel (alias: `text`).
</ParamField>
<ParamField path="value" type="string">
  Waarde die bij een klik wordt teruggestuurd en als actie-ID wordt gebruikt (aliassen: `callback_data`, `callbackData`). Vereist voor een aanklikbare knop, tenzij `url` is ingesteld.
</ParamField>
<ParamField path="url" type="string">
  Linkknop; wordt als `label: url`-tekst in de berichttekst weergegeven in plaats van als interactieve knop.
</ParamField>
<ParamField path="style" type='"primary" | "secondary" | "success" | "danger"'>
  Knopstijl. Mattermost past de standaardopmaak toe op waarden die het niet ondersteunt.
</ParamField>

Voeg `inlineButtons` toe aan de kanaalmogelijkheden om knopondersteuning in de systeemprompt van de agent kenbaar te maken:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

Wanneer een gebruiker op een knop klikt:

<Steps>
  <Step title="Toegangscontrole">
    Degene die klikt, moet dezelfde DM-/groepsbeleidscontroles doorstaan als een afzender van een bericht; ongeautoriseerde klikken krijgen een tijdelijke melding en worden genegeerd.
  </Step>
  <Step title="Knoppen vervangen door bevestiging">
    Alle knoppen worden vervangen door een bevestigingsregel (bijvoorbeeld "✓ **Ja** geselecteerd door @user").
  </Step>
  <Step title="Agent ontvangt de selectie">
    De agent ontvangt de selectie als een inkomend bericht (plus een systeemgebeurtenis) en reageert.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Implementatieopmerkingen">
    - Knopcallbacks gebruiken HMAC-SHA256-verificatie (automatisch, geen configuratie nodig).
    - Bij een klik wordt het volledige bijlageblok vervangen, zodat alle knoppen tegelijk worden verwijderd - gedeeltelijke verwijdering is niet mogelijk.
    - Actie-ID's met koppeltekens of onderstrepingstekens worden automatisch opgeschoond (beperking van Mattermost-routering).
    - Klikken waarvan `action_id` niet overeenkomt met een actie in het oorspronkelijke bericht, worden geweigerd met `403` ("Onbekende actie").

  </Accordion>
  <Accordion title="Configuratie en bereikbaarheid">
    - `channels.mattermost.capabilities`: reeks met mogelijkheden. Voeg `"inlineButtons"` toe om de beschrijving van de knoptool in de systeemprompt van de agent in te schakelen.
    - `channels.mattermost.interactions.callbackBaseUrl`: optionele externe basis-URL voor knopcallbacks (bijvoorbeeld `https://gateway.example.com`). Gebruik dit wanneer Mattermost de Gateway niet rechtstreeks via de bindhost kan bereiken.
    - Bij configuraties met meerdere accounts kun je hetzelfde veld ook instellen onder `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl`.
    - Als `interactions.callbackBaseUrl` wordt weggelaten, leidt OpenClaw de callback-URL af van `gateway.customBindHost` + `gateway.port` (standaard 18789) en valt daarna terug op `http://localhost:<port>`. Het callbackpad is `/mattermost/interactions/<accountId>`.
    - Bereikbaarheidsregel: de URL voor de knopcallback moet bereikbaar zijn vanaf de Mattermost-server. `localhost` werkt alleen wanneer Mattermost en OpenClaw op dezelfde host/netwerknaamruimte draaien.
    - `channels.mattermost.interactions.allowedSourceIps`: toelatingslijst met bron-IP-adressen voor knopcallbacks. Zonder deze lijst worden alleen loopbackbronnen (`127.0.0.1`, `::1`) geaccepteerd. Een externe Mattermost-server moet hier dus aan de toelatingslijst worden toegevoegd, anders worden de klikken geweigerd met `403`. Stel achter een reverse proxy ook `gateway.trustedProxies` in, zodat het werkelijke client-IP-adres uit doorgestuurde headers wordt afgeleid.
    - Als je callbackdoel privé/op een tailnet/intern is, voeg je de host/het domein ervan toe aan `ServiceSettings.AllowedUntrustedInternalConnections` van Mattermost.

  </Accordion>
</AccordionGroup>

### Directe API-integratie (externe scripts)

Externe scripts en webhooks kunnen knoppen rechtstreeks via de Mattermost REST-API plaatsen in plaats van via de `message`-tool van de agent. Geef de voorkeur aan de `message`-tool van OpenClaw. Importeer voor directe integraties `buildButtonAttachments` uit `@openclaw/mattermost/api.js`; volg bij het plaatsen van onbewerkte JSON deze regels:

**Payloadstructuur:**

```json5
{
  channel_id: "<channelId>",
  message: "Kies een optie:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // alleen alfanumeriek - zie hieronder
            type: "button", // vereist, anders worden klikken stilzwijgend genegeerd
            name: "Goedkeuren", // weergavelabel
            style: "primary", // optioneel: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // moet overeenkomen met knop-ID
                action: "approve",
                // ... eventuele aangepaste velden ...
                _token: "<hmac>", // zie het gedeelte over HMAC hieronder
              },
            },
          },
        ],
      },
    ],
  },
}
```

<Warning>
**Kritieke regels**

1. Bijlagen horen in `props.attachments`, niet in `attachments` op het hoogste niveau (wordt stilzwijgend genegeerd).
2. Elke actie heeft `type: "button"` nodig - zonder dit worden klikken stilzwijgend ingeslikt.
3. Elke actie heeft een `id`-veld nodig - Mattermost negeert acties zonder ID's.
4. Actie-`id` moet **uitsluitend alfanumeriek** zijn (`[a-zA-Z0-9]`). Koppeltekens en onderstrepingstekens verstoren de serverroutering van acties in Mattermost (retourneert 404). Verwijder ze vóór gebruik.
5. `context.action_id` moet overeenkomen met de `id` van de knop; de Gateway weigert klikken waarvan `action_id` niet in het bericht bestaat.
6. `context.action_id` is vereist - de interactiehandler retourneert zonder dit veld 400.
7. Het bron-IP-adres van de callback moet zijn toegestaan (zie `interactions.allowedSourceIps` hierboven).

</Warning>

**HMAC-token genereren**

De Gateway verifieert knopklikken met HMAC-SHA256. Externe scripts moeten tokens genereren die overeenkomen met de verificatielogica van de Gateway:

<Steps>
  <Step title="Het geheim afleiden van het bottoken">
    `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`, hexadecimaal gecodeerd.
  </Step>
  <Step title="Het contextobject opbouwen">
    Bouw het contextobject op met alle velden **behalve** `_token`.
  </Step>
  <Step title="Serialiseren met gesorteerde sleutels">
    Serialiseer met **recursief gesorteerde sleutels** en **zonder spaties** (de Gateway canonicaliseert ook geneste objecten en produceert compacte JSON).
  </Step>
  <Step title="De payload ondertekenen">
    `HMAC-SHA256(key=secret, data=serializedContext)`
  </Step>
  <Step title="Het token toevoegen">
    Voeg de resulterende hexadecimale digest als `_token` aan de context toe.
  </Step>
</Steps>

Python-voorbeeld:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

<AccordionGroup>
  <Accordion title="Veelvoorkomende HMAC-valkuilen">
    - Python's `json.dumps` voegt standaard spaties toe (`{"key": "val"}`). Gebruik `separators=(",", ":")` zodat dit overeenkomt met de compacte uitvoer van JavaScript (`{"key":"val"}`).
    - Onderteken altijd **alle** contextvelden (behalve `_token`). De Gateway verwijdert `_token` en ondertekent vervolgens alles wat overblijft. Het ondertekenen van een subset veroorzaakt een stille verificatiefout.
    - Gebruik `sort_keys=True`: de Gateway sorteert sleutels vóór ondertekening en Mattermost kan contextvelden opnieuw ordenen wanneer de payload wordt opgeslagen.
    - Leid het geheim af van het bottoken (deterministisch), niet van willekeurige bytes. Het geheim moet hetzelfde zijn in het proces dat knoppen maakt en de Gateway die ze verifieert.

  </Accordion>
</AccordionGroup>

## Directory-adapter

De Mattermost-plugin bevat een directory-adapter die kanaal- en gebruikersnamen via de Mattermost-API omzet. Hierdoor kunnen `#channel-name`- en `@username`-doelen worden gebruikt in `openclaw message send` en leveringen via cron/webhook.

Er is geen configuratie nodig: de adapter gebruikt het bottoken uit de accountconfiguratie.

## Meerdere accounts

Mattermost ondersteunt meerdere accounts onder `channels.mattermost.accounts`:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

Accountwaarden overschrijven velden op het hoogste niveau; `channels.mattermost.defaultAccount` bepaalt welk account wordt gebruikt wanneer er geen account is opgegeven.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Geen antwoorden in kanalen">
    Zorg dat de bot zich in het kanaal bevindt en vermeld deze (oncall), gebruik een triggerprefix (onchar) of stel `chatmode: "onmessage"` in.
  </Accordion>
  <Accordion title="Authenticatie- of multi-accountfouten">
    - Controleer het bottoken, de basis-URL en of het account is ingeschakeld.
    - Problemen met meerdere accounts: omgevingsvariabelen zijn alleen van toepassing op het `default`-account.
    - Privé-/LAN-hosts van Mattermost vereisen `network.dangerouslyAllowPrivateNetwork: true` (de SSRF-beveiliging blokkeert standaard privé-IP-adressen).

  </Accordion>
  <Accordion title="Native slash-opdrachten mislukken">
    - `Unauthorized: invalid command token.`: OpenClaw heeft het callbacktoken niet geaccepteerd. Gebruikelijke oorzaken:
      - de registratie van de slash-opdracht is mislukt of bij het opstarten slechts gedeeltelijk voltooid
      - de callback bereikt de verkeerde Gateway of het verkeerde account
      - Mattermost heeft nog oude opdrachten die naar een eerder callbackdoel verwijzen
      - de Gateway is opnieuw gestart zonder de slash-opdrachten opnieuw te activeren
    - Als native slash-opdrachten niet meer werken, controleer dan de logboeken op `mattermost: failed to register slash commands` of `mattermost: native slash commands enabled but no commands could be registered`.
    - Als `callbackUrl` is weggelaten en de logboeken waarschuwen dat de callback is omgezet naar een loopback-URL zoals `http://localhost:18789/...`, is die URL waarschijnlijk alleen bereikbaar wanneer Mattermost op dezelfde host/netwerknaamruimte als OpenClaw draait. Stel in plaats daarvan een expliciete, extern bereikbare `commands.callbackUrl` in.

  </Accordion>
  <Accordion title="Problemen met knoppen">
    - Knoppen worden als witte vakken of helemaal niet weergegeven: de knopgegevens zijn ongeldig. Elke presentatieknop heeft een `label` en een `value` nodig (knoppen waarbij een van beide ontbreekt, worden verwijderd).
    - Knoppen worden weergegeven, maar klikken doet niets: controleer of de Gateway bereikbaar is vanaf de Mattermost-server, of het IP-adres van de Mattermost-server is opgenomen in `channels.mattermost.interactions.allowedSourceIps` (zonder deze instelling wordt alleen loopback geaccepteerd) en of `ServiceSettings.AllowedUntrustedInternalConnections` de callbackhost voor privédoelen bevat.
    - Knoppen retourneren 404 wanneer erop wordt geklikt: de `id` van de knop bevat waarschijnlijk koppeltekens of underscores. De actierouter van Mattermost werkt niet met niet-alfanumerieke ID's. Gebruik alleen `[a-zA-Z0-9]`.
    - De Gateway logt `rejected callback source`: de klik kwam van een IP-adres buiten `interactions.allowedSourceIps`. Voeg de Mattermost-server of je ingress toe aan de toelatingslijst en stel `gateway.trustedProxies` in achter een reverse proxy.
    - De Gateway logt `invalid _token`: HMAC komt niet overeen. Controleer of je alle contextvelden ondertekent (niet slechts een subset), gesorteerde sleutels gebruikt en compacte JSON gebruikt (zonder spaties). Zie het gedeelte over HMAC hierboven.
    - De Gateway logt `missing _token in context`: het veld `_token` bevindt zich niet in de context van de knop. Zorg dat dit veld wordt opgenomen bij het opbouwen van de integratiepayload.
    - De Gateway weigert de klik met `Unknown action`: `context.action_id` komt niet overeen met een `id` van een actie in het bericht. Stel beide in op dezelfde opgeschoonde waarde.
    - De agent biedt geen knoppen aan: voeg `capabilities: ["inlineButtons"]` toe aan de Mattermost-kanaalconfiguratie.

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Kanaalroutering](/nl/channels/channel-routing) - sessieroutering voor berichten
- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Groepen](/nl/channels/groups) - gedrag van groepschats en vermeldingstoegang
- [Koppeling](/nl/channels/pairing) - DM-authenticatie en koppelingsflow
- [Beveiliging](/nl/gateway/security) - toegangsmodel en beveiligingsmaatregelen
