---
read_when:
    - Slack instellen of de socket-, HTTP- of relaymodus van Slack debuggen
summary: Slack-configuratie en runtimegedrag (Socket Mode, HTTP Request URLs en relaymodus)
title: Slack
x-i18n:
    generated_at: "2026-07-27T04:57:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

Slack-ondersteuning omvat DM's en kanalen via Slack-appintegraties. Het standaardtransport is Socket Mode; HTTP Request URLs worden ook ondersteund. Relay-modus is bedoeld voor beheerde implementaties waarbij een vertrouwde router de Slack-ingang beheert.

<CardGroup cols={3}>
  <Card title="Koppelen" icon="link" href="/nl/channels/pairing">
    Slack-DM's gebruiken standaard de koppelingsmodus.
  </Card>
  <Card title="Slash-opdrachten" icon="terminal" href="/nl/tools/slash-commands">
    Gedrag van native opdrachten en opdrachtencatalogus.
  </Card>
  <Card title="Problemen met kanalen oplossen" icon="wrench" href="/nl/channels/troubleshooting">
    Diagnostiek en herstelprocedures voor meerdere kanalen.
  </Card>
</CardGroup>

## Een transport kiezen

Socket Mode en HTTP Request URLs bieden dezelfde functionaliteit voor berichten, slash-opdrachten, App Home en interactiviteit. Kies op basis van de implementatievorm, niet van de functies.

| Aandachtspunt                 | Socket Mode (standaard)                                                                                                                              | HTTP Request URLs                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Openbare Gateway-URL         | Niet vereist                                                                                                                                         | Vereist (DNS, TLS, reverse proxy of tunnel)                                                                    |
| Uitgaand netwerk             | Uitgaande WSS naar `wss-primary.slack.com` moet bereikbaar zijn                                                                                           | Geen uitgaande WS; alleen inkomende HTTPS                                                                      |
| Vereiste tokens              | Botidentiteit: bottoken + App-Level Token met `connections:write`; gebruikersidentiteit: gebruikerstoken + App-Level Token                            | Botidentiteit: bottoken + Signing Secret; gebruikersidentiteit: gebruikerstoken + Signing Secret               |
| Ontwikkellaptop / achter firewall | Werkt zonder aanpassingen                                                                                                                       | Vereist een openbare tunnel (ngrok, Cloudflare Tunnel, Tailscale Funnel) of staging-Gateway                    |
| Horizontaal schalen          | Eén Socket Mode-sessie per app per host; meerdere Gateways vereisen afzonderlijke Slack-apps                                                         | Stateless POST-handler; meerdere Gateway-replica's kunnen één app achter een loadbalancer delen                |
| Meerdere accounts op één Gateway | Ondersteund; elk account opent een eigen WS                                                                                                      | Ondersteund; elk account vereist een unieke `webhookPath` (standaard `/slack/events`), zodat registraties niet conflicteren |
| Transport voor slash-opdrachten | Geleverd via de WS-verbinding; `slash_commands[].url` wordt genegeerd                                                                                | Slack stuurt POST-verzoeken naar `slash_commands[].url`; het veld is vereist om de opdracht door te sturen         |
| Ondertekening van verzoeken  | Niet gebruikt (authenticatie verloopt via de App-Level Token)                                                                                        | Slack ondertekent elk verzoek; OpenClaw verifieert dit met `signingSecret`                                  |
| Herstel na verbroken verbinding | Automatisch opnieuw verbinden door de Slack SDK is ingeschakeld; OpenClaw herstart mislukte Socket Mode-sessies ook met begrensde back-off. Transportafstemming voor pong-time-outs is van toepassing. | Er is geen permanente verbinding die kan worden verbroken; nieuwe pogingen worden per verzoek door Slack uitgevoerd |

<Note>
  **Kies Socket Mode** voor hosts met één Gateway, ontwikkellaptops en on-premisesnetwerken die `*.slack.com` uitgaand kunnen bereiken, maar geen inkomende HTTPS kunnen accepteren.

**Kies HTTP Request URLs** wanneer je meerdere Gateway-replica's achter een loadbalancer uitvoert, wanneer uitgaande WSS is geblokkeerd maar inkomende HTTPS is toegestaan, of wanneer je Slack-webhooks al op een reverse proxy afhandelt.
</Note>

<Warning>
  Slack kan meerdere Socket Mode-verbindingen voor één app onderhouden en elke payload aan om het even welke verbinding leveren. Afzonderlijke OpenClaw-gateways die een Slack-app delen, moeten daarom gelijkwaardige routerings- en autorisatieconfiguraties hebben. Gebruik anders een afzonderlijke Slack-app per gateway, één centrale relay-ingang of HTTP Request URLs achter een loadbalancer. Zie [Socket Mode gebruiken](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections).
</Warning>

### Relay-modus

Relay-modus scheidt de Slack-ingang van de OpenClaw-gateway. Een vertrouwde router beheert de enige Slack Socket Mode-verbinding, kiest een bestemmingsgateway en stuurt een getypeerde gebeurtenis door via een geauthenticeerde websocket. De gateway gebruikt nog steeds zijn eigen bottoken voor uitgaande aanroepen naar de Slack Web API.

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

De relay-URL moet `wss://` gebruiken, tenzij deze naar localhost verwijst. Beschouw het bearertoken en de routeringstabel van de router als onderdeel van de Slack-autorisatiegrens: gerouteerde gebeurtenissen komen de normale Slack-berichtenhandler binnen als geautoriseerde activeringen. Een door de router verstrekte `slack_identity` in het websocket-`hello`-frame kan de standaard uitgaande gebruikersnaam en het pictogram instellen; een expliciete identiteit die door de aanroeper wordt verstrekt, heeft nog steeds voorrang. De relay-verbinding maakt opnieuw verbinding met dezelfde begrensde back-offtiming als Socket Mode en wist de door de router verstrekte identiteit telkens wanneer de verbinding wordt verbroken.

### Organisatiebrede installaties voor Enterprise Grid

Eén Slack-account kan berichten ontvangen uit elke workspace die onder een
organisatiebrede Enterprise Grid-installatie valt. Kies rechtstreekse Socket Mode
of HTTP Request URLs; relay-modus wordt niet ondersteund voor enterprise-accounts.
Beide onderstaande manifesten met minimale bevoegdheden schakelen alleen het V1-pad
voor `message`- en `app_mention`-gebeurtenissen, directe antwoorden en
door de listener beheerde statusreacties in.

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Laat een Enterprise Grid Org Admin of Org Owner de app goedkeuren, op
organisatieniveau installeren en de workspaces kiezen die onder de installatie vallen.
Controleer voordat je OpenClaw start of de app in elke beoogde workspace beschikbaar is.
Genereer voor Socket Mode een app-level token met `connections:write` en kopieer
vervolgens het bottoken uit de organisatie-installatie. Configureer het account dat
het organisatiebreed geïnstalleerde bottoken gebruikt:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### HTTP Request URLs

Gebruik de HTTP-modus wanneer de Gateway een openbaar HTTPS-eindpunt heeft en geen
Socket Mode-verbinding opent. Vervang de voorbeeld-URL door de openbare
`webhookPath`-URL van de Gateway (standaard `/slack/events`):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

Laat een Enterprise Grid Org Admin of Org Owner de app goedkeuren, op
organisatieniveau installeren en de workspaces kiezen die onder de installatie vallen.
Nadat Slack de Request URL heeft geverifieerd, kopieer je het bottoken van de
organisatie-installatie en de **Basic Information -> App Credentials -> Signing Secret**
van de app. Configureer het enterprise-account met hetzelfde pad voor de Request URL:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

Bij het opstarten verifieert OpenClaw `enterpriseOrgInstall` met Slack
`auth.test`. Een organisatiebreed geïnstalleerd token zonder de vlag, of een
workspace-token met de vlag, zorgt ervoor dat het opstarten mislukt. Slack blijft de
gezaghebbende bron voor welke workspaces de installatie hebben toegestaan; OpenClaw past
vervolgens het geconfigureerde beleid voor kanalen, gebruikers, DM's en vermeldingen toe
op elke geleverde gebeurtenis. Enterprise V1 weigert vóór het doorsturen alle door bots
geschreven `message`- en `app_mention`-gebeurtenissen, ongeacht
`allowBots`, omdat organisatie-installaties geen stabiele, aan een workspace
gekoppelde botidentiteit bieden om lussen te voorkomen.

Enterprise-ondersteuning is bewust beperkt tot rechtstreekse Socket Mode- of HTTP-
`message`- en `app_mention`-gebeurtenissen en de directe antwoorden daarop.
Relay-modus, slash-opdrachten, interacties, App Home, listeners voor reactiegebeurtenissen,
vastgezette items, Slack-actietools, native Slack-goedkeuringen, bindingen, levering in de
wachtrij of volgens een planning en proactieve verzendingen zijn niet beschikbaar voor
een enterprise-account. Uitgaande bevestigings-, typ- en statusreacties worden ondersteund
via de door de listener beheerde Slack-client en vereisen `reactions:write`; inkomende
reactiemeldingen en reactie-actietools blijven niet beschikbaar.

Directe antwoorden hergebruiken het standaardbezorggedrag van Slack voor fragmenten,
media, metagegevens, identiteitsterugval, linkvoorvertoningen en ontvangstbevestigingen, maar alleen zolang de
gevalideerde client die eigendom is van de listener in de actieve gebeurtenisafhandeling blijft. De
verzendwachtrij in het geheugen en registraties van deelname aan threads worden gepartitioneerd op basis van de
workspace van die gebeurtenis; de client zelf wordt nooit geserialiseerd of persistent opgeslagen.

Kanaalbeleidsleutels en `dm.groupChannels`-vermeldingen moeten onbewerkte, stabiele Slack-kanaal-ID's of de
`channel:<id>`-vorm gebruiken. OpenClaw normaliseert beide vormen naar het onbewerkte kanaal-ID voor
runtimeovereenkomsten; voorvoegsels `slack:`, `group:` en `mpim:` laten het opstarten mislukken.
Gebruikersbeleidsvermeldingen moeten stabiele Slack-gebruikers-ID's gebruiken; namen, slugs, weergavenamen
en e-mailadressen laten het opstarten mislukken. ID's moeten het canonieke Slack-voorvoegsel in hoofdletters
en de canonieke hoofdtekst gebruiken (bijvoorbeeld `C0123456789` of `U0123456789`); varianten in kleine letters en
korte lookalikes laten het opstarten mislukken. Enterprise-accounts kunnen
`dangerouslyAllowNameMatching` niet inschakelen. Enterprise-accounts mogen de algemene
`mentionPatterns.mode` instellen, maar `mentionPatterns.allowIn` en
`mentionPatterns.denyIn` laten het opstarten mislukken omdat kale Slack-kanaal-ID's niet
aan een workspace zijn gekoppeld en in meerdere workspaces kunnen worden hergebruikt. Workspace-installaties
behouden het bestaande, begrensde gedrag voor vermeldingspatronen. Elke geaccepteerde workspace
krijgt een afzonderlijke identiteit voor routering, sessies, transcripties, deduplicatie, geschiedenis en caching,
zelfs wanneer Slack-ID's overlappen. Binnen de `message`-stream worden gewone gebruikersberichten
en door gebruikers aangemaakte `file_share`-gebeurtenissen ondersteund; andere berichtsubtypen worden
vóór autorisatie of verwerking van systeemgebeurtenissen geweigerd.

Enterprise-DM's moeten uitgeschakeld zijn (`dm.enabled=false` of
`dmPolicy="disabled"`) of expliciet geopend zijn met `dmPolicy="open"` en
een effectieve account-`allowFrom` die de letterlijke waarde `"*"` bevat. Een lege
toegestane lijst of gebruikersspecifieke ID's zonder `"*"` laten het opstarten mislukken. Koppeling en
DM-toegestane lijsten per gebruiker worden geweigerd omdat Slack-gebruikers-ID's in die autorisatieopslag
niet aan een workspace zijn gekoppeld. Kanaal- en afzenderbeleid
blijft van toepassing op kanaalberichten.

## Installatie

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` registreert en activeert de Plugin. Deze doet niets totdat je de Slack-app en onderstaande kanaalinstellingen configureert. Zie [Plugins](/nl/tools/plugin) voor algemene regels voor het installeren van plugins.

## Snelle configuratie

De manifesten in deze sectie maken een installatie die tot één workspace is beperkt. Gebruik voor een
installatie voor een volledige Enterprise Grid-organisatie in plaats daarvan het speciale
[organisatiebrede manifest en de bijbehorende workflow](#enterprise-grid-org-wide-installs).

<Tabs>
  <Tab title="Socket Mode (standaard)">
    <Steps>
      <Step title="Een nieuwe Slack-app maken">
        Open [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → selecteer je workspace → plak een van de onderstaande manifesten → **Next** → **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw koppelt Slack Agent View-gesprekken aan OpenClaw-agents.",
      "suggested_prompts": [
        { "title": "Wat kun je doen?", "message": "Waarmee kun je me helpen?" },
        {
          "title": "Dit kanaal samenvatten",
          "message": "Vat de recente activiteit in dit kanaal samen."
        },
        { "title": "Een antwoord opstellen", "message": "Help me een antwoord op te stellen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw koppelt Slack Agent View-gesprekken aan OpenClaw-agents.",
      "suggested_prompts": [
        { "title": "Wat kun je doen?", "message": "Waarmee kun je me helpen?" },
        {
          "title": "Dit kanaal samenvatten",
          "message": "Vat de recente activiteit in dit kanaal samen."
        },
        { "title": "Een antwoord opstellen", "message": "Help me een antwoord op te stellen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Recommended** komt overeen met de volledige functieset van de Slack-plugin: App Home, slash-opdrachten, bestanden, reacties, vastgezette items, groeps-DM's en het lezen van emoji's/gebruikersgroepen. Kies **Minimal** wanneer het workspacebeleid scopes beperkt — deze optie omvat DM's, kanaal-/groepsgeschiedenis, vermeldingen en slash-opdrachten, maar laat bestanden, reacties, vastgezette items, groeps-DM's (`mpim:*`), `emoji:read` en `usergroups:read` weg. Zie [Checklist voor manifest en scopes](#manifest-and-scope-checklist) voor de onderbouwing per scope en aanvullende opties, zoals extra slash-opdrachten.
        </Note>

        Nadat Slack de app heeft gemaakt:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**: voeg `connections:write` toe, sla op en kopieer het App-Level Token.
        - **Install App -> Install to Workspace**: kopieer het Bot User OAuth Token.

      </Step>

      <Step title="OpenClaw configureren">

        Aanbevolen SecretRef-configuratie:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        Terugval op omgevingsvariabelen (alleen standaardaccount):

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP Request URLs">
    <Steps>
      <Step title="Een nieuwe Slack-app maken">
        Open [api.slack.com/apps](https://api.slack.com/apps/new) → **Create New App** → **From a manifest** → selecteer je workspace → plak een van de onderstaande manifesten → vervang `https://gateway-host.example.com/slack/events` door je openbare Gateway-URL → **Next** → **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw koppelt Slack Agent View-gesprekken aan OpenClaw-agents.",
      "suggested_prompts": [
        { "title": "Wat kun je doen?", "message": "Waarmee kun je me helpen?" },
        {
          "title": "Dit kanaal samenvatten",
          "message": "Vat de recente activiteit in dit kanaal samen."
        },
        { "title": "Een antwoord opstellen", "message": "Help me een antwoord op te stellen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindt gesprekken in Slack Agent View met OpenClaw-agents.",
      "suggested_prompts": [
        { "title": "Wat kun je doen?", "message": "Waarmee kun je me helpen?" },
        {
          "title": "Dit kanaal samenvatten",
          "message": "Vat de recente activiteit in dit kanaal samen."
        },
        { "title": "Een antwoord opstellen", "message": "Help me een antwoord op te stellen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **Aanbevolen** komt overeen met de volledige functieset van de Slack-plugin; **Minimaal** laat bestanden, reacties, vastgemaakte items, groeps-DM's (`mpim:*`), `emoji:read` en `usergroups:read` weg voor restrictieve werkruimten. Zie [Checklist voor manifest en scopes](#manifest-and-scope-checklist) voor de onderbouwing per scope.
        </Note>

        <Info>
          De drie URL-velden (`slash_commands[].url`, `event_subscriptions.request_url` en `interactivity.request_url` / `message_menu_options_url`) verwijzen allemaal naar hetzelfde OpenClaw-eindpunt. Het manifestschema van Slack vereist dat ze afzonderlijk worden benoemd, maar OpenClaw routeert op payloadtype, zodat één `webhookPath` (standaard `/slack/events`) volstaat. Slash-opdrachten zonder `slash_commands[].url` doen in HTTP-modus ongemerkt niets.
        </Info>

        Nadat Slack de app heeft aangemaakt:

        - **Basic Information → App Credentials**: kopieer het **Signing Secret** voor verzoekverificatie.
        - **Install App -> Install to Workspace**: kopieer het Bot User OAuth Token.

      </Step>

      <Step title="OpenClaw configureren">

        Aanbevolen SecretRef-configuratie:

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        Gebruik unieke webhookpaden voor HTTP met meerdere accounts

        Geef elk account een afzonderlijke `webhookPath` (standaard `/slack/events`), zodat registraties niet conflicteren.
        </Note>

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Gebruikersidentiteit (plaatsen als een echte persoon)

Met een gebruikersidentiteit kan OpenClaw lezen en berichten plaatsen als de persoon die de Slack-app autoriseert. De `userToken` is de handelende identiteit; een bijbehorende Slack-app verwerkt verkeer van de Events API via Socket Mode of een HTTP Request URL. De bijbehorende app heeft geen botgebruiker of bottoken nodig.

Stel de bijbehorende app als volgt in:

1. Voeg onder **OAuth & Permissions -> User Token Scopes** deze machtigingen met gebruikersscope toe:

   - geschiedenis: `channels:history`, `groups:history`, `im:history`, `mpim:history`
   - gesprekken opzoeken: `channels:read`, `groups:read`, `im:read`, `mpim:read`
   - personen: `users:read`
   - berichten plaatsen: `chat:write` (berichten worden geplaatst als de autoriserende gebruiker)
   - DM's openen: `im:write`, `mpim:write`

2. Voeg onder **Event Subscriptions -> Subscribe to events on behalf of users** deze gebruikersgebeurtenissen toe. Voeg ze niet alleen toe aan de lijst met botgebeurtenissen:

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. Kies één gebeurtenistransport:

   - **Socket Mode:** schakel Socket Mode in en maak een token op appniveau met `connections:write`. Configureer dit als `appToken`.
   - **HTTP Request URL:** laat Event Subscriptions verwijzen naar het openbare Slack-eindpunt van OpenClaw en kopieer **Basic Information -> App Credentials -> Signing Secret**. Configureer dit als `signingSecret`.

4. Installeer de app of installeer deze opnieuw, autoriseer deze als de beoogde persoon en kopieer het resulterende OAuth-gebruikerstoken naar `userToken`.

Configuratie voor Socket Mode:

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

Configuratie voor HTTP Request URL:

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  DM's en groeps-DM's werken alleen via het abonnement op gebeurtenissen met gebruikersscope hierboven. Een bot kan niet deelnemen aan een menselijke 1:1-DM en kan niet worden toegevoegd aan een bestaande groeps-DM. De bijbehorende app is onzichtbare infrastructuur: andere Slack-leden zien berichten van de autoriserende persoon, niet van een OpenClaw-bot.
</Warning>

OpenClaw negeert automatisch berichtgebeurtenissen met gebruikersscope die door de herkende menselijke identiteit zijn gemaakt, zodat verzonden berichten geen antwoorden aan zichzelf activeren.

## Transportafstemming voor Socket Mode

OpenClaw stelt voor Socket Mode de pong-time-out van de Slack SDK-client standaard in op 15 seconden. Pas de transportinstellingen alleen aan wanneer werkruimte- of hostspecifieke afstemming nodig is:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

Gebruik dit alleen voor Socket Mode-werkruimten die time-outs voor websocket-pongs/serverpings van Slack registreren of die draaien op hosts met bekende uithongering van de eventloop. `clientPingTimeout` is de wachttijd voor een pong nadat de SDK een clientping heeft verzonden; `serverPingTimeout` is de wachttijd voor serverpings van Slack. Appberichten en gebeurtenissen blijven applicatiestatus, geen signalen voor de beschikbaarheid van het transport.

Opmerkingen:

- `socketMode` wordt genegeerd in de HTTP Request URL-modus.
- Basisinstellingen voor `channels.slack.socketMode` gelden voor alle Slack-accounts, tenzij ze worden overschreven. Overschrijvingen per account gebruiken `channels.slack.accounts.<accountId>.socketMode`; omdat dit een objectoverschrijving is, moet je elk veld voor socketafstemming opnemen dat je voor dat account wilt gebruiken.
- Alleen `clientPingTimeout` heeft een OpenClaw-standaardwaarde (`15000`). `serverPingTimeout` en `pingPongLoggingEnabled` worden alleen aan de Slack SDK doorgegeven wanneer ze zijn geconfigureerd.
- De wachttijd voor het opnieuw starten van Socket Mode begint rond 2 seconden en loopt op tot maximaal ongeveer 30 seconden. Herstelbare fouten bij het starten, wachten op het starten en verbreken van de verbinding worden opnieuw geprobeerd totdat het kanaal stopt. Permanente account- en referentiefouten, zoals ongeldige authenticatie, ingetrokken tokens of ontbrekende scopes, mislukken direct in plaats van eindeloos opnieuw te worden geprobeerd.

## Checklist voor manifest en scopes

Het basismanifest van de Slack-app is hetzelfde voor Socket Mode en HTTP Request URL's. Alleen het blok `settings` (en `url` van de slash-opdracht) verschilt.

Basismanifest (standaard voor Socket Mode):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack-connector voor OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw verbindt gesprekken in Slack Agent View met OpenClaw-agents.",
      "suggested_prompts": [
        { "title": "Wat kun je doen?", "message": "Waarmee kun je me helpen?" },
        {
          "title": "Dit kanaal samenvatten",
          "message": "Vat de recente activiteit in dit kanaal samen."
        },
        { "title": "Een antwoord opstellen", "message": "Help me een antwoord op te stellen." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

Vervang voor de **HTTP Request URL-modus** `settings` door de HTTP-variant en voeg `url` toe aan elke slash-opdracht. Openbare URL vereist:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Een bericht naar OpenClaw sturen",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### Aanvullende manifestinstellingen

Maak verschillende functies beschikbaar die de bovenstaande standaardinstellingen uitbreiden.

Het standaardmanifest schakelt het tabblad **Home** van Slack App Home in en abonneert zich op `app_home_opened`. Wanneer een werkruimtelid het tabblad Home opent, publiceert OpenClaw een veilige standaardweergave voor Home met `views.publish`; er worden geen gespreksgegevens of privéconfiguratie opgenomen. Wanneer de modus met één slash-opdracht is ingeschakeld, gebruikt de opdrachthint `channels.slack.slashCommand.name`; installaties die native opdrachten of geen slash-opdrachten gebruiken, laten die hint weg. Het tabblad **Messages** blijft ingeschakeld voor Slack-DM's. Nieuwe apps gebruiken Slack Agent View via `features.agent_view`, `assistant:write` en `app_context_changed`. Elke zichtbare hoofdweergave van Agent View wordt naar een eigen OpenClaw-threadsessie gerouteerd en de geordende actieve-weergave-entiteiten van Slack bereiken de agent uitsluitend als niet-vertrouwde context.

Bestaande apps die `features.assistant_view` al gebruiken, kunnen hun huidige manifest behouden. OpenClaw blijft `assistant_thread_started` en `assistant_thread_context_changed` voor die installaties afhandelen. Slack maakt de migratie van Assistant View naar Agent View onomkeerbaar en vereist dat gebruikers daarna een harde vernieuwing uitvoeren. Vervang `assistant_view` daarom pas in een bestaande app wanneer je de volledige werkruimte wilt migreren.

<AccordionGroup>
  <Accordion title="Optionele native slash-opdrachten">

    Meerdere [native slash-opdrachten](#commands-and-slash-behavior) kunnen met enige nuance worden gebruikt in plaats van één geconfigureerde opdracht:

    - Gebruik `/agentstatus` in plaats van `/status`, omdat de opdracht `/status` is gereserveerd.
    - Er kunnen niet meer dan 25 slash-opdrachten tegelijk voor een Slack-app worden geregistreerd (limiet van het Slack-platform).

    OpenClaw registreert handlers voor ingeschakelde native opdrachten, maar de manifestvermeldingen van Slack blijven door de beheerder beheerd en worden niet tijdens runtime gesynchroniseerd. Voeg `/login` handmatig aan het manifest toe; het onderstaande voorbeeld bevat deze in plaats van de optionele alias `/side` om op 25 opdrachten te blijven. `/login` kan overal worden weergegeven, maar geeft alleen koppelcodes uit in privéchats of de webinterface.

    Vervang je bestaande sectie `features.slash_commands` door een subset van de [beschikbare opdrachten](/nl/tools/slash-commands#command-list):

    <Tabs>
      <Tab title="Socket Mode (standaard)">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Een nieuwe sessie starten",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "De huidige sessie opnieuw instellen"
    },
    {
      "command": "/compact",
      "description": "De sessiecontext comprimeren",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "De huidige uitvoering stoppen"
    },
    {
      "command": "/session",
      "description": "De vervaltijd van de threadkoppeling beheren",
      "usage_hint": "inactief <duration|off> of maximale leeftijd <duration|off>"
    },
    {
      "command": "/think",
      "description": "Het denkniveau instellen",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "Uitgebreide uitvoer in- of uitschakelen",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "Snelle modus weergeven of instellen",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "Zichtbaarheid van redeneringen in- of uitschakelen",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "Verhoogde modus in- of uitschakelen",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "Standaardwaarden voor uitvoering weergeven of instellen",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "Openstaande goedkeuringsverzoeken goedkeuren of weigeren",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "Het model weergeven of instellen",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "Providers/modellen weergeven",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "De korte helpsamenvatting weergeven"
    },
    {
      "command": "/commands",
      "description": "De gegenereerde opdrachtencatalogus weergeven"
    },
    {
      "command": "/tools",
      "description": "Weergeven wat de huidige agent nu kan gebruiken",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "Runtimestatus weergeven, inclusief providergebruik/quotum indien beschikbaar"
    },
    {
      "command": "/tasks",
      "description": "Actieve/recente achtergrondtaken voor de huidige sessie weergeven"
    },
    {
      "command": "/context",
      "description": "Uitleggen hoe de context wordt samengesteld",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "Je afzenderidentiteit weergeven"
    },
    {
      "command": "/skill",
      "description": "Een skill op naam uitvoeren",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "Een zijvraag stellen zonder de sessiecontext te wijzigen",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "Codex-aanmelding koppelen",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "De gebruiksvoettekst beheren of het kostenoverzicht weergeven",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="HTTP-aanvraag-URL's">
        Gebruik dezelfde lijst `slash_commands` als bij Socket Mode hierboven en voeg `"url": "https://gateway-host.example.com/slack/events"` aan elke vermelding toe. Voorbeeld:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "Een nieuwe sessie starten",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "De korte helpsamenvatting weergeven",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        Herhaal die waarde `url` voor elke opdracht in de lijst.

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="Optionele auteurschapsbereiken (schrijfbewerkingen)">
    Voeg het botbereik `chat:write.customize` toe als je wilt dat uitgaande berichten de identiteit van de actieve agent gebruiken (aangepaste gebruikersnaam en pictogram) in plaats van de standaardidentiteit van de Slack-app.

    Als je een emoji-pictogram gebruikt, verwacht Slack de syntaxis `:emoji_name:`.

  </Accordion>
  <Accordion title="Optionele gebruikerstokenbereiken (leesbewerkingen)">
    Als je `channels.slack.userToken` configureert, zijn gebruikelijke leesbereiken:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (als je afhankelijk bent van zoekleesbewerkingen van Slack)

  </Accordion>
</AccordionGroup>

## Tokenmodel

- Botidentiteit (standaard) vereist `botToken` + `appToken` voor Socket Mode, of `botToken` + `signingSecret` voor HTTP-modus.
- Gebruikersidentiteit vereist `userToken` + `appToken` voor Socket Mode, of `userToken` + `signingSecret` voor HTTP-modus. Hierbij wordt geen bottoken gebruikt.
- Relaymodus vereist `botToken` plus `relay.url`, `relay.authToken` en `relay.gatewayId`; hierbij wordt geen apptoken of ondertekeningsgeheim gebruikt.
- `botToken`, `appToken`, `signingSecret`, `relay.authToken` en `userToken` accepteren tekenreeksen met platte tekst
  of SecretRef-objecten.
- Tokens in de configuratie overschrijven de terugval op omgevingsvariabelen.
- De terugval via de omgevingsvariabelen `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` en `SLACK_USER_TOKEN` geldt telkens alleen voor het standaardaccount.
- `userToken` gebruikt standaard alleen-lezen gedrag (`userTokenReadOnly: true`).

Gedrag van de statusmomentopname:

- Slack-accountinspectie houdt per aanmeldgegeven de velden `*Source` en `*Status` bij
  (`botToken`, `appToken`, `signingSecret`, `userToken`).
- De status is `available`, `configured_unavailable` of `missing`.
- `configured_unavailable` betekent dat het account via SecretRef
  of een andere niet-inline bron voor geheimen is geconfigureerd, maar dat het huidige opdracht-/runtimepad
  de daadwerkelijke waarde niet kon omzetten.
- In HTTP-modus wordt `signingSecretStatus` opgenomen. Socket Mode gebruikt
  `botTokenStatus` + `appTokenStatus` voor botidentiteit en
  `userTokenStatus` + `appTokenStatus` voor gebruikersidentiteit.

<Tip>
Voor botidentiteit kunnen acties en directoryleesbewerkingen bij voorkeur een optioneel gebruikerstoken gebruiken; schrijfbewerkingen blijven het bottoken gebruiken, tenzij `userTokenReadOnly: false` terugval toestaat. Voor `identity: "user"` gebruiken lees- en schrijfbewerkingen altijd `userToken`.
</Tip>

## Acties en poorten

Slack-acties worden beheerd door `channels.slack.actions.*`.

Beschikbare actiegroepen in de huidige Slack-tooling:

| Groep      | Standaard      |
| ---------- | -------------- |
| messages   | ingeschakeld   |
| reactions  | ingeschakeld   |
| pins       | ingeschakeld   |
| memberInfo | ingeschakeld   |
| emojiList  | ingeschakeld   |

De huidige Slack-berichtacties omvatten `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info` en `emoji-list`. `download-file` accepteert Slack-bestands-ID's die in tijdelijke aanduidingen voor inkomende bestanden worden weergegeven en retourneert afbeeldingsvoorbeelden voor afbeeldingen of lokale bestandsmetadata voor andere bestandstypen.

## Toegangsbeheer en routering

<Tabs>
  <Tab title="DM-beleid">
    `channels.slack.dmPolicy` beheert DM-toegang. `channels.slack.allowFrom` is de canonieke DM-toegestane lijst.

    - `pairing` (standaard)
    - `allowlist`
    - `open` (vereist dat `channels.slack.allowFrom` `"*"` bevat)
    - `disabled`

    DM-vlaggen:

    - `dm.enabled` (standaard true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (verouderd)
    - `dm.groupEnabled` (groeps-DM's standaard false)
    - `dm.groupChannels` (optionele MPIM-toegestane lijst)

    Voorrang bij meerdere accounts:

    - `channels.slack.accounts.default.allowFrom` geldt alleen voor het account `default`.
    - Benoemde accounts nemen `channels.slack.allowFrom` over wanneer hun eigen `allowFrom` niet is ingesteld.
    - Benoemde accounts nemen `channels.slack.accounts.default.allowFrom` niet over.

    De verouderde `channels.slack.dm.policy` en `channels.slack.dm.allowFrom` worden voor compatibiliteit nog steeds gelezen. `openclaw doctor --fix` migreert ze naar `dmPolicy` en `allowFrom` wanneer dat zonder wijziging van de toegang mogelijk is.

    Voor koppeling in DM's wordt `openclaw pairing approve slack <code>` gebruikt.

  </Tab>

  <Tab title="Kanaalbeleid">
    `channels.slack.groupPolicy` beheert de kanaalafhandeling:

    - `open`
    - `allowlist`
    - `disabled`

    De toegestane lijst voor kanalen bevindt zich onder `channels.slack.channels` en **moet stabiele Slack-kanaal-ID's gebruiken** (bijvoorbeeld `C12345678`) als configuratiesleutels.

    Runtime-opmerking: als `channels.slack` volledig ontbreekt (configuratie uitsluitend via omgevingsvariabelen), valt de runtime terug op `groupPolicy="allowlist"` en registreert deze een waarschuwing (zelfs als `channels.defaults.groupPolicy` is ingesteld).

    Naam-/ID-omzetting:

    - vermeldingen in de toegestane lijst voor kanalen en de toegestane lijst voor DM's worden bij het opstarten omgezet wanneer tokentoegang dit toestaat
    - niet-omgezette vermeldingen met kanaalnamen blijven geconfigureerd, maar worden standaard genegeerd voor routering
    - inkomende autorisatie en kanaalroutering gebruiken standaard eerst ID's; directe overeenkomsten met gebruikersnamen/slugs vereisen `channels.slack.dangerouslyAllowNameMatching: true`

    <Warning>
    Op namen gebaseerde sleutels (`#channel-name` of `channel-name`) komen onder `groupPolicy: "allowlist"` **niet** overeen. Het opzoeken van kanalen gebeurt standaard eerst op ID, waardoor een op namen gebaseerde sleutel nooit correct wordt gerouteerd en alle berichten in dat kanaal stilzwijgend worden geblokkeerd. Dit verschilt van `groupPolicy: "open"`, waarbij de kanaalsleutel niet vereist is voor routering en een op namen gebaseerde sleutel lijkt te werken.

    Gebruik altijd de Slack-kanaal-ID als sleutel. Je vindt deze als volgt: klik met de rechtermuisknop op het kanaal in Slack → **Copy link** — de ID (`C...`) staat aan het einde van de URL.

    Juist:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    Onjuist (stilzwijgend geblokkeerd onder `groupPolicy: "allowlist"`):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="Vermeldingen en kanaalgebruikers">
    Kanaalberichten vereisen standaard een vermelding.

    Bronnen voor vermeldingen:

    - expliciete appvermelding (`<@botId>`)
    - Slack-gebruikersgroepvermelding (`<!subteam^S...>`) wanneer de botgebruiker lid is van die gebruikersgroep; vereist `usergroups:read`
    - reguliere-expressiepatronen voor vermeldingen (`agents.entries.*.groupChat.mentionPatterns`, terugvaloptie `messages.groupChat.mentionPatterns`)
    - antwoorden op het eigen Slack-bericht van de bot (`implicitMentions.replyToBot`)
    - vervolgberichten in threads waaraan de bot heeft deelgenomen (`implicitMentions.threadParticipation`)

    Instellingen per kanaal (`channels.slack.channels.<id>`; namen alleen via omzetting bij het opstarten of `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode` (`off|first|all|batched`; overschrijft de antwoordmodus voor het account/chattype voor dit kanaal)
    - `users` (toelatingslijst)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - sleutelindeling voor `toolsBySender`: `channel:`, `id:`, `e164:`, `username:`, `name:` of het jokerteken `"*"`
      (verouderde sleutels zonder voorvoegsel worden nog steeds alleen aan `id:` gekoppeld)

    `ignoreOtherMentions` (standaard `false`) negeert kanaalberichten die een andere gebruiker of gebruikersgroep vermelden, maar niet deze bot. Privéberichten en groepsprivéberichten (MPIM's) worden niet beïnvloed. Het filter vereist een omgezette botgebruikers-ID uit `auth.test`; als die identiteit niet beschikbaar is (bijvoorbeeld bij een identiteit met alleen een gebruikerstoken), wordt de beperking open overgeslagen en worden berichten ongewijzigd doorgelaten.

    `allowBots` hanteert een behoudende aanpak voor kanalen en privékanalen: door bots geschreven ruimteberichten worden alleen geaccepteerd wanneer de verzendende bot expliciet voorkomt in de `users`-toelatingslijst van die ruimte, of wanneer ten minste één expliciete Slack-eigenaars-ID uit `channels.slack.allowFrom` momenteel lid is van de ruimte. Jokertekens en eigenaarsvermeldingen met weergavenamen gelden niet als aanwezigheid van een eigenaar. De aanwezigheid van een eigenaar gebruikt Slack `conversations.members`; zorg dat de app het bijbehorende leesbereik voor het ruimtetype heeft (`channels:read` voor openbare kanalen, `groups:read` voor privékanalen). Als het opzoeken van leden mislukt, negeert OpenClaw het door een bot geschreven ruimtebericht.

    Geaccepteerde door bots geschreven Slack-berichten gebruiken gedeelde [bescherming tegen botlussen](/nl/channels/bot-loop-protection). Configureer `channels.defaults.botLoopProtection` voor het standaardbudget en overschrijf dit vervolgens met `channels.slack.botLoopProtection` of `channels.slack.channels.<id>.botLoopProtection` wanneer een werkruimte of kanaal een andere limiet nodig heeft.

  </Tab>
</Tabs>

## Threads, sessies en antwoordtags

- Privéberichten worden gerouteerd als `direct`; kanalen als `channel`; MPIM's als `group`.
- Slack-routekoppelingen accepteren onbewerkte peer-ID's plus Slack-doelvormen zoals `channel:C12345678`, `user:U12345678` en `<@U12345678>`.
- Met de standaardwaarde `session.dmScope=main` worden gewone Slack-privéberichten samengevoegd in de hoofdsessie van de agent. Hoofdelementen van Agent View en bestaande threads van Assistant View blijven geïsoleerd als `:thread:<threadTs>`-sessies.
- Kanaalsessies: `agent:<agentId>:slack:channel:<channelId>`.
- Gewone kanaalberichten op het hoogste niveau blijven in de sessie per kanaal, zelfs wanneer `replyToMode` niet `off` is.
- Antwoorden in threads van Slack-kanalen, MPIM's, Agent View en Assistant View gebruiken de bovenliggende Slack-`thread_ts` voor sessieachtervoegsels (`:thread:<threadTs>`). Gewone antwoordthreads in privéberichten blijven een UI-voorziening binnen de basisprivéberichtsessie.
- OpenClaw voegt een geschikt kanaalhoofdelement op het hoogste niveau toe aan `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>` wanneer wordt verwacht dat dit hoofdelement een zichtbare Slack-thread start, zodat het hoofdelement en latere antwoorden in de thread één OpenClaw-sessie delen. Dit geldt voor `app_mention`-gebeurtenissen, expliciete botvermeldingen of overeenkomsten met geconfigureerde vermeldingspatronen, en `requireMention: false`-kanalen met een niet-`off` `replyToMode`.
- De standaardwaarde van `channels.slack.thread.historyScope` is `thread`; de standaardwaarde van `thread.inheritParent` is `false`.
- `channels.slack.thread.initialHistoryLimit` bepaalt hoeveel bestaande threadberichten worden opgehaald wanneer een nieuwe threadsessie begint (standaard `20`; stel in op `0` om uit te schakelen).
- `channels.slack.implicitMentions.replyToBot` bepaalt of een antwoord op het eigen bericht van de bot de vereiste vermelding omzeilt (standaard `true`).
- `channels.slack.implicitMentions.threadParticipation` bepaalt of vervolgberichten in een thread waarin de bot heeft geantwoord de vereiste vermelding omzeilen (standaard `true`). Stel dit in op `false` om in die vervolgberichten een nieuwe expliciete vermelding te vereisen. `openclaw doctor --fix` migreert de voormalige sleutel `channels.slack.thread.requireExplicitMention` naar deze positieve canonieke vlag.
- Accountoverschrijvingen staan onder `channels.slack.accounts.<id>.implicitMentions`; gedeelde standaardwaarden staan onder `channels.defaults.implicitMentions`.

Instellingen voor antwoordthreads:

- `channels.slack.channels.<id>.replyToMode`: overschrijving per kanaal voor berichten in Slack-kanalen/privékanalen
- `channels.slack.replyToMode`: `off|first|all|batched` (standaard `off`)
- `channels.slack.replyToModeByChatType`: per `direct|group|channel`
- verouderde terugvaloptie voor directe chats: `channels.slack.dm.replyToMode`

Handmatige antwoordtags worden ondersteund:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

Stel voor expliciete Slack-threadantwoorden vanuit het hulpprogramma `message` `replyBroadcast: true` in met `action: "send"` en `threadId` of `replyTo` om Slack te vragen het threadantwoord ook naar het bovenliggende kanaal te verspreiden. Dit wordt gekoppeld aan Slacks `chat.postMessage`-vlag `reply_broadcast` en wordt alleen ondersteund voor verzendingen met tekst of Block Kit, niet voor media-uploads.

Wanneer een aanroep van het hulpprogramma `message` binnen een Slack-thread wordt uitgevoerd en op hetzelfde kanaal is gericht, neemt OpenClaw normaal gesproken de huidige Slack-thread over volgens de effectieve `replyToMode` voor het account, chattype of kanaal. Automatische antwoorden en aanroepen van `send` of `upload-file` binnen hetzelfde kanaal gebruiken dezelfde overschrijving per kanaal. Stel `topLevel: true` in op `action: "send"` of `action: "upload-file"` om in plaats daarvan een nieuw bericht in het bovenliggende kanaal af te dwingen. `threadId: null` wordt geaccepteerd als dezelfde afmelding op het hoogste niveau.

<Note>
`replyToMode="off"` schakelt optionele uitgaande Slack-antwoordthreads uit, inclusief expliciete `[[reply_to_*]]`-tags. Agent View en Assistant View zijn door Slack beheerde threadervaringen, waardoor hun antwoorden en status ongeacht deze instelling op het zichtbare hoofdelement blijven. Andere binnenkomende Slack-threadsessies worden hierdoor niet afgevlakt. Dit verschilt van Telegram, waar expliciete tags nog steeds worden gehonoreerd in de modus `"off"`. Slack-threads verbergen berichten voor het kanaal, terwijl Telegram-antwoorden inline zichtbaar blijven.
</Note>

## Bevestigingsreacties

`ackReaction` verzendt een bevestigingsemoji terwijl OpenClaw een binnenkomend bericht verwerkt. `ackReactionScope` bepaalt _wanneer_ die emoji daadwerkelijk wordt verzonden.

Standaard blijft de bevestiging statisch terwijl de ingebouwde threadstatus van de Slack-agent/-assistent de voortgang toont met wisselende laadberichten. Stel `messages.statusReactions.enabled: true` in om in plaats daarvan de reactielevenscyclus in wachtrij/nadenken/hulpprogramma/voltooid/fout te gebruiken.

### Emoji (`ackReaction`)

Volgorde van omzetting:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- terugvalemoji voor de agentidentiteit (`agents.entries.*.identity.emoji`, anders `"eyes"` / 👀)

Opmerkingen:

- Slack verwacht shortcodes (bijvoorbeeld `"eyes"`).
- Gebruik `""` om de reactie voor het Slack-account of globaal uit te schakelen.

### Bereik (`messages.ackReactionScope`)

De Slack-provider leest het bereik uit `messages.ackReactionScope` (standaard `"group-mentions"`). Momenteel bestaat er geen overschrijving op Slack-account- of Slack-kanaalniveau; de waarde geldt globaal voor de Gateway.

Waarden:

- `"all"`: reageer in privéberichten en groepen, inclusief omgevingsgebeurtenissen in ruimtes.
- `"direct"`: reageer alleen in privéberichten.
- `"group-all"`: reageer op elk groepsbericht behalve omgevingsgebeurtenissen in ruimtes (geen privéberichten).
- `"group-mentions"` (standaard): reageer in groepen, maar alleen wanneer de bot wordt vermeld (of in groepsvermeldingen waarvoor deelname is ingeschakeld). **Privéberichten zijn uitgesloten.**
- `"off"` / `"none"`: reageer nooit.

<Note>
Het standaardbereik (`"group-mentions"`) activeert geen bevestigingsreacties in directe berichten of omgevingsgebeurtenissen in ruimtes. Om de geconfigureerde `ackReaction` (bijvoorbeeld `"eyes"`) te zien bij binnenkomende Slack-privéberichten en stille ruimtegebeurtenissen, stel je `messages.ackReactionScope` in op `"all"`. `messages.ackReactionScope` wordt gelezen wanneer de Slack-provider opstart. De Gateway moet daarom opnieuw worden gestart voordat de wijziging van kracht wordt.
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // reageer in privéberichten en groepen
  },
}
```

## Tekststreaming

`channels.slack.streaming` bepaalt het gedrag van livevoorbeelden:

- `off`: schakel het streamen van livevoorbeelden uit.
- `partial` (standaard): vervang de voorbeeldtekst door de nieuwste gedeeltelijke uitvoer.
- `block`: voeg voorbeeldupdates in delen toe.
- `progress`: toon voortgangsstatustekst tijdens het genereren en verzend daarna de definitieve tekst.
- `streaming.preview.toolProgress`: wanneer een conceptvoorbeeld actief is, worden updates van hulpprogramma's en voortgang naar hetzelfde bewerkte voorbeeldbericht gerouteerd (standaard: `true`). Stel `false` in om afzonderlijke berichten voor hulpprogramma's en voortgang te behouden.
- `streaming.preview.commandText` / `streaming.progress.commandText`: stel in op `status` om compacte voortgangsregels voor hulpprogramma's te behouden en onbewerkte opdracht-/uitvoeringstekst te verbergen (standaard: `raw`).

Verberg onbewerkte opdracht-/uitvoeringstekst en behoud compacte voortgangsregels:

```json
{
  "channels": {
    "slack": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

`channels.slack.streaming.nativeTransport` bepaalt het ingebouwde streamen van tekst in Slack wanneer `channels.slack.streaming.mode` `partial` is (standaard: `true`).

Ingebouwde Slack-taakkaarten voor voortgang moeten expliciet worden ingeschakeld voor de voortgangsmodus. Stel `channels.slack.streaming.progress.nativeTaskCards` in op `true` met `channels.slack.streaming.mode="progress"` om tijdens het werk een ingebouwde plan-/taakkaart van Slack te verzenden en dezelfde taakkaart na voltooiing bij te werken. Zonder deze vlag behoudt de voortgangsmodus het overdraagbare gedrag voor conceptvoorbeelden.

- Er moet een antwoordthread beschikbaar zijn om native tekststreaming en de Slack-assistentthreadstatus weer te geven. De threadselectie volgt nog steeds `replyToMode`.
- Kanalen, groepschats en DM-hoofdberichten kunnen nog steeds het normale conceptvoorbeeld gebruiken wanneer native streaming niet beschikbaar is of er geen antwoordthread bestaat.
- Slack-DM's op hoofdniveau blijven standaard buiten een thread, zodat ze Slacks threadachtige native streaming-/statusvoorbeeld niet weergeven; in plaats daarvan plaatst en bewerkt OpenClaw een conceptvoorbeeld in de DM.
- Media en niet-tekstuele payloads vallen terug op normale aflevering.
- Definitieve media-/foutpayloads annuleren wachtende voorbeeldbewerkingen; geschikte definitieve tekst-/blokpayloads worden alleen verzonden wanneer ze het voorbeeld ter plaatse kunnen bewerken.
- Als streaming halverwege een antwoord mislukt, valt OpenClaw voor de resterende payloads terug op normale aflevering.

Gebruik een conceptvoorbeeld in plaats van native tekststreaming van Slack:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

Schakel native Slack-taakkaarten voor voortgang in:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

Verouderde sleutels:

- `channels.slack.streamMode` (`replace | status_final | append`) is een verouderde alias voor `channels.slack.streaming.mode`.
- boolean `channels.slack.streaming` is een verouderde alias voor `channels.slack.streaming.mode` en `channels.slack.streaming.nativeTransport`.
- `channels.slack.chunkMode` en `channels.slack.nativeStreaming` op hoofdniveau zijn verouderde aliassen voor `channels.slack.streaming.chunkMode` en `channels.slack.streaming.nativeTransport`.
- Verouderde aliassen worden tijdens runtime niet gelezen; voer `openclaw doctor --fix` uit om de opgeslagen Slack-streamingconfiguratie te herschrijven naar de canonieke sleutels.

## Terugvalreactie voor typen

`typingReaction` voegt tijdelijk een reactie toe aan het binnenkomende Slack-bericht terwijl OpenClaw een antwoord verwerkt en verwijdert deze wanneer de uitvoering is voltooid. Dit is vooral nuttig buiten antwoorden in threads, die standaard de statusindicator "is aan het typen..." gebruiken.

Volgorde van omzetting:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

Opmerkingen:

- Slack verwacht shortcodes (bijvoorbeeld `"hourglass_flowing_sand"`).
- De reactie is op basis van beste inspanning en er wordt automatisch geprobeerd deze op te ruimen nadat het antwoord- of foutpad is voltooid.

## Spraakinvoer

Als je OpenClaw momenteel in Slack met spraak wilt gebruiken, stuur je een Slack-audiofragment naar de OpenClaw-app. De dicteermicrofoon van Slackbot is een afzonderlijke functie van Slack en geen app-API.

- **[Spraakdicteren met Slackbot](https://slack.com/help/articles/202026038-How-to-use-Slackbot)** vindt plaats in het privégesprek van de gebruiker met Slackbot. Slack zet de opname om in een Slackbot-prompt, maar verstuurt via de Events API geen audiobestand, dicteergebeurtenis, prompt of invoerbronmarkering naar Slack-apps van derden. De OpenClaw-Plugin voor Slack kan deze functie niet inschakelen of ontvangen.
- **[Slack-audiofragmenten](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)** zijn opgeslagen Slack-bestanden die in een OpenClaw-DM, kanaal of thread kunnen worden geplaatst. OpenClaw downloadt een toegankelijk fragment met het bottoken, normaliseert Slacks MIME-metadata voor fragmenten en stuurt het door de gedeelde [pijplijn voor audiotranscriptie](/nl/nodes/audio). Het aanbevolen app-manifest bevat het vereiste bereik `files:read`.

Audiofragmenten en Slackbot-dicteren hebben verschillende privacykenmerken: fragmenten vallen onder Slacks bewaarbeleid voor bestanden en OpenClaw downloadt ze voor transcriptie, terwijl Slack aangeeft dat dicteeraudio niet wordt opgeslagen.

In een kanaal met `requireMention: true` kan een audiofragment zonder bijschrift aan de voorwaarde voldoen door een geconfigureerd vermeldingspatroon uit te spreken (`agents.entries.*.groupChat.mentionPatterns`, met terugval op `messages.groupChat.mentionPatterns`). OpenClaw autoriseert de afzender voordat het fragment wordt gedownload of getranscribeerd en laat het vervolgens alleen toe wanneer het transcript overeenkomt. Een mislukt of niet-overeenkomend voorlopig transcript wordt samen met het gedownloade fragment verwijderd; het wordt niet bewaard in de kanaalgeschiedenis. De native Slack-identiteit `@bot` kan niet uit spraak worden afgeleid, dus configureer een patroon voor een gesproken naam of voeg een getypte vermelding toe. Als het terugsturen van het transcript is ingeschakeld, wordt dit pas na toelating verzonden.

## Media, opsplitsing en aflevering

<AccordionGroup>
  <Accordion title="Binnenkomende bijlagen">
    Slack-bestandsbijlagen worden gedownload van door Slack gehoste privé-URL's (via een met een token geauthenticeerde aanvraag) en naar de mediaopslag geschreven wanneer het ophalen slaagt en de groottelimieten dit toestaan. Bestandsplaatsaanduidingen bevatten de Slack-`fileId`, zodat agents het oorspronkelijke bestand kunnen ophalen met `download-file`.

    Downloads gebruiken begrensde time-outs voor inactiviteit en totale duur. Als het ophalen van Slack-bestanden vastloopt of mislukt, blijft OpenClaw het bericht verwerken en valt het terug op de bestandsplaatsaanduiding.

    De limiet voor de grootte van binnenkomende runtimegegevens is standaard `20MB`, tenzij deze wordt overschreven door `channels.slack.mediaMaxMb`.

  </Accordion>

  <Accordion title="Uitgaande tekst en bestanden">
    - tekstfragmenten gebruiken `channels.slack.textChunkLimit` (standaard `8000`, begrensd op Slacks eigen limiet voor berichtlengte)
    - `channels.slack.streaming.chunkMode="newline"` schakelt opsplitsing met voorrang voor alinea's in
    - bestanden worden verzonden via Slacks upload-API's en kunnen antwoorden in threads bevatten (`thread_ts`)
    - lange bestandsbijschriften gebruiken het eerste voor Slack veilige tekstfragment als uploadopmerking en verzenden de resterende fragmenten als vervolgberichten
    - de limiet voor uitgaande media volgt `channels.slack.mediaMaxMb` wanneer deze is geconfigureerd; anders gebruiken kanaalverzendingen de standaardwaarden per MIME-type uit de mediapijplijn

  </Accordion>

  <Accordion title="Afleveringsdoelen">
    Voorkeursdoelen die expliciet zijn opgegeven:

    - `user:<id>` voor DM's
    - `channel:<id>` voor kanalen

    Slack-DM's met alleen tekst/blokken kunnen rechtstreeks naar gebruikers-ID's worden geplaatst; bij bestandsuploads en verzendingen in threads wordt de DM eerst via Slacks conversatie-API's geopend, omdat voor deze paden een concrete conversatie-ID vereist is.

  </Accordion>
</AccordionGroup>

## Opdrachten en slashgedrag

Slashopdrachten verschijnen in Slack als één geconfigureerde opdracht of als meerdere native opdrachten. Configureer `channels.slack.slashCommand` om de standaardwaarden voor opdrachten te wijzigen:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

Voor native opdrachten zijn [aanvullende manifestinstellingen](#additional-manifest-settings) in je Slack-app vereist. Ze worden ingeschakeld met `channels.slack.commands.native: true` of `commands.native: true` in algemene configuraties.

- De automatische modus voor native opdrachten staat voor Slack **uit**, zodat `commands.native: "auto"` native Slack-opdrachten niet inschakelt.

```txt
/help
```

Menu's met native argumenten worden in de volgende prioriteitsvolgorde weergegeven:

- 3-5 opties die kort genoeg zijn: een overloopmenu ("...")
- meer dan 100 opties, wanneer asynchrone filtering van opties beschikbaar is: externe selectie
- 1-2 opties, of een optie waarvan de gecodeerde waarde te lang is voor een selectie: knopblokken
- anders (6-100 opties, of meer dan 100 zonder asynchrone filtering): statisch selectiemenu, opgesplitst in 100 opties per menu

```txt
/think
```

Slashsessies gebruiken geïsoleerde sleutels zoals `agent:<agentId>:slack:slash:<userId>` en leiden de uitvoering van opdrachten nog steeds met `CommandTargetSessionKey` naar de sessie van het doelgesprek.

## Native grafieken

Slacks openbare [`data_visualization` Block Kit-blok](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
geeft lijn-, staaf-, vlak- en cirkeldiagrammen weer in berichten. OpenClaw zet het overdraagbare
`presentation` `chart`-blok om naar die native vorm; buiten de normale
`chat:write`-berichttoegang zijn geen aanvullend OAuth-bereik,
bestandsupload, afbeeldingsrenderer of Slack-configuratie vereist.

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "Kwartaalomzet",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "Omzet", "values": [120, 145] }],
      "xLabel": "Kwartaal"
    }
  ]
}
```

Slacks limieten worden vóór native weergave afgedwongen:

- titel en optionele aslabels: 50 tekens
- cirkel: 1-12 positieve segmenten
- lijn/staaf/vlak: 1-12 reeksen met unieke namen en 1-20 gedeelde categorieën
- labels voor segmenten, categorieën en reeksen: 20 tekens
- elke reeks moet voor elke categorie één eindige waarde bevatten; niet-cirkelwaarden
  mogen negatief zijn

Elke native grafiek bevat ook een tekstuele weergave op hoofdniveau voor
schermlezers, meldingen, sessiespiegeling en clients die het blok niet kunnen
weergeven. Standaardpresentaties die naar andere OpenClaw-kanalen worden verzonden, ontvangen
dezelfde deterministische grafiekgegevens als tekst, tenzij ze ondersteuning voor native grafieken
aangeven. Als Slack tijdens een gefaseerde uitrol de grafiek afwijst met `invalid_blocks`, verwijdert OpenClaw
de afgewezen native gegevensblokken, behoudt het eventuele aangrenzende bedieningselementen en verzendt het
de volledige grafiekweergave als zichtbare tekst.

Slack accepteert momenteel maximaal twee `data_visualization`-blokken per bericht. Wanneer
een presentatie meer dan twee geldige grafieken bevat, behoudt OpenClaw hun volgorde
en gaat de native weergave verder in vervolgberichten, met maximaal twee
grafieken in elk bericht.

Slacks [lancering voor ontwikkelaars](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
beschrijft het blok als een appgerichte Block Kit-functie en vermeldt geen beperking
tot betaalde abonnementen. De tekst over beschikbaarheid voor Business+/Enterprise is van toepassing op
Slakbots automatische AI-grafiekgeneratie, die losstaat van een app die
een reeds gestructureerde Block Kit-grafiek verzendt. Grafieken zijn uitsluitend berichtblokken, geen inhoud
voor App Home, modals of Canvas.

## Native tabellen

Slacks huidige [`data_table` Block Kit-blok](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
geeft gestructureerde rijen en kolommen weer in berichten. OpenClaw zet een expliciet
overdraagbaar `presentation` `table`-blok om naar `data_table`; het gebruikt niet Slacks
verouderde [`table`-blok](https://docs.slack.dev/reference/block-kit/blocks/table-block/).
Buiten de normale `chat:write`-berichttoegang is geen aanvullend OAuth-bereik of
Slack-configuratie vereist.

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "Open pijplijn",
      "headers": ["Account", "Fase", "ARR"],
      "rows": [
        ["Acme", "Gewonnen", 125000],
        ["Globex", "Beoordeling", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw zet koptekst- en tekenreekscellen om naar Slack-`raw_text`-cellen. Numerieke cellen
worden omgezet naar `raw_number`, waarbij de eindige numerieke waarde behouden blijft voor native sortering
en filtering. `rowHeaderColumnIndex` markeert, indien aanwezig, die op nul gebaseerde
kolom als Slack-rijkoppen.

Slacks gepubliceerde `data_table`-limieten worden vóór native weergave afgedwongen:

- 1-20 kolommen
- 1-100 gegevensrijen, plus de koprij
- hetzelfde aantal cellen in elke rij
- maximaal 10.000 tekens in totaal voor alle tabelcellen in één bericht

Meerdere geldige tabelblokken kunnen native worden weergegeven zolang het bericht
binnen de totale tekenlimiet blijft. Een tabel die niet binnen de
native grenzen kan worden weergegeven, wordt volledige deterministische tekst in plaats van rijen of
cellen te verliezen. Als die tekst langer is dan één Slack-bericht, gebruiken verzendingen en slashantwoorden
geordende tekstfragmenten. Tabelbewerkingen mislukken met een expliciete groottefout in plaats van
stilzwijgend rijen uit een bestaand bericht af te kappen.

Elke systeemeigen tabel die uit draagbare presentatie wordt gegenereerd, bevat ook een tekstweergave op het hoogste niveau
voor schermlezers, meldingen, sessiespiegeling en
clients die het blok niet kunnen weergeven. Ruwe diagram- en tabelwaarden blijven letterlijk
in de terugvalweergave, zodat celgegevens zoals `<@U123>` geen Slack-vermelding worden.
Als Slack systeemeigen diagram- of tabelblokken weigert met `invalid_blocks`, verwijdert OpenClaw
alle systeemeigen gegevensblokken in één begrensde herstelstap, behoudt het geldige
naastliggende blokken zoals knoppen en selecties, en verzendt het volledige zichtbare diagram-
en tabeltekst met Slack-opmaak uitgeschakeld. Levering via slash-opdrachten
houdt voor de hele opdracht het budget van vijf aanroepen van Slack voor `response_url` bij. Vóór elke
antwoordbatch selecteert OpenClaw een volledig plan dat binnen de resterende aanroepen past, of mislukt het
voordat die batch wordt geplaatst.

Alleen expliciete `presentation`-tabelblokken worden naar systeemeigen tabellen gepromoveerd.
Markdown-tabellen met sluistekens blijven geschreven tekst; OpenClaw doet geen aannames over de tabelstructuur
of celtypen. Bestaande vertrouwde producenten van systeemeigen Slack-inhoud kunnen ruwe blokken blijven
doorgeven via `channelData.slack.blocks`; OpenClaw leidt terugvaltekst
af uit geldige ruwe `data_table`-cellen, terwijl onjuist gevormde aangepaste blokken mogelijk
terugvallen op hun bijschrift of de algemene Block Kit-terugvalweergave. Draagbare uitvoer van agents, de CLI
en plugins moet `presentation` gebruiken.

## Interactieve antwoorden

Slack kan door agents gemaakte interactieve antwoordbedieningselementen weergeven, maar deze functie is standaard uitgeschakeld.
Geef voor nieuwe uitvoer van agents, de CLI en plugins de voorkeur aan de gedeelde
`presentation`-knoppen of selectieblokken. Ze gebruiken hetzelfde Slack-interactiepad
en vallen ook bruikbaar terug op andere kanalen.

Schakel dit globaal in:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

Of schakel dit alleen voor één Slack-account in:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

Als dit is ingeschakeld, kunnen agents nog steeds verouderde, uitsluitend voor Slack bestemde antwoordrichtlijnen uitvoeren:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

Deze richtlijnen worden gecompileerd naar Slack Block Kit en sturen klikken of selecties
terug via het bestaande gebeurtenispad voor Slack-interacties. Behoud ze voor oude
prompts en Slack-specifieke uitwijkmogelijkheden; gebruik gedeelde presentatie voor nieuwe
draagbare bedieningselementen.

De API's van de richtlijncompiler zijn ook verouderd voor nieuwe producentcode:

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

Gebruik `presentation`-payloads en `buildSlackPresentationBlocks(...)` voor nieuwe
in Slack weergegeven bedieningselementen.

Opmerkingen:

- Dit is verouderde Slack-specifieke UI. Andere kanalen vertalen Slack Block
  Kit-richtlijnen niet naar hun eigen knopsystemen.
- De interactieve callbackwaarden zijn door OpenClaw gegenereerde ondoorzichtige tokens, geen ruwe door agents gemaakte waarden.
- Als gegenereerde interactieve blokken de limieten van Slack Block Kit zouden overschrijden, valt OpenClaw terug op het oorspronkelijke tekstantwoord in plaats van een ongeldige blokkenpayload te verzenden.

### Door plugins beheerde modale inzendingen

Slack-plugins die een interactieve handler registreren, kunnen ook modale
`view_submission`- en `view_closed`-levenscyclusgebeurtenissen ontvangen voordat OpenClaw
de payload comprimeert voor de systeemgebeurtenis die voor de agent zichtbaar is. Gebruik een van deze routeringspatronen
bij het openen van een modaal venster in Slack:

- Stel `callback_id` in op `openclaw:<namespace>:<payload>`.
- Of behoud een bestaande `callback_id` en plaats `pluginInteractiveData:
"<namespace>:<payload>"` in de modale `private_metadata`.

De handler ontvangt `ctx.interaction.kind` als `view_submission` of
`view_closed`, genormaliseerde `inputs` en het volledige ruwe `stateValues`-object van
Slack. Routering uitsluitend op callback-id volstaat om de pluginhandler aan te roepen; neem
de bestaande modale `private_metadata`-routeringsvelden voor gebruiker/sessie op wanneer het
modale venster ook een voor de agent zichtbare systeemgebeurtenis moet produceren. De agent ontvangt een
compacte, geredigeerde `Slack interaction: ...`-systeemgebeurtenis. Als de handler
`systemEvent.summary`, `systemEvent.reference` of `systemEvent.data` retourneert, worden die
velden opgenomen in die compacte gebeurtenis, zodat de agent kan verwijzen naar
door plugins beheerde opslag zonder de volledige formulierpayload te zien.

## Systeemeigen goedkeuringen in Slack

Slack kan fungeren als systeemeigen goedkeuringsclient met interactieve knoppen en interacties, in plaats van terug te vallen op de web-UI of terminal.

- Uitvoerings- en plugingoedkeuringen kunnen worden weergegeven als systeemeigen Slack-prompts in Block Kit.
- `channels.slack.execApprovals.*` blijft de configuratie voor inschakeling van de systeemeigen goedkeuringsclient voor uitvoering en routering naar DM/kanaal.
- DM's voor uitvoeringsgoedkeuring gebruiken `channels.slack.execApprovals.approvers` of `commands.ownerAllowFrom`.
- Plugingoedkeuringen gebruiken systeemeigen Slack-knoppen wanneer Slack is ingeschakeld als systeemeigen goedkeuringsclient voor de oorspronkelijke sessie, of wanneer `approvals.plugin` naar de oorspronkelijke Slack-sessie of een Slack-doel routeert.
- DM's voor plugingoedkeuring gebruiken Slack-plugingoedkeurders uit `channels.slack.allowFrom`, `allowFrom` voor een benoemd account, of de standaardroute van het account.
- Autorisatie van goedkeurders wordt nog steeds afgedwongen: goedkeurders die alleen uitvoering mogen goedkeuren, kunnen pluginverzoeken niet goedkeuren tenzij ze ook plugingoedkeurders zijn.

Dit gebruikt hetzelfde gedeelde oppervlak voor goedkeuringsknoppen als andere kanalen. Wanneer `interactivity` is ingeschakeld in de instellingen van je Slack-app, worden goedkeuringsprompts rechtstreeks als Block Kit-knoppen in het gesprek weergegeven.
Wanneer die knoppen aanwezig zijn, vormen ze de primaire goedkeurings-UX; OpenClaw
mag alleen een handmatige `/approve`-opdracht opnemen wanneer het gereedschapsresultaat aangeeft dat chatgoedkeuringen
niet beschikbaar zijn of handmatige goedkeuring de enige mogelijkheid is.

Configuratiepad:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (optioneel; valt indien mogelijk terug op `commands.ownerAllowFrom`)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, standaard: `dm`)
- `agentFilter`, `sessionFilter`

Slack schakelt systeemeigen uitvoeringsgoedkeuringen automatisch in wanneer `enabled` niet is ingesteld of `"auto"` is, en ten minste één
uitvoeringsgoedkeurder kan worden bepaald. Slack kan via dit systeemeigen-clientpad ook systeemeigen plugingoedkeuringen afhandelen
wanneer Slack-plugingoedkeurders kunnen worden bepaald en het verzoek overeenkomt met de systeemeigen-clientfilters. Stel
`enabled: false` in om Slack expliciet uit te schakelen als systeemeigen goedkeuringsclient. Stel `enabled: true` in om
systeemeigen goedkeuringen geforceerd in te schakelen wanneer goedkeurders kunnen worden bepaald. Het uitschakelen van Slack-uitvoeringsgoedkeuringen schakelt
de levering van systeemeigen Slack-plugingoedkeuringen die via `approvals.plugin` is ingeschakeld niet uit; voor de levering van plugingoedkeuringen
worden in plaats daarvan Slack-plugingoedkeurders gebruikt.

Standaardgedrag zonder expliciete Slack-configuratie voor uitvoeringsgoedkeuring:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

Expliciete systeemeigen Slack-configuratie is alleen nodig wanneer je goedkeurders wilt overschrijven, filters wilt toevoegen of
levering in de oorspronkelijke chat wilt inschakelen:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

Gedeeld doorsturen via `approvals.exec` staat hiervan los. Gebruik dit alleen wanneer prompts voor uitvoeringsgoedkeuring ook
naar andere chats of expliciete externe doelen moeten worden gerouteerd. Gedeeld doorsturen via `approvals.plugin` staat eveneens
los hiervan; systeemeigen Slack-levering onderdrukt die terugval alleen wanneer Slack het verzoek om
plugingoedkeuring systeemeigen kan afhandelen.

`/approve` in dezelfde chat werkt ook in Slack-kanalen en DM's die al opdrachten ondersteunen. Zie [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals) voor het volledige model voor het doorsturen van goedkeuringen.

## Gebeurtenissen en operationeel gedrag

- Bewerkingen/verwijderingen van berichten worden omgezet in systeemgebeurtenissen.
- Threaduitzendingen (threadantwoorden met "Also send to channel") worden verwerkt als normale gebruikersberichten.
- Gebeurtenissen voor het toevoegen/verwijderen van reacties worden omgezet in systeemgebeurtenissen.
- Gebeurtenissen voor het toetreden/verlaten van leden, het maken/hernoemen van kanalen en het toevoegen/verwijderen van pins worden omgezet in systeemgebeurtenissen.
- Optionele aanwezigheidspeiling kan een waargenomen overgang van `away` naar `active` van een menselijke deelnemer omzetten in een gebeurtenis in de meest recent actieve, geschikte Slack-sessie van die deelnemer. Dit is standaard uitgeschakeld.
- `channel_id_changed` kan kanaalconfiguratiesleutels migreren wanneer `configWrites` is ingeschakeld.
- Metadata voor kanaalonderwerp/-doel wordt behandeld als niet-vertrouwde context en kan in de routeringscontext worden geïnjecteerd.
- Agent View-`app_context`-entiteiten worden gevalideerd in de relevantievolgorde van Slack en uitsluitend beschikbaar gesteld als gestructureerde, niet-vertrouwde context; bij een ontbrekende context wordt de beurt gewist in plaats van verouderde entiteiten opnieuw te gebruiken.
- De threadstarter en initiële contextvulling uit de threadgeschiedenis worden, indien van toepassing, gefilterd op basis van geconfigureerde afzenderstoegestane lijsten.
- Blokacties, snelkoppelingen en modale interacties genereren gestructureerde `Slack interaction: ...`-systeemgebeurtenissen met uitgebreide payloadvelden:
  - blokacties: geselecteerde waarden, labels, kiezerwaarden en `workflow_*`-metadata
  - globale snelkoppelingen: callback- en actormetadata, gerouteerd naar de directe sessie van de actor
  - berichtsnelkoppelingen: callback, actor, kanaal, thread en context van het geselecteerde bericht
  - modale `view_submission`- en `view_closed`-gebeurtenissen met gerouteerde kanaalmetadata en formulierinvoer

Definieer globale of berichtsnelkoppelingen in de configuratie van je Slack-app en gebruik een niet-lege callback-ID. OpenClaw bevestigt overeenkomende snelkoppelingspayloads, past hetzelfde afzenderbeleid voor DM's/kanalen toe als voor andere Slack-interacties en plaatst de opgeschoonde gebeurtenis in de wachtrij voor de gerouteerde agentsessie. Trigger-ID's en antwoord-URL's worden uit de agentcontext geredigeerd.

### Aanwezigheidsgebeurtenissen

Slack verstuurt aanwezigheidswijzigingen niet via de Events API of Socket Mode. OpenClaw kan in plaats daarvan [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) peilen voor menselijke deelnemers van wie de berichten de normale Slack-toegangs- en routeringscontroles hebben doorstaan.

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off` (standaard): geen aanwezigheidstimer of Slack-API-aanroepen.
- `auto`: bewaak DM's, MPIM's en Slack-threads die in de afgelopen 24 uur actief waren, met maximaal 8 waargenomen menselijke deelnemers. Kanaalsessies op het hoogste niveau zijn uitgesloten.
- `on`: bewaak dezelfde gesprekken zonder deelnemerslimiet en neem kanaalsessies op het hoogste niveau mee. Gebruik een overschrijving per kanaal om één kanaal af te dwingen of te onderdrukken.

OpenClaw peilt per Slack-account maximaal 45 unieke gebruikers per minuut, initialiseert het eerste resultaat zonder de agent te activeren en activeert de agent alleen bij een waargenomen overgang van `away` naar `active`. Per Slack-account en gebruiker geldt een duurzame afkoelperiode van 8 uur, zelfs als die persoon aan meerdere threads deelneemt. De gebeurtenis wordt alleen naar het meest recent actieve, geschikte gesprek van die persoon gerouteerd en instrueert de agent om het geheugen/de wiki en bekende tijdzonecontext te raadplegen voordat deze beslist of één korte begroeting wordt verzonden. De agent mag stil blijven.

Het bottoken heeft `users:read` nodig, dat al in het aanbevolen manifest is opgenomen. Aanwezigheidsgebeurtenissen zijn niet beschikbaar voor organisatiebrede Enterprise Grid-installaties.

## Configuratiereferentie

Primaire referentie: [Configuratiereferentie - Slack](/nl/gateway/config-channels#slack).

<Accordion title="Belangrijkste Slack-velden">

- modus/authenticatie: `identity`, `mode`, `enterpriseOrgInstall`, `botToken`, `appToken`, `userToken`, `signingSecret`, `webhookPath`, `accounts.*`
- DM-toegang: `dm.enabled`, `dmPolicy`, `allowFrom` (verouderd: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
- compatibiliteitsschakelaar: `dangerouslyAllowNameMatching` (noodvoorziening; uitgeschakeld laten tenzij nodig)
- kanaaltoegang: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`, `implicitMentions.*`
- threads/geschiedenis: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- activering door aanwezigheid: `presenceEvents.mode`, `channels.*.presenceEvents.mode` (`off|auto|on`; standaard `off`)
- bezorging: `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `streaming`, `streaming.nativeTransport`, `streaming.preview.toolProgress`
- voorvertoningen: `unfurlLinks` (standaard: `false`), `unfurlMedia` voor beheer van link-/mediavoorvertoningen via `chat.postMessage`; stel `unfurlLinks: true` in om linkvoorvertoningen weer in te schakelen
- beheer/functies: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

</Accordion>

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Geen antwoorden in kanalen">
    Controleer in deze volgorde:

    - `groupPolicy`
    - toegestane kanalen (`channels.slack.channels`) — **sleutels moeten kanaal-ID's zijn** (`C12345678`), geen namen (`#channel-name`). Op namen gebaseerde sleutels werken stilzwijgend niet onder `groupPolicy: "allowlist"`, omdat kanaalroutering standaard eerst het ID gebruikt. Een ID vinden: klik met de rechtermuisknop op het kanaal in Slack → **Copy link** — de waarde `C...` aan het einde van de URL is het kanaal-ID.
    - `requireMention`
    - toegestane lijst `users` per kanaal
    - `messages.groupChat.visibleReplies`: normale groeps-/kanaalverzoeken gebruiken standaard `"automatic"`. Als je `"message_tool"` hebt ingeschakeld en de logboeken assistenttekst zonder aanroep van `message(action=send)` tonen, heeft het model het zichtbare berichttoolpad gemist. Definitieve tekst blijft in deze modus privé; controleer het uitgebreide Gateway-logboek op onderdrukte payloadmetadata, of stel dit in op `"automatic"` als je wilt dat elk normaal definitief assistentantwoord via het verouderde pad wordt geplaatst.
    - `messages.groupChat.unmentionedInbound`: als dit `"room_event"` is, vormt niet-vermeld toegestaan kanaalverkeer omgevingscontext en blijft het stil, tenzij de agent het hulpmiddel `message` aanroept. Zie [Omgevingsgebeurtenissen in ruimtes](/nl/channels/ambient-room-events).

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    Nuttige opdrachten:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DM-berichten worden genegeerd">
    Controleer:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (of het verouderde `channels.slack.dm.policy`)
    - koppelingsgoedkeuringen/vermeldingen in de toegestane lijst (`dmPolicy: "open"` vereist nog steeds `channels.slack.allowFrom: ["*"]`)
    - groeps-DM's gebruiken MPIM-afhandeling; schakel `channels.slack.dm.groupEnabled` in en neem, indien geconfigureerd, de MPIM op in `channels.slack.dm.groupChannels`
    - DM-gebeurtenissen van Slack Assistant: uitgebreide logboeken die `drop message_changed` vermelden,
      betekenen meestal dat Slack een bewerkte Assistant-threadgebeurtenis heeft verzonden zonder een
      herstelbare menselijke afzender in de berichtmetadata

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socketmodus maakt geen verbinding">
    Valideer de bot- en app-tokens en controleer of Socket Mode is ingeschakeld in de instellingen van de Slack-app.
    Het App-Level Token heeft `connections:write` nodig en het Bot User OAuth Token
    moet bij dezelfde Slack-app/werkruimte horen als het app-token.

    Als `openclaw channels status --probe --json` `botTokenStatus` of
    `appTokenStatus: "configured_unavailable"` toont, is het Slack-account
    geconfigureerd, maar kon de huidige runtime de door SecretRef ondersteunde
    waarde niet vinden.

    Logboeken zoals `slack socket mode failed to start; retry ...` duiden op herstelbare
    opstartfouten. Ontbrekende bereiken, ingetrokken tokens en ongeldige authenticatie leiden
    daarentegen direct tot een fout. Een logboekvermelding `slack token mismatch ...` betekent dat het bot-token en app-token
    bij verschillende Slack-apps lijken te horen; corrigeer de inloggegevens van de Slack-app.

  </Accordion>

  <Accordion title="HTTP-modus ontvangt geen gebeurtenissen">
    Valideer:

    - ondertekeningsgeheim
    - Webhook-pad
    - Slack Request URLs (Events + Interactivity + Slash Commands)
    - unieke `webhookPath` per HTTP-account
    - de openbare URL beëindigt TLS en stuurt verzoeken door naar het Gateway-pad
    - het pad `request_url` van de Slack-app komt exact overeen met `channels.slack.webhookPath` (standaard `/slack/events`)

    Als `signingSecretStatus: "configured_unavailable"` in accountmomentopnamen
    voorkomt, is het HTTP-account geconfigureerd, maar kon de huidige runtime het door
    SecretRef ondersteunde ondertekeningsgeheim niet vinden.

    Een herhaalde logboekvermelding `slack: webhook path ... already registered` betekent dat twee HTTP-
    accounts dezelfde `webhookPath` gebruiken; geef elk account een afzonderlijk pad.

  </Accordion>

  <Accordion title="Native/slash-opdrachten worden niet uitgevoerd">
    Controleer of je het volgende bedoelde:

    - native-opdrachtmodus (`channels.slack.commands.native: true`) met overeenkomende slash-opdrachten die in Slack zijn geregistreerd
    - of modus voor één slash-opdracht (`channels.slack.slashCommand.enabled: true`)

    Slack maakt of verwijdert slash-opdrachten niet automatisch. `commands.native: "auto"` schakelt native Slack-opdrachten niet in; gebruik `true` en maak de overeenkomende opdrachten in de Slack-app. In HTTP-modus moet elke Slack-slash-opdracht de Gateway-URL bevatten. In Socket Mode komen opdrachtpayloads binnen via de websocket en negeert Slack `slash_commands[].url`.

    Controleer ook `commands.useAccessGroups`, DM-autorisatie, toegestane kanalen
    en toegestane lijsten `users` per kanaal. Slack retourneert tijdelijke fouten voor
    geblokkeerde afzenders van slash-opdrachten, waaronder:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## Naslaginformatie voor bijlagemedia

Slack kan gedownloade media aan de agentbeurt toevoegen wanneer het downloaden van Slack-bestanden slaagt en de groottelimieten dit toestaan. Audioclips kunnen worden getranscribeerd, afbeeldingsbestanden kunnen via het pad voor mediabegrip of rechtstreeks naar een antwoordmodel met beeldondersteuning worden doorgegeven, en andere bestanden blijven beschikbaar als downloadbare bestandscontext.

### Ondersteunde mediatypen

| Mediatype                      | Bron                 | Huidig gedrag                                                                    | Opmerkingen                                                                    |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Slack-audioclips               | Slack-bestands-URL   | Gedownload en via de gedeelde audiotranscriptie gerouteerd                       | Vereist `files:read` en een werkend `tools.media.audio`-model of CLI      |
| JPEG-/PNG-/GIF-/WebP-afbeeldingen | Slack-bestands-URL | Gedownload en aan de beurt gekoppeld voor verwerking met beeldondersteuning     | Limiet per bestand: `channels.slack.mediaMaxMb` (standaard 20 MB)                 |
| PDF-bestanden                  | Slack-bestands-URL   | Gedownload en beschikbaar gesteld als bestandscontext voor hulpmiddelen zoals `download-file` of `pdf` | Inkomende Slack-verwerking zet PDF's niet automatisch om in beeldinvoer |
| Andere bestanden              | Slack-bestands-URL   | Waar mogelijk gedownload en beschikbaar gesteld als bestandscontext             | Binaire bestanden worden niet als beeldinvoer behandeld                  |
| Threadantwoorden              | Bestanden van de threadstarter | Bestanden van het hoofdbericht kunnen als context worden geladen wanneer het antwoord geen directe media bevat | Starters met alleen bestanden gebruiken een bijlageplaceholder |
| Berichten met meerdere bestanden | Meerdere Slack-bestanden | Elk bestand wordt afzonderlijk beoordeeld                                      | Slack-verwerking is beperkt tot acht bestanden per bericht               |

### Inkomende pijplijn

Wanneer een Slack-bericht met bestandsbijlagen binnenkomt:

1. OpenClaw downloadt het bestand vanaf de privé-URL van Slack met behulp van het bot-token.
2. Na een geslaagde download wordt het bestand naar de mediaopslag geschreven.
3. Paden en inhoudstypen van gedownloade media worden aan de inkomende context toegevoegd.
4. Audioclips worden naar de gedeelde transcriptiepijplijn gerouteerd; model-/hulpmiddelpaden met beeldondersteuning kunnen afbeeldingsbijlagen uit dezelfde context gebruiken.
5. Andere bestanden blijven beschikbaar als bestandsmetadata of mediaverwijzingen voor hulpmiddelen die ze kunnen verwerken.

### Overerving van bijlagen uit het hoofdbericht van een thread

Wanneer een bericht binnenkomt in een thread (met een bovenliggend `thread_ts`):

- Als het antwoord zelf geen directe media bevat en het opgenomen hoofdbericht bestanden bevat, kan Slack de hoofdbestanden laden als context van de threadstarter.
- Hoofdbestanden worden alleen geladen bij het initialiseren van een nieuwe of opnieuw ingestelde threadsessie. Latere antwoorden met alleen tekst hergebruiken de bestaande sessiecontext en koppelen hoofdbestanden niet opnieuw als nieuwe media.
- Directe antwoordbijlagen hebben voorrang op bijlagen van het hoofdbericht.
- Een hoofdbericht dat alleen bestanden en geen tekst bevat, wordt weergegeven met een bijlageplaceholder, zodat de terugvaloptie de bestanden toch kan opnemen.

### Verwerking van meerdere bijlagen

Wanneer één Slack-bericht meerdere bestandsbijlagen bevat:

- Elke bijlage wordt afzonderlijk via de mediapijplijn verwerkt.
- Verwijzingen naar gedownloade media worden samengevoegd in de berichtcontext.
- De verwerkingsvolgorde volgt de bestandsvolgorde van Slack in de gebeurtenispayload.
- Een mislukte download van één bijlage blokkeert de andere niet.

### Limieten voor grootte, downloaden en modellen

- **Groottelimiet**: standaard 20 MB per bestand. Configureerbaar via `channels.slack.mediaMaxMb`.
- **Limiet voor audiotranscriptie**: de `maxBytes` van de geselecteerde vermelding `tools.media.models[]` met audio-ondersteuning is ook van toepassing wanneer het gedownloade bestand naar een transcriptieprovider of CLI wordt verzonden.
- **Downloadfouten**: bestanden die Slack niet kan leveren, verlopen URL's, ontoegankelijke bestanden, te grote bestanden en HTML-antwoorden voor Slack-authenticatie/-aanmelding worden overgeslagen in plaats van als niet-ondersteunde indelingen te worden gemeld.
- **Beeldmodel**: voor beeldanalyse wordt het actieve antwoordmodel gebruikt wanneer dit beeld ondersteunt, of het afbeeldingsmodel dat is geconfigureerd bij `agents.defaults.imageModel`.

### Bekende beperkingen

| Scenario                                      | Huidig gedrag                                                                      | Tijdelijke oplossing                                                               |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Verlopen Slack-bestands-URL                   | Bestand overgeslagen; er wordt geen fout weergegeven                               | Upload het bestand opnieuw in Slack                                                |
| Audiotranscriptie niet beschikbaar            | Fragment blijft bijgevoegd, maar er wordt geen transcriptie gemaakt                | Configureer `tools.media.audio` of installeer een ondersteunde lokale transcriptie-CLI |
| Fragment zonder bijschrift doorstaat een vermeldingscontrole niet | Verwijderd na persoonlijke speculatieve transcriptie; transcriptie en download verwijderd | Configureer een vermeldingspatroon voor de uitgesproken naam, voeg een getypte botvermelding toe of gebruik een DM |
| Visueel model niet geconfigureerd             | Afbeeldingsbijlagen worden opgeslagen als mediaverwijzingen, maar niet als afbeeldingen geanalyseerd | Configureer `agents.defaults.imageModel` of gebruik een antwoordmodel met visuele mogelijkheden |
| Zeer grote afbeeldingen (> 20 MB standaard)   | Overgeslagen vanwege de groottelimiet                                               | Verhoog `channels.slack.mediaMaxMb` als Slack dit toestaat                                  |
| Doorgestuurde/gedeelde bijlagen               | Tekst en door Slack gehoste afbeeldings-/bestandsmedia worden waar mogelijk verwerkt | Deel ze opnieuw rechtstreeks in de OpenClaw-thread                                 |
| PDF-bijlagen                                  | Opgeslagen als bestands-/mediacontext, niet automatisch verwerkt door het visuele afbeeldingsmodel | Gebruik `download-file` voor bestandsmetadata of de tool `pdf` voor PDF-analyse |

### Gerelateerde documentatie

- [Pijplijn voor mediabegrip](/nl/nodes/media-understanding)
- [Audio- en spraaknotities](/nl/nodes/audio)
- [PDF-tool](/nl/tools/pdf)

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Koppelen" icon="link" href="/nl/channels/pairing">
    Koppel een Slack-gebruiker aan de Gateway.
  </Card>
  <Card title="Groepen" icon="users" href="/nl/channels/groups">
    Gedrag van kanaal- en groeps-DM's.
  </Card>
  <Card title="Kanaalroutering" icon="route" href="/nl/channels/channel-routing">
    Routeer inkomende berichten naar agents.
  </Card>
  <Card title="Beveiliging" icon="shield" href="/nl/gateway/security">
    Dreigingsmodel en beveiliging.
  </Card>
  <Card title="Configuratie" icon="sliders" href="/nl/gateway/configuration">
    Configuratie-indeling en prioriteitsvolgorde.
  </Card>
  <Card title="Slash-opdrachten" icon="terminal" href="/nl/tools/slash-commands">
    Opdrachtencatalogus en gedrag.
  </Card>
</CardGroup>
