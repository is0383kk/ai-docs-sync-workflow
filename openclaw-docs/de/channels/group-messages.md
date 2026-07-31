---
read_when:
    - WhatsApp-Gruppen gezielt konfigurieren
    - Ändern der WhatsApp-Aktivierungsmodi (`mention` vs. `always`)
    - Optimierung von WhatsApp-Gruppensitzungsschlüsseln oder des Kontexts ausstehender Nachrichten
sidebarTitle: WhatsApp groups
summary: Handhabung von WhatsApp-Gruppennachrichten — Aktivierung, Zulassungslisten, Sitzungen und Kontextinjektion
title: WhatsApp-Gruppennachrichten
x-i18n:
    generated_at: "2026-07-26T17:38:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

Das kanalübergreifende Gruppenmodell (Discord, iMessage, Matrix, Microsoft Teams, QQBot, Signal, Slack, Telegram, WhatsApp, Zalo) wird unter [Gruppen](/de/channels/groups) beschrieben. Diese Seite behandelt das WhatsApp-spezifische Verhalten, das dieses Modell ergänzt: Aktivierung, Gruppen-Zulassungslisten, gruppenspezifische Sitzungsschlüssel und die Einbindung des Kontexts aus ausstehenden Nachrichten.

Ziel: OpenClaw soll in WhatsApp-Gruppen präsent sein, nur bei direkter Ansprache aktiv werden und diesen Gesprächsverlauf von der persönlichen Direktnachrichtensitzung getrennt halten.

<Note>
`agents.entries.*.groupChat.mentionPatterns` wird gemeinsam mit der Erwähnungssteuerung der anderen Kanäle verwendet. Legen Sie den Wert bei Multi-Agent-Konfigurationen für jeden Agenten einzeln fest oder verwenden Sie `messages.groupChat.mentionPatterns` als globalen Rückfallwert. Wenn keiner der beiden Werte festgelegt ist, werden die Muster aus dem Namen und Emoji der Agentenidentität abgeleitet.
</Note>

## Verhalten

- Aktivierungsmodi: `mention` (Standard) oder `always`. `mention` erfordert eine direkte Ansprache: eine echte WhatsApp-@-Erwähnung (`mentionedJids`), ein konfiguriertes Regex-Muster, die E.164-Ziffern des Bots an beliebiger Stelle im Text oder eine zitierte Antwort auf eine Nachricht des Bots (außer bei Selbstchat-Konfigurationen mit gemeinsam genutzter Nummer). `always` aktiviert den Agenten bei jeder Nachricht, die eingebundene Gruppenanweisung weist ihn jedoch an, nur zu antworten, wenn dies einen Mehrwert bietet, und andernfalls exakt das Stille-Token `NO_REPLY` zurückzugeben (Groß-/Kleinschreibung wird nicht berücksichtigt). Die Standardwerte stammen aus der Konfiguration (`channels.whatsapp.groups` `requireMention`) und können über `/activation` für jede Gruppe überschrieben werden.
- Gruppen-Zulassungsliste: Wenn `channels.whatsapp.groups` festgelegt ist, werden nur die aufgeführten Gruppen-JIDs zugelassen (fügen Sie `"*"` hinzu, um alle zuzulassen); Nachrichten aus nicht aufgeführten Gruppen werden verworfen und ein Hinweis wird protokolliert.
- Gruppenrichtlinie: `channels.whatsapp.groupPolicy` steuert, ob Gruppennachrichten akzeptiert werden (`open|disabled|allowlist`). `allowlist` verwendet `channels.whatsapp.groupAllowFrom` (Rückfallwert: explizites `channels.whatsapp.allowFrom`). Der Standardwert ist `allowlist` (blockiert, bis Sie Absender hinzufügen).
- Gruppenspezifische Sitzungen: Sitzungsschlüssel sehen wie `agent:<agentId>:whatsapp:group:<jid>` aus (bei Konten, die nicht dem Standardkonto entsprechen, wird `:thread:whatsapp-account-<accountId>` angehängt). Daher gelten Anweisungen wie `/verbose on`, `/trace on` oder `/think high` (als eigenständige Nachrichten gesendet) nur für diese Gruppe; der Zustand persönlicher Direktnachrichten bleibt unverändert.
- Kontexteinbindung: Gruppennachrichten, die **nur ausstehend** sind (standardmäßig 50) und _keinen_ Lauf ausgelöst haben, werden unter `[Chat messages since your last reply - for context]` vorangestellt; die auslösende Zeile steht unter `[Current message - respond to this]`. Das Fenster ausstehender Nachrichten wird nach dem Lauf geleert; bereits in der Sitzung vorhandene Nachrichten werden nicht erneut eingebunden.
- Absenderzuordnung: Jede Gruppenzeile enthält die Absenderbezeichnung innerhalb des Nachrichtenumschlags, z. B. `[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`. Die Absenderidentität sowie der Gruppenbetreff und die Mitglieder werden außerdem im Block mit nicht vertrauenswürdigen Konversationsmetadaten übermittelt.
- Flüchtig/einmalig anzeigbar: Wrapper werden vor dem Extrahieren von Text und Erwähnungen entfernt, sodass darin enthaltene direkte Ansprachen weiterhin auslösen.
- Gruppen-Systemanweisung: Beim ersten Durchlauf einer Gruppensitzung und bei jedem Durchlauf, nachdem `/activation` den Modus geändert hat, werden Aktivierungshinweise in die Systemanweisung eingebunden (`Activation: trigger-only ...` oder `Activation: always-on ...` sowie „den jeweiligen Absender direkt ansprechen“). Dauerhafte Hinweise zur Zustellung in Gruppenchats („Sie befinden sich in einem WhatsApp-Gruppenchat ...“) sind immer enthalten.

## Konfigurationsbeispiel (WhatsApp)

So funktionieren direkte Ansprachen über den Anzeigenamen auch dann, wenn WhatsApp das sichtbare `@` aus dem Textinhalt entfernt:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // Fenster für ausstehenden Gruppenkontext (Standardwert: 50)
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

Hinweise:

- Die regulären Ausdrücke unterscheiden nicht zwischen Groß- und Kleinschreibung und verwenden dieselben Schutzmechanismen für sichere reguläre Ausdrücke wie andere Regex-Konfigurationsflächen; ungültige Muster und unsichere verschachtelte Wiederholungen werden ignoriert.
- WhatsApp sendet weiterhin kanonische Erwähnungen über `mentionedJids`, wenn jemand auf den Kontakt tippt. Daher wird der Nummern-Rückfallwert nur selten benötigt, stellt jedoch eine nützliche Absicherung dar.
- Das Fenster für ausstehenden Kontext wird in folgender Reihenfolge aufgelöst: `channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50.

### Aktivierungsbefehl (nur Eigentümer)

Verwenden Sie den Gruppenchatbefehl:

- `/activation mention`
- `/activation always`

Nur Eigentümernummern (aus `channels.whatsapp.allowFrom` oder, falls nicht festgelegt, die eigene E.164-Nummer des Bots) können dies ändern; `/activation` von anderen Personen wird ignoriert und lediglich als Kontext gespeichert. Senden Sie `/status` als eigenständige Nachricht in der Gruppe, um den aktuellen Aktivierungsmodus anzuzeigen.

## Verwendung

1. Fügen Sie Ihr WhatsApp-Konto, auf dem OpenClaw ausgeführt wird, der Gruppe hinzu.
2. Schreiben Sie `@openclaw ...` (oder geben Sie die Nummer an). Nur Absender auf der Zulassungsliste können den Agenten auslösen, sofern Sie nicht `groupPolicy: "open"` festlegen.
3. Die Agentenanweisung enthält den ausstehenden Gruppenkontext und mit Absenderbezeichnungen versehene Zeilen, sodass der Agent die richtige Person ansprechen kann.
4. Sitzungsanweisungen (`/verbose on`, `/trace on`, `/think high`, `/new` oder `/reset`, `/compact`) gelten nur für die Sitzung dieser Gruppe. Senden Sie sie als eigenständige Nachrichten, damit sie registriert werden. Ihre persönliche Direktnachrichtensitzung bleibt davon unabhängig.

## Tests/Überprüfung

- Manueller Kurztest:
  - Senden Sie in der Gruppe eine direkte Ansprache mit `@openclaw` und vergewissern Sie sich, dass die Antwort auf den Namen des Absenders Bezug nimmt.
  - Senden Sie eine zweite direkte Ansprache und prüfen Sie, ob der Verlaufsblock enthalten ist und beim darauffolgenden Durchlauf geleert wurde.
- Prüfen Sie die Gateway-Protokolle (Ausführung mit `--verbose`) auf `inbound web message`-Einträge, die `from: <groupJid>` und den mit einer Absenderbezeichnung versehenen Inhalt enthalten.

## Bekannte Aspekte

- Heartbeats werden in der Hauptsitzung des Agenten ausgeführt; Gruppensitzungen erhalten niemals Heartbeat-Läufe.
- Die Echounterdrückung speichert die kombinierte Anweisung (Verlauf und aktuelle Nachricht) für jede Sitzung, damit die vom Bot selbst zugestellten Nachrichten ihn nicht erneut auslösen. Ein identischer wiederholter Stapel kann als Echo übersprungen werden.
- Einträge im Sitzungsspeicher werden im agentenspezifischen SQLite-Sitzungsspeicher als `agent:<agentId>:whatsapp:group:<jid>` angezeigt. Ein fehlender Eintrag bedeutet lediglich, dass die Gruppe noch keinen Lauf ausgelöst hat.
- Eingabeindikatoren richten sich nach `agents.entries.*.typingMode` / `agents.defaults.typingMode`. Wenn sichtbare Antworten ausschließlich über das Nachrichtentool erfolgen sollen, beginnt die Eingabeanzeige standardmäßig sofort. So können Gruppenmitglieder erkennen, dass der Agent arbeitet, auch wenn keine automatische abschließende Antwort veröffentlicht wird. Eine explizite Konfiguration des Eingabemodus hat weiterhin Vorrang.

## Verwandte Themen

- [Gruppen](/de/channels/groups)
- [Kanalrouting](/de/channels/channel-routing)
- [Broadcast-Gruppen](/de/channels/broadcast-groups)
