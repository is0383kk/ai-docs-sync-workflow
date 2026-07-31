---
read_when:
    - OpenClaw voor het eerst instellen
    - Zoeken naar veelgebruikte configuratiepatronen
    - Navigeren naar specifieke configuratiesecties
summary: 'Configuratieoverzicht: veelvoorkomende taken, snelle installatie en links naar de volledige referentie'
title: Configuratie
x-i18n:
    generated_at: "2026-07-27T05:33:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw leest een optionele <Tooltip tip="JSON5 ondersteunt opmerkingen en afsluitende komma's">**JSON5**</Tooltip>-configuratie uit `~/.openclaw/openclaw.json`. Als het bestand ontbreekt, gebruikt OpenClaw veilige standaardwaarden.

Het actieve configuratiepad moet een normaal bestand zijn. Schrijfbewerkingen van OpenClaw vervangen het atomair (door het naar het pad te hernoemen), waardoor bij een symbolische koppeling voor `openclaw.json` het doel wordt vervangen in plaats van via de koppeling te worden beschreven. Vermijd daarom configuratie-indelingen met symbolische koppelingen. Als je de configuratie buiten de standaardstatusmap bewaart, laat `OPENCLAW_CONFIG_PATH` dan rechtstreeks naar het echte bestand wijzen.

Veelvoorkomende redenen om een configuratie toe te voegen:

- Kanalen verbinden en bepalen wie de bot berichten kan sturen
- Modellen, tools, sandboxing of automatisering instellen (cron, hooks)
- Sessies, media, netwerken of de gebruikersinterface afstemmen

Zie de [volledige naslaginformatie](/nl/gateway/configuration-reference) voor elk beschikbaar veld.

De configuratie volgt een regel met twee categorieën: hoofdelementen bevatten infrastructuur en standaardwaarden voor meerdere agents, terwijl `agents.defaults` het gedrag van de agentlus bevat. Vermeldingen onder `agents.entries` kunnen beide categorieën overschrijven waar het schema een overschrijving per agent ondersteunt.

Agents en automatisering moeten `config.schema.lookup` gebruiken voor exacte documentatie
op veldniveau voordat ze de configuratie bewerken. Gebruik deze pagina voor taakgerichte richtlijnen en
de [configuratienaslag](/nl/gateway/configuration-reference) voor het bredere
overzicht van velden en standaardwaarden.

<Tip>
**Nieuw met configuratie?** Begin met `openclaw onboard` voor interactieve installatie of bekijk de handleiding [Configuratievoorbeelden](/nl/gateway/configuration-examples) voor volledige configuraties die je kunt kopiëren en plakken.
</Tip>

## Minimale configuratie

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Configuratie bewerken

<Tabs>
  <Tab title="Interactieve wizard">
    ```bash
    openclaw onboard       # volledige onboardingflow
    openclaw configure     # configuratiewizard
    ```
  </Tab>
  <Tab title="CLI (opdrachten van één regel)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="Bedieningsinterface">
    Open [http://127.0.0.1:18789](http://127.0.0.1:18789) en gebruik het tabblad **Config**.
    De bedieningsinterface genereert een formulier op basis van het live configuratieschema, inclusief
    documentatiemetagegevens voor de velden `title` / `description` en, indien
    beschikbaar, schema's voor plugins en kanalen, met een **Raw JSON**-editor als uitweg. Voor
    detailinterfaces en andere tools stelt de Gateway ook `config.schema.lookup` beschikbaar om
    één schemaknooppunt voor een specifiek pad op te halen, plus samenvattingen van de directe onderliggende knooppunten.
    Bij de instellingen worden algemene velden eerst weergegeven. Elke sectie houdt de geavanceerde velden
    in een ingeklapte groep **Advanced (N)**; gebruik **Show advanced** om alle
    groepen uit te vouwen. Bij het zoeken in de instellingen worden altijd beide niveaus doorzocht en wordt zo nodig
    de overeenkomende geavanceerde groep geopend.
  </Tab>
  <Tab title="Rechtstreeks bewerken">
    Bewerk `~/.openclaw/openclaw.json` rechtstreeks. De Gateway bewaakt het bestand en past wijzigingen automatisch toe (zie [direct herladen](#config-hot-reload)).
  </Tab>
</Tabs>

## Strikte validatie

<Warning>
OpenClaw accepteert alleen configuraties die volledig met het schema overeenkomen. Onbekende sleutels, onjuist gevormde typen of ongeldige waarden zorgen ervoor dat de Gateway **weigert te starten**. De enige uitzondering op hoofdniveau is `$schema` (tekenreeks), zodat editors metagegevens van JSON Schema kunnen toevoegen.
</Warning>

`openclaw config schema` toont het canonieke JSON Schema dat door de bedieningsinterface
en validatie wordt gebruikt. `config.schema.lookup` haalt één knooppunt voor een specifiek pad op, plus
samenvattingen van onderliggende knooppunten voor tools met detailweergaven. Documentatiemetagegevens van de velden `title`/`description`
worden doorgegeven aan geneste objecten, jokertekens (`*`), array-items (`[]`) en vertakkingen van `anyOf`/
`oneOf`/`allOf`. Runtimeschema's van plugins en kanalen worden samengevoegd wanneer het
manifestregister is geladen.

Elk eindveld van de configuratie heeft een algemeen of geavanceerd presentatieniveau in `uiHints`.
`advanced: false` markeert algemene instellingen en `advanced: true` markeert geavanceerde
instellingen. Een eindveld neemt het niveau van de dichtstbijzijnde bovenliggende structuur over als het geen directe aanwijzing heeft;
paden zonder gedeclareerde bovenliggende structuur krijgen standaard het niveau geavanceerd. Dit is alleen van invloed op de presentatie,
niet op validatie, standaardwaarden, herlaadgedrag of de mogelijkheid om de sleutel in te stellen.

Wanneer validatie mislukt:

- De Gateway start niet op
- Alleen diagnostische opdrachten werken (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- Voer `openclaw doctor` uit om de exacte problemen te bekijken
- Voer `openclaw doctor --fix` uit (`--repair` is dezelfde vlag; `--yes` slaat vragen over) om reparaties toe te passen

De Gateway bewaart na elke geslaagde start een vertrouwde kopie van de laatst bekende werkende configuratie,
maar bij het starten en direct herladen wordt deze niet automatisch hersteld; alleen `openclaw doctor --fix`
doet dit. Als `openclaw.json` niet door de validatie komt (inclusief lokale validatie van plugins), mislukt het
starten van de Gateway of wordt het herladen overgeslagen en blijft de huidige runtime de laatst geaccepteerde
configuratie gebruiken. Een geweigerde schrijfbewerking wordt voor inspectie ook opgeslagen als `<path>.rejected.<timestamp>`.
De Gateway blokkeert schrijfbewerkingen die op onbedoeld overschrijven lijken — het verwijderen van `gateway.mode`,
het verliezen van het blok `meta` of het met meer dan de helft verkleinen van het bestand — tenzij de schrijfbewerking
destructieve wijzigingen expliciet toestaat. Promotie tot laatst bekende werkende configuratie wordt overgeslagen wanneer een
kandidaat een tijdelijke aanduiding voor een geredigeerd geheim bevat, zoals `***` of `[redacted]`.

## Veelvoorkomende taken

<AccordionGroup>
  <Accordion title="Een kanaal instellen (WhatsApp, Telegram, Discord enzovoort)">
    Elk kanaal heeft een eigen configuratiesectie onder `channels.<provider>`. Zie de specifieke kanaalpagina voor de installatiestappen:

    - [Discord](/nl/channels/discord) - `channels.discord`
    - [Feishu](/nl/channels/feishu) - `channels.feishu`
    - [Google Chat](/nl/channels/googlechat) - `channels.googlechat`
    - [iMessage](/nl/channels/imessage) - `channels.imessage`
    - [Mattermost](/nl/channels/mattermost) - `channels.mattermost`
    - [Microsoft Teams](/nl/channels/msteams) - `channels.msteams`
    - [Signal](/nl/channels/signal) - `channels.signal`
    - [Slack](/nl/channels/slack) - `channels.slack`
    - [Telegram](/nl/channels/telegram) - `channels.telegram`
    - [WhatsApp](/nl/channels/whatsapp) - `channels.whatsapp`

    Alle kanalen gebruiken hetzelfde patroon voor DM-beleid:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // alleen voor allowlist/open
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Modellen kiezen en configureren">
    Stel het primaire model en optionele terugvalmodellen in:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` bewaart aliassen en instellingen per model; het toevoegen van een vermelding beperkt overschrijvingen via `/model` of `--model` nooit.
    - `agents.defaults.modelPolicy.allow` is de expliciete toelatingslijst voor overschrijvingen en modelkiezers. Deze accepteert exacte verwijzingen en jokertekens van `provider/*`; laat dit weg of gebruik `[]` om elk model toe te staan.
    - Modelverwijzingen gebruiken de indeling `provider/model` (bijvoorbeeld `anthropic/claude-opus-4-6`).
    - `agents.defaults.imageMaxDimensionPx` bepaalt het verkleinen van afbeeldingen uit transcripties/tools (standaard `1200`); lagere waarden verminderen doorgaans het gebruik van vision-tokens bij uitvoeringen met veel schermafbeeldingen.
    - Zie [CLI voor modellen](/nl/concepts/models) voor het wisselen van model in een chat en [Modelomschakeling bij storingen](/nl/concepts/model-failover) voor autorisatierotatie en terugvalgedrag.
    - Zie voor aangepaste/zelfgehoste providers [Aangepaste providers](/nl/gateway/config-tools#custom-providers-and-base-urls) in de naslaginformatie.

  </Accordion>

  <Accordion title="Bepalen wie de bot berichten kan sturen">
    DM-toegang wordt per kanaal geregeld via `dmPolicy` (standaard `"pairing"`):

    - `"pairing"`: onbekende afzenders krijgen een eenmalige koppelingscode ter goedkeuring
    - `"allowlist"`: alleen afzenders in `allowFrom` (of de opslag met gekoppelde toegestane afzenders)
    - `"open"`: alle inkomende DM's toestaan (vereist `allowFrom: ["*"]`)
    - `"disabled"`: alle DM's negeren

    Gebruik voor groepen `groupPolicy` (`"allowlist" | "open" | "disabled"`) plus `groupAllowFrom` of kanaalspecifieke toelatingslijsten.

    Zie de [volledige naslaginformatie](/nl/gateway/config-channels#dm-and-group-access) voor details per kanaal.

  </Accordion>

  <Accordion title="Vermeldingsvereisten voor groepschats instellen">
    Voor groepsberichten is standaard een **vermelding vereist**. Configureer activeringspatronen per agent. Normale antwoorden in groepen/kanalen worden automatisch geplaatst; schakel het pad via de berichtentool in voor gedeelde ruimten waarin de agent moet beslissen wanneer deze spreekt:

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // stel "message_tool" in om overal verzending via de berichtentool te vereisen
        groupChat: {
          visibleReplies: "message_tool", // opt-in; zichtbare uitvoer vereist message(action=send)
          unmentionedInbound: "room_event", // groepsgesprekken zonder vermelding die altijd actief zijn, vormen stille context
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **Vermeldingen in metagegevens**: systeemeigen @-vermeldingen (tikken om te vermelden in WhatsApp, Telegram @bot enzovoort)
    - **Tekstpatronen**: veilige reguliere-expressiepatronen in `mentionPatterns`
    - **Zichtbare antwoorden**: `messages.visibleReplies` kan verzending via de berichtentool globaal verplichten; `messages.groupChat.visibleReplies` overschrijft dit voor groepen/kanalen.
    - Zie de [volledige naslaginformatie](/nl/gateway/config-channels#group-chat-mention-gating) voor modi voor zichtbare antwoorden, overschrijvingen per kanaal en de zelfchatmodus.

  </Accordion>

  <Accordion title="Skills per agent beperken">
    Gebruik `agents.defaults.skills` voor een gedeelde basis en overschrijf vervolgens specifieke
    agents met `agents.entries.*.skills`:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // neemt github, weather over
          { id: "docs", skills: ["docs-search"] }, // vervangt de standaardwaarden
          { id: "locked-down", skills: [] }, // geen skills
        ],
      },
    }
    ```

    - Laat `agents.defaults.skills` weg voor standaard onbeperkte skills.
    - Laat `agents.entries.*.skills` weg om de standaardwaarden over te nemen.
    - Stel `agents.entries.*.skills: []` in voor geen skills.
    - Zie [Skills](/nl/tools/skills), [Skills-configuratie](/nl/tools/skills-config) en
      de [configuratienaslag](/nl/gateway/config-agents#agents-defaults-skills).

  </Accordion>

  <Accordion title="Statusbewaking per kanaal configureren">
    Schakel automatische herstarts voor de status van een kanaal of account uit of in:

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - Gebruik `channels.<provider>.healthMonitor.enabled` of `channels.<provider>.accounts.<id>.healthMonitor.enabled` om automatische herstarts voor één kanaal of account te regelen.
    - Zie [Statuscontroles](/nl/gateway/health) voor operationele foutopsporing en de [volledige naslaginformatie](/nl/gateway/configuration-reference#gateway) voor alle velden.

  </Accordion>

  <Accordion title="Sessies en resets configureren">
    Sessies bepalen de continuïteit en isolatie van gesprekken:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // aanbevolen voor meerdere gebruikers
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (gedeeld) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: algemene standaardwaarden voor sessieroutering die aan threads is gekoppeld. `/focus`, `/unfocus`, `/agents`, `/session idle` en `/session max-age` koppelen, ontkoppelen, vermelden en configureren dit per sessie (Discord koppelt threads, Telegram koppelt onderwerpen/gesprekken).
    - Zie [Sessiebeheer](/nl/concepts/session) voor bereik, identiteitskoppelingen en verzendbeleid.
    - Zie de [volledige referentie](/nl/gateway/config-agents#session) voor alle velden.

  </Accordion>

  <Accordion title="Sandboxing inschakelen">
    Voer agentsessies uit in geïsoleerde sandboxruntimes:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    Bouw eerst de image: voer vanuit een broncheckout `scripts/sandbox-setup.sh` uit, of raadpleeg bij een npm-installatie de inlineopdracht `docker build` in [Sandboxing § Images en configuratie](/nl/gateway/sandboxing#images-and-setup).

    Zie [Sandboxing](/nl/gateway/sandboxing) voor de volledige handleiding en de [volledige referentie](/nl/gateway/config-agents#agentsdefaultssandbox) voor alle opties.

  </Accordion>

  <Accordion title="Relaygebaseerde push inschakelen voor officiële iOS-builds">
    Relaygebaseerde push voor openbare App Store-builds gebruikt de gehoste OpenClaw-relay: `https://ios-push-relay.openclaw.ai`.

    Aangepaste relayimplementaties vereisen een bewust afzonderlijk iOS-build- en implementatiepad waarvan de relay-URL overeenkomt met de relay-URL van de Gateway. Als je een aangepaste relaybuild gebruikt, stel je dit in de Gateway-configuratie in:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // Optioneel. Standaard: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    CLI-equivalent:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    Wat dit doet:

    - Hiermee kan de Gateway `push.test`, activeringssignalen en herverbindingssignalen via de externe relay verzenden.
    - Gebruikt een verzendmachtiging met registratiebereik die door de gekoppelde iOS-app wordt doorgestuurd. De Gateway heeft geen implementatiebreed relaytoken nodig.
    - Koppelt elke relaygebaseerde registratie aan de Gateway-identiteit waarmee de iOS-app is gekoppeld, zodat een andere Gateway de opgeslagen registratie niet kan hergebruiken.
    - Houdt lokale/handmatige iOS-builds op directe APNs. Relaygebaseerde verzending is alleen van toepassing op officieel gedistribueerde builds die via de relay zijn geregistreerd.
    - Moet overeenkomen met de relaybasis-URL die in de iOS-build is ingebouwd, zodat registratie- en verzendverkeer dezelfde relayimplementatie bereiken.

    End-to-end-flow:

    1. Installeer de officiële iOS-app.
    2. Optioneel: configureer `gateway.push.apns.relay.baseUrl` op de Gateway alleen wanneer je een bewust afzonderlijke aangepaste relaybuild gebruikt.
    3. Koppel de iOS-app aan de Gateway en laat zowel Node- als operatorsessies verbinding maken.
    4. De iOS-app haalt de Gateway-identiteit op, registreert zich bij de relay met App Attest en het appbewijs en publiceert vervolgens de relaygebaseerde `push.apns.register`-payload naar de gekoppelde Gateway.
    5. De Gateway slaat de relayhandle en verzendmachtiging op en gebruikt deze vervolgens voor `push.test`, activeringssignalen en herverbindingssignalen.

    Operationele opmerkingen:

    - Als je de iOS-app naar een andere Gateway overschakelt, verbind je de app opnieuw zodat deze een nieuwe, aan die Gateway gekoppelde relayregistratie kan publiceren.
    - Als je een nieuwe iOS-build uitbrengt die naar een andere relayimplementatie verwijst, vernieuwt de app de gecachte relayregistratie in plaats van de oude relayoorsprong opnieuw te gebruiken.

    Compatibiliteitsopmerking:

    - `OPENCLAW_APNS_RELAY_BASE_URL` en `OPENCLAW_APNS_RELAY_TIMEOUT_MS` werken nog steeds als tijdelijke omgevingsoverschrijvingen.
    - Aangepaste relay-URL's voor de Gateway moeten overeenkomen met de relaybasis-URL die in de iOS-build is ingebouwd; het openbare App Store-releasekanaal weigert aangepaste iOS-relay-URL-overschrijvingen.
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` blijft een uitsluitend voor loopback bestemde noodoplossing voor ontwikkeling; sla HTTP-relay-URL's niet permanent op in de configuratie.

    Zie [iOS-app](/nl/platforms/ios#relay-backed-push-for-official-builds) voor de end-to-end-flow en [Authenticatie- en vertrouwensflow](/nl/platforms/ios#authentication-and-trust-flow) voor het beveiligingsmodel van de relay.

  </Accordion>

  <Accordion title="Heartbeat instellen (periodieke check-ins)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: duurtekenreeks (`30m`, `2h`). Stel `0m` in om dit uit te schakelen. Standaard: `30m`.
    - `target`: `last` | `none` | `<channel-id>` (bijvoorbeeld `discord`, `matrix`, `telegram` of `whatsapp`)
    - `directPolicy`: `allow` (standaard) of `block` voor Heartbeat-doelen in DM-stijl
    - Zie [Heartbeat](/nl/gateway/heartbeat) voor de volledige handleiding.

  </Accordion>

  <Accordion title="Cron-taken configureren">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: verwijder voltooide geïsoleerde uitvoeringssessies uit SQLite-sessierijen (standaard `24h`; stel `false` in om dit uit te schakelen).
    - De uitvoeringsgeschiedenis bewaart automatisch de nieuwste 2000 terminalrijen per taak; verloren rijen behouden hun opschoningsvenster van 24 uur.
    - Zie [Cron-taken](/nl/automation/cron-jobs) voor een functieoverzicht en CLI-voorbeelden.

  </Accordion>

  <Accordion title="Webhooks (hooks) instellen">
    Schakel HTTP-webhook-eindpunten op de Gateway in:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    Beveiligingsopmerking:
    - Behandel alle inhoud van hook-/webhook-payloads als niet-vertrouwde invoer.
    - Gebruik een afzonderlijke `hooks.token`; gebruik actieve authenticatiegeheimen van de Gateway niet opnieuw (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` of `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`).
    - Hookauthenticatie werkt uitsluitend via headers (`Authorization: Bearer ...` of `x-openclaw-token`); tokens in queryreeksen worden geweigerd.
    - `hooks.path` kan niet `/` zijn; houd webhookinvoer op een afzonderlijk subpad, zoals `/hooks`.
    - Laat omzeilingsvlaggen voor onveilige inhoud uitgeschakeld (`hooks.gmail.allowUnsafeExternalContent`, `hooks.mappings[].allowUnsafeExternalContent`), tenzij je strikt afgebakende foutopsporing uitvoert.
    - Als je `hooks.allowRequestSessionKey` inschakelt, stel dan ook `hooks.allowedSessionKeyPrefixes` in om door aanroepers geselecteerde sessiesleutels te begrenzen.
    - Geef voor door hooks aangestuurde agents de voorkeur aan sterke, moderne modelniveaus en een strikt toolbeleid (bijvoorbeeld alleen berichten plus sandboxing waar mogelijk).

    Zie de [volledige referentie](/nl/gateway/configuration-reference#hooks) voor alle toewijzingsopties en Gmail-integratie.

  </Accordion>

  <Accordion title="Routering voor meerdere agents configureren">
    Voer meerdere geïsoleerde agents uit met afzonderlijke werkruimten en sessies:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    Zie [Multi-Agent](/nl/concepts/multi-agent) en de [volledige referentie](/nl/gateway/config-agents#multi-agent-routing) voor koppelingsregels en toegangsprofielen per agent.

  </Accordion>

  <Accordion title="Configuratie over meerdere bestanden verdelen ($include)">
    Gebruik `$include` om grote configuraties te organiseren:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **Eén bestand**: vervangt het omsluitende object
    - **Bestandsarray**: wordt op volgorde diep samengevoegd (de laatste wint), tot 10 geneste niveaus diep
    - **Sleutels op hetzelfde niveau**: worden na de includes samengevoegd (overschrijven opgenomen waarden)
    - **Relatieve paden**: worden relatief ten opzichte van het includerende bestand omgezet
    - **Padindeling**: includepaden mogen geen nullbytes bevatten en moeten vóór en na omzetting strikt korter zijn dan 4096 tekens
    - **Door OpenClaw beheerde schrijfbewerkingen**: wanneer een schrijfbewerking slechts één sectie op het hoogste niveau wijzigt
      die wordt ondersteund door een include van één bestand, zoals `plugins: { $include: "./plugins.json5" }`,
      werkt OpenClaw dat opgenomen bestand bij en laat het `openclaw.json` intact
    - **Niet-ondersteund doorschrijven**: root-includes, includearrays en includes
      met overschrijvingen op hetzelfde niveau worden bij door OpenClaw beheerde schrijfbewerkingen
      veilig geweigerd in plaats van de configuratie af te vlakken
    - **Beperking**: `$include`-paden moeten worden omgezet naar een locatie onder de map die
      `openclaw.json` bevat. Als je een boomstructuur tussen machines of gebruikers wilt delen, stel je
      `OPENCLAW_INCLUDE_ROOTS` in op een padenlijst (`:` op POSIX, `;` op Windows) met
      aanvullende mappen waarnaar includes mogen verwijzen. Symbolische koppelingen worden omgezet
      en opnieuw gecontroleerd. Een pad dat zich lexicaal in een configuratiemap bevindt, maar waarvan
      het werkelijke doel buiten elke toegestane root valt, wordt daarom nog steeds geweigerd.
    - **Foutafhandeling**: duidelijke fouten voor ontbrekende bestanden, parseerfouten, circulaire includes, een ongeldige padindeling en een te grote lengte

  </Accordion>
</AccordionGroup>

## Configuratie dynamisch herladen

De Gateway bewaakt `~/.openclaw/openclaw.json` en past wijzigingen automatisch toe. Voor de meeste instellingen is geen handmatige herstart nodig.

Rechtstreekse bestandsbewerkingen worden als niet-vertrouwd behandeld totdat ze zijn gevalideerd. De bewaker wacht
tot tijdelijke schrijf- en hernoemactiviteiten van de editor zijn gestabiliseerd, leest het definitieve bestand en weigert
ongeldige externe bewerkingen zonder `openclaw.json` te herschrijven. Door OpenClaw beheerde schrijfbewerkingen van de configuratie
gebruiken vóór het schrijven dezelfde schemavalidatie (zie [Strikte validatie](#strict-validation)
voor de regels voor overschrijven en terugdraaien die op elke schrijfbewerking van toepassing zijn).

Als je `config reload skipped (invalid config)` ziet of bij het opstarten `Invalid
config` wordt gemeld, controleer je de configuratie, voer je `openclaw config validate` uit en voer je vervolgens `openclaw
doctor --fix` uit om deze te herstellen. Zie [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting#gateway-rejected-invalid-config)
voor de controlelijst.

### Herlaadmodi

| Modus                  | Gedrag                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| **`hybrid`** (standaard) | Past veilige wijzigingen direct toe. Start bij kritieke wijzigingen automatisch opnieuw.   |
| **`hot`**              | Past alleen veilige wijzigingen direct toe. Logt een waarschuwing wanneer opnieuw starten nodig is; je handelt dit zelf af. |
| **`restart`**          | Start de Gateway opnieuw bij elke configuratiewijziging, veilig of niet.                    |
| **`off`**              | Schakelt bestandsbewaking uit. Wijzigingen worden van kracht bij de volgende handmatige herstart. |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### Wat direct wordt toegepast en waarvoor opnieuw starten nodig is

De meeste velden worden zonder downtime direct toegepast; bij sommige direct toegepaste secties wordt alleen dat
subsysteem (kanaal, Cron, Heartbeat, statusmonitor) opnieuw gestart in plaats van de hele Gateway. In de
modus `hybrid` worden wijzigingen waarvoor de Gateway opnieuw moet worden gestart automatisch afgehandeld.

| Categorie           | Velden                                                                  | Gateway opnieuw starten nodig? |
| ------------------- | ----------------------------------------------------------------------- | ------------------------------ |
| Kanalen             | `channels.*`, `web` (WhatsApp) - alle ingebouwde kanalen en plugin-kanalen | Nee (start dat kanaal opnieuw) |
| Agent en modellen   | `agent`, `agents`, `models`, `routing`                                  | Nee                            |
| Automatisering      | `hooks`, `cron`, `agent.heartbeat`                                      | Nee (start dat subsysteem opnieuw) |
| Sessies en berichten | `session`, `messages`                                                   | Nee                            |
| Hulpmiddelen en media | `tools`, `skills`, `mcp`, `audio`, `talk`                               | Nee                            |
| Plugin-configuratie | `plugins.entries.*`, `plugins.allow`, `plugins.deny`, `plugins.enabled` | Nee (laadt de plugin-runtime opnieuw) |
| UI en overige       | `ui`, `logging`, `identity`, `bindings`                                 | Nee                            |
| Gateway-server      | `gateway.*` (poort, binding, authenticatie, Tailscale, TLS, HTTP, push) | **Ja**                         |
| Infrastructuur      | `discovery`, `browser`, `plugins.load`, `plugins.installs`              | **Ja**                         |

<Note>
`gateway.reload` en `gateway.remote` zijn uitzonderingen onder `gateway.*`: wijzigingen hiervan activeren **geen** herstart. Afzonderlijke plugins kunnen deze tabel ook overschrijven: een geladen plugin kan eigen configuratievoorvoegsels opgeven die een herstart activeren (de meegeleverde Canvas-plugin start de Gateway bijvoorbeeld opnieuw voor `plugins.enabled`, `plugins.allow` en `plugins.deny`, niet alleen voor zijn eigen `plugins.entries.canvas`), waardoor het werkelijke gedrag afhangt van welke plugins actief zijn.
</Note>

### Planning voor opnieuw laden

Wanneer je een bronbestand bewerkt waarnaar via `$include` wordt verwezen, plant OpenClaw
het opnieuw laden op basis van de in de bron vastgelegde indeling, niet op basis van de afgevlakte weergave in het geheugen.
Zo blijven beslissingen over direct opnieuw laden (direct toepassen of opnieuw starten) voorspelbaar, zelfs wanneer één
sectie op hoofdniveau in een afzonderlijk opgenomen bestand staat, zoals
`plugins: { $include: "./plugins.json5" }`. De planning voor opnieuw laden wordt bij een
dubbelzinnige bronindeling uit veiligheidsoverwegingen afgebroken.

## Configuratie-RPC (programmatische updates)

Gebruik voor hulpmiddelen die configuratie via de Gateway-API schrijven bij voorkeur deze werkwijze:

- `config.schema.lookup` om één substructuur te inspecteren (ondiep schemaknooppunt + samenvattingen van onderliggende knooppunten)
- `config.get` om de huidige momentopname plus `hash` op te halen
- `config.patch` voor gedeeltelijke updates (JSON-samenvoegpatch: objecten worden samengevoegd, `null`
  verwijdert, matrices worden vervangen wanneer dit expliciet wordt bevestigd met `replacePaths` als
  vermeldingen zouden worden verwijderd)
- `config.apply` alleen wanneer je de volledige configuratie wilt vervangen
- `update.run` voor een expliciete zelfupdate plus herstart; neem `continuationMessage` op wanneer de sessie na de herstart nog één vervolgstap moet uitvoeren
- `update.status` om de nieuwste herstartmarkering voor updates te inspecteren en na een herstart de actieve versie te verifiëren

Agents moeten `config.schema.lookup` als eerste raadplegen voor exacte
documentatie en beperkingen op veldniveau. Gebruik [Configuratiereferentie](/nl/gateway/configuration-reference)
wanneer ze de bredere configuratiestructuur, standaardwaarden of links naar specifieke
subsysteemreferenties nodig hebben.

<Note>
Schrijfbewerkingen op het besturingsvlak (`config.apply`, `config.patch`, `update.run`) zijn
beperkt tot 30 verzoeken per 60 seconden, per methode, per
`deviceId+clientIp`; zie [Snelheidsbeperking](/gateway/security/rate-limiting). Herstartverzoeken
worden samengevoegd, waarna tussen herstartcycli een afkoelperiode van 30 seconden geldt.
`update.status` is alleen-lezen, maar vereist beheerdersrechten omdat de herstartmarkering
samenvattingen van updatestappen en de laatste regels van opdrachtuitvoer kan bevatten.
</Note>

Voorbeeld van een gedeeltelijke patch:

```bash
openclaw gateway call config.get --params '{}'  # leg payload.hash vast
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

Zowel `config.apply` als `config.patch` accepteren `raw`, `baseHash`, `sessionKey`,
`note` en `restartDelayMs`. `baseHash` is voor beide methoden vereist zodra er al een
configuratiebestand bestaat (bij een eerste schrijfbewerking zonder bestaande configuratie wordt de controle overgeslagen).

`config.patch` accepteert ook `replacePaths`, een matrix met configuratiepaden waarvan het
vervangen van de matrix opzettelijk is. Als een patch een bestaande matrix zou vervangen of verwijderen
door een matrix met minder vermeldingen, weigert de Gateway de schrijfbewerking tenzij dat exacte pad in
`replacePaths` voorkomt; geneste matrices binnen matrixvermeldingen gebruiken `[]`, zoals
`agents.entries.*.skills`. Dit voorkomt dat afgekorte `config.get`-momentopnamen
routerings- of toelatingslijstmatrices ongemerkt overschrijven. Gebruik `config.apply` wanneer je
de volledige configuratie wilt vervangen.

## Omgevingsvariabelen

OpenClaw leest omgevingsvariabelen uit het bovenliggende proces en daarnaast uit:

- `.env` uit de huidige werkmap (indien aanwezig)
- `~/.openclaw/.env` (algemene terugvaloptie)

Geen van beide bestanden overschrijft bestaande omgevingsvariabelen. Je kunt ook inline omgevingsvariabelen in de configuratie instellen:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="Shell-omgevingsvariabelen importeren (optioneel)">
  Als dit is ingeschakeld en verwachte sleutels niet zijn ingesteld, voert OpenClaw je login-shell uit en importeert het alleen de ontbrekende sleutels:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

Equivalent als omgevingsvariabele: `OPENCLAW_LOAD_SHELL_ENV=1`. Standaard `timeoutMs`: `15000`.
</Accordion>

<Accordion title="Omgevingsvariabelen vervangen in configuratiewaarden">
  Verwijs in elke tekenreekswaarde van de configuratie naar omgevingsvariabelen met `${VAR_NAME}`:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

Regels:

- Alleen namen in hoofdletters komen overeen: `[A-Z_][A-Z0-9_]*`
- Ontbrekende of lege variabelen veroorzaken tijdens het laden een fout
- Escape met `$${VAR}` voor letterlijke uitvoer
- Werkt binnen `$include`-bestanden
- Inline vervanging: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="Geheime verwijzingen (omgeving, bestand, uitvoering)">
  Voor velden die SecretRef-objecten ondersteunen, kun je het volgende gebruiken:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

Details over SecretRef (waaronder `secrets.providers` voor `env`/`file`/`exec`) staan in [Beheer van geheimen](/nl/gateway/secrets).
Ondersteunde referentiepaden voor aanmeldgegevens staan vermeld in [SecretRef-oppervlak voor aanmeldgegevens](/nl/reference/secretref-credential-surface).
</Accordion>

Zie [Omgeving](/nl/help/environment) voor de volledige prioriteitsvolgorde en bronnen.

## Volledige referentie

Zie **[Configuratiereferentie](/nl/gateway/configuration-reference)** voor de volledige referentie per veld.

---

_Gerelateerd: [Configuratievoorbeelden](/nl/gateway/configuration-examples) · [Configuratiereferentie](/nl/gateway/configuration-reference) · [Doctor](/nl/gateway/doctor)_

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/configuration-reference)
- [Configuratievoorbeelden](/nl/gateway/configuration-examples)
- [Gateway-draaiboek](/nl/gateway)
