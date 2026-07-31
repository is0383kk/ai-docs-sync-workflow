---
read_when:
    - Gruppenchat-Verhalten oder Erwähnungsfilter ändern
    - mentionPatterns auf bestimmte Gruppenunterhaltungen beschränken
sidebarTitle: Groups
summary: Gruppenchat-Verhalten auf verschiedenen Plattformen (Discord/iMessage/Matrix/Microsoft Teams/QQBot/Signal/Slack/Telegram/WhatsApp/Zalo)
title: Gruppen
x-i18n:
    generated_at: "2026-07-26T17:38:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 146378f0fc31e129b6504df6778ab8633c048cd4d02af02a5e6da1bfef640d3f
    source_path: channels/groups.md
    workflow: 16
---

OpenClaw wendet dieselben Gruppenregeln auf alle gruppenfähigen Kanäle an, darunter Discord, iMessage, Matrix, Microsoft Teams, QQBot, Signal, Slack, Telegram, WhatsApp und Zalo.

Informationen zu dauerhaft aktiven Räumen, die unaufdringlich Kontext bereitstellen sollen, sofern der Agent nicht ausdrücklich eine sichtbare Nachricht sendet, finden Sie unter [Umgebungsbezogene Raumereignisse](/de/channels/ambient-room-events).

## Einführung für Einsteiger (2 Minuten)

OpenClaw „lebt“ in Ihren eigenen Messaging-Konten. Es gibt keinen separaten WhatsApp-Bot-Benutzer: Wenn **Sie** Mitglied einer Gruppe sind, kann OpenClaw diese Gruppe sehen und dort antworten.

Standardverhalten:

- Gruppen sind eingeschränkt (`groupPolicy: "allowlist"`); Absender in Gruppen werden blockiert, bis sie auf die Zulassungsliste gesetzt werden.
- Antworten erfordern eine Erwähnung, sofern Sie die Erwähnungsprüfung für eine Gruppe nicht deaktivieren.
- Der endgültige Antworttext wird automatisch im Raum veröffentlicht (`visibleReplies: "automatic"`).

Das bedeutet: Zugelassene Absender können OpenClaw auslösen, indem sie es erwähnen.

<Note>
**Kurzfassung**

- Der **DM-Zugriff** wird durch `*.allowFrom` gesteuert.
- Der **Gruppenzugriff** wird durch `*.groupPolicy` und Zulassungslisten (`*.groups`, `*.groupAllowFrom`) gesteuert.
- Das **Auslösen von Antworten** wird durch die Erwähnungsprüfung (`requireMention`, `/activation`) gesteuert.

</Note>

Kurzer Ablauf (was mit einer Gruppennachricht geschieht):

```text
groupPolicy? disabled -> verwerfen
groupPolicy? allowlist -> Gruppe zugelassen? nein -> verwerfen
requireMention? ja -> erwähnt? nein -> nur als Kontext speichern
Erwähnung/Antwort/Befehl/DM -> Benutzeranfrage
Unterhaltungen in dauerhaft aktiven Gruppen -> Benutzeranfrage oder, wenn konfiguriert, Raumereignis
```

## Sichtbare Antworten

Bei normalen Gruppen-/Kanalanfragen verwendet OpenClaw standardmäßig `messages.groupChat.visibleReplies: "automatic"`: Der endgültige Assistententext wird als sichtbare Antwort im Raum veröffentlicht.

Verwenden Sie `messages.groupChat.visibleReplies: "message_tool"`, wenn der Agent in einem gemeinsam genutzten Raum selbst entscheiden soll, wann er spricht, indem er `message(action=send)` aufruft. Dies funktioniert am besten mit Modellen, die Tools zuverlässig verwenden (beispielsweise GPT-5.6 Sol). Wenn das Modell das Tool nicht verwendet und stattdessen substanziellen endgültigen Text zurückgibt, hält OpenClaw diesen Text privat, anstatt ihn im Raum zu veröffentlichen.

Verwenden Sie `"automatic"` für Modelle oder Laufzeitumgebungen, die eine ausschließlich toolbasierte Zustellung nicht zuverlässig befolgen: Normale endgültige Textantworten werden direkt im Raum veröffentlicht, und der Agent kann weiterhin `message(action=send)` für Dateien, Bilder oder andere Anhänge aufrufen, die nicht zusammen mit dem endgültigen Text übertragen werden können.

Wenn das Nachrichtentool gemäß der aktiven Tool-Richtlinie nicht verfügbar ist, greift OpenClaw auf automatische sichtbare Antworten zurück, anstatt die Antwort stillschweigend zu unterdrücken. `openclaw doctor` warnt vor dieser Diskrepanz.

Für Direktchats und alle anderen Quellereignisse wendet `messages.visibleReplies: "message_tool"` dasselbe ausschließlich toolbasierte Verhalten global an; `messages.groupChat.visibleReplies` bleibt die spezifischere Überschreibung für Gruppen-/Kanalräume. Interne direkte WebChat-Interaktionen verwenden standardmäßig die automatische Zustellung endgültiger Antworten, sodass Pi und Codex denselben Vertrag für sichtbare Antworten erhalten.

Der ausschließlich toolbasierte Modus ersetzt das frühere Muster, das Modell bei den meisten Interaktionen im stillen Beobachtungsmodus zu einer Antwort mit `NO_REPLY` zu zwingen. Im ausschließlich toolbasierten Modus definiert der Prompt keinen `NO_REPLY`-Vertrag; nichts sichtbar zu tun bedeutet lediglich, das Nachrichtentool nicht aufzurufen.

Plugin-eigene Konversationsbindungen bilden die Ausnahme. Sobald ein Plugin einen Thread bindet und die eingehende Interaktion übernimmt, ist die vom Plugin zurückgegebene Antwort die sichtbare Bindungsantwort; sie benötigt `message(action=send)` nicht. Diese Antwort ist eine Ausgabe der Plugin-Laufzeitumgebung und kein privater endgültiger Modelltext.

Bei direkten Gruppenanfragen werden weiterhin Tippindikatoren gesendet. Umgebungsbezogene Ereignisse dauerhaft aktiver Räume bleiben, wenn aktiviert, strikt und unaufdringlich, sofern der Agent nicht das Nachrichtentool aufruft.

Sitzungen unterdrücken standardmäßig ausführliche Tool-/Fortschrittszusammenfassungen. Verwenden Sie beim Debuggen `/verbose on` (oder `/verbose full`), um sie für die aktuelle Sitzung anzuzeigen, und `/verbose off`, um zum Verhalten mit ausschließlich endgültigen Antworten zurückzukehren. Der Ausführlichkeitsstatus gilt pro Sitzung und funktioniert in Direktchats, Gruppen, Kanälen und Forenthemen gleich.

Um nicht erwähnte Unterhaltungen in dauerhaft aktiven Gruppen als unaufdringlichen Raumkontext statt als Benutzeranfragen zu übermitteln, verwenden Sie [Umgebungsbezogene Raumereignisse](/de/channels/ambient-room-events):

```json5
{
  messages: {
    groupChat: {
      unmentionedInbound: "room_event",
    },
  },
}
```

Der Standardwert ist `unmentionedInbound: "user_request"`. Erwähnte Nachrichten, Befehle, Abbruchanfragen und DMs bleiben Benutzeranfragen.

So erzwingen Sie, dass sichtbare Ausgaben für Gruppen-/Kanalanfragen über das Nachrichtentool gesendet werden:

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "message_tool",
    },
  },
}
```

So erzwingen Sie dies für jeden Quellchat:

```json5
{
  messages: {
    visibleReplies: "message_tool",
  },
}
```

Das Gateway übernimmt Änderungen an der `messages`-Konfiguration nach dem Speichern der Datei ohne Neustart. Starten Sie nur neu, wenn das erneute Laden der Konfiguration deaktiviert ist (`gateway.reload.mode: "off"`).

Befehlsinteraktionen umgehen `visibleReplies: "message_tool"` und antworten immer sichtbar: Sowohl native Slash-Befehle (Discord, Telegram und andere Oberflächen mit nativer Befehlsunterstützung) als auch autorisierte `/...`-Textbefehle veröffentlichen ihre Antwort im Quellchat. Nicht autorisierte `/...`-Textinteraktionen in Gruppen bleiben ausschließlich nachrichtentoolbasiert; gewöhnliche Chatinteraktionen folgen dem konfigurierten Standard.

## Kontextsichtbarkeit und Zulassungslisten

Für die Sicherheit von Gruppen sind zwei unterschiedliche Steuerungen relevant:

- **Auslöseberechtigung**: Wer den Agenten auslösen kann (`groupPolicy`, `groups`, `groupAllowFrom`, kanalspezifische Zulassungslisten).
- **Kontextsichtbarkeit**: Welcher ergänzende Kontext in das Modell eingespeist wird (Antwort-/Zitattext, Threadverlauf, weitergeleitete Metadaten).

Standardmäßig behält OpenClaw den Kontext so bei, wie er empfangen wurde: Zulassungslisten bestimmen, wer Aktionen auslösen kann, nicht welche zitierten oder historischen Ausschnitte das Modell sieht. Um auch ergänzenden Kontext zu filtern, legen Sie `contextVisibility` fest:

| Modus                | Verhalten                                                                         |
| ------------------- | -------------------------------------------------------------------------------- |
| `"all"` (Standard)   | Ergänzenden Kontext so beibehalten, wie er empfangen wurde.                                           |
| `"allowlist"`       | Nur Verlaufs-, Thread-, Zitat- und weitergeleiteten Kontext von zugelassenen Absendern einspeisen.     |
| `"allowlist_quote"` | `allowlist`; zusätzlich die ausdrücklich zitierte oder beantwortete Nachricht jedes Absenders beibehalten. |

Legen Sie dies pro Kanal (`channels.<channel>.contextVisibility`), pro Konto (`channels.<channel>.accounts.<accountId>.contextVisibility`) oder global (`channels.defaults.contextVisibility`) fest. Kanäle, die ergänzenden Kontext abrufen (Discord, Feishu, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp), wenden die Richtlinie beim Erstellen des eingehenden Kontexts an; unbekannte Richtlinienkombinationen schlagen nach dem Fail-Closed-Prinzip fehl und lassen den Kontext weg.

Diese Modi filtern nur den vom Kanal bereitgestellten ergänzenden Kontext. Die Tool-Richtlinie und das ausschließlich dem Eigentümer vorbehaltene Tool-Inventar werden weiterhin anhand des ursprünglichen Anfragenden der aktuellen Interaktion ausgewählt, nicht anhand jedes im Prompt dargestellten Absenders. Siehe [Auf den Anfragenden beschränkte Steuerungen und Prompt-Kontext](/de/gateway/security#requester-scoped-controls-and-prompt-context).

![Ablauf von Gruppennachrichten](/images/groups-flow.svg)

Wenn Sie Folgendes erreichen möchten ...

| Ziel                                         | Festzulegender Wert                                                |
| -------------------------------------------- | ---------------------------------------------------------- |
| Alle Gruppen zulassen, aber nur auf @Erwähnungen antworten | `groups: { "*": { requireMention: true } }`                |
| Alle Gruppenantworten deaktivieren                    | `groupPolicy: "disabled"`                                  |
| Nur bestimmte Gruppen                         | `groups: { "<group-id>": { ... } }` (kein Schlüssel `"*"`)         |
| Nur Sie können den Agenten in Gruppen auslösen               | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |
| Eine vertrauenswürdige Absendergruppe kanalübergreifend wiederverwenden | `groupAllowFrom: ["accessGroup:operators"]`                |

Informationen zu wiederverwendbaren Absender-Zulassungslisten finden Sie unter [Zugriffsgruppen](/de/channels/access-groups).

## Sitzungsschlüssel

- Gruppensitzungen verwenden `agent:<agentId>:<channel>:group:<id>`-Sitzungsschlüssel (Räume/Kanäle verwenden `agent:<agentId>:<channel>:channel:<id>`).
- Telegram-Forenthemen fügen der Gruppen-ID `:topic:<threadId>` hinzu, sodass jedes Thema über eine eigene Sitzung verfügt.
- Direktchats verwenden die Hauptsitzung (oder Sitzungen pro Absender, wenn `session.dmScope` konfiguriert ist).
- Heartbeats werden in der konfigurierten Heartbeat-Sitzung ausgeführt (Standard: Hauptsitzung des Agenten); Gruppensitzungen führen keine eigenen Heartbeats aus.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Muster: persönliche DMs + öffentliche Gruppen (ein Agent)

Ja — dies funktioniert gut, wenn Ihr „persönlicher“ Datenverkehr aus **DMs** und Ihr „öffentlicher“ Datenverkehr aus **Gruppen** besteht.

Grund: Im Einzelagentenmodus landen DMs üblicherweise unter dem **Hauptsitzungs**-Schlüssel (`agent:main:main`), während Gruppen immer **Nicht-Hauptsitzungs**-Schlüssel (`agent:main:<channel>:group:<id>`) verwenden. Wenn Sie Sandboxing mit `mode: "non-main"` aktivieren, werden diese Gruppensitzungen im konfigurierten Sandbox-Backend ausgeführt, während Ihre DM-Hauptsitzung auf dem Host verbleibt. Docker ist das Standard-Backend, wenn Sie kein anderes auswählen.

Dadurch erhalten Sie ein gemeinsames „Gehirn“ des Agenten (gemeinsamer Arbeitsbereich und Speicher), aber zwei Ausführungsmodi:

- **DMs**: vollständige Tools (Host)
- **Gruppen**: Sandbox und eingeschränkte Tools

<Note>
Wenn Sie vollständig getrennte Arbeitsbereiche/Personas benötigen („persönlich“ und „öffentlich“ dürfen sich niemals vermischen), verwenden Sie einen zweiten Agenten und Bindungen. Siehe [Multi-Agenten-Routing](/de/concepts/multi-agent).
</Note>

<Tabs>
  <Tab title="DMs auf dem Host, Gruppen in der Sandbox">
    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main", // Gruppen/Kanäle sind keine Hauptsitzungen -> in der Sandbox
            scope: "session", // stärkste Isolation (ein Container pro Gruppe/Kanal)
            workspaceAccess: "none",
          },
        },
      },
      tools: {
        sandbox: {
          tools: {
            // Wenn allow nicht leer ist, wird alles andere blockiert (deny hat weiterhin Vorrang).
            allow: ["group:messaging", "group:sessions"],
            deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Gruppen sehen nur einen zugelassenen Ordner">
    Sollen Gruppen „nur Ordner X sehen“ können, statt „keinen Hostzugriff“ zu haben? Behalten Sie `workspaceAccess: "none"` bei und binden Sie nur zugelassene Pfade in die Sandbox ein:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",
            scope: "session",
            workspaceAccess: "none",
            docker: {
              binds: [
                // Hostpfad:Containerpfad:Modus
                "/home/user/FriendsShared:/data:ro",
              ],
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

Verwandte Themen:

- Konfigurationsschlüssel und Standardwerte: [Gateway-Konfiguration](/de/gateway/config-agents#agentsdefaultssandbox)
- Debugging, warum ein Tool blockiert wird: [Sandbox im Vergleich zu Tool-Richtlinie und erhöhten Berechtigungen](/de/gateway/sandbox-vs-tool-policy-vs-elevated)
- Details zu Bind-Mounts: [Sandboxing](/de/gateway/sandboxing#custom-bind-mounts)

## Anzeigebezeichnungen

- UI-Bezeichnungen verwenden `displayName`, sofern verfügbar, formatiert als `<channel>:<token>`.
- `#room` ist für Räume/Kanäle reserviert; Gruppenchats verwenden `g-<slug>` (Kleinbuchstaben, Leerzeichen -> `-`, `#@+._-` beibehalten). Sehr lange undurchsichtige IDs werden zu einem stabilen Token verkürzt, statt vollständige Routing-IDs in der UI offenzulegen.

## Gruppenrichtlinie

Steuern Sie pro Kanal, wie Gruppen-/Raumnachrichten behandelt werden:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numerische Telegram-Benutzer-ID (Einrichtung löst @username auf)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { enabled: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { enabled: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { enabled: true },
        "#alias:example.org": { enabled: true },
      },
    },
  },
}
```

| Richtlinie        | Verhalten                                                     |
| ------------- | ------------------------------------------------------------ |
| `"open"`      | Gruppen umgehen Zulassungslisten; das Erwähnungs-Gating gilt weiterhin.      |
| `"disabled"`  | Alle Gruppennachrichten vollständig blockieren.                           |
| `"allowlist"` | Nur Gruppen/Räume zulassen, die der konfigurierten Zulassungsliste entsprechen. |

<AccordionGroup>
  <Accordion title="Hinweise pro Kanal">
    - `groupPolicy` ist vom Erwähnungs-Gating getrennt (das @Erwähnungen erfordert).
    - WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: `groupAllowFrom` verwenden (Fallback: explizites `allowFrom`).
    - Signal: `groupAllowFrom` kann entweder der eingehenden Signal-Gruppen-ID oder der Telefonnummer/UUID des Absenders entsprechen.
    - Genehmigungen für DM-Kopplungen (Einträge im Speicher `*-allowFrom`) gelten nur für den DM-Zugriff; die Autorisierung von Gruppenabsendern bleibt explizit an Gruppen-Zulassungslisten gebunden.
    - Discord: Die Zulassungsliste verwendet `channels.discord.guilds.<id>.channels`.
    - Slack: Die Zulassungsliste verwendet `channels.slack.channels`.
    - Matrix: Die Zulassungsliste verwendet `channels.matrix.groups`. Verwenden Sie Raum-IDs (`!room:server`) oder Aliase (`#alias:server`); Schlüssel mit Raumnamen stimmen nur mit `channels.matrix.dangerouslyAllowNameMatching: true` überein, und nicht aufgelöste Einträge werden zur Laufzeit ignoriert. Verwenden Sie `channels.matrix.groupAllowFrom`, um Absender einzuschränken; raumspezifische `users`-Zulassungslisten werden ebenfalls unterstützt.
    - Gruppen-DMs werden separat gesteuert (`channels.discord.dm.*`, `channels.slack.dm.*`: `groupEnabled`, `groupChannels`).
    - Telegram: Absender-Zulassungslisten akzeptieren nur numerische Benutzer-IDs (`"123456789"`; die Präfixe `telegram:`/`tg:` werden ohne Berücksichtigung der Groß-/Kleinschreibung entfernt). `@username`-Einträge stimmen zur Laufzeit nicht überein und protokollieren eine Warnung; die Einrichtung löst `@username` in IDs auf. Negative Chat-IDs gehören unter `channels.telegram.groups`, nicht in Absender-Zulassungslisten.
    - Der Standardwert ist `groupPolicy: "allowlist"`; wenn Ihre Gruppen-Zulassungsliste leer ist, werden Gruppennachrichten blockiert.
    - Laufzeitsicherheit: Wenn ein Provider-Block vollständig fehlt (`channels.<provider>` nicht vorhanden), wird die Gruppenrichtlinie geschlossen auf `allowlist` gesetzt, statt `channels.defaults.groupPolicy` zu übernehmen, und das Gateway protokolliert den Fallback einmal pro Konto.

  </Accordion>
</AccordionGroup>

Kurzes mentales Modell (Auswertungsreihenfolge für Gruppennachrichten):

<Steps>
  <Step title="groupPolicy">
    `groupPolicy` (open/disabled/allowlist).
  </Step>
  <Step title="Gruppen-Zulassungslisten">
    Gruppen-Zulassungslisten (`*.groups`, `*.groupAllowFrom`, kanalspezifische Zulassungsliste).
  </Step>
  <Step title="Erwähnungs-Gating">
    Erwähnungs-Gating (`requireMention`, `/activation`).
  </Step>
</Steps>

## Erwähnungs-Gating (Standard)

Gruppennachrichten erfordern eine Erwähnung, sofern dies nicht pro Gruppe überschrieben wird. Standardwerte befinden sich je Subsystem unter `*.groups."*"`.

Unterstützte implizite Erwähnungsfakten sind kanalspezifisch:

| Fakt                  | Aktuelle integrierte Erzeuger                       |
| --------------------- | ------------------------------------------------ |
| Antwort an den Bot      | Discord, Microsoft Teams, QQBot, Slack, Telegram |
| Zitat des Bots      | WhatsApp, Zalo Personal                          |
| Bot ist dem Thread beigetreten | Mattermost, Slack, Tlon                          |

Jeder Fakt ist standardmäßig aktiviert, wenn der Kanal ihn erzeugt. Setzen Sie das entsprechende `implicitMentions`-Flag auf `false`, damit dieser Fakt das Erwähnungs-Gating nicht mehr umgeht; native explizite Erwähnungen bleiben davon unberührt. Ein Flag hat keine Auswirkung auf Kanäle, die diesen Fakt nicht erzeugen.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    },
  },
}
```

## Konfigurierte Erwähnungsmuster eingrenzen

Konfigurierte `mentionPatterns` sind Regex-Fallback-Auslöser. Verwenden Sie sie, wenn die
Plattform keine native Bot-Erwähnung bereitstellt oder wenn Klartext wie
`openclaw:` als Erwähnung gelten soll. Native Plattform-Erwähnungen sind separat:
Wenn Discord, Slack, Telegram, Matrix, Signal oder ein anderer Kanal nachweisen kann, dass die Nachricht
den Bot explizit erwähnt hat, löst diese native Erwähnung weiterhin aus, selbst wenn
konfigurierte Regex-Muster verweigert werden.

Standardmäßig gelten konfigurierte Erwähnungsmuster überall dort, wo der Kanal Provider- und Konversationsfakten an die Erwähnungserkennung übergibt. Um zu verhindern, dass weit gefasste Muster den Agenten in jeder Gruppe aktivieren, grenzen Sie sie pro Kanal mit `channels.<channel>.mentionPatterns` ein.

Verwenden Sie `mode: "deny"`, wenn Regex-Erwähnungsmuster für einen Kanal standardmäßig deaktiviert sein sollen, und aktivieren Sie sie anschließend mit `allowIn` für bestimmte Räume:

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b", "\\bops bot\\b"],
    },
  },
  channels: {
    slack: {
      mentionPatterns: {
        mode: "deny",
        allowIn: ["C0123OPS"],
      },
    },
  },
}
```

Verwenden Sie den Standardwert `mode: "allow"` (oder lassen Sie `mode` weg), wenn Regex-Erwähnungsmuster allgemein gelten sollen, und deaktivieren Sie sie anschließend mit `denyIn` in störungsreichen Räumen:

```json5
{
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
  channels: {
    telegram: {
      mentionPatterns: {
        denyIn: ["-1001234567890", "-1001234567890:topic:42"],
      },
    },
  },
}
```

Richtlinienauflösung:

| Feld           | Auswirkung                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| `mode: "allow"` | Regex-Erwähnungsmuster sind aktiviert, sofern sich die Konversations-ID nicht in `denyIn` befindet. Dies ist der Standardwert.                    |
| `mode: "deny"`  | Regex-Erwähnungsmuster sind deaktiviert, sofern sich die Konversations-ID nicht in `allowIn` befindet.                                       |
| `allowIn`       | Konversations-IDs, für die Regex-Erwähnungsmuster im Verweigerungsmodus aktiviert sind.                                               |
| `denyIn`        | Konversations-IDs, für die Regex-Erwähnungsmuster deaktiviert sind. `denyIn` hat Vorrang vor `allowIn`, wenn beide dieselbe ID enthalten. |

Derzeit unterstützte eingegrenzte Regex-Richtlinien:

| Kanal  | In `allowIn` / `denyIn` verwendete IDs                             |
| -------- | ------------------------------------------------------------ |
| Discord  | Discord-Kanal-IDs.                                         |
| Matrix   | Matrix-Raum-IDs.                                             |
| Slack    | Slack-Kanal-IDs.                                           |
| Telegram | Gruppenchat-IDs oder `chatId:topic:threadId` für Forumsthemen. |
| WhatsApp | WhatsApp-Konversations-IDs wie `123@g.us`.                |

Kanalkonfigurationen auf Kontoebene können dieselbe Richtlinie unter `channels.<channel>.accounts.<accountId>.mentionPatterns` festlegen, wenn der Kanal mehrere Konten unterstützt. Die Kontorichtlinie hat für dieses Konto Vorrang vor der Kanalrichtlinie der obersten Ebene.

<AccordionGroup>
  <Accordion title="Hinweise zum Erwähnungs-Gating">
    - `mentionPatterns` sind sichere Regex-Muster ohne Berücksichtigung der Groß-/Kleinschreibung; ungültige Muster und unsichere Formen mit verschachtelten Wiederholungen werden ignoriert (mit einer Warnung).
    - Musterpriorität: `agents.entries.*.groupChat.mentionPatterns` (nützlich, wenn mehrere Agenten eine Gruppe gemeinsam verwenden) überschreibt `messages.groupChat.mentionPatterns`; wenn keines von beiden festgelegt ist, werden Muster aus dem Namen/Emoji der Agentenidentität abgeleitet.
    - Das Erwähnungs-Gating wird nur erzwungen, wenn die Erkennung von Erwähnungen möglich ist (native Erwähnungen oder konfigurierte `mentionPatterns`).
    - Das Aufnehmen einer Gruppe oder eines Absenders in die Zulassungsliste deaktiviert das Erwähnungs-Gating nicht; setzen Sie `requireMention` dieser Gruppe auf `false`, wenn alle Nachrichten auslösen sollen.
    - Der automatische Prompt-Kontext für Gruppenchats enthält bei jedem Durchlauf die aufgelöste Anweisung für stille Antworten; Workspace-Dateien sollten die `NO_REPLY`-Mechanik nicht duplizieren.
    - Gruppen, in denen automatische stille Antworten zulässig sind, behandeln saubere leere oder reine Reasoning-Modellläufe als still, entsprechend `NO_REPLY`. Direkte Chats erhalten nie `NO_REPLY`-Anweisungen, und reine Nachrichtentool-Gruppenantworten bleiben still, indem `message(action=send)` nicht aufgerufen wird.
    - Ständige Hintergrundkommunikation in Gruppen verwendet standardmäßig die Semantik von Benutzeranfragen. Setzen Sie `messages.groupChat.unmentionedInbound: "room_event"`, um sie stattdessen als stillen Kontext zu übermitteln. Einrichtungsbeispiele finden Sie unter [Raumereignisse im Hintergrund](/de/channels/ambient-room-events).
    - Raumereignisse werden nicht als fingierte Benutzeranfragen gespeichert, und privater Assistententext aus Raumereignissen ohne Nachrichtentool wird nicht als Chatverlauf erneut wiedergegeben.
    - Die Discord-Standardwerte befinden sich in `channels.discord.guilds."*"` (pro Guild/Kanal überschreibbar).
    - Der Kontext des Gruppenverlaufs wird kanalübergreifend einheitlich umschlossen. Gruppen mit Erwähnungs-Gating behalten ausstehende übersprungene Nachrichten; ständig aktive Gruppen können auch kürzlich verarbeitete Raumnachrichten beibehalten, wenn der Kanal dies unterstützt. Verwenden Sie `messages.groupChat.historyLimit` für den globalen Standardwert und `channels.<channel>.historyLimit` (oder `channels.<channel>.accounts.*.historyLimit`) für Überschreibungen. Setzen Sie `0`, um dies zu deaktivieren.

  </Accordion>
</AccordionGroup>

## Tool-Einschränkungen für Gruppen/Kanäle (optional)

Einige Kanalkonfigurationen unterstützen die Einschränkung, welche Tools **innerhalb einer bestimmten Gruppe/eines bestimmten Raums/Kanals** verfügbar sind.

- `tools`: Tools für die gesamte Gruppe zulassen/verweigern (`allow`, `alsoAllow`, `deny`; Verweigerung hat Vorrang).
- `toolsBySender`: Absenderspezifische Überschreibungen innerhalb der Gruppe. Verwenden Sie explizite Schlüsselpräfixe: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` und den Platzhalter `"*"`. Kanal-IDs verwenden kanonische OpenClaw-Kanal-IDs; Aliase wie `teams` werden zu `msteams` normalisiert. Veraltete Schlüssel ohne Präfix werden weiterhin akzeptiert, nur als `id:` abgeglichen und protokollieren eine Veraltungswarnung.

Auflösungsreihenfolge (die spezifischste Regel hat Vorrang):

<Steps>
  <Step title="Gruppen-toolsBySender">
    Übereinstimmung mit Gruppen-/Kanal-`toolsBySender`.
  </Step>
  <Step title="Gruppen-Tools">
    Gruppen-/Kanal-`tools`.
  </Step>
  <Step title="Standard-toolsBySender">
    Übereinstimmung mit `toolsBySender` im Standardwert (`"*"`).
  </Step>
  <Step title="Standard-Tools">
    `tools` im Standardwert (`"*"`).
  </Step>
</Steps>

Beispiel (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

<Note>
Werkzeugbeschränkungen für Gruppen/Kanäle werden zusätzlich zur globalen bzw. agentenspezifischen Werkzeugrichtlinie angewendet (Verbote haben weiterhin Vorrang). Einige Kanäle verwenden für Räume/Kanäle eine andere Verschachtelung (z. B. Discord `guilds.*.channels.*`, Slack `channels.*`, Microsoft Teams `teams.*.channels.*`).
</Note>

## Gruppen-Zulassungslisten

Wenn `channels.whatsapp.groups`, `channels.telegram.groups` oder `channels.imessage.groups` konfiguriert ist, dienen die Schlüssel als Gruppen-Zulassungsliste. Verwenden Sie `"*"`, um alle Gruppen zuzulassen und dennoch das standardmäßige Erwähnungsverhalten festzulegen.

<Warning>
Häufiges Missverständnis: Die Genehmigung einer DM-Kopplung ist nicht mit der Gruppenautorisierung gleichzusetzen. Bei Kanälen, die DM-Kopplung unterstützen, schaltet der Kopplungsspeicher ausschließlich DMs frei. Gruppenbefehle erfordern weiterhin eine ausdrückliche Autorisierung des Absenders über Konfigurations-Zulassungslisten wie `groupAllowFrom` oder den dokumentierten Konfigurations-Fallback des jeweiligen Kanals.
</Warning>

Gängige Anwendungsfälle (zum Kopieren und Einfügen):

<Tabs>
  <Tab title="Alle Gruppenantworten deaktivieren">
    ```json5
    {
      channels: { whatsapp: { groupPolicy: "disabled" } },
    }
    ```
  </Tab>
  <Tab title="Nur bestimmte Gruppen zulassen (WhatsApp)">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: {
            "123@g.us": { requireMention: true },
            "456@g.us": { requireMention: false },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Alle Gruppen zulassen, aber Erwähnung voraussetzen">
    ```json5
    {
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Auslösung nur durch Eigentümer (WhatsApp)">
    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Aktivierung (nur Eigentümer)

Gruppeneigentümer können die Aktivierung für jede Gruppe mit einer eigenständigen Nachricht umschalten:

- `/activation mention`
- `/activation always`

`/activation` ist ein Kernbefehl, der Eigentümern vorbehalten ist und nur in Gruppenchats gilt. Eigentümer bedeutet, dass der Absender `commands.ownerAllowFrom` entspricht; `allowFrom`-Listen des Kanals steuern lediglich den gewöhnlichen Kanal- und Befehlszugriff. Der gespeicherte Modus überschreibt auf Kanälen, die ihn berücksichtigen (Google Chat, QQBot, Telegram, WhatsApp), den Wert `requireMention` dieser Gruppe, und die Einleitung des Gruppen-System-Prompts gibt überall den aktiven Modus wieder.

## Kontextfelder

Eingehende Gruppen-Payloads legen Folgendes fest:

- `ChatType=group`
- `GroupSubject` (falls bekannt)
- `GroupMembers` (falls bekannt)
- `WasMentioned` (Ergebnis der Erwähnungsprüfung)
- Telegram-Forenthemen enthalten außerdem `MessageThreadId` und `IsForum`.

Der Agent-System-Prompt enthält im ersten Durchlauf einer neuen Gruppensitzung (und nach Änderungen an `/activation`) eine Gruppeneinleitung. Sie erinnert das Modell daran, wie ein Mensch zu antworten, Leerzeilen zu minimieren, die üblichen Chat-Abstände einzuhalten und die wörtliche Eingabe von `\n`-Sequenzen zu vermeiden. Kanäle, deren deklarierter Tabellenmodus native oder unformatierte Tabellen nicht beibehält, raten außerdem von Markdown-Tabellen ab. Aus Kanälen stammende Gruppennamen und Teilnehmerbezeichnungen werden als eingezäunte, nicht vertrauenswürdige Metadaten dargestellt, nicht als eingebettete Systemanweisungen.

## iMessage-Besonderheiten

- Bevorzugen Sie `chat_id:<id>` für Routing oder Zulassungslisten.
- Chats auflisten: `imsg chats --limit 20`.
- Gruppenantworten werden immer an dieselbe `chat_id` zurückgesendet.

## WhatsApp-System-Prompts

Die maßgeblichen Regeln für WhatsApp-System-Prompts, einschließlich der Auflösung von Gruppen- und Direkt-Prompts, des Platzhalterverhaltens und der Semantik von Kontoüberschreibungen, finden Sie unter [WhatsApp](/de/channels/whatsapp#system-prompts).

## WhatsApp-Besonderheiten

WhatsApp-spezifisches Verhalten (Verlaufseinfügung und Details zur Verarbeitung von Erwähnungen) finden Sie unter [Gruppennachrichten](/de/channels/group-messages).

## Verwandte Themen

- [Broadcast-Gruppen](/de/channels/broadcast-groups)
- [Kanal-Routing](/de/channels/channel-routing)
- [Gruppennachrichten](/de/channels/group-messages)
- [Kopplung](/de/channels/pairing)
