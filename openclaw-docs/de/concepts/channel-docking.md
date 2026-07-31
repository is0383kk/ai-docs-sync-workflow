---
read_when:
    - Sie möchten Antworten für eine aktive Sitzung von Telegram zu Discord, Slack, Mattermost oder einem anderen verknüpften Kanal verschieben
    - Sie konfigurieren `session.identityLinks` für kanalübergreifende Direktnachrichten
    - Ein /dock-Befehl meldet, dass der Absender nicht verknüpft ist oder keine aktive Sitzung vorhanden ist.
summary: Antwort-Route einer OpenClaw-Sitzung zwischen verknüpften Chat-Kanälen verschieben
title: Kanal-Andocken
x-i18n:
    generated_at: "2026-07-26T17:44:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d7af3a59b95b2c73cb74a9529584e51caed055719db2df8aad2ba8e8c9b0593
    source_path: concepts/channel-docking.md
    workflow: 16
---

Channel-Docking ist eine Anrufweiterleitung für eine OpenClaw-Sitzung. Dabei bleibt derselbe
Konversationskontext erhalten, aber der Zustellort künftiger Antworten für diese Sitzung
ändert sich. Docking funktioniert nur aus einem direkten Chat heraus; in einem Gruppenchat
wird es nicht ausgeführt.

## Beispiel

Alice kann OpenClaw über Telegram und Discord Nachrichten senden:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456"],
    },
  },
}
```

Wenn Alice Folgendes aus einem direkten Telegram-Chat sendet:

```text
/dock_discord
```

behält OpenClaw den aktuellen Sitzungskontext bei und ändert die Antwortroute:

| Vor dem Docking               | Nach `/dock_discord`       |
| ---------------------------- | --------------------------- |
| Antworten werden an Telegram `123` gesendet | Antworten werden an Discord `456` gesendet |

Die Sitzung wird nicht neu erstellt. Der Transkriptverlauf bleibt mit derselben
Sitzung verknüpft.

## Verwendungszweck

Verwenden Sie Docking, wenn eine Aufgabe in einer Chat-App beginnt, die nächsten Antworten
aber an anderer Stelle eingehen sollen.

Typischer Ablauf:

1. Starten Sie eine Agentenaufgabe über Telegram.
2. Wechseln Sie zu Discord, wo Sie die Arbeit koordinieren.
3. Senden Sie `/dock_discord` aus dem direkten Telegram-Chat.
4. Behalten Sie dieselbe OpenClaw-Sitzung bei, empfangen Sie künftige Antworten jedoch in Discord.

## Erforderliche Konfiguration

Docking erfordert `session.identityLinks`. Der ursprüngliche Absender und der Ziel-Peer
müssen derselben Identitätsgruppe angehören:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456", "slack:U123"],
    },
  },
}
```

Die Werte sind Peer-IDs mit vorangestelltem Kanalpräfix:

| Wert          | Bedeutung                      |
| -------------- | ---------------------------- |
| `telegram:123` | Telegram-Absender-ID `123`     |
| `discord:456`  | ID des direkten Discord-Peers `456` |
| `slack:U123`   | Slack-Benutzer-ID `U123`         |

Der kanonische Schlüssel (`alice` oben) ist lediglich der gemeinsame Name der Identitätsgruppe. Dock-
Befehle verwenden die Werte mit Kanalpräfix, um nachzuweisen, dass der ursprüngliche Absender und
der Ziel-Peer dieselbe Person sind.

## Befehle

OpenClaw generiert für jedes geladene Kanal-Plugin, das native Befehle
unterstützt, einen `/dock-<channel>`-Befehl. Daher wächst die Liste, wenn Plugins hinzugefügt werden. Mitgelieferte
Plugins, die dies derzeit unterstützen:

| Zielkanal | Befehl            | Alias              |
| -------------- | ------------------ | ------------------ |
| Discord        | `/dock-discord`    | `/dock_discord`    |
| Mattermost     | `/dock-mattermost` | `/dock_mattermost` |
| Slack          | `/dock-slack`      | `/dock_slack`      |
| Telegram       | `/dock-telegram`   | `/dock_telegram`   |

Die Form mit Unterstrich ist zugleich der Name des nativen Befehls auf Oberflächen wie Telegram,
die Slash-Befehle direkt bereitstellen.

## Was sich ändert

Docking aktualisiert die Zustellungsfelder der aktiven Sitzung:

| Sitzungsfeld   | Beispiel nach `/dock_discord`            |
| --------------- | ---------------------------------------- |
| `lastChannel`   | `discord`                                |
| `lastTo`        | `456`                                    |
| `lastAccountId` | das Zielkanalkonto oder `default` |

Diese Felder werden im Sitzungsspeicher persistiert und für die spätere Zustellung von Antworten
dieser Sitzung verwendet.

## Was sich nicht ändert

Docking bewirkt Folgendes nicht:

- Kanalkonten erstellen
- einen neuen Discord-, Telegram-, Slack- oder Mattermost-Bot verbinden
- einem Benutzer Zugriff gewähren
- Kanal-Zulassungslisten oder Richtlinien für Direktnachrichten umgehen
- den Transkriptverlauf in eine andere Sitzung verschieben
- nicht miteinander verbundene Benutzer eine Sitzung gemeinsam verwenden lassen

Es ändert lediglich die Zustellroute der aktuellen Sitzung.

## Fehlerbehebung

**Der Befehl meldet, dass der Absender nicht verknüpft ist.**

Fügen Sie sowohl den aktuellen Absender als auch den Ziel-Peer derselben
`session.identityLinks`-Gruppe hinzu. Wenn beispielsweise der Telegram-Absender `123` an den
Discord-Peer `456` andocken soll, schließen Sie sowohl `telegram:123` als auch `discord:456` ein.

**Der Befehl meldet, dass Docking nur aus direkten Chats verfügbar ist.**

Senden Sie den Dock-Befehl aus einem direkten Chat mit OpenClaw und nicht aus einem Gruppenchat.

**Der Befehl meldet, dass keine aktive Sitzung vorhanden ist.**

Führen Sie das Docking aus einer bestehenden Direktchat-Sitzung durch. Der Befehl benötigt einen aktiven Sitzungseintrag,
damit er die neue Route persistieren kann.

**Antworten werden weiterhin an den alten Kanal gesendet.**

Prüfen Sie, ob der Befehl mit einer Erfolgsmeldung geantwortet hat, und bestätigen Sie, dass die ID des Ziel-
Peers mit der von diesem Kanal verwendeten ID übereinstimmt. Docking ändert nur die Route der aktiven
Sitzung; eine andere Sitzung kann weiterhin an einen anderen Ort weiterleiten.

**Ich muss zurückwechseln.**

Senden Sie den entsprechenden Befehl für den ursprünglichen Kanal, etwa `/dock_telegram` oder
`/dock-telegram`, von einem verknüpften Absender.
