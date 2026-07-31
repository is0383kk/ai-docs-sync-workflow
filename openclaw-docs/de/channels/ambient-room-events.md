---
read_when:
    - Konfigurieren von dauerhaft aktiven Gruppen- oder Kanalräumen
    - Sie möchten, dass der Agent den Chat im Raum verfolgt, ohne automatisch endgültigen Text zu posten
    - Fehlersuche bei Tippstatus und Token-Nutzung ohne sichtbare Nachricht im Raum
sidebarTitle: Ambient room events
summary: Unterstützte Gruppenräume sollen unaufdringlich Kontext bereitstellen, sofern der Agent nicht mit dem Nachrichten-Tool sendet
title: Raumereignisse in der Umgebung
x-i18n:
    generated_at: "2026-07-26T18:18:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 15c083c139058c9bd2c651794965bd8252d74691e536db2ad2a2ae0b4ac886e8
    source_path: channels/ambient-room-events.md
    workflow: 16
---

Umgebungsraumereignisse ermöglichen es OpenClaw, nicht erwähnte Unterhaltungen in Gruppen oder Kanälen als stillen Kontext zu verarbeiten. Der Agent kann den Speicher und den Sitzungsstatus aktualisieren, aber der Raum bleibt still, sofern der Agent nicht ausdrücklich das Tool `message` aufruft.

Kombinieren Sie für dauerhaft aktive Gruppenchats `messages.groupChat.unmentionedInbound: "room_event"` mit `messages.groupChat.visibleReplies: "message_tool"`. Der Agent hört zu, entscheidet, wann eine Antwort sinnvoll ist, und benötigt nicht mehr das alte Prompt-Muster, mit `NO_REPLY` zu antworten.

Derzeit unterstützt: Discord-Guild-Kanäle, Slack-Kanäle und private Kanäle, Slack-Direktnachrichten mit mehreren Personen sowie Telegram-Gruppen oder -Supergruppen. Andere Gruppenkanäle behalten ihr bestehendes Gruppenverhalten bei, sofern auf ihrer Kanalseite nicht angegeben ist, dass sie Umgebungsraumereignisse unterstützen.

## Empfohlene Einrichtung

Legen Sie das globale Verhalten für Gruppenchats fest:

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
}
```

Machen Sie den Raum anschließend dauerhaft aktiv, indem Sie die Erwähnungsbeschränkung für diesen Raum deaktivieren. Der Raum muss weiterhin seine normalen `groupPolicy`, die Raum-Zulassungsliste und die Absender-Zulassungsliste erfüllen.

Nach dem Speichern der Konfiguration übernimmt der Gateway die Einstellungen für `messages` dynamisch. Starten Sie nur neu, wenn die Dateiüberwachung oder das erneute Laden der Konfiguration deaktiviert ist (`gateway.reload.mode: "off"`).

## Was sich ändert

Mit `messages.groupChat.unmentionedInbound: "room_event"`:

- werden nicht erwähnte, zugelassene Gruppen- oder Kanalnachrichten zu stillen Raumereignissen
- bleiben Nachrichten mit Erwähnungen Benutzeranfragen
- bleiben textbasierte Steuerbefehle und native Befehle Benutzeranfragen
- bleiben Abbruch- oder Stoppanfragen Benutzeranfragen
- bleiben Direktnachrichten Benutzeranfragen

Raumereignisse verwenden eine strikte sichtbare Zustellung. Der endgültige Assistententext bleibt privat. Der Agent muss `message(action=send)` aufrufen, um im Raum zu posten.

Reaktionen für Tipp- und Lebenszyklusstatus bleiben bei Raumereignissen unterdrückt. Die einzige ausdrückliche Empfangsausnahme ist `messages.ackReactionScope: "all"`, wodurch die konfigurierte Bestätigungsreaktion gesendet wird; verwenden Sie einen engeren Geltungsbereich oder `"off"`, wenn der Raum vollständig still bleiben muss.

## Discord-Beispiel

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          requireMention: false,
          users: ["<YOUR_DISCORD_USER_ID>"],
        },
      },
    },
  },
}
```

Verwenden Sie die Discord-Konfiguration pro Kanal, wenn nur ein Kanal als Umgebungsraum dienen soll. Unter `groupPolicy: "allowlist"` wird der Kanal durch seine Auflistung zugelassen (`enabled: false` deaktiviert einen Eintrag):

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "<DISCORD_SERVER_ID>": {
          channels: {
            "<DISCORD_CHANNEL_ID_OR_NAME>": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

## Slack-Beispiel

Slack-Kanal-Zulassungslisten verwenden vorrangig IDs. Verwenden Sie Kanal-IDs wie `C12345678`, nicht `#channel-name`. Durch das Auflisten des Kanals unter `channels.slack.channels` wird er zugelassen (`enabled: false` deaktiviert einen Eintrag):

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    slack: {
      groupPolicy: "allowlist",
      channels: {
        "<SLACK_CHANNEL_ID>": {
          requireMention: false,
        },
      },
    },
  },
}
```

## Telegram-Beispiel

Bei Telegram-Gruppen muss der Bot normale Gruppennachrichten sehen können. Wenn `requireMention: false`, deaktivieren Sie den Datenschutzmodus von BotFather oder verwenden Sie eine andere Telegram-Einrichtung, die den vollständigen Gruppenverkehr an den Bot übermittelt.

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
      visibleReplies: "message_tool",
      historyLimit: 50,
    },
  },
  channels: {
    telegram: {
      groups: {
        "<TELEGRAM_GROUP_CHAT_ID>": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

Telegram-Gruppen-IDs sind normalerweise negative Zahlen wie `-1001234567890`. Lesen Sie `chat.id` aus `openclaw logs --follow`, leiten Sie eine Gruppennachricht an einen ID-Hilfsbot weiter oder prüfen Sie `getUpdates` der Bot API.

## Agentenspezifische Richtlinie

Verwenden Sie eine Agentenüberschreibung, wenn mehrere Agenten denselben Raum nutzen, aber nur einer nicht erwähnte Unterhaltungen als Umgebungskontext behandeln soll:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          unmentionedInbound: "room_event",
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
}
```

Der agentenspezifische Wert `agents.entries.*.groupChat.unmentionedInbound` überschreibt `messages.groupChat.unmentionedInbound` für diesen Agenten.

## Modi für sichtbare Antworten

`messages.groupChat.visibleReplies` verwendet standardmäßig `"automatic"` für normale Benutzeranfragen in Gruppen oder Kanälen. Behalten Sie diese Standardeinstellung bei, wenn der endgültige Assistententext ohne ausdrücklichen Aufruf des Nachrichten-Tools sichtbar gepostet werden soll.

Für dauerhaft aktive Umgebungsräume wird `messages.groupChat.visibleReplies: "message_tool"` weiterhin empfohlen, insbesondere mit Modellen der neuesten Generation, die Tools zuverlässig verwenden, etwa GPT-5.6 Sol. Dadurch kann der Agent durch Aufrufen des Nachrichten-Tools entscheiden, wann er sich äußert. Wenn das Modell endgültigen Text zurückgibt, ohne das Tool aufzurufen, hält OpenClaw diesen endgültigen Text privat und protokolliert Metadaten zur unterdrückten Zustellung.

Raumereignisse bleiben auch dann strikt, wenn andere Gruppenanfragen automatische Antworten verwenden. Nicht erwähnte Umgebungsraumereignisse erfordern für eine sichtbare Ausgabe immer `message(action=send)`.

## Verlauf

`messages.groupChat.historyLimit` legt den globalen Standardwert für den Gruppenverlauf fest (50, wenn nicht festgelegt; muss eine positive Ganzzahl sein). Kanäle können ihn mit `channels.<channel>.historyLimit` überschreiben, und einige Kanäle unterstützen außerdem Verlaufslimits pro Konto. Setzen Sie `historyLimit: 0` auf Kanalebene, um den Kontext des Gruppenverlaufs für diesen Kanal zu deaktivieren.

Unterstützte Kanäle für Raumereignisse behalten aktuelle Nachrichten des Umgebungsraums als Kontext bei. Telegram verwaltet ein dauerhaft aktives, rollierendes Fenster pro Gruppe, das durch `historyLimit` begrenzt wird; Durchläufe für Benutzeranfragen wählen Einträge nach der letzten aufgezeichneten Antwort des Bots aus, während Durchläufe für Raumereignisse das vollständige aktuelle Fenster erhalten, damit das Modell seine eigenen aktuellen Beiträge sehen kann. Der eingestellte Telegram-Modusschlüssel `includeGroupHistoryContext` wird durch `openclaw doctor --fix` entfernt.

## Fehlerbehebung

Wenn der Raum eine Tippanzeige oder Token-Nutzung, aber keine sichtbare Nachricht anzeigt:

1. Bestätigen Sie, dass der Raum durch die Kanal-Zulassungsliste und die Absender-Zulassungsliste zugelassen ist.
2. Bestätigen Sie, dass `requireMention: false` auf der erwarteten Raumebene festgelegt ist.
3. Prüfen Sie, ob `messages.groupChat.unmentionedInbound` oder die Agentenüberschreibung auf `"room_event"` gesetzt ist.
4. Prüfen Sie die Protokolle auf Metadaten zur unterdrückten endgültigen Nutzlast oder `didSendViaMessagingTool: false`.
5. Behalten Sie bei normalen Gruppenanfragen `messages.groupChat.visibleReplies: "automatic"` bei oder stellen Sie es wieder her, wenn endgültige Antworten automatisch gepostet werden sollen. Verwenden Sie für Umgebungsräume mit `message_tool` ein Modell oder eine Laufzeit, das beziehungsweise die Tools zuverlässig aufruft.

Wenn Telegram-Umgebungsräume überhaupt nicht ausgelöst werden, prüfen Sie den Datenschutzmodus von BotFather und stellen Sie sicher, dass der Gateway normale Gruppennachrichten empfängt.

Wenn Slack-Umgebungsräume nicht ausgelöst werden, überprüfen Sie, ob der Kanalschlüssel die Slack-Kanal-ID ist und die App über den Verlaufsberechtigungsbereich für diesen Raumtyp verfügt: `channels:history` (öffentlich), `groups:history` (privat) oder `mpim:history` (Direktnachrichten mit mehreren Personen).

## Verwandte Themen

- [Gruppen](/de/channels/groups)
- [Discord](/de/channels/discord)
- [Slack](/de/channels/slack)
- [Telegram](/de/channels/telegram)
- [Fehlerbehebung für Kanäle](/de/channels/troubleshooting)
- [Referenz zur Kanalkonfiguration](/de/gateway/config-channels)
