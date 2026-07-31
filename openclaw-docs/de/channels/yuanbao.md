---
read_when:
    - Sie möchten einen Yuanbao-Bot verbinden
    - Sie konfigurieren den Yuanbao-Kanal
summary: Überblick, Funktionen und Konfiguration des Yuanbao-Bots
title: Yuanbao
x-i18n:
    generated_at: "2026-07-26T18:15:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 43488834f588530206b290cb0fb185fd1fe2e1f214ab4a4ccccc49b9b549b6ac
    source_path: channels/yuanbao.md
    workflow: 16
---

Tencent Yuanbao ist die KI-Assistentenplattform von Tencent. Das von der Community gepflegte Plugin `openclaw-plugin-yuanbao` verbindet Yuanbao-Bots über WebSocket mit OpenClaw und ermöglicht Direktnachrichten und Gruppenchats.

**Status:** produktionsreif für Bot-Direktnachrichten und Gruppenchats. WebSocket ist der einzige unterstützte Verbindungsmodus. Dieses Plugin wird vom Tencent-Yuanbao-Team als externer Katalogeintrag gepflegt, nicht vom OpenClaw-Kernteam; die nachfolgenden Konfigurations- und Verhaltensdetails (abgesehen von der Installation und der generischen CLI-Oberfläche) stammen aus der eigenen Dokumentation des Plugins und wurden nicht anhand des OpenClaw-Kernquellcodes überprüft.

## Schnellstart

Erfordert OpenClaw 2026.4.10 oder höher. Prüfen Sie dies mit `openclaw --version`; führen Sie das Upgrade mit `openclaw update` durch.

<Steps>
  <Step title="Yuanbao-Kanal mit Ihren Zugangsdaten hinzufügen">
  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  `--token` verwendet durch Doppelpunkte getrennte `appKey:appSecret`. Sie erhalten diese aus der Yuanbao-App, indem Sie in Ihren Anwendungseinstellungen einen Bot erstellen.
  </Step>

  <Step title="Gateway neu starten, um die Änderung anzuwenden">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

### Interaktive Einrichtung (Alternative)

```bash
openclaw channels login --channel yuanbao
```

Folgen Sie den Eingabeaufforderungen, um Ihre App ID und Ihr App Secret einzugeben.

## Zugriffskontrolle

### Direktnachrichten

`channels.yuanbao.dm.policy`:

| Wert             | Verhalten                                                        |
| ---------------- | ---------------------------------------------------------------- |
| `open` (Standard) | Alle Benutzer zulassen                                           |
| `pairing`        | Unbekannte Benutzer erhalten einen Kopplungscode; über CLI genehmigen |
| `allowlist`      | Nur Benutzer in `allowFrom` können chatten                       |
| `disabled`       | Alle Direktnachrichten deaktivieren                              |

So genehmigen Sie eine Kopplungsanfrage:

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### Gruppenchats

`channels.yuanbao.requireMention` (Standard `true`): Erfordert eine @Erwähnung, bevor der Bot in einer Gruppe antwortet. Eine Antwort auf die eigene Nachricht des Bots wird als implizite Erwähnung behandelt.

## Konfigurationsbeispiele

Grundlegende Einrichtung mit offener Richtlinie für Direktnachrichten:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

Direktnachrichten auf bestimmte Benutzer beschränken:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

Anforderung einer @Erwähnung in Gruppen deaktivieren:

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

Ausgehende Zustellung abstimmen:

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // puffern, bis diese Zeichenanzahl erreicht ist
      maxChars: 3000, // oberhalb dieses Grenzwerts Aufteilung erzwingen
      idleMs: 5000, // nach Inaktivitätszeitüberschreitung automatisch senden (ms)
    },
  },
}
```

Setzen Sie `outboundQueueStrategy: "immediate"`, um jeden Abschnitt ohne Pufferung zu senden.

## Häufig verwendete Befehle

| Befehl     | Beschreibung                 |
| ---------- | ---------------------------- |
| `/help`    | Verfügbare Befehle anzeigen  |
| `/status`  | Bot-Status anzeigen          |
| `/new`     | Neue Sitzung starten         |
| `/stop`    | Aktuellen Lauf stoppen       |
| `/restart` | OpenClaw neu starten         |
| `/compact` | Sitzungskontext komprimieren |

Yuanbao unterstützt native Schrägstrichbefehlsmenüs; die Befehle werden beim Start des Gateways automatisch mit der Plattform synchronisiert.

## Fehlerbehebung

**Bot antwortet nicht in Gruppenchats:**

1. Bestätigen Sie, dass der Bot der Gruppe hinzugefügt wurde
2. Bestätigen Sie, dass Sie den Bot mit @ erwähnen (standardmäßig erforderlich)
3. Prüfen Sie die Protokolle: `openclaw logs --follow`

**Bot empfängt keine Nachrichten:**

1. Bestätigen Sie, dass der Bot in der Yuanbao-App erstellt und genehmigt wurde
2. Bestätigen Sie, dass `appKey` und `appSecret` korrekt konfiguriert sind
3. Bestätigen Sie, dass das Gateway ausgeführt wird: `openclaw gateway status`
4. Prüfen Sie die Protokolle: `openclaw logs --follow`

**Bot sendet leere Antworten oder Ausweichantworten:**

1. Prüfen Sie, ob das KI-Modell gültige Inhalte zurückgibt
2. Standardmäßige Ausweichantwort: „暂时无法解答，你可以换个问题问问我哦“
3. Passen Sie sie mit `channels.yuanbao.fallbackReply` an

**App Secret offengelegt:**

1. Setzen Sie das App Secret in der Yuanbao-App zurück
2. Aktualisieren Sie den Wert in Ihrer Konfiguration
3. Starten Sie das Gateway neu: `openclaw gateway restart`

## Erweiterte Konfiguration

### Mehrere Konten

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "Primary bot",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` legt fest, welches Konto verwendet wird, wenn ausgehende APIs keine `accountId` angeben.

### Nachrichtenlimits

- `maxChars`: maximale Zeichenanzahl einer einzelnen Nachricht (Standard `3000`)
- `mediaMaxMb`: Limit für das Hoch-/Herunterladen von Medien (Standard `20` MB)
- `overflowPolicy`: Verhalten, wenn eine Nachricht das Limit überschreitet, `"split"` (Standard) oder `"stop"`

### Streaming

Yuanbao unterstützt Streaming-Ausgaben auf Blockebene; der Bot sendet den Text während der Generierung abschnittsweise.

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // Block-Streaming aktiviert (Standard)
    },
  },
}
```

Setzen Sie `disableBlockStreaming: true`, um die vollständige Antwort in einer einzigen Nachricht zu senden.

### Verlaufskontext für Gruppenchats

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // Standard: 100, zum Deaktivieren auf 0 setzen
    },
  },
}
```

Legt fest, wie viele frühere Nachrichten bei Gruppenchats in den KI-Kontext aufgenommen werden.

### Antwortbezugsmodus

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all" (Standard: "first")
    },
  },
}
```

| Wert    | Verhalten                                                      |
| ------- | -------------------------------------------------------------- |
| `off`   | Keine zitierte Antwort                                         |
| `first` | Nur die erste Antwort pro eingehender Nachricht zitieren (Standard) |
| `all`   | Jede Antwort zitieren                                          |

### Einfügen eines Markdown-Hinweises

Standardmäßig fügt der Bot dem System-Prompt eine Anweisung hinzu, die verhindert, dass das Modell die gesamte Antwort in einen Markdown-Codeblock einschließt.

```json5
{
  channels: {
    yuanbao: {
      markdownHintEnabled: true, // Standard: true
    },
  },
}
```

### Debug-Modus

```json5
{
  channels: {
    yuanbao: {
      debugBotIds: ["bot_user_id_1", "bot_user_id_2"],
    },
  },
}
```

Aktiviert unbereinigte Protokollausgaben für die aufgeführten Bot-IDs.

### Multi-Agent-Routing

Verwenden Sie `bindings`, um Yuanbao-Direktnachrichten oder -Gruppen an verschiedene Agenten weiterzuleiten:

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "yuanbao",
        peer: { kind: "direct", id: "user_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "yuanbao",
        peer: { kind: "group", id: "group_zzz" },
      },
    },
  ],
}
```

- `match.channel`: `"yuanbao"`
- `match.peer.kind`: `"direct"` (Direktnachricht) oder `"group"` (Gruppenchat)
- `match.peer.id`: Benutzer-ID oder Gruppencode

## Konfigurationsreferenz

Vollständige Konfiguration: [Gateway-Konfiguration](/de/gateway/configuration)

| Einstellung                                | Beschreibung                                              | Standard                               |
| ------------------------------------------ | --------------------------------------------------------- | -------------------------------------- |
| `channels.yuanbao.enabled`                 | Kanal aktivieren/deaktivieren                             | `true`                                 |
| `channels.yuanbao.defaultAccount`          | Standardkonto für ausgehendes Routing                     | `default`                              |
| `channels.yuanbao.accounts.<id>.appKey`    | App Key (Signierung + Ticket-Erzeugung)                    | -                                      |
| `channels.yuanbao.accounts.<id>.appSecret` | App Secret (Signierung)                                    | -                                      |
| `channels.yuanbao.accounts.<id>.token`     | Vorsigniertes Token (überspringt automatische Ticket-Signierung) | -                                      |
| `channels.yuanbao.accounts.<id>.name`      | Anzeigename des Kontos                                    | -                                      |
| `channels.yuanbao.accounts.<id>.enabled`   | Bestimmtes Konto aktivieren/deaktivieren                  | `true`                                 |
| `channels.yuanbao.dm.policy`               | Richtlinie für Direktnachrichten                          | `open`                                 |
| `channels.yuanbao.dm.allowFrom`            | Zulassungsliste für Direktnachrichten (Liste von Benutzer-IDs) | -                                      |
| `channels.yuanbao.requireMention`          | @Erwähnung in Gruppen erforderlich                        | `true`                                 |
| `channels.yuanbao.overflowPolicy`          | Verarbeitung langer Nachrichten (`split` oder `stop`) | `split`                                |
| `channels.yuanbao.replyToMode`             | Strategie für Antwortbezüge in Gruppen (`off`, `first`, `all`) | `first`                                |
| `channels.yuanbao.outboundQueueStrategy`   | Ausgehende Strategie (`merge-text` oder `immediate`) | `merge-text`                           |
| `channels.yuanbao.minChars`                | Textzusammenführung: Mindestzeichenanzahl zum Auslösen des Sendens | `2800`                                 |
| `channels.yuanbao.maxChars`                | Textzusammenführung: maximale Zeichenanzahl pro Nachricht | `3000`                                 |
| `channels.yuanbao.idleMs`                  | Textzusammenführung: Inaktivitätszeitüberschreitung vor automatischem Senden (ms) | `5000`                                 |
| `channels.yuanbao.mediaMaxMb`              | Größenlimit für Medien (MB)                               | `20`                                   |
| `channels.yuanbao.historyLimit`            | Einträge des Verlaufskontexts für Gruppenchats            | `100`                                  |
| `channels.yuanbao.disableBlockStreaming`   | Streaming-Ausgabe auf Blockebene deaktivieren             | `false`                                |
| `channels.yuanbao.fallbackReply`           | Ausweichantwort, wenn das Modell keine Inhalte zurückgibt | `暂时无法解答，你可以换个问题问问我哦` |
| `channels.yuanbao.markdownHintEnabled`     | Markdown-Anweisungen gegen vollständige Codeblock-Einschließung einfügen | `true`                                 |
| `channels.yuanbao.debugBotIds`             | Bot-IDs der Debug-Zulassungsliste (unbereinigte Protokolle) | `[]`                                   |

## Unterstützte Nachrichtentypen

**Empfangen:** Text, Bilder, Dateien, Audio/Sprachnachrichten, Video, Sticker/benutzerdefinierte Emojis, benutzerdefinierte Elemente (Linkkarten).

**Senden:** Text (Markdown), Bilder, Dateien, Audio, Video, Sticker.

**Threads und Antworten:** zitierte Antworten (über `replyToMode` konfigurierbar); Thread-Antworten werden von der Plattform nicht unterstützt.

## Verwandte Themen

- [Kanalübersicht](/de/channels) - alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) - DM-Authentifizierung und Kopplungsablauf
- [Gruppen](/de/channels/groups) - Verhalten von Gruppenchats und Erwähnungsfilterung
- [Kanal-Routing](/de/channels/channel-routing) - Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) - Zugriffsmodell und Absicherung
