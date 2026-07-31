---
read_when:
    - Sie möchten OpenClaw mit IRC-Kanälen oder Direktnachrichten verbinden
    - Sie konfigurieren IRC-Zulassungslisten, Gruppenrichtlinien oder die Erwähnungssteuerung
summary: Einrichtung des IRC-Plugins, Zugriffskontrollen und Fehlerbehebung
title: IRC
x-i18n:
    generated_at: "2026-07-26T18:19:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

Verwenden Sie IRC, wenn Sie OpenClaw in klassischen Kanälen (`#room`) und Direktnachrichten einsetzen möchten.
Installieren Sie das offizielle IRC-Plugin und konfigurieren Sie es anschließend unter `channels.irc`.

## Schnellstart

1. Installieren Sie das Plugin:

```bash
openclaw plugins install @openclaw/irc
```

2. Legen Sie in `~/.openclaw/openclaw.json` mindestens Host, Nick und die beizutretenden Kanäle fest:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. Starten Sie den Gateway bzw. starten Sie ihn neu:

```bash
openclaw gateway run
```

Bevorzugen Sie für die Bot-Koordination einen privaten IRC-Server. Wenn Sie absichtlich ein öffentliches IRC-Netzwerk verwenden, gehören Libera.Chat, OFTC und Snoonet zu den gängigen Optionen. Vermeiden Sie vorhersehbare öffentliche Kanäle für den Backchannel-Datenverkehr von Bots oder Schwärmen.

## Dauerhafte Verarbeitung eingehender Nachrichten

OpenClaw schreibt jedes akzeptierte IRC-`PRIVMSG` vor den regulären Richtlinienprüfungen und der Agent-Weiterleitung in seine dauerhafte Eingangswarteschlange. Ausstehende oder erneut zustellbare Nachrichten überstehen einen Neustart des Gateways und werden weiterhin pro Kanal oder Direktnachrichten-Gegenstelle serialisiert.

IRC stellt keine erneut abrufbare Zustellungs-ID bereit und sendet keine Nachrichten erneut, die ein nicht verbundener Client verpasst hat. OpenClaw weist daher eine lokale ID zu, die nur innerhalb der aktuellen TCP-Verbindung stabil ist. Die Warteschlange schützt das lokale Zeitfenster zwischen Annahme und Weiterleitung. Sie kann keine Nachricht wiederherstellen, die OpenClaw nie erreicht hat, und keine erneute Serverzustellung über mehrere Verbindungen hinweg deduplizieren.

## Verbindungseinstellungen

| Schlüssel                     | Standardwert                  | Hinweise                                                    |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | keiner (erforderlich)         | Hostname des IRC-Servers                                    |
| `port`                        | `6697` mit TLS, `6667` unverschlüsselt | 1-65535                                                     |
| `tls`                         | `true`                        | Setzen Sie `false` nur bei beabsichtigter Klartextübertragung |
| `nick`                        | keiner (erforderlich)         | Bot-Nick                                                    |
| `username`                    | Nick, andernfalls `openclaw`  | IRC-Benutzername                                            |
| `realname`                    | `OpenClaw`                    | Realname-/GECOS-Feld                                        |
| `password` / `passwordFile`   | keiner                        | Serverpasswort; die Datei muss eine reguläre Datei sein     |
| `channels`                    | keiner                        | Beizutretende Kanäle (`["#openclaw"]`)                      |
| `accounts` / `defaultAccount` | keiner                        | Einrichtung mehrerer Konten; Umgebungsvariablen befüllen nur das Standardkonto |

## Sicherheitsstandardwerte

- IRC verwendet unformatierte TCP-/TLS-Sockets außerhalb des vom OpenClaw-Betreiber verwalteten Forward-Proxy-Routings. Legen Sie in Bereitstellungen, die sämtlichen ausgehenden Datenverkehr durch diesen Forward-Proxy leiten müssen, `channels.irc.enabled=false` fest, sofern direkter ausgehender IRC-Datenverkehr nicht ausdrücklich genehmigt wurde.
- `channels.irc.dmPolicy` verwendet standardmäßig `"pairing"`: Unbekannte Absender von Direktnachrichten erhalten einen Kopplungscode, den Sie mit `openclaw pairing approve irc <code>` genehmigen.
- `channels.irc.groupPolicy` verwendet standardmäßig `"allowlist"`.
- Legen Sie bei `groupPolicy="allowlist"` mit `channels.irc.groups` die zulässigen Kanäle fest.
- Verwenden Sie TLS (`channels.irc.tls=true`), sofern Sie nicht bewusst eine Klartextübertragung akzeptieren.

## Zugriffskontrolle

Für IRC-Kanäle gibt es zwei separate „Schranken“:

1. **Kanalzugriff** (`groupPolicy` + `groups`): ob der Bot überhaupt Nachrichten aus einem Kanal akzeptiert.
2. **Absenderzugriff** (`groupAllowFrom` / kanalspezifisch `groups["#channel"].allowFrom`): wer den Bot innerhalb dieses Kanals auslösen darf.

Konfigurationsschlüssel:

- Zulassungsliste für Direktnachrichten (Absenderzugriff für Direktnachrichten): `channels.irc.allowFrom`
- Zulassungsliste für Gruppenabsender (Absenderzugriff für Kanäle): `channels.irc.groupAllowFrom`
- Kanalspezifische Steuerung (Regeln für Kanal, Absender und Erwähnungen): `channels.irc.groups["#channel"]` mit `requireMention`, `allowFrom`, `enabled`, `tools`, `toolsBySender`, `skills` und `systemPrompt`
- `channels.irc.groupPolicy="open"` erlaubt nicht konfigurierte Kanäle (**standardmäßig weiterhin nur bei Erwähnung**)

Einträge in Zulassungslisten sollten stabile Absenderidentitäten verwenden (`nick!user@host`).
Der Abgleich ausschließlich anhand des Nicks ist veränderlich und nur aktiviert, wenn `channels.irc.dangerouslyAllowNameMatching: true`.

### Häufiger Stolperstein: `allowFrom` gilt für Direktnachrichten, nicht für Kanäle

Wenn Protokolleinträge wie der folgende angezeigt werden:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

...bedeutet dies, dass der Absender für **Gruppen-/Kanalnachrichten** nicht zugelassen war. Beheben Sie dies durch:

- Festlegen von `channels.irc.groupAllowFrom` (global für alle Kanäle) oder
- Festlegen kanalspezifischer Absender-Zulassungslisten: `channels.irc.groups["#channel"].allowFrom`

Beispiel (allen Personen in `#openclaw` die Kommunikation mit dem Bot erlauben):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## Auslösen von Antworten (Erwähnungen)

Selbst wenn ein Kanal zulässig ist (über `groupPolicy` + `groups`) und der Absender zugelassen ist, verwendet OpenClaw in Gruppenkontexten standardmäßig eine **Erwähnungsschranke**. Der Bot gilt als erwähnt, wenn die Nachricht den Nick des verbundenen Bots enthält oder Ihren konfigurierten Erwähnungsmustern entspricht.

Daher werden möglicherweise Protokolleinträge wie `drop channel … (missing-mention)` angezeigt, sofern die Nachricht kein zum Bot passendes Erwähnungsmuster enthält.

Deaktivieren Sie die Erwähnungsschranke für den Kanal, damit der Bot in einem IRC-Kanal **ohne erforderliche Erwähnung** antwortet:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

Oder erlauben Sie **alle** IRC-Kanäle (ohne kanalspezifische Zulassungsliste) und antworten Sie weiterhin ohne Erwähnungen:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## Sicherheitshinweis (für öffentliche Kanäle empfohlen)

Wenn Sie `allowFrom: ["*"]` in einem öffentlichen Kanal zulassen, kann jede Person dem Bot Prompts senden.
Beschränken Sie die Werkzeuge für diesen Kanal, um das Risiko zu verringern.

### Dieselben Werkzeuge für alle Personen im Kanal

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### Unterschiedliche Werkzeuge je Absender (der Besitzer erhält mehr Befugnisse)

Verwenden Sie `toolsBySender`, um auf `"*"` eine strengere Richtlinie und auf Ihren Nick eine weniger strenge Richtlinie anzuwenden:

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

Hinweise:

- Schlüssel für `toolsBySender` sollten explizite Präfixe verwenden (`channel:`, `id:`, `e164:`, `username:`, `name:`). Verwenden Sie für IRC `id:` mit dem Wert der Absenderidentität: `id:alice` oder `id:alice!~alice@203.0.113.7` für einen zuverlässigeren Abgleich.
- Veraltete Schlüssel ohne Präfix werden weiterhin akzeptiert, nur als `id:` abgeglichen und lösen eine Veraltungswarnung aus.
- Die erste passende Absenderrichtlinie wird angewendet; `"*"` dient als Platzhalter-Ausweichregel.

Weitere Informationen zum Gruppenzugriff und zur Erwähnungsschranke sowie zu deren Zusammenspiel finden Sie unter: [/channels/groups](/de/channels/groups).

## NickServ

So identifizieren Sie sich nach dem Verbindungsaufbau bei NickServ:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

Die NickServ-Identifizierung wird standardmäßig immer ausgeführt, wenn ein Passwort festgelegt ist (`enabled` muss nur zum Deaktivieren auf `false` gesetzt werden). `service` verwendet standardmäßig `NickServ`; `passwordFile` ist eine Alternative zum direkt angegebenen `password`.

Optionale einmalige Registrierung beim Verbindungsaufbau (`register: true` erfordert `registerEmail`):

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

Deaktivieren Sie `register`, nachdem der Nick registriert wurde, um wiederholte REGISTER-Versuche zu vermeiden.

## Umgebungsvariablen

Das Standardkonto unterstützt:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (durch Kommas getrennt)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

`IRC_HOST` kann nicht über eine `.env` des Arbeitsbereichs festgelegt werden; siehe [`.env`-Dateien des Arbeitsbereichs](/de/gateway/security).

## Fehlerbehebung

- Wenn der Bot eine Verbindung herstellt, aber nie in Kanälen antwortet, überprüfen Sie `channels.irc.groups` **und** ob die Erwähnungsschranke Nachrichten verwirft (`missing-mention`). Wenn der Bot ohne Ping antworten soll, legen Sie für den Kanal `requireMention:false` fest.
- Wenn die Anmeldung fehlschlägt, überprüfen Sie die Verfügbarkeit des Nicks und das Serverpasswort.
- Wenn TLS in einem benutzerdefinierten Netzwerk fehlschlägt, überprüfen Sie Host, Port und Zertifikatseinrichtung.

## Verwandte Themen

- [Übersicht der Kanäle](/de/channels) — alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) — Authentifizierung von Direktnachrichten und Kopplungsablauf
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungsschranke
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Absicherung
