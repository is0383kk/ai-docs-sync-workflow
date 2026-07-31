---
read_when:
    - Von Bots verfasste Kanalnachrichten konfigurieren
    - Abstimmung des Schutzes vor Bot-zu-Bot-Schleifen
sidebarTitle: Bot loop protection
summary: Standardeinstellungen für den Schutz vor Bot-zu-Bot-Schleifen und kanalspezifische Überschreibungen
title: Bot-Schleifenschutz
x-i18n:
    generated_at: "2026-07-26T18:47:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d59d3b48dd5506e774282b880334df8970b05c4d001261ff7107e8e1678894db
    source_path: channels/bot-loop-protection.md
    workflow: 16
---

OpenClaw kann auf Kanälen, die `allowBots` unterstützen, von anderen Bots verfasste Nachrichten annehmen. Wenn dieser Pfad aktiviert ist, verhindert der Schleifenschutz für Bot-Paare, dass zwei Bot-Identitäten einander unbegrenzt antworten.

Der Schutz wird vom zentralen Runner für eingehende Antworten durchgesetzt. Jeder unterstützende Kanal bildet sein eingehendes Ereignis auf generische Fakten ab: Konto oder Geltungsbereich, Konversations-ID, Bot-ID des Absenders und Bot-ID des Empfängers. Der Kern verfolgt das Teilnehmerpaar in beiden Richtungen (A zu B und B zu A gelten als dasselbe Paar), wendet ein Budget mit gleitendem Zeitfenster an und unterdrückt das Paar für eine Abklingzeit, nachdem das Budget überschritten wurde.

## Standardwerte

Der Schleifenschutz für Bot-Paare ist immer aktiv, wenn ein Kanal zulässt, dass von Bots verfasste Nachrichten zur Verarbeitung weitergeleitet werden. Integrierte Standardwerte:

| Schlüssel             | Standardwert | Bedeutung                                          |
| -------------------- | ------- | --------------------------------------------------- |
| `enabled`            | `true`  | Schutz für Kanäle aktiv, die ihn unterstützen.      |
| `maxEventsPerWindow` | `20`    | Ereignisse, die ein Bot-Paar im Zeitfenster austauschen kann. |
| `windowSeconds`      | `60`    | Länge des gleitenden Zeitfensters.                  |
| `cooldownSeconds`    | `60`    | Unterdrückungsdauer, nachdem das Paar das Budget überschritten hat. |

Der Schutz wirkt sich nicht auf von Menschen verfasste Nachrichten, Bereitstellungen mit nur einem Bot, die Filterung eigener Nachrichten oder Bot-Antworten aus, die unter dem Budget bleiben.

## Gemeinsame Standardwerte konfigurieren

Legen Sie `channels.defaults.botLoopProtection` einmal fest, um allen unterstützenden Kanälen dieselbe Ausgangskonfiguration zuzuweisen. Kanäle können auch spezifischere Überschreibungen bereitstellen; Feishu verwendet absichtlich nur diese gemeinsame Ausgangskonfiguration.

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
        windowSeconds: 60,
        cooldownSeconds: 60,
      },
    },
  },
}
```

Legen Sie `enabled: false` nur fest, wenn Ihre Kanalrichtlinie Bot-zu-Bot-Konversationen bewusst ohne automatische Unterdrückung zulässt.

## Nach Kanal, Konto oder Raum überschreiben

Unterstützende Kanäle legen ihre eigene Konfiguration Schlüssel für Schlüssel über den gemeinsamen Standardwert. Prioritätsreihenfolge, beginnend mit der spezifischsten Ebene:

1. `channels.<channel>.<room-or-space>.botLoopProtection`, wenn der Kanal konversationsspezifische Überschreibungen unterstützt
2. `channels.<channel>.accounts.<account>.botLoopProtection`, wenn der Kanal Konten unterstützt
3. `channels.<channel>.botLoopProtection`, wenn der Kanal Standardwerte auf oberster Ebene unterstützt
4. `channels.defaults.botLoopProtection`
5. integrierte Standardwerte

```json5
{
  channels: {
    defaults: {
      botLoopProtection: {
        maxEventsPerWindow: 20,
      },
    },
    discord: {
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
      accounts: {
        secondary: {
          allowBots: true,
          botLoopProtection: {
            maxEventsPerWindow: 5,
            cooldownSeconds: 90,
          },
        },
      },
    },
    googlechat: {
      allowBots: true,
      groups: {
        "spaces/AAAA": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    matrix: {
      allowBots: "mentions",
      groups: {
        "!roomid:example.org": {
          botLoopProtection: {
            maxEventsPerWindow: 5,
          },
        },
      },
    },
    slack: {
      allowBots: "mentions",
      botLoopProtection: {
        maxEventsPerWindow: 8,
      },
    },
  },
}
```

## Kanalunterstützung

- Discord: native `author.bot`-Fakten, nach Discord-Konto, Kanal und Bot-Paar verschlüsselt.
- Feishu: native `sender_type=bot`-Fakten für zugelassene, von Bots verfasste Gruppennachrichten, nach Feishu-Konto, Chat und Bot-Paar verschlüsselt. Feishu verwendet nur `channels.defaults.botLoopProtection`.
- Google Chat: native `sender.type=BOT`-Fakten für akzeptierte, von Bots verfasste Nachrichten, nach Konto, Space und Bot-Paar verschlüsselt.
- Matrix: konfigurierte Matrix-Bot-Konten, nach Matrix-Konto, Raum und konfiguriertem Bot-Paar verschlüsselt.
- Slack: native `bot_id`-Fakten für akzeptierte, von Bots verfasste Nachrichten, nach Slack-Konto, Kanal und Bot-Paar verschlüsselt.

Kanäle, die keine zuverlässige Identität des eingehenden Bots bereitstellen, verwenden weiterhin ihre normalen Filter für eigene Nachrichten und Zugriffsrichtlinien. Sie sollten diesen Schutz erst aktivieren, wenn sie beide Teilnehmer des Bot-Paars identifizieren können.

Implementierungsdetails für Plugins finden Sie unter [SDK-Laufzeit](/de/plugins/sdk-runtime#reusable-runtime-utilities).
