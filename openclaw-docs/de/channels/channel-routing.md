---
read_when:
    - Ändern des Kanal-Routings oder des Posteingangsverhaltens
summary: Routingregeln pro Kanal (WhatsApp, Telegram, Discord, Slack) und gemeinsamer Kontext
title: Kanalrouting
x-i18n:
    generated_at: "2026-07-26T17:38:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# Kanäle und Routing

OpenClaw leitet Antworten **an den Kanal zurück, aus dem eine Nachricht stammt**. Das
Modell wählt keinen Kanal aus; das Routing ist deterministisch und wird durch die
Host-Konfiguration gesteuert. Beim standardmäßigen DM-Geltungsbereich laufen Direktnachrichten aus allen
Kanälen in der [Hauptsitzung](/de/concepts/main-session) des Agenten zusammen.

## Schlüsselbegriffe

- **Kanal**: ein mitgeliefertes Kanal-Plugin wie `discord`, `googlechat`, `imessage`, `irc`, `line`, `signal`, `slack`, `telegram` oder `whatsapp` sowie installierte Plugin-Kanäle. `webchat` ist der interne WebChat-UI-Kanal und kein konfigurierbarer ausgehender Kanal.
- **AccountId**: Konto-Instanz pro Kanal (sofern unterstützt).
- Optionales Standardkonto des Kanals: `channels.<channel>.defaultAccount` bestimmt,
  welches Konto verwendet wird, wenn ein ausgehender Pfad keine `accountId` angibt.
  - Legen Sie in Konfigurationen mit mehreren Konten einen expliziten Standard fest (`defaultAccount` oder ein Konto namens `default`), wenn zwei oder mehr Konten konfiguriert sind. Andernfalls kann das Fallback-Routing die erste normalisierte Konto-ID auswählen.
- **AgentId**: ein isolierter Arbeitsbereich mit Sitzungsspeicher („Gehirn“).
- **SessionKey**: der Bereichsschlüssel, mit dem Kontext gespeichert und Nebenläufigkeit gesteuert wird.

## Präfixe für ausgehende Ziele

Explizite ausgehende Ziele können ein Provider-Präfix enthalten, etwa `telegram:123` oder `tg:123`. Der Core behandelt dieses Präfix nur dann als Hinweis zur Kanalauswahl, wenn der ausgewählte Kanal `last` oder anderweitig nicht aufgelöst ist, und nur, wenn das geladene Plugin dieses Präfix ausweist. Hat der Aufrufer bereits einen expliziten Kanal ausgewählt, muss das Provider-Präfix mit diesem Kanal übereinstimmen; kanalübergreifende Kombinationen wie eine WhatsApp-Zustellung an `telegram:123` schlagen vor der Plugin-spezifischen Zielnormalisierung fehl.

Präfixe für Zielart und Dienst wie `channel:<id>`, `user:<id>`, `room:<id>`, `thread:<id>`, `imessage:<handle>` und `sms:<number>` bleiben Bestandteil der Grammatik des ausgewählten Kanals. Sie wählen den Provider nicht selbstständig aus.

## Formen von Sitzungsschlüsseln (Beispiele)

Direktnachrichten werden standardmäßig in der **Hauptsitzung** des Agenten zusammengeführt:

- `agent:<agentId>:<mainKey>` (Standard: `agent:main:main`)

`session.dmScope` steuert die Zusammenführung von DMs: `main` (Standard) verwendet eine gemeinsame Hauptsitzung,
während `per-peer`, `per-channel-peer` und `per-account-channel-peer`
DMs in separaten Sitzungen halten. Eine Routing-Bindung kann den Geltungsbereich für die von ihr
erfassten Kommunikationspartner über `bindings[].session.dmScope` überschreiben.

Selbst wenn der Gesprächsverlauf von Direktnachrichten mit der Hauptsitzung geteilt wird, verwenden Sandbox- und
Tool-Richtlinien für externe DMs einen abgeleiteten, kontospezifischen Laufzeitschlüssel für Direktchats,
damit von Kanälen stammende Nachrichten nicht wie lokale Ausführungen der Hauptsitzung behandelt werden.

Gruppen und Kanäle bleiben pro Kanal isoliert:

- Gruppen: `agent:<agentId>:<channel>:group:<id>`
- Kanäle/Räume: `agent:<agentId>:<channel>:channel:<id>`

Threads:

- Slack-/Discord-Threads hängen `:thread:<threadId>` an den Basisschlüssel an.
- Telegram-Forenthemen betten `:topic:<topicId>` in den Gruppenschlüssel ein.

Beispiele:

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## Bindung der Haupt-DM-Route

Wenn `session.dmScope` den Wert `main` hat, können Direktnachrichten eine gemeinsame Hauptsitzung verwenden.
Um zu verhindern, dass die `lastRoute` der Sitzung durch DMs von Nicht-Eigentümern überschrieben wird,
leitet OpenClaw aus `allowFrom` einen fest gebundenen Eigentümer ab, wenn alle folgenden Bedingungen erfüllt sind:

- `allowFrom` enthält genau einen Eintrag ohne Platzhalter.
- Der Eintrag kann für diesen Kanal zu einer konkreten Absender-ID normalisiert werden.
- Der Absender der eingehenden DM stimmt nicht mit diesem fest gebundenen Eigentümer überein.

Bei einer solchen Abweichung zeichnet OpenClaw weiterhin die Metadaten der eingehenden Sitzung auf,
überspringt jedoch die Aktualisierung der `lastRoute` der Hauptsitzung.

## Geschützte Aufzeichnung eingehender Nachrichten

Kanal-Plugins können einen eingehenden Sitzungseintrag als `createIfMissing: false`
kennzeichnen, wenn ein geschützter Pfad keine neue OpenClaw-Sitzung erstellen darf. In diesem Modus
kann OpenClaw Metadaten und `lastRoute` für eine vorhandene Sitzung aktualisieren, erstellt jedoch
nicht allein deshalb einen ausschließlich für das Routing bestimmten Sitzungseintrag, weil eine Nachricht beobachtet wurde.

## Routing-Regeln (wie ein Agent ausgewählt wird)

Das Routing wählt für jede eingehende Nachricht **einen Agenten** aus:

1. **Exakte Übereinstimmung des Kommunikationspartners** (`bindings` mit `peer.kind` + `peer.id`).
2. **Übereinstimmung des übergeordneten Kommunikationspartners** (Thread-Vererbung).
3. **Platzhalterübereinstimmung des Kommunikationspartners** (`peer.id: "*"` für eine Kommunikationspartnerart).
4. **Übereinstimmung von Guild und Rollen** (Discord) über `guildId` + `roles`.
5. **Guild-Übereinstimmung** (Discord) über `guildId`.
6. **Team-Übereinstimmung** (Slack) über `teamId`.
7. **Kontoübereinstimmung** (`accountId` im Kanal).
8. **Kanalübereinstimmung** (beliebiges Konto in diesem Kanal, `accountId: "*"`).
9. **Standard-Agent** (`agents.entries.*.default`, andernfalls erster Listeneintrag, mit Fallback auf `main`).

Wenn eine Bindung mehrere Übereinstimmungsfelder enthält (`peer`, `guildId`, `teamId`, `roles`), müssen **alle angegebenen Felder übereinstimmen**, damit diese Bindung angewendet wird.

Der übereinstimmende Agent bestimmt, welcher Arbeitsbereich und Sitzungsspeicher verwendet werden.

## Broadcast-Gruppen (mehrere Agenten ausführen)

Mit Broadcast-Gruppen können Sie **mehrere Agenten** für denselben Kommunikationspartner ausführen, **wenn OpenClaw normalerweise antworten würde** (zum Beispiel in WhatsApp-Gruppen nach der Erwähnungs-/Aktivierungsprüfung).

Konfiguration:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

Siehe: [Broadcast-Gruppen](/de/channels/broadcast-groups).

## Konfigurationsübersicht

- `agents.entries`: benannte Agentendefinitionen (Arbeitsbereich, Modell usw.).
- `bindings`: ordnet eingehende Kanäle/Konten/Kommunikationspartner Agenten zu.

Beispiel:

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## Sitzungsspeicherung

Laufzeit-Sitzungszeilen befinden sich in der SQLite-Datenbank jedes Agenten unter dem
Statusverzeichnis (Standard: `~/.openclaw`):

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

Ältere Installationen können veraltete JSONL-Transkriptdateien und einen `sessions.json`-Zeilenspeicher
unter `~/.openclaw/agents/<agentId>/sessions/` enthalten. Beim Start des Gateway und durch
`openclaw doctor --fix` werden aktive veraltete Zeilen/Verläufe automatisch in SQLite importiert.
Verwenden Sie `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` und die
[Doctor](/de/cli/doctor#session-sqlite-migration)-Validierungssequenz, wenn Sie
explizite Migrationsnachweise benötigen.
Über `session.store` und die `{agentId}`-Vorlagenbildung können Sie weiterhin
einen veralteten Speicherpfad für Migrations- und Offline-Wartungsabläufe auswählen.

Die Sitzungsfindung von Gateway und ACP durchsucht außerdem datenträgergestützte Agentenspeicher unter dem
standardmäßigen `agents/`-Stammverzeichnis und unter aus Vorlagen erzeugten `session.store`-Stammverzeichnissen. Gefundene
Speicher müssen innerhalb dieses aufgelösten Agentenstammverzeichnisses verbleiben und eine reguläre veraltete
`sessions.json`-Datei verwenden. Symlinks und Pfade außerhalb des Stammverzeichnisses werden ignoriert.

## WebChat-Verhalten

WebChat wird mit dem **ausgewählten Agenten** verbunden und verwendet standardmäßig die Hauptsitzung
des Agenten. Dadurch können Sie in WebChat den kanalübergreifenden Kontext dieses
Agenten an einer zentralen Stelle einsehen.

## Antwortkontext

Eingehende Antworten enthalten:

- `ReplyToId`, `ReplyToBody` und `ReplyToSender`, sofern verfügbar.
- Der zitierte Kontext wird als `[Replying to ...]`-Block an `Body` angehängt.

Dies ist über alle Kanäle hinweg einheitlich.

## Verwandte Themen

- [Gruppen](/de/channels/groups)
- [Broadcast-Gruppen](/de/channels/broadcast-groups)
- [Kopplung](/de/channels/pairing)
