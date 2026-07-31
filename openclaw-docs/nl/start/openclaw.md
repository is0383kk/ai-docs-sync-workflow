---
read_when:
    - Een nieuwe assistentinstantie onboarden
    - Gevolgen voor veiligheid en machtigingen beoordelen
summary: End-to-endhandleiding voor het gebruik van OpenClaw als persoonlijke assistent, met veiligheidswaarschuwingen
title: Installatie van persoonlijke assistent
x-i18n:
    generated_at: "2026-07-27T06:35:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ed3e267971fc1ee5c9154194e5b1f98db8c7a7edca8182871a2057a778614217
    source_path: start/openclaw.md
    workflow: 16
---

OpenClaw is een zelfgehoste Gateway die Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo en meer met AI-agents verbindt. Deze handleiding behandelt de configuratie als 'persoonlijke assistent': een speciaal WhatsApp-nummer dat zich gedraagt als je altijd beschikbare AI-assistent.

## Veiligheid voorop

Als je een agent toegang geeft tot een kanaal, kan deze opdrachten uitvoeren op je machine (afhankelijk van je toolbeleid), bestanden in je werkruimte lezen/schrijven en berichten terugsturen via elk verbonden kanaal. Begin voorzichtig:

- Stel altijd `channels.whatsapp.allowFrom` in (stel je persoonlijke Mac nooit open voor de hele wereld).
- Gebruik een speciaal WhatsApp-nummer voor de assistent.
- Heartbeats worden standaard elke 30 minuten uitgevoerd. Schakel ze uit totdat je de configuratie vertrouwt door `agents.defaults.heartbeat.every: "0m"` in te stellen.

## Vereisten

- OpenClaw is geïnstalleerd en het onboardingproces is voltooid — zie [Aan de slag](/nl/start/getting-started) als je dit nog niet hebt gedaan
- Een tweede telefoonnummer (simkaart/eSIM/prepaid) voor de assistent

## De configuratie met twee telefoons (aanbevolen)

Dit is het gewenste resultaat:

```mermaid
flowchart TB
    A["<b>Je telefoon (persoonlijk)<br></b><br>Je WhatsApp<br>+1-555-YOU"] -- bericht --> B["<b>Tweede telefoon (assistent)<br></b><br>WhatsApp van assistent<br>+1-555-ASSIST"]
    B -- gekoppeld via QR --> C["<b>Je Mac (openclaw)<br></b><br>AI-agent"]
```

Als je je persoonlijke WhatsApp aan OpenClaw koppelt, wordt elk bericht aan jou 'agentinvoer'. Dat is zelden wat je wilt.

## Snel starten in 5 minuten

1. Koppel WhatsApp Web (toont een QR-code; scan deze met de telefoon van de assistent):

```bash
openclaw channels login
```

2. Start de Gateway (laat deze actief):

```bash
openclaw gateway --port 18789
```

3. Plaats een minimale configuratie in `~/.openclaw/openclaw.json`:

```json5
{
  gateway: { mode: "local" },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

Stuur nu vanaf je telefoon op de toelatingslijst een bericht naar het nummer van de assistent.

Wanneer de onboarding is voltooid, opent OpenClaw automatisch het dashboard en toont het een nette link (zonder token). Als het dashboard om authenticatie vraagt, plak je het geconfigureerde gedeelde geheim in de instellingen van de Control UI. Onboarding gebruikt standaard een token (`gateway.auth.token`), maar authenticatie met een wachtwoord werkt ook als je `gateway.auth.mode` hebt gewijzigd in `password`. Later opnieuw openen: `openclaw dashboard`.

## Geef de agent een werkruimte (AGENTS)

OpenClaw leest bedieningsinstructies en 'geheugen' uit de werkruimtemap.

Standaard gebruikt OpenClaw `~/.openclaw/workspace` als werkruimte voor de agent en maakt deze tijdens de onboarding of de eerste agentuitvoering automatisch aan (inclusief de initiële bestanden `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md` en `USER.md`). `BOOTSTRAP.md` wordt alleen voor een volledig nieuwe werkruimte aangemaakt en hoort niet terug te komen nadat je het hebt verwijderd. `MEMORY.md` is optioneel en wordt nooit automatisch aangemaakt; indien aanwezig wordt het voor normale sessies geladen. Subagentsessies injecteren alleen `AGENTS.md` en `TOOLS.md`.

<Tip>
Behandel deze map als het geheugen van OpenClaw en maak er een git-repository van (bij voorkeur privé), zodat er een back-up wordt gemaakt van je `AGENTS.md` en geheugenbestanden. Als git is geïnstalleerd, worden volledig nieuwe werkruimten automatisch geïnitialiseerd met `git init`.
</Tip>

Zo maak je de werkruimte- en configuratiemappen aan zonder de volledige onboardingwizard uit te voeren:

```bash
openclaw setup --baseline
```

(Kale `openclaw setup` is een alias voor `openclaw onboard` en voert de volledige interactieve wizard uit.)

Volledige indeling van de werkruimte en back-uphandleiding: [Agentwerkruimte](/nl/concepts/agent-workspace)
Geheugenworkflow: [Geheugen](/nl/concepts/memory)

Optioneel: kies een andere werkruimte met `agents.defaults.workspace` (ondersteunt `~`).

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

Als je al je eigen werkruimtebestanden vanuit een repository levert, kun je het aanmaken van bootstrapbestanden volledig uitschakelen:

```json5
{
  agents: {
    defaults: {
      skipBootstrap: true,
    },
  },
}
```

## De configuratie die er 'een assistent' van maakt

OpenClaw gebruikt standaard een goede assistentconfiguratie, maar meestal wil je het volgende aanpassen:

- persona/instructies in [`SOUL.md`](/nl/concepts/soul)
- standaardinstellingen voor denkwerk (indien gewenst)
- heartbeats (zodra je ze vertrouwt)

Voorbeeld:

```json5
{
  logging: { level: "info" },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-5" },
      workspace: "~/.openclaw/workspace",
      thinkingDefault: "high",
      timeoutSeconds: 1800,
      // Begin met 0; schakel dit later in.
      heartbeat: { every: "0m" },
    },
    list: [
      {
        id: "main",
        default: true,
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  session: {
    scope: "per-sender",
    resetTriggers: ["/new", "/reset"],
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080,
    },
  },
}
```

## Sessies en geheugen

- Sessierijen, transcriptrijen en metagegevens (tokengebruik, laatste route enzovoort): `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Verouderde/gearchiveerde transcriptartefacten: `~/.openclaw/agents/<agentId>/sessions/`
- Migratiebron voor verouderde rijen: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- `/new` of `/reset` start een nieuwe sessie voor die chat (configureerbaar via `session.resetTriggers`). Als je dit afzonderlijk verzendt, bevestigt OpenClaw de reset zonder het model aan te roepen.
- `/compact [instructions]` voert Compaction uit op de sessiecontext en rapporteert het resterende contextbudget.

## Heartbeats (proactieve modus)

OpenClaw voert standaard elke 30 minuten een Heartbeat uit met de prompt:
`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
Stel `agents.defaults.heartbeat.every: "0m"` in om dit uit te schakelen. Heartbeat-checklists bevinden zich in de Cron-kladruimte van de monitor (zie [Heartbeat](/nl/gateway/heartbeat)); `openclaw doctor --fix` migreert een verouderde `HEARTBEAT.md` uit de werkruimte hiernaartoe.

- Als de kladruimte van de monitor bestaat maar feitelijk leeg is (alleen lege regels, Markdown-/HTML-opmerkingen, Markdown-koppen zoals `# Heading`, hekmarkeringen of lege checklistitems), slaat OpenClaw de Heartbeat-uitvoering over om API-aanroepen te besparen.
- Als er geen kladruimte bestaat, wordt de Heartbeat toch uitgevoerd en bepaalt het model wat er moet gebeuren.
- Als de agent antwoordt met `HEARTBEAT_OK` (eventueel met een korte aanvulling; zie `agents.defaults.heartbeat.ackMaxChars`), onderdrukt OpenClaw de uitgaande bezorging voor die Heartbeat.
- Standaard is Heartbeat-bezorging aan DM-achtige `user:<id>`-doelen toegestaan. Stel `agents.defaults.heartbeat.directPolicy: "block"` in om bezorging aan directe doelen te onderdrukken terwijl Heartbeat-uitvoeringen actief blijven.
- Heartbeats voeren volledige agentbeurten uit — kortere intervallen verbruiken meer tokens.

```json5
{
  agents: {
    defaults: {
      heartbeat: { every: "30m" },
    },
  },
}
```

## Media ontvangen en verzenden

Binnenkomende bijlagen (afbeeldingen/audio/documenten) kunnen via sjablonen aan je opdracht beschikbaar worden gesteld:

- `{{AttachmentPath}}` (lokaal tijdelijk bestandspad)
- `{{AttachmentUrl}}` (oorspronkelijke URL of providerreferentie)
- `{{AttachmentContentType}}` (MIME-inhoudstype)
- `{{AttachmentDir}}` (map die het lokale pad bevat)
- `{{AttachmentIndex}}` (op nul gebaseerde index van het bronfeit)
- `{{Transcript}}` (als audiotranscriptie is ingeschakeld)

De oudere namen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` en `{{MediaDir}}`
blijven beschikbaar als verouderde compatibiliteitsaliassen.

Uitgaande bijlagen van de agent gebruiken gestructureerde mediavelden in de berichtentool of antwoordpayload, zoals `media`, `mediaUrl`, `mediaUrls`, `path` of `filePath`. Voorbeeldargumenten voor de berichtentool:

```json
{
  "message": "Hier is de schermafbeelding.",
  "mediaUrl": "https://example.com/screenshot.png"
}
```

OpenClaw verzendt gestructureerde media samen met de tekst. Verouderde definitieve antwoorden van de assistent kunnen nog steeds voor compatibiliteit worden genormaliseerd, maar uitvoer van tools, browseruitvoer, streamingblokken en berichtacties interpreteren tekst niet als bijlageopdrachten.

Het gedrag voor lokale paden volgt hetzelfde vertrouwensmodel voor bestandstoegang als de agent:

- Als `tools.fs.workspaceOnly` `true` is, blijven uitgaande lokale mediapaden beperkt tot de tijdelijke hoofdmap van OpenClaw, de mediacache, paden in de werkruimte van de agent en door de sandbox gegenereerde bestanden.
- Als `tools.fs.workspaceOnly` `false` is, kunnen uitgaande lokale media bestanden op de host gebruiken die de agent al mag lezen.
- Lokale paden kunnen absoluut, relatief aan de werkruimte of relatief aan de thuismap met `~/` zijn.
- Bij verzending vanaf de host zijn nog steeds alleen media en veilige documenttypen toegestaan (afbeeldingen, audio, video, PDF, Office-documenten en gevalideerde tekstdocumenten zoals Markdown/MD, TXT, JSON, YAML en YML). Dit is een uitbreiding van de bestaande vertrouwensgrens voor lezen vanaf de host, geen scanner voor geheimen: als de agent een lokaal `secret.txt`- of `config.json`-bestand op de host kan lezen, kan de agent dat bestand toevoegen wanneer de extensie- en inhoudsvalidatie overeenkomen.

Bewaar gevoelige bestanden buiten het voor de agent leesbare bestandssysteem, of behoud `tools.fs.workspaceOnly: true` voor strengere verzending via lokale paden.

## Controlelijst voor beheer

```bash
openclaw status          # lokale status (referenties, sessies, gebeurtenissen in de wachtrij)
openclaw status --all    # volledige diagnose (alleen-lezen, geschikt om te plakken)
openclaw status --deep   # kanalen controleren (WhatsApp Web + Telegram + Discord + Slack + Signal)
openclaw health --json   # momentopname van Gateway-status via de WS-verbinding
```

Logboeken bevinden zich onder `/tmp/openclaw/`: `openclaw-YYYY-MM-DD.log` voor het standaardprofiel
en `openclaw-<profile>-YYYY-MM-DD.log` voor benoemde profielen.

## Volgende stappen

- WebChat: [WebChat](/nl/web/webchat)
- Gateway-beheer: [Gateway-draaiboek](/nl/gateway)
- Cron + activeringen: [Cron-taken](/nl/automation/cron-jobs)
- macOS-menubalkapp: [OpenClaw-app voor macOS](/nl/platforms/macos)
- iOS-Node-app: [iOS-app](/nl/platforms/ios)
- Android-Node-app: [Android-app](/nl/platforms/android)
- Windows Hub: [Windows](/nl/platforms/windows)
- Linux-status: [Linux-app](/nl/platforms/linux)
- Beveiliging: [Beveiliging](/nl/gateway/security)

## Gerelateerd

- [Aan de slag](/nl/start/getting-started)
- [Configuratie](/nl/start/setup)
- [Overzicht van kanalen](/nl/channels)
