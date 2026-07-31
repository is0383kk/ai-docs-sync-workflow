---
read_when:
    - Konfigurieren eines Kanal-Plugins (Authentifizierung, Zugriffskontrolle, mehrere Konten)
    - Fehlerbehebung bei kanalspezifischen Konfigurationsschlüsseln
    - Überprüfung von DM-Richtlinien, Gruppenrichtlinien oder der Erwähnungssteuerung
summary: 'Kanalkonfiguration: Zugriffskontrolle, Kopplung und kanalspezifische Schlüssel für Slack, Discord, Telegram, WhatsApp, Matrix, iMessage und weitere Kanäle'
title: Konfiguration — Kanäle
x-i18n:
    generated_at: "2026-07-26T18:57:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

Konfigurationsschlüssel pro Kanal unter `channels.*`: DM- und Gruppenzugriff, Mehrkontoeinrichtungen, Erwähnungssteuerung sowie kanalspezifische Schlüssel für Slack, Discord, Telegram, WhatsApp, Matrix, iMessage und andere Kanal-Plugins.

Informationen zu Agenten, Tools, Gateway-Laufzeit und anderen Schlüsseln der obersten Ebene finden Sie in der [Konfigurationsreferenz](/de/gateway/configuration-reference).

## Kanäle

Jeder Kanal startet automatisch, wenn sein Konfigurationsabschnitt vorhanden ist (sofern nicht `enabled: false`). Telegram und iMessage sind im zentralen `openclaw`-Paket enthalten. Andere offizielle Kanäle (Discord, Slack, WhatsApp, Matrix, Microsoft Teams, IRC, Google Chat, Signal, Mattermost und weitere) werden mit `openclaw plugins install <spec>` als separate Plugins installiert; die vollständige Liste und Installationsspezifikationen finden Sie unter [Kanäle](/de/channels).

### DM- und Gruppenzugriff

Alle Kanäle unterstützen DM- und Gruppenrichtlinien:

| DM-Richtlinie           | Verhalten                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (Standard) | Unbekannte Absender erhalten einen einmaligen Kopplungscode; der Eigentümer muss ihn genehmigen |
| `allowlist`         | Nur Absender in `allowFrom` (oder im Speicher gekoppelter Zulassungen)             |
| `open`              | Alle eingehenden DMs zulassen (erfordert `allowFrom: ["*"]`)             |
| `disabled`          | Alle eingehenden DMs ignorieren                                          |

| Gruppenrichtlinie          | Verhalten                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (Standard) | Nur Gruppen, die der konfigurierten Zulassungsliste entsprechen          |
| `open`                | Gruppenzulassungslisten umgehen (Erwähnungssteuerung gilt weiterhin) |
| `disabled`            | Alle Gruppen-/Raumnachrichten blockieren                          |

<Note>
`channels.defaults.groupPolicy` legt den Standardwert fest, wenn `groupPolicy` eines Providers nicht gesetzt ist.
Kopplungscodes laufen nach 1 Stunde ab. Ausstehende Kopplungsanfragen sind auf **3 pro Konto** begrenzt (nach Kanal und Konto-ID getrennt).
Wenn ein Provider-Block vollständig fehlt (`channels.<provider>` nicht vorhanden), fällt die Gruppenrichtlinie der Laufzeit mit einer Startwarnung auf `allowlist` zurück (standardmäßig ablehnend).
</Note>

### Kanalspezifische Modellüberschreibungen

Verwenden Sie `channels.modelByChannel`, um bestimmte Kanal-IDs oder Direktnachrichtenkontakte an ein Modell zu binden. Als Werte werden `provider/model` oder konfigurierte Modellaliase akzeptiert. Die Kanalzuordnung gilt nur, wenn eine Sitzung noch keine aktive Modellüberschreibung besitzt (beispielsweise eine über `/model` festgelegte).

Bei Gruppen-/Thread-Unterhaltungen sind die Schlüssel kanalspezifische Gruppen-IDs, Themen-IDs oder Kanalnamen. Bei Direktnachrichten-Unterhaltungen (DM) sind die Schlüssel Kontaktkennungen, die aus der Absenderidentität des Kanals abgeleitet werden (`nativeDirectUserId`, `origin.from`, `origin.to`, `OriginatingTo`, `From` oder `SenderId`). Die genaue Schlüsselform hängt vom Kanal ab:

| Kanal  | Form des DM-Schlüssels         | Beispiel                                      |
| -------- | ------------------- | -------------------------------------------- |
| Discord  | unverarbeitete Benutzer-ID         | `987654321`                                  |
| Feishu   | `feishu:ou_...`     | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85` |
| Matrix   | Matrix-Benutzer-ID      | `@user:matrix.org`                           |
| Slack    | `user:U...`         | `user:U12345`                                |
| Telegram | unverarbeitete Benutzer-ID         | `123456789`                                  |
| WhatsApp | Telefonnummer oder JID | `15551234567`                                |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

DM-spezifische Schlüssel stimmen nur in Direktnachrichten-Unterhaltungen überein; sie beeinflussen nicht das Routing von Gruppen oder Threads.

### Kanalstandards und Heartbeat

Verwenden Sie `channels.defaults` für gemeinsames Verhalten von Gruppenrichtlinien, impliziten Erwähnungen und Heartbeat über mehrere Provider hinweg:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: Ersatz-Gruppenrichtlinie, wenn `groupPolicy` auf Providerebene nicht gesetzt ist.
- `channels.defaults.contextVisibility`: standardmäßiger Modus für die Sichtbarkeit ergänzenden Kontexts für alle Kanäle. Werte: `all` (Standard; gesamten zitierten, Thread- und Verlaufskontext einbeziehen), `allowlist` (nur Kontext von Absendern auf der Zulassungsliste einbeziehen), `allowlist_quote` (wie die Zulassungsliste, aber expliziten Zitat-/Antwortkontext beibehalten). Kanalspezifische Überschreibung: `channels.<channel>.contextVisibility`.
- `channels.defaults.implicitMentions`: steuert, welche unterstützten eingehenden Fakten als Erwähnungen gelten. `replyToBot`, `quotedBot` und `threadParticipation` verwenden jeweils standardmäßig `true`, wodurch das aktuelle Verhalten erhalten bleibt. Überschreiben Sie dies pro Kanal mit `channels.<channel>.implicitMentions` oder pro Konto mit `channels.<channel>.accounts.<id>.implicitMentions`; jedes Flag wird unabhängig in der Reihenfolge Konto -> Kanal -> Standardwerte aufgelöst. Die Namen sind positiv formuliert: Setzen Sie ein Flag auf `false`, damit dieser Fakt die Erwähnungssteuerung nicht mehr umgeht. Native explizite Erwähnungen sind immer zulässig, und ein Flag hat keine Wirkung, wenn der Kanal diesen Fakt nicht erzeugt. Die aktuelle Erzeugermatrix finden Sie unter [Erwähnungssteuerung](/de/channels/groups#mention-gating-default). Diese Einstellungen ändern weder die Modi für ausgehende Antworten/Threads noch die Verarbeitung autorisierter Befehle.
- `channels.defaults.heartbeat.showOk`: fehlerfreie Kanalstatus in die Heartbeat-Ausgabe aufnehmen (Standard `false`).
- `channels.defaults.heartbeat.showAlerts`: beeinträchtigte/fehlerhafte Status in die Heartbeat-Ausgabe aufnehmen (Standard `true`).
- `channels.defaults.heartbeat.useIndicator`: kompakte Heartbeat-Ausgabe im Indikatorstil darstellen (Standard `true`).

### WhatsApp

WhatsApp wird über den Webkanal des Gateways ausgeführt (Baileys Web). Es startet automatisch, sobald eine verknüpfte Sitzung vorhanden ist.

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- Einträge der obersten Ebene in `bindings[]` mit `type: "acp"` konfigurieren persistente ACP-Bindungen für WhatsApp-DMs und -Gruppen. Verwenden Sie in `match.peer.id` eine direkte Nummer im E.164-Format oder eine WhatsApp-Gruppen-JID. Die Feldsemantik wird unter [ACP-Agenten](/de/tools/acp-agents#persistent-channel-bindings) beschrieben.

<Accordion title="WhatsApp mit mehreren Konten">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Ausgehende Befehle verwenden standardmäßig das Konto `default`, falls vorhanden; andernfalls die erste konfigurierte Konto-ID (sortiert).
- Das optionale `channels.whatsapp.defaultAccount` überschreibt diese standardmäßige Auswahl des Ersatzkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.
- Das veraltete Baileys-Authentifizierungsverzeichnis für ein einzelnes Konto wird durch `openclaw doctor` nach `whatsapp/default` migriert.
- Kontospezifische Überschreibungen: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress (default: partial)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bot-Token: `channels.telegram.botToken` oder `channels.telegram.tokenFile` (nur reguläre Datei; symbolische Links werden abgelehnt), mit `TELEGRAM_BOT_TOKEN` als Ersatz für das Standardkonto.
- `apiRoot` ist ausschließlich die Wurzel der Telegram Bot API. Verwenden Sie `https://api.telegram.org` oder die Wurzel Ihrer selbst gehosteten beziehungsweise Proxy-Instanz, nicht `https://api.telegram.org/bot<TOKEN>`; `openclaw doctor --fix` entfernt ein versehentlich angehängtes `/bot<TOKEN>`-Suffix.
- Bei einem selbst gehosteten Bot-API-Server im `--local`-Modus listet `trustedLocalFileRoots` die Hostpfade auf, die OpenClaw lesen darf. Binden Sie das Datenvolume des Servers auf dem OpenClaw-Host ein und konfigurieren Sie entweder dessen Datenstamm oder das Token-spezifische Verzeichnis; Containerpfade unter `/var/lib/telegram-bot-api` werden diesen Stammverzeichnissen zugeordnet. Andere absolute Pfade werden weiterhin abgelehnt.
- Das optionale `channels.telegram.defaultAccount` überschreibt die Auswahl des Standardkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.
- Legen Sie bei Mehrkontoeinrichtungen (2+ Konto-IDs) einen expliziten Standardwert fest (`channels.telegram.defaultAccount` oder `channels.telegram.accounts.default`), um Ersatz-Routing zu vermeiden; `openclaw doctor` warnt, wenn dieser fehlt oder ungültig ist.
- `configWrites: false` blockiert von Telegram initiierte Konfigurationsschreibvorgänge (Supergruppen-ID-Migrationen, `/config set|unset`).
- Einträge der obersten Ebene in `bindings[]` mit `type: "acp"` konfigurieren persistente ACP-Bindungen für Forenthemen (verwenden Sie das kanonische `chatId:topic:topicId` in `match.peer.id`). Die Feldsemantik wird unter [ACP-Agenten](/de/tools/acp-agents#persistent-channel-bindings) beschrieben.
- Telegram-Streamingvorschauen verwenden `sendMessage` + `editMessageText` (funktioniert in Direkt- und Gruppenchats).
- `network.dnsResultOrder` verwendet standardmäßig `"ipv4first"`, um häufige IPv6-Abruffehler zu vermeiden.
- Wiederholungsrichtlinie: siehe [Wiederholungsrichtlinie](/de/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Nur kurze Antworten.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress (Discord-Standard: progress)
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Token: `channels.discord.token`, mit `DISCORD_BOT_TOKEN` als Fallback für das Standardkonto.
- Direkte ausgehende Aufrufe, die explizit ein Discord-`token` angeben, verwenden dieses Token für den Aufruf; Einstellungen für Wiederholungsversuche und Richtlinien des Kontos stammen weiterhin aus dem ausgewählten Konto im aktiven Runtime-Snapshot.
- Das optionale `channels.discord.defaultAccount` überschreibt die Auswahl des Standardkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.
- Verwenden Sie `user:<id>` (DM) oder `channel:<id>` (Guild-Kanal) für Zustellungsziele; reine numerische IDs werden abgelehnt.
- Guild-Slugs bestehen aus Kleinbuchstaben, wobei Leerzeichen durch `-` ersetzt werden; Kanalschlüssel verwenden den Namen als Slug (ohne `#`). Bevorzugen Sie Guild-IDs.
- Von Bots verfasste Nachrichten werden standardmäßig ignoriert. `allowBots: true` aktiviert sie; verwenden Sie `allowBots: "mentions"`, um nur Bot-Nachrichten zu akzeptieren, in denen der Bot erwähnt wird (eigene Nachrichten werden weiterhin herausgefiltert).
- Kanäle, die von Bots verfasste eingehende Nachrichten unterstützen, können den gemeinsamen [Bot-Schleifenschutz](/de/channels/bot-loop-protection) verwenden. Legen Sie `channels.defaults.botLoopProtection` für grundlegende Paarbudgets fest und überschreiben Sie anschließend den Kanal oder das Konto nur, wenn eine Oberfläche andere Grenzwerte benötigt.
- `channels.discord.guilds.<id>.ignoreOtherMentions` (und Kanalüberschreibungen) verwirft Nachrichten, die einen anderen Benutzer oder eine andere Rolle, aber nicht den Bot erwähnen (ausgenommen @everyone/@here).
- `channels.discord.mentionAliases` ordnet stabilen ausgehenden `@handle`-Text vor dem Senden Discord-Benutzer-IDs zu, sodass bekannte Teammitglieder auch dann deterministisch erwähnt werden können, wenn der temporäre Verzeichnis-Cache leer ist. Kontospezifische Überschreibungen befinden sich unter `channels.discord.accounts.<accountId>.mentionAliases`.
- `maxLinesPerMessage` (Standard: `17`) teilt hohe Nachrichten auch dann auf, wenn sie weniger als 2000 Zeichen umfassen.
- `channels.discord.suppressEmbeds` ist standardmäßig `true`, sodass ausgehende URLs nicht zu Discord-Linkvorschauen erweitert werden, sofern dies nicht deaktiviert wird. Explizite `embeds`-Payloads werden weiterhin normal gesendet; Tool-Aufrufe für einzelne Nachrichten können dies mit `suppressEmbeds` überschreiben.
- `channels.discord.threadBindings` steuert das an Discord-Threads gebundene Routing:
  - `enabled`: Discord-Überschreibung für an Threads gebundene Sitzungsfunktionen (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` sowie gebundene Zustellung und gebundenes Routing)
  - `idleHours`: Discord-Überschreibung für das automatische Aufheben des Fokus nach Inaktivität in Stunden (`0` deaktiviert dies)
  - `maxAgeHours`: Discord-Überschreibung für das feste Höchstalter in Stunden (`0` deaktiviert dies)
  - `spawnSessions`: Schalter für `sessions_spawn({ thread: true })` und die automatische Thread-Erstellung/-Bindung beim Erzeugen von ACP-Threads (Standard: `true`)
  - `defaultSpawnContext`: nativer Subagent-Kontext für an Threads gebundene Erzeugungen (standardmäßig `"fork"`)
- Einträge der obersten Ebene unter `bindings[]` mit `type: "acp"` konfigurieren persistente ACP-Bindungen für Kanäle und Threads (verwenden Sie die Kanal-/Thread-ID in `match.peer.id`). Die Feldsemantik wird unter [ACP-Agenten](/de/tools/acp-agents#persistent-channel-bindings) gemeinsam erläutert.
- `channels.discord.ui.components.accentColor` legt die Akzentfarbe für Discord-Komponentencontainer der Version 2 fest.
- `channels.discord.agentComponents.ttlMs` steuert, wie lange Callbacks gesendeter Discord-Komponenten registriert bleiben. Standard: `1800000` (30 Minuten), Maximum: `86400000` (24 Stunden). Kontospezifische Überschreibungen befinden sich unter `channels.discord.accounts.<accountId>.agentComponents.ttlMs`. Bevorzugen Sie die kürzeste TTL, die für den Workflow ausreicht.
- `channels.discord.voice` aktiviert Unterhaltungen in Discord-Sprachkanälen sowie optionale Überschreibungen für automatischen Beitritt, LLM und TTS. Reine Textkonfigurationen für Discord lassen Sprachfunktionen standardmäßig deaktiviert; setzen Sie `channels.discord.voice.enabled=true`, um sie zu aktivieren.
- `channels.discord.voice.model` überschreibt optional das LLM-Modell, das für Antworten in Discord-Sprachkanälen verwendet wird.
- `channels.discord.voice.daveEncryption` (Standard: `true`) und `channels.discord.voice.decryptionFailureTolerance` (Standard: `24`) werden an die DAVE-Optionen von `@discordjs/voice` durchgereicht.
- `channels.discord.voice.connectTimeoutMs` steuert die anfängliche Wartezeit auf den `@discordjs/voice`-Ready-Zustand für `/vc join` und Versuche zum automatischen Beitritt (Standard: `30000`).
- `channels.discord.voice.reconnectGraceMs` steuert, wie lange eine getrennte Sprachsitzung benötigen darf, um in die Signalisierung für die Wiederverbindung einzutreten, bevor OpenClaw sie beendet (Standard: `15000`).
- Die Discord-Sprachwiedergabe wird nicht durch das Ereignis zum Sprechbeginn eines anderen Benutzers unterbrochen. Um Rückkopplungsschleifen zu vermeiden, ignoriert OpenClaw neue Sprachaufnahmen, während TTS wiedergegeben wird.
- OpenClaw versucht zusätzlich, den Sprachempfang nach wiederholten Entschlüsselungsfehlern wiederherzustellen, indem es eine Sprachsitzung verlässt und ihr erneut beitritt.
- `channels.discord.streaming` ist der kanonische Schlüssel für den Streaming-Modus. Discord verwendet standardmäßig `streaming.mode: "progress"`, sodass der Fortschritt von Tools und Arbeiten in einer einzigen bearbeiteten Vorschaunachricht erscheint; setzen Sie `streaming.mode: "off"`, um dies zu deaktivieren. Veraltete flache Schlüssel (`streamMode`, `chunkMode`, `blockStreaming`, `draftChunk`, `blockStreamingCoalesce`) werden zur Laufzeit nicht mehr gelesen; führen Sie `openclaw doctor --fix` aus, um die persistierte Konfiguration zu migrieren.
- `channels.discord.autoPresence` ordnet die Runtime-Verfügbarkeit der Bot-Präsenz zu (fehlerfrei => online, beeinträchtigt => idle, erschöpft => dnd) und erlaubt optionale Überschreibungen des Statustexts.
- `channels.discord.guilds.<id>.presenceEvents` leitet Ereignisse zur Verfügbarkeit von Personen als Agent-Systemereignisse an einen konfigurierten Discord-Kanal weiter. Berechtigte Mitglieder müssen `channelId` sehen können; öffentliche Threads erben die Sichtbarkeit des übergeordneten Elements, während private Threads zusätzlich eine Mitgliedschaft oder Manage Threads erfordern. `users` kann diesen Personenkreis weiter einschränken. Die Funktion initialisiert aktuell online befindliche Mitglieder aus vollständigen `GUILD_CREATE`-Snapshots, leitet beobachtete Übergänge von offline zu online weiter und behandelt ein erstes späteres Online-Signal eines zuvor nicht gesehenen Mitglieds als neu verfügbar, ohne festzustellen, ob die Person online gegangen oder erst nach dem Snapshot beigetreten ist. Guilds oberhalb des Discord-Snapshot-Limits von 75,000 Mitgliedern benötigen zunächst eine explizite Offline-Aktualisierung. Optionen zur Drosselung: `reconnectSuppressSeconds` (Ruhefenster nach einer neuen Gateway-Sitzung, während der Guild-Präsenzstatus neu aufgebaut wird; Standard: 300, `0` deaktiviert es) und `burstLimit`/`burstWindowSeconds` (Ratenbegrenzung für erfolgreich in die Warteschlange gestellte Ereignisse pro Guild; Standard: 8 Ereignisse pro gleitendem 60s-Fenster). Fortgesetzte Sitzungen starten das Unterdrückungsfenster für Wiederverbindungen nicht. Die bestehende Abklingzeit für erneute Begrüßungen pro Benutzer bleibt bei acht Stunden. Erforderlich sind `channels.discord.intents.presence=true`, der privilegierte Presence Intent im Developer Portal von Discord und ein aktivierter Agent-Heartbeat.
- `channels.discord.dangerouslyAllowNameMatching` aktiviert die veränderliche Namens-/Tag-Zuordnung erneut (Break-Glass-Kompatibilitätsmodus).
- `channels.discord.execApprovals`: Discord-native Zustellung von Ausführungsgenehmigungen und Autorisierung der Genehmigenden.
  - `enabled`: `true`, `false` oder `"auto"` (Standard). Im automatischen Modus werden Ausführungsgenehmigungen aktiviert, wenn Genehmigende über `approvers` oder `commands.ownerAllowFrom` aufgelöst werden können.
  - `approvers`: Discord-Benutzer-IDs, die Ausführungsanfragen genehmigen dürfen. Fällt bei Auslassung auf `commands.ownerAllowFrom` zurück.
  - `agentFilter`: optionale Positivliste für Agent-IDs. Lassen Sie sie weg, um Genehmigungen für alle Agenten weiterzuleiten.
  - `sessionFilter`: optionale Muster für Sitzungsschlüssel (Teilzeichenfolge oder regulärer Ausdruck).
  - `target`: Ziel für Genehmigungsaufforderungen. `"dm"` (Standard) sendet sie an die DMs der Genehmigenden, `"channel"` sendet sie an den Ursprungskanal und `"both"` sendet sie an beide. Wenn das Ziel `"channel"` umfasst, können die Schaltflächen nur von den ermittelten Genehmigenden verwendet werden.
  - `cleanupAfterResolve`: Wenn `true` gesetzt ist, werden Genehmigungs-DMs nach Genehmigung, Ablehnung oder Zeitüberschreitung gelöscht.

**Modi für Reaktionsbenachrichtigungen:** `off` (keine), `own` (Nachrichten des Bots, Standard), `all` (alle Nachrichten), `allowlist` (von `guilds.<id>.users` bei allen Nachrichten).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- Dienstkonto-JSON: inline (`serviceAccount`) oder dateibasiert (`serviceAccountFile`).
- `serviceAccount` akzeptiert direkt eine SecretRef.
- Umgebungs-Fallbacks: `GOOGLE_CHAT_SERVICE_ACCOUNT` oder `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (nur für das Standardkonto).
- Verwenden Sie `spaces/<spaceId>` oder `users/<userId>` für Zustellungsziele.
- `channels.googlechat.dangerouslyAllowNameMatching` aktiviert die veränderliche Zuordnung von E-Mail-Principals erneut (Break-Glass-Kompatibilitätsmodus).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Nur kurze Antworten.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // off | partial | block | progress
        chunkMode: "length", // length | newline
        nativeTransport: true, // native Slack-Streaming-API verwenden, wenn mode=partial
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket-Modus** erfordert sowohl `botToken` als auch `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` für den Umgebungsvariablen-Fallback des Standardkontos).
- **HTTP-Modus** erfordert `botToken` sowie `signingSecret` (auf Stammebene oder pro Konto).
- **Benutzeridentität** (`identity: "user"`) veröffentlicht und liest als der autorisierende Mensch. Sie erfordert `userToken` sowie `appToken` im Socket-Modus oder `userToken` sowie `signingSecret` im HTTP-Modus. Weder ein Bot-Token noch ein Bot-Benutzer ist erforderlich. Informationen zu Benutzerberechtigungen und Ereignisabonnements finden Sie unter [Benutzeridentität](/de/channels/slack#user-identity-post-as-a-real-person).
- `enterpriseOrgInstall: true` bindet ein Konto in den organisationsweiten Ereignispfad von Slack Enterprise Grid ein. Beim Start wird das Bot-Token mit `auth.test` überprüft;
  der Start schlägt fehl, wenn der konfigurierte Modus nicht mit der Installationsidentität von Slack übereinstimmt.
  Unternehmens-DMs müssen deaktiviert sein oder `dmPolicy: "open"` mit einem wirksamen
  `allowFrom: ["*"]` verwenden. Kanal- und Benutzerrichtlinien müssen stabile Slack-IDs verwenden;
  veränderliche Namen und nicht unterstützte Kanalpräfixe führen zum Fehlschlagen des Starts. V1 verarbeitet nur
  direkte Socket-Modus- oder HTTP-Ereignisse vom Typ `message` und `app_mention` mit sofortigen
  Antworten; Relay, Befehle, Interaktionen, App Home, Listener für Reaktionsereignisse,
  Pins, Aktionswerkzeuge, native Genehmigungen, Bindungen, verzögerte Zustellung und
  proaktives Senden sind nicht verfügbar. Listener-eigene Bestätigungen, Tipp- und
  Statusreaktionen bleiben mit `reactions:write` verfügbar; eingehende
  Reaktionsbenachrichtigungen und Reaktionsaktionswerkzeuge sind nicht verfügbar. Informationen zum
  Least-Privilege-Manifest, Einrichtungsablauf und zu allen Einschränkungen finden Sie unter
  [Organisationsweite Enterprise-Grid-Installationen](/de/channels/slack#enterprise-grid-org-wide-installs).
- `socketMode` übergibt die Transportoptimierung des Slack-SDK-Socket-Modus an die öffentliche Bolt-Receiver-API. Verwenden Sie dies nur zur Untersuchung von Ping/Pong-Zeitüberschreitungen oder veraltetem WebSocket-Verhalten. `clientPingTimeout` verwendet standardmäßig `15000`; `serverPingTimeout` und `pingPongLoggingEnabled` werden nur übergeben, wenn sie konfiguriert sind.
- `botToken`, `appToken`, `signingSecret` und `userToken` akzeptieren Klartextzeichenfolgen
  oder SecretRef-Objekte.
- Slack-Kontomomentaufnahmen stellen quell- und statusbezogene Felder pro Anmeldedaten bereit, beispielsweise
  `botTokenSource`, `botTokenStatus`, `userTokenSource`, `userTokenStatus`,
  `appTokenStatus` und im HTTP-Modus `signingSecretStatus`.
  `configured_unavailable` bedeutet, dass das Konto
  über SecretRef konfiguriert ist, der aktuelle Befehls- bzw. Laufzeitpfad den
  Geheimniswert jedoch nicht auflösen konnte.
- `configWrites: false` blockiert von Slack initiierte Konfigurationsschreibvorgänge.
- Das optionale `channels.slack.defaultAccount` überschreibt die Auswahl des Standardkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.
- `channels.slack.streaming.mode` ist der kanonische Schlüssel für den Slack-Stream-Modus (Standard: `"partial"`). `channels.slack.streaming.nativeTransport` steuert den nativen Streaming-Transport von Slack (Standard: `true`). Veraltete Werte für `streamMode`, den booleschen Wert `streaming`, `chunkMode`, `blockStreaming`, `blockStreamingCoalesce` und `nativeStreaming` werden zur Laufzeit nicht mehr gelesen; führen Sie `openclaw doctor --fix` aus, um die persistierte Konfiguration zu `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}` zu migrieren.
- `unfurlLinks` und `unfurlMedia` übergeben die booleschen Slack-Werte `chat.postMessage` für Link- und Medienvorschauen bei Bot-Antworten. `unfurlLinks` verwendet standardmäßig `false`, sodass ausgehende Bot-Links nicht inline erweitert werden, sofern dies nicht aktiviert ist; `unfurlMedia` wird weggelassen, sofern es nicht konfiguriert ist. Legen Sie einen der Werte unter `channels.slack.accounts.<accountId>` fest, um den Wert der obersten Ebene für ein einzelnes Konto zu überschreiben.
- Verwenden Sie `user:<id>` (DM) oder `channel:<id>` als Zustellungsziele.

**Modi für Reaktionsbenachrichtigungen:** `off`, `own` (Standard), `all`, `allowlist` (von `reactionAllowlist`).

**Thread-Sitzungsisolierung:** `thread.historyScope` gilt pro Thread (Standard) oder wird kanalweit geteilt. `thread.inheritParent` kopiert das Transkript des übergeordneten Kanals in neue Threads. `thread.initialHistoryLimit` (Standard: `20`) begrenzt die Anzahl vorhandener Thread-Nachrichten, die beim Start einer neuen Thread-Sitzung abgerufen werden; `0` deaktiviert das Abrufen des Thread-Verlaufs.

- Natives Slack-Streaming sowie der assistentenähnliche Thread-Status „is typing...“ von Slack erfordern einen Antwort-Thread als Ziel. DMs auf oberster Ebene bleiben standardmäßig außerhalb von Threads, sodass sie weiterhin über Slack-Entwurfsvorschauen mit Veröffentlichung und Bearbeitung streamen können, anstatt die native, threadbasierte Stream-/Statusvorschau anzuzeigen.
- `typingReaction` fügt der eingehenden Slack-Nachricht vorübergehend eine Reaktion hinzu, während eine Antwort ausgeführt wird, und entfernt sie nach Abschluss. Verwenden Sie einen Slack-Emoji-Shortcode wie `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: Slack-native Zustellung an den Genehmigungsclient und Autorisierung ausführender Genehmiger. Dasselbe Schema wie bei Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (Slack-Benutzer-IDs), `agentFilter`, `sessionFilter` und `target` (`"dm"`, `"channel"` oder `"both"`). Plugin-Genehmigungen können diesen nativen Clientpfad für von Slack stammende Anfragen verwenden, wenn Slack-Plugin-Genehmiger aufgelöst werden; die Slack-native Zustellung von Plugin-Genehmigungen kann außerdem über `approvals.plugin` für von Slack stammende Sitzungen oder Slack-Ziele aktiviert werden. Plugin-Genehmigungen verwenden Slack-Plugin-Genehmiger aus `allowFrom` und das Standardrouting, nicht die ausführenden Genehmiger.

| Aktionsgruppe | Standard   | Hinweise                         |
| ------------- | ---------- | -------------------------------- |
| reactions     | aktiviert  | Reagieren + Reaktionen auflisten |
| messages      | aktiviert  | Lesen/Senden/Bearbeiten/Löschen  |
| pins          | aktiviert  | Anheften/Lösen/Auflisten         |
| memberInfo    | aktiviert  | Mitgliedsinformationen           |
| emojiList     | aktiviert  | Liste benutzerdefinierter Emojis |

### Mattermost

Mattermost wird genau wie Discord, Slack und WhatsApp als separates Plugin installiert:

```bash
openclaw plugins install @openclaw/mattermost
```

Prüfen Sie vor dem Festlegen einer Version die aktuellen Dist-Tags unter [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost).

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // optional aktivieren
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Optionale explizite URL für Reverse-Proxy-/öffentliche Bereitstellungen
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

Chatmodi: `oncall` (bei @-Erwähnung antworten, Standard), `onmessage` (jede Nachricht), `onchar` (Nachrichten, die mit einem Auslösepräfix beginnen).

Wenn native Mattermost-Befehle aktiviert sind:

- `commands.callbackPath` muss ein Pfad sein (beispielsweise `/api/channels/mattermost/command`), keine vollständige URL.
- `commands.callbackUrl` muss zum OpenClaw-Gateway-Endpunkt aufgelöst werden und vom Mattermost-Server erreichbar sein.
- Native Slash-Callbacks werden mit den befehlsspezifischen Tokens authentifiziert, die
  Mattermost bei der Registrierung von Slash-Befehlen zurückgibt. Wenn die Registrierung fehlschlägt oder keine
  Befehle aktiviert werden, lehnt OpenClaw Callbacks mit
  `Unauthorized: invalid command token.` ab.
- Bei privaten Callback-Hosts bzw. Callback-Hosts im Tailnet oder internen Netz kann Mattermost verlangen,
  dass `ServiceSettings.AllowedUntrustedInternalConnections` den Callback-Host bzw. die Callback-Domain enthält.
  Verwenden Sie Host-/Domainwerte, keine vollständigen URLs.
- `channels.mattermost.configWrites`: von Mattermost initiierte Konfigurationsschreibvorgänge zulassen oder verweigern.
- `channels.mattermost.requireMention`: `@mention` vor dem Antworten in Kanälen voraussetzen.
- `channels.mattermost.groups.<channelId>.requireMention`: kanalspezifische Überschreibung der Erwähnungspflicht (`"*"` für den Standard).
- Das optionale `channels.mattermost.defaultAccount` überschreibt die Auswahl des Standardkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // optionale Kontobindung
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Modi für Reaktionsbenachrichtigungen:** `off`, `own` (Standard), `all`, `allowlist` (von `reactionAllowlist`).

- `channels.signal.account`: den Kanalstart an eine bestimmte Signal-Kontoidentität binden.
- `channels.signal.configWrites`: von Signal initiierte Konfigurationsschreibvorgänge zulassen oder verweigern.
- Das optionale `channels.signal.defaultAccount` überschreibt die Auswahl des Standardkontos, wenn es mit einer konfigurierten Konto-ID übereinstimmt.

### iMessage

OpenClaw startet `imsg rpc` (JSON-RPC über stdio). Es ist weder ein Daemon noch ein Port erforderlich. Dies ist der bevorzugte Pfad für neue OpenClaw-iMessage-Einrichtungen, wenn der Host Berechtigungen für die Messages-Datenbank und Automation erteilen kann.

Die Unterstützung für BlueBubbles wurde entfernt. `channels.bluebubbles` ist im aktuellen OpenClaw keine unterstützte Laufzeitkonfigurationsoberfläche. Migrieren Sie alte Konfigurationen zu `channels.imessage`; die Kurzfassung finden Sie unter [Entfernung von BlueBubbles und der imsg-iMessage-Pfad](/de/announcements/bluebubbles-imessage), die vollständige Übersetzungstabelle unter [Umstieg von BlueBubbles](/de/channels/imessage-from-bluebubbles).

Wenn das Gateway nicht auf dem bei Messages angemeldeten Mac ausgeführt wird, behalten Sie `channels.imessage.enabled=true` bei und setzen Sie `channels.imessage.cliPath` auf einen SSH-Wrapper, der `imsg "$@"` auf diesem Mac ausführt. Der standardmäßige lokale Pfad `imsg` ist ausschließlich für macOS vorgesehen.

Bevor Sie sich für produktive Sendevorgänge auf einen SSH-Wrapper verlassen, verifizieren Sie einen ausgehenden `imsg send` über genau diesen Wrapper. In einigen macOS-TCC-Zuständen wird die Nachrichtenautomatisierung `/usr/libexec/sshd-keygen-wrapper` zugewiesen, wodurch Lesevorgänge und Prüfungen funktionieren können, während Sendevorgänge mit AppleEvents `-1743` fehlschlagen; siehe den Abschnitt zur Fehlerbehebung für SSH-Wrapper unter [iMessage](/de/channels/imessage).

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- Optional überschreibt `channels.imessage.defaultAccount` die standardmäßige Kontoauswahl, wenn der Wert mit einer konfigurierten Konto-ID übereinstimmt.
- Erfordert vollständigen Festplattenzugriff auf die Nachrichtendatenbank.
- Bevorzugen Sie `chat_id:<id>`-Ziele. Verwenden Sie `imsg chats --limit 20`, um Chats aufzulisten.
- `cliPath` kann auf einen SSH-Wrapper verweisen; legen Sie `remoteHost` (`host` oder `user@host`) für den Abruf von Anhängen per SCP fest.
- `attachmentRoots` und `remoteAttachmentRoots` beschränken die Pfade eingehender Anhänge (Standard: `/Users/*/Library/Messages/Attachments`).
- SCP verwendet eine strikte Hostschlüsselprüfung. Stellen Sie daher sicher, dass der Hostschlüssel des Relay-Hosts bereits in `~/.ssh/known_hosts` vorhanden ist.
- `channels.imessage.configWrites`: Von iMessage initiierte Konfigurationsschreibvorgänge zulassen oder verweigern.
- `channels.imessage.sendTransport`: bevorzugter `imsg`-RPC-Sendetransport für normale ausgehende Antworten. `auto` (Standard) verwendet für bestehende Chats die IMCore-Bridge, wenn sie ausgeführt wird, und greift anschließend auf AppleScript zurück; `bridge` erfordert die Zustellung über eine private API; `applescript` erzwingt den öffentlichen Automatisierungspfad von Nachrichten.
- `channels.imessage.actions.*`: Private API-Aktionen aktivieren, die zusätzlich durch `imsg status` / `openclaw channels status --probe` eingeschränkt werden.
- `channels.imessage.includeAttachments` ist standardmäßig deaktiviert; legen Sie den Wert auf `true` fest, bevor Sie eingehende Medien in Agentendurchläufen erwarten.
- Die Wiederherstellung eingehender Nachrichten nach einem Neustart der Bridge/des Gateways erfolgt automatisch (GUID-Deduplizierung plus Altersgrenze für veraltete Rückstände). Bestehende `channels.imessage.catchup.enabled: true`-Konfigurationen werden weiterhin als veraltetes Kompatibilitätsprofil berücksichtigt; `catchup` ist standardmäßig deaktiviert.
- `channels.imessage.groups`: Gruppenregister und gruppenspezifische Einstellungen. Konfigurieren Sie bei `groupPolicy: "allowlist"` entweder explizite `chat_id`-Schlüssel oder einen `"*"`-Platzhaltereintrag, damit Gruppennachrichten die Registerprüfung passieren können.
- Einträge der obersten Ebene unter `bindings[]` mit `type: "acp"` können iMessage-Unterhaltungen an persistente ACP-Sitzungen binden. Verwenden Sie in `match.peer.id` ein normalisiertes Handle oder ein explizites Chatziel (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`). Gemeinsame Feldsemantik: [ACP-Agenten](/de/tools/acp-agents#persistent-channel-bindings).

<Accordion title="Beispiel für einen iMessage-SSH-Wrapper">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix wird durch ein Plugin bereitgestellt und unter `channels.matrix` konfiguriert.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- Die Token-Authentifizierung verwendet `accessToken`; die Passwortauthentifizierung verwendet `userId` + `password`.
- `channels.matrix.proxy` leitet den Matrix-HTTP-Datenverkehr über einen expliziten HTTP(S)-Proxy. Benannte Konten können dies mit `channels.matrix.accounts.<id>.proxy` überschreiben.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` lässt private/interne Homeserver zu. `proxy` und diese Netzwerkfreigabe sind voneinander unabhängige Steuerelemente.
- `channels.matrix.defaultAccount` wählt das bevorzugte Konto in Konfigurationen mit mehreren Konten aus.
- `channels.matrix.autoJoin` verwendet standardmäßig `"off"`, sodass Einladungen zu Räumen und neue DM-ähnliche Einladungen ignoriert werden, bis Sie `autoJoin: "allowlist"` mit `autoJoinAllowlist` oder `autoJoin: "always"` festlegen.
- `channels.matrix.execApprovals`: Matrix-native Übermittlung von Ausführungsgenehmigungen und Autorisierung der Genehmigenden.
  - `enabled`: `true`, `false` oder `"auto"` (Standard). Im automatischen Modus werden Ausführungsgenehmigungen aktiviert, wenn die Genehmigenden aus `approvers` oder `commands.ownerAllowFrom` ermittelt werden können.
  - `approvers`: Matrix-Benutzer-IDs (z. B. `@owner:example.org`), die Ausführungsanfragen genehmigen dürfen.
  - `agentFilter`: optionale Positivliste für Agenten-IDs. Lassen Sie sie weg, um Genehmigungen für alle Agenten weiterzuleiten.
  - `sessionFilter`: optionale Muster für Sitzungsschlüssel (Teilzeichenfolge oder regulärer Ausdruck).
  - `target`: Ziel für Genehmigungsaufforderungen. `"dm"` (Standard), `"channel"` (Ursprungsraum) oder `"both"`.
  - Kontospezifische Überschreibungen: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` steuert, wie Matrix-DMs zu Sitzungen gruppiert werden: `per-user` (Standard) verwendet eine gemeinsame Sitzung pro weitergeleitetem Kommunikationspartner, während `per-room` jeden DM-Raum isoliert.
- Matrix-Statusprüfungen und Live-Verzeichnissuchen verwenden dieselbe Proxy-Richtlinie wie der Laufzeitdatenverkehr.
- Die vollständige Matrix-Konfiguration, Zielregeln und Einrichtungsbeispiele sind unter [Matrix](/de/channels/matrix) dokumentiert.

### Microsoft Teams

Microsoft Teams wird durch ein Plugin bereitgestellt und unter `channels.msteams` konfiguriert.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // siehe /channels/msteams
    },
  },
}
```

- Hier behandelte zentrale Schlüsselpfade: `channels.msteams`, `channels.msteams.configWrites`.
- Die vollständige Teams-Konfiguration (Anmeldedaten, Webhook, DM-/Gruppenrichtlinie sowie team- und kanalspezifische Überschreibungen) ist unter [Microsoft Teams](/de/channels/msteams) dokumentiert.

### IRC

IRC wird durch ein Plugin bereitgestellt und unter `channels.irc` konfiguriert.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Hier behandelte zentrale Schlüsselpfade: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- Optional überschreibt `channels.irc.defaultAccount` die standardmäßige Kontoauswahl, wenn der Wert mit einer konfigurierten Konto-ID übereinstimmt.
- Die vollständige IRC-Kanalkonfiguration (Host/Port/TLS/Kanäle/Positivlisten/Erwähnungsprüfung) ist unter [IRC](/de/channels/irc) dokumentiert.

### Mehrere Konten (alle Kanäle)

Führen Sie mehrere Konten pro Kanal aus (jeweils mit einem eigenen `accountId`):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default` wird verwendet, wenn `accountId` weggelassen wird (CLI + Routing).
- Umgebungsvariablen-Token gelten nur für das **Standardkonto**.
- Die grundlegenden Kanaleinstellungen gelten für alle Konten, sofern sie nicht kontospezifisch überschrieben werden.
- Verwenden Sie `bindings[].match.accountId`, um jedes Konto an einen anderen Agenten weiterzuleiten.
- Wenn Sie über `openclaw channels add` (oder die Kanaleinrichtung) ein nicht standardmäßiges Konto hinzufügen, während weiterhin eine Einkontokonfiguration des Kanals auf oberster Ebene verwendet wird, verschiebt OpenClaw zunächst die kontospezifischen Einkontowerte der obersten Ebene in die Kontozuordnung des Kanals, damit das ursprüngliche Konto weiterhin funktioniert. Die meisten Kanäle verschieben sie nach `channels.<channel>.accounts.default`; Matrix kann stattdessen ein vorhandenes passendes benanntes Ziel oder Standardziel beibehalten.
- Bestehende rein kanalbezogene Bindungen (ohne `accountId`) stimmen weiterhin mit dem Standardkonto überein; kontospezifische Bindungen bleiben optional.
- `openclaw doctor --fix` repariert außerdem gemischte Strukturen, indem kontospezifische Einkontowerte der obersten Ebene in das für diesen Kanal ausgewählte hochgestufte Konto verschoben werden. Die meisten Kanäle verwenden `accounts.default`; Matrix kann stattdessen ein vorhandenes passendes benanntes Ziel oder Standardziel beibehalten.

### Weitere Plugin-Kanäle

Viele Plugin-Kanäle werden als `channels.<id>` konfiguriert und auf ihren jeweiligen Kanalseiten dokumentiert (zum Beispiel Feishu, LINE, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Twitch und Zalo).
Den vollständigen Kanalindex finden Sie unter [Kanäle](/de/channels).

### Erwähnungsprüfung in Gruppenchats

Gruppennachrichten **erfordern standardmäßig eine Erwähnung** (Metadaten-Erwähnung oder sichere Regex-Muster). Dies gilt für Gruppenchats in WhatsApp, Telegram, Discord, Google Chat und iMessage.

Sichtbare Antworten werden separat gesteuert. Normale direkte Anfragen aus Gruppen, Kanälen und dem internen WebChat verwenden standardmäßig die automatische abschließende Zustellung: Der endgültige Assistententext wird über den bisherigen Pfad für sichtbare Antworten veröffentlicht. Aktivieren Sie `messages.visibleReplies: "message_tool"` oder `messages.groupChat.visibleReplies: "message_tool"`, wenn vom Modell verfasste Antworten an die Quelle erst veröffentlicht werden sollen, nachdem der Agent `message(action=send)` aufgerufen hat. Wenn das Modell in einem aktivierten Nur-Tool-Modus eine inhaltlich relevante endgültige Antwort zurückgibt, ohne das Nachrichtenwerkzeug aufzurufen, bleibt dieser endgültige Text privat, das ausführliche Gateway-Protokoll zeichnet Metadaten der unterdrückten Nutzlast auf und OpenClaw stellt einen Wiederherstellungsversuch in die Warteschlange, der das Modell auffordert, dieselbe Antwort über `message(action=send)` zuzustellen.

Die Nur-Tool-Richtlinie gilt für Assistentenantworten an die Quelle und generische Tool-Medien. Sie unterdrückt keine laufzeiteigenen Terminalausgaben wie autorisierte Befehlsantworten, dauerhafte Abschlussbenachrichtigungen oder Provider-native Artefakte, die das zuständige Ausführungssystem ausdrücklich als hosteigen klassifiziert. Hosteigene Artefakte werden über den normalen Kanalversandpfad zugestellt und berücksichtigen weiterhin eine ausgehende Verweigerung durch `sendPolicy`. Umgebungsgesteuerte `room_event`-Durchläufe bleiben still, sofern es sich nicht um explizite Befehle handelt, selbst wenn Laufzeitausgaben als hosteigen markiert sind.

Nur-Tool-Antworten erfordern ein Modell/eine Laufzeit, das bzw. die Werkzeuge zuverlässig aufruft, und werden für gemeinsam genutzte, umgebungsgesteuerte Räume mit Modellen der neuesten Generation wie GPT-5.6 Sol empfohlen. Einige schwächere Modelle können endgültigen Text ausgeben, verstehen jedoch nicht, dass für die Quelle sichtbare Ausgaben mit `message(action=send)` gesendet werden müssen. OpenClaw stellt den häufigen Fall einer nicht zugestellten endgültigen Antwort standardmäßig nur dann wieder her, wenn die endgültige Antwort inhaltlich relevant ist, der Quelldurchlauf kein Raumereignis war, die Senderichtlinie die Zustellung nicht verweigert hat und noch keine Antwort an die Quelle gesendet wurde. Die Wiederherstellung ist auf einen Versuch begrenzt; sie unterdrückt die Persistierung der synthetischen Wiederholungsaufforderung und schließt diesen Wiederholungsversuch von der Sammelverarbeitung aus, sodass er nicht mit unabhängigen Aufforderungen in der Warteschlange zusammengeführt werden kann. Wenn auch der Wiederholungsversuch nicht zugestellt oder nicht in die Warteschlange gestellt werden kann, liefert OpenClaw lediglich eine bereinigte Diagnose wie „Ich habe eine Antwort erstellt, konnte sie jedoch nicht an diesen Chat zustellen. Bitte versuchen Sie es erneut.“ Der ursprüngliche private endgültige Text wird niemals für eine automatische Zustellung an die Quelle markiert. Verwenden Sie bei Modellen, die Antworten wiederholt nicht zustellen, `"automatic"`, sodass der letzte Assistentendurchlauf als Pfad für sichtbare Antworten dient, wechseln Sie zu einem leistungsfähigeren Modell für Tool-Aufrufe, prüfen Sie das ausführliche Gateway-Protokoll auf die Zusammenfassung der unterdrückten Nutzlast oder legen Sie `messages.groupChat.visibleReplies: "automatic"` fest, um für jede Gruppen-/Kanalanfrage sichtbare endgültige Antworten zu verwenden.

Wenn das Nachrichten-Tool gemäß der aktiven Tool-Richtlinie nicht verfügbar ist, greift OpenClaw auf automatische sichtbare Antworten zurück, statt die Antwort stillschweigend zu unterdrücken. `openclaw doctor` warnt vor dieser Diskrepanz.

Diese Regel gilt für den normalen finalen Agententext. Plugin-eigene Konversationsbindungen verwenden bei beanspruchten Nachrichten in gebundenen Threads die vom zuständigen Plugin zurückgegebene Antwort als sichtbare Antwort; das Plugin muss für diese Bindungsantworten `message(action=send)` nicht aufrufen.

**Fehlerbehebung: Eine @Erwähnung in einer Gruppe löst eine Tippanzeige aus, danach bleibt es still (kein Fehler)**

Symptom: Eine @Erwähnung in einer Gruppe/einem Kanal zeigt die Tippanzeige an und das Gateway-Protokoll meldet `dispatch complete (queuedFinal=false, replies=0)`, aber im Raum kommt keine Nachricht an. Direktnachrichten an denselben Agenten werden normal beantwortet.

Ursache: Der Modus für sichtbare Antworten in Gruppen/Kanälen wird zu `"message_tool"` aufgelöst. OpenClaw führt die Verarbeitung daher aus, unterdrückt aber den finalen Assistententext, sofern der Agent nicht `message(action=send)` aufruft. In diesem Modus gibt es keinen `NO_REPLY`-Vertrag; ohne Aufruf des Nachrichten-Tools bleibt der ursprüngliche finale Text privat. Bei substanziellen Quellnachrichten versucht OpenClaw nun eine einzelne abgesicherte Wiederholung zur Wiederherstellung; kurze Notizen, explizites Schweigen, Raumereignisse, aufgrund der Senderichtlinie abgelehnte Verarbeitungen und bereits zugestellte Verarbeitungen werden nicht wiederholt. Normale Gruppen- und Kanalverarbeitungen verwenden standardmäßig `"automatic"`, sodass dieses Symptom nur auftritt, wenn `messages.groupChat.visibleReplies` (oder global `messages.visibleReplies`) explizit auf `"message_tool"` gesetzt ist. Harness-`defaultVisibleReplies` gilt hier nicht – der Resolver für Gruppen/Kanäle ignoriert es; es wirkt sich nur auf Direkt-/Quellchats aus (das Codex-Harness unterdrückt auf diese Weise finale Antworten in Direktchats).

Behebung: Wählen Sie entweder ein Modell, das Tools zuverlässiger aufruft, entfernen Sie die explizite `"message_tool"`-Überschreibung, um auf den Standardwert `"automatic"` zurückzufallen, oder setzen Sie `messages.groupChat.visibleReplies: "automatic"`, um sichtbare Antworten für jede Gruppen-/Kanalanfrage zu erzwingen. Eine substanzielle, nicht zugestellte finale Antwort sollte nicht mehr als stillschweigender Erfolg enden; sie sollte entweder durch eine einzelne `message(action=send)`-Wiederholung wiederhergestellt werden oder die bereinigte Diagnose zum Zustellungsfehler anzeigen. Das Gateway lädt die `messages`-Konfiguration nach dem Speichern der Datei dynamisch neu; starten Sie das Gateway nur dann neu, wenn die Dateiüberwachung oder das erneute Laden der Konfiguration in der Bereitstellung deaktiviert ist.

**Erwähnungstypen:**

- **Metadaten-Erwähnungen**: Native @Erwähnungen der Plattform. Werden im WhatsApp-Selbstchatmodus ignoriert.
- **Textmuster**: Sichere Regex-Muster in `agents.entries.*.groupChat.mentionPatterns`. Ungültige Muster und unsichere verschachtelte Wiederholungen werden ignoriert.
- Die Erwähnungsbeschränkung wird nur durchgesetzt, wenn eine Erkennung möglich ist (native Erwähnungen oder mindestens ein Muster).

```json5
{
  messages: {
    visibleReplies: "automatic", // alte automatische finale Antworten für Direkt-/Quellchats erzwingen
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // dauerhaft aktives, nicht erwähnendes Raumgeplauder wird zu stillem Kontext
      visibleReplies: "message_tool", // Opt-in; message(action=send) für sichtbare Raumantworten voraussetzen
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` legt den globalen Standardwert fest. Kanäle können ihn mit `channels.<channel>.historyLimit` (oder pro Konto) überschreiben. Setzen Sie `0`, um ihn zu deaktivieren.

`messages.groupChat.unmentionedInbound: "room_event"` übermittelt nicht erwähnende, dauerhaft aktive Gruppen-/Kanalnachrichten auf unterstützten Kanälen als stillen Raumkontext. Erwähnte Nachrichten, Befehle und Direktnachrichten bleiben Benutzeranfragen. Vollständige Beispiele für Discord, Slack und Telegram finden Sie unter [Umgebende Raumereignisse](/de/channels/ambient-room-events).

`messages.visibleReplies` ist der globale Standardwert für Quellereignisse; `messages.groupChat.visibleReplies` überschreibt ihn für Gruppen-/Kanalquellereignisse. Wenn `messages.visibleReplies` nicht gesetzt ist, verwenden Direkt-/Quellchats den ausgewählten Runtime- oder Harness-Standardwert, interne direkte WebChat-Verarbeitungen verwenden jedoch die automatische finale Zustellung, um die Prompt-Parität zwischen Pi und Codex sicherzustellen. Setzen Sie `messages.visibleReplies: "message_tool"`, um für sichtbare Ausgaben absichtlich `message(action=send)` vorauszusetzen. Kanal-Zulassungslisten und die Erwähnungsbeschränkung bestimmen weiterhin, ob ein Ereignis verarbeitet wird.

#### Verlaufslimits für Direktnachrichten

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Auflösungsreihenfolge: Überschreibung pro Direktnachricht → Provider-Standardwert → kein Limit (alles wird aufbewahrt).

Dieser Resolver liest `channels.<provider>.dmHistoryLimit` und `channels.<provider>.dms.<id>.historyLimit` für jeden Kanal, dessen Sitzungsschlüssel der standardmäßigen Form `provider:direct:<id>` (oder der veralteten Form `provider:dm:<id>`) entspricht. Daher funktioniert er gleichermaßen für gebündelte und Plugin-Kanäle und nicht nur für eine feste Liste.

#### Selbstchatmodus

Nehmen Sie Ihre eigene Nummer in `allowFrom` auf, um den Selbstchatmodus zu aktivieren (native @Erwähnungen werden ignoriert, es wird nur auf Textmuster geantwortet):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Befehle (Verarbeitung von Chatbefehlen)

```json5
{
  commands: {
    native: "auto", // native Befehle registrieren, sofern unterstützt
    nativeSkills: "auto", // native Skill-Befehle registrieren, sofern unterstützt
    text: true, // /commands in Chatnachrichten parsen
    bash: false, // ! zulassen (Alias: /bash)
    bashForegroundMs: 2000,
    config: false, // /config zulassen
    mcp: false, // /mcp zulassen
    plugins: false, // /plugins zulassen
    debug: false, // /debug zulassen
    restart: true, // /restart und externe SIGUSR1-Neustartanforderungen zulassen
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Befehlsdetails">

- Dieser Block konfiguriert Befehlsoberflächen. Den aktuellen integrierten und gebündelten Befehlskatalog finden Sie unter [Slash-Befehle](/de/tools/slash-commands).
- Diese Seite ist eine **Referenz der Konfigurationsschlüssel**, nicht der vollständige Befehlskatalog. Kanal-/Plugin-eigene Befehle wie QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, Gerätekopplung `/pair`, Speicher `/dreaming`, Telefonsteuerung `/phone` und Talk `/voice` sind auf den jeweiligen Kanal-/Plugin-Seiten sowie unter [Slash-Befehle](/de/tools/slash-commands) dokumentiert.
- Textbefehle müssen **eigenständige** Nachrichten mit vorangestelltem `/` sein.
- `native: "auto"` aktiviert native Befehle für Discord/Telegram und lässt sie für Slack deaktiviert.
- `nativeSkills: "auto"` aktiviert native Skill-Befehle für Discord/Telegram und lässt sie für Slack deaktiviert.
- Überschreibung pro Kanal: `channels.discord.commands.native` (boolescher Wert oder `"auto"`). Bei Discord überspringt `false` beim Start die Registrierung und Bereinigung nativer Befehle.
- Überschreiben Sie die Registrierung nativer Skills pro Kanal mit `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` fügt zusätzliche Einträge zum Bot-Menü von Telegram hinzu.
- `bash: true` aktiviert `! <cmd>` für die Host-Shell. Erfordert `tools.elevated.enabled` und einen Absender in `tools.elevated.allowFrom.<channel>`.
- `config: true` aktiviert `/config` (liest/schreibt `openclaw.json`). Für Gateway-`chat.send`-Clients erfordern persistente `/config set|unset`-Schreibvorgänge zusätzlich `operator.admin`; schreibgeschützte `/config show`-Vorgänge bleiben für normale Operator-Clients mit Schreibberechtigung verfügbar.
- `mcp: true` aktiviert `/mcp` für die von OpenClaw verwaltete MCP-Serverkonfiguration unter `mcp.servers`.
- `plugins: true` aktiviert `/plugins` für die Erkennung, Installation sowie Aktivierungs-/Deaktivierungssteuerung von Plugins.
- `channels.<provider>.configWrites` beschränkt Konfigurationsänderungen pro Kanal (Standardwert: true).
- Bei Kanälen mit mehreren Konten beschränkt `channels.<provider>.accounts.<id>.configWrites` außerdem Schreibvorgänge, die auf dieses Konto abzielen (beispielsweise `/allowlist --config --account <id>` oder `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` deaktiviert `/restart` und externe `SIGUSR1`-Neustartanforderungen. Standardwert: `true`.
- `ownerAllowFrom` ist die explizite Eigentümer-Zulassungsliste für ausschließlich Eigentümern vorbehaltene Befehle und eigentümerbeschränkte Kanalaktionen. Sie ist von `allowFrom` getrennt.
- `ownerDisplay: "hash"` hasht Eigentümer-IDs im System-Prompt. Setzen Sie `ownerDisplaySecret`, um das Hashing zu steuern.
- `allowFrom` gilt pro Provider. Wenn es gesetzt ist, bildet es die **einzige** Autorisierungsquelle (Kanal-Zulassungslisten/Kopplung und `useAccessGroups` werden ignoriert).
- `useAccessGroups: false` erlaubt Befehlen, Richtlinien für Zugriffsgruppen zu umgehen, wenn `allowFrom` nicht gesetzt ist.
- Übersicht der Befehlsdokumentation:
  - integrierter und gebündelter Katalog: [Slash-Befehle](/de/tools/slash-commands)
  - kanalspezifische Befehlsoberflächen: [Kanäle](/de/channels)
  - QQ-Bot-Befehle: [QQ Bot](/de/channels/qqbot)
  - Kopplungsbefehle: [Kopplung](/de/channels/pairing)
  - LINE-Kartenbefehl: [LINE](/de/channels/line)
  - Speicher-Dreaming: [Dreaming](/de/concepts/dreaming)

</Accordion>

---

## Verwandte Themen

- [Konfigurationsreferenz](/de/gateway/configuration-reference) — Schlüssel der obersten Ebene
- [Konfiguration — Agenten](/de/gateway/config-agents)
- [Kanalübersicht](/de/channels)
