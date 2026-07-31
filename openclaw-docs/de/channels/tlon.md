---
read_when:
    - Arbeiten an Funktionen für den Tlon-/Urbit-Kanal
summary: Status, Funktionen und Konfiguration der Tlon-/Urbit-Unterstützung
title: Tlon
x-i18n:
    generated_at: "2026-07-26T18:20:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d742628d6cf9aaf82d79a8d96b1685229905e9452c9fc4d3a494d2dee8d69943
    source_path: channels/tlon.md
    workflow: 16
---

Tlon ist ein dezentraler Messenger, der auf Urbit basiert. OpenClaw verbindet sich mit Ihrem Urbit-Ship und
antwortet auf Direktnachrichten und Gruppenchat-Nachrichten. Für Gruppenantworten ist standardmäßig eine @-Erwähnung erforderlich,
zusätzlich gelten Autorisierungsregeln und ein Genehmigungsablauf durch den Eigentümer.

Status: gebündeltes Plugin. Direktnachrichten, Gruppenerwähnungen, Threads, Rich Text, Bildupload/-download und ein
Genehmigungssystem für den Eigentümer werden unterstützt. Reaktionen und Umfragen werden nicht unterstützt.

## Gebündeltes Plugin

Tlon ist in aktuellen OpenClaw-Versionen gebündelt; für paketierte Builds ist keine separate Installation erforderlich.

Installieren Sie bei einem älteren Build oder einer benutzerdefinierten Installation, die es ausschließt, das Paket von npm:

```bash
openclaw plugins install @openclaw/tlon
```

Verwenden Sie den reinen Paketnamen, um dem aktuellen Release-Tag zu folgen. Fixieren Sie eine Version (`@openclaw/tlon@x.y.z`)
nur für reproduzierbare Installationen.

Aus einem lokalen Checkout:

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

Details: [Plugins](/de/tools/plugin)

## Einrichtung

```bash
openclaw channels add --channel tlon --ship ~sampel-palnet --url https://your-ship-host --code lidlut-tabwed-pillex-ridrup
```

Oder bearbeiten Sie die Konfiguration direkt:

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // empfohlen: Ihr Ship, immer autorisiert
    },
  },
}
```

Starten Sie den Gateway nach der direkten Bearbeitung der Konfiguration neu. Senden Sie dem Bot anschließend eine Direktnachricht oder erwähnen Sie ihn mit @ in einem
Gruppenkanal.

## Dauerhaftigkeit eingehender Nachrichten

OpenClaw speichert akzeptierte Tlon-Ereignisse aus Direktnachrichten und Gruppenchats dauerhaft, bevor sie an den Agenten weitergeleitet werden. Ausstehende oder erneut ausführbare Vorgänge überstehen einen Neustart des Gateway, und die Verarbeitung bleibt pro Gruppenkanal oder direktem Kommunikationspartner serialisiert. Stabile Urbit-Nachrichten-IDs unterdrücken außerdem ein erneut zugestelltes Ereignis, solange der zugehörige Warteschlangeneintrag oder der aufbewahrte Abschlussdatensatz vorhanden ist.

Die Zustellung über die Grenze zwischen Warteschlange und Agent erfolgt mindestens einmal: Ein Absturz während der Übergabe kann einen Vorgang erneut ausführen. Agentenaktionen mit externen Nebenwirkungen sollten daher, soweit praktikabel, idempotent bleiben.

## Private Ships/LAN-Ships

OpenClaw blockiert zum Schutz vor SSRF standardmäßig private/interne Hostnamen und IP-Bereiche. Wenn Ihr
Ship in einem privaten Netzwerk ausgeführt wird (localhost, LAN-IP, interner Hostname), müssen Sie dies ausdrücklich zulassen:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
    },
  },
}
```

Dies gilt für Ziele wie `http://localhost:8080`, `http://192.168.x.x:8080` und
`http://my-ship.local:8080`. Aktivieren Sie dies nur für eine Ship-URL, der Sie vertrauen; dadurch wird der SSRF-Schutz
für die HTTP-Anfragen dieses Kontos deaktiviert.

<Note>
`channels.tlon.allowPrivateNetwork` (flacher Schlüssel) wird nicht mehr verwendet. `openclaw doctor --fix` verschiebt ihn automatisch nach
`channels.tlon.network.dangerouslyAllowPrivateNetwork`.
</Note>

## Gruppenkanäle

Heften Sie Kanäle manuell an oder aktivieren Sie die automatische Erkennung:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
      autoDiscoverChannels: true,
    },
  },
}
```

`autoDiscoverChannels` verwendet standardmäßig `false`, wenn der Wert in der Konfiguration nicht festgelegt ist; im Einrichtungsassistenten lautet die
Standardantwort auf die Abfrage „Ja“, und `true` wird ausdrücklich geschrieben. Wenn die Option aktiviert ist, fragt OpenClaw beim Start beigetretene Gruppen ab,
überwacht neue Kanäle, sobald Gruppeneinladungen angenommen werden, und prüft alle 2 Minuten erneut.

## Zugriffskontrolle

Zulassungsliste für Direktnachrichten (leer = keine Direktnachrichten zulässig, außer der Absender ist `ownerShip`):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

Die Gruppenautorisierung verwendet standardmäßig `restricted` pro Kanal. Legen Sie `defaultAuthorizedShips` als
Ausgangsbasis fest und überschreiben Sie den Wert pro Kanal-Nest:

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

Sobald der Bot innerhalb eines Threads geantwortet hat, antwortet er weiterhin auf spätere Nachrichten in diesem Thread,
ohne dass eine weitere Erwähnung erforderlich ist.

Legen Sie `channels.tlon.implicitMentions.threadParticipation: false` fest, um für diese Folgenachrichten
eine neue ausdrückliche Erwähnung zu verlangen. Kontoüberschreibungen verwenden `channels.tlon.accounts.<id>.implicitMentions`. Tlon
erzeugt derzeit keine Fakten vom Typ `replyToBot` oder `quotedBot`, daher haben diese Optionen hier keine Auswirkung.

## Eigentümer- und Genehmigungssystem

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

Das Ship des Eigentümers ist überall autorisiert: Einladungen zu Direktnachrichten werden immer automatisch angenommen, Gruppeneinladungen werden
immer automatisch angenommen und Kanalnachrichten bestehen stets die Autorisierungsprüfung. Das Ship des Eigentümers muss nicht in
`dmAllowlist`, `defaultAuthorizedShips` oder `groupInviteAllowlist` enthalten sein.

Wenn `ownerShip` festgelegt ist, werden nicht autorisierte Anfragen nicht einfach verworfen – sie werden als ausstehende
Genehmigung in die Warteschlange gestellt und dem Eigentümer per Direktnachricht gesendet:

- Direktnachrichtenanfragen von Ships, die nicht in `dmAllowlist` enthalten sind
- Erwähnungen in Kanälen, in denen der Absender die Autorisierungsprüfung nicht besteht
- Gruppeneinladungen von Ships, die nicht in `groupInviteAllowlist` enthalten sind (wenn die automatische Annahme deaktiviert ist oder aktiviert ist, aber der
  Einladende nicht auf der Zulassungsliste steht)

Der Eigentümer antwortet per Direktnachricht, um eine Anfrage zu bearbeiten:

| Antwort des Eigentümers      | Wirkung                                                      |
| ---------------------------- | ------------------------------------------------------------ |
| `approve` / `deny` / `block` | Bearbeitet die neueste ausstehende Genehmigung               |
| `approve <id>` / `deny <id>` | Bearbeitet eine bestimmte Genehmigung anhand ihrer ID        |
| `block`                      | Blockiert das Ship zusätzlich nativ, sodass es keine erneute Verbindung herstellen kann |
| `unblock ~ship`              | Hebt eine native Blockierung auf                             |
| `blocked`                    | Listet die derzeit blockierten Ships auf                    |
| `pending`                    | Listet ausstehende Genehmigungsanfragen auf                  |

Wenn `ownerShip` nicht konfiguriert ist, werden nicht autorisierte Direktnachrichten und Kanalerwähnungen einfach verworfen und protokolliert;
es wird keine Genehmigungsabfrage angezeigt.

## Einstellungen für die automatische Annahme

Nehmen Sie Einladungen zu Direktnachrichten von Ships, die bereits in `dmAllowlist` enthalten sind, automatisch an (das Ship des Eigentümers wird unabhängig
von dieser Option immer automatisch angenommen):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

Nehmen Sie Gruppeneinladungen aus einer Zulassungsliste automatisch an (standardmäßig abweisend: Bei `autoAcceptGroupInvites: true` und
einer leeren `groupInviteAllowlist` wird keine Einladung eines anderen Ships als des Eigentümers angenommen):

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
      groupInviteAllowlist: ["~zod"],
    },
  },
}
```

## Hot-Reload über den Urbit-Einstellungsspeicher

Die meisten der oben genannten Einstellungen (`dmAllowlist`, `groupInviteAllowlist`, `groupChannels`,
`defaultAuthorizedShips`, `autoDiscoverChannels`, `autoAcceptDmInvites`,
`autoAcceptGroupInvites`, `ownerShip`, `showModelSignature`) werden beim ersten Start in den
`%settings`-Agenten des Ships (Desk `moltbot`, Bucket `tlon`) gespiegelt und anschließend live von dort gelesen,
sodass Änderungen über einen Landscape-Client oder die Einstellungsbefehle des gebündelten Skills ohne einen
Neustart des Gateway wirksam werden. `channelRules` und ausstehende Genehmigungen werden dort ebenfalls als JSON dauerhaft gespeichert. Die
Dateikonfiguration bleibt die maßgebliche Quelle für Werte, die nie in den Einstellungsspeicher geschrieben wurden.

## Zustellungsziele (CLI/Cron)

Verwenden Sie diese mit `openclaw message send` oder für die Cron-Zustellung:

- Direktnachricht: `~sampel-palnet` oder `dm/~sampel-palnet`
- Gruppe: `chat/~host-ship/channel` oder `group:~host-ship/channel`

## Gebündelter Skill

Das Plugin enthält [`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill), eine CLI für
direkte Urbit-Operationen, die nach der Installation des Plugins automatisch verfügbar ist:

- **Aktivität**: Erwähnungen, Antworten, ungelesene Nachrichten
- **Kanäle**: auflisten, erstellen, umbenennen
- **Kontakte**: Profile auflisten/abrufen/aktualisieren
- **Gruppen**: erstellen, beitreten, Einladungs-/Anfrageabläufe, Rollen
- **Hooks**: Kanal-Hooks verwalten
- **Nachrichten**: Verlauf, Suche
- **Direktnachrichten**: senden, reagieren, annehmen/ablehnen
- **Beiträge**: reagieren, löschen
- **Notizbuch**: in Tagebuchkanälen veröffentlichen
- **Einstellungen**: Plugin-Konfiguration über den oben genannten Einstellungsspeicher per Hot-Reload aktualisieren

## Funktionen

| Funktion              | Status                                                 |
| --------------------- | ------------------------------------------------------ |
| Direktnachrichten     | Unterstützt                                            |
| Gruppen/Kanäle        | Unterstützt (standardmäßig nur bei Erwähnung)          |
| Threads               | Unterstützt (antwortet nach dem Beitritt weiterhin)    |
| Rich Text             | Markdown wird in das native Format von Tlon konvertiert |
| Bilder                | Eingehend heruntergeladen, ausgehend hochgeladen       |
| Reaktionen            | Nur über den [gebündelten Skill](#bundled-skill)       |
| Umfragen              | Nicht unterstützt                                      |
| Native Befehle        | Standardmäßig nur für den Eigentümer                   |

## Fehlerbehebung

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

Häufige Fehler:

- **Direktnachrichten werden ignoriert**: Der Absender ist nicht in `dmAllowlist` enthalten, und für den Genehmigungsablauf ist kein `ownerShip` konfiguriert.
- **Gruppennachrichten werden ignoriert**: Der Kanal wurde nicht erkannt/angeheftet, oder der Absender besteht die Autorisierungsprüfung nicht und es gibt kein
  `ownerShip`, um eine Genehmigung in die Warteschlange zu stellen.
- **Verbindungsfehler**: Prüfen Sie, ob die Ship-URL erreichbar ist; legen Sie
  `network.dangerouslyAllowPrivateNetwork` für lokale Ships fest.
- **Authentifizierungsfehler**: Anmeldecodes wechseln regelmäßig – kopieren Sie den aktuellen Code von Ihrem Ship.

## Konfigurationsreferenz

Vollständige Konfiguration: [Konfiguration](/de/gateway/configuration)

| Schlüssel                                              | Bedeutung                                                      |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `channels.tlon.enabled`                                | Aktiviert/deaktiviert den Start des Kanals.                    |
| `channels.tlon.ship`                                   | Urbit-Ship-Name des Bots (z. B. `~sampel-palnet`).             |
| `channels.tlon.url`                                    | Ship-URL (z. B. `https://sampel-palnet.tlon.network`).                            |
| `channels.tlon.code`                                   | Anmeldecode des Ships.                                         |
| `channels.tlon.network.dangerouslyAllowPrivateNetwork` | Erlaubt localhost-/LAN-Ship-URLs (ausdrückliche SSRF-Freigabe). |
| `channels.tlon.ownerShip`                              | Ship des Eigentümers: immer autorisiert, empfängt Genehmigungsanfragen. |
| `channels.tlon.dmAllowlist`                            | Ships, die Direktnachrichten senden dürfen (leer = außer dem Eigentümer keine). |
| `channels.tlon.autoAcceptDmInvites`                    | Nimmt Direktnachrichten von Ships in `dmAllowlist` automatisch an. |
| `channels.tlon.autoAcceptGroupInvites`                 | Nimmt Gruppeneinladungen von `groupInviteAllowlist` automatisch an. |
| `channels.tlon.groupInviteAllowlist`                   | Ships, deren Gruppeneinladungen automatisch angenommen werden. |
| `channels.tlon.autoDiscoverChannels`                   | Erkennt beigetretene Gruppenkanäle automatisch (Standard: `false`). |
| `channels.tlon.implicitMentions.threadParticipation`   | Lässt Folgenachrichten in Threads mit Beteiligung die Erwähnungspflicht umgehen. |
| `channels.tlon.groupChannels`                          | Manuell angeheftete Kanal-Nests.                               |
| `channels.tlon.defaultAuthorizedShips`                 | Für alle Kanäle autorisierte Ships (wird verwendet, wenn keine Regel zutrifft). |
| `channels.tlon.authorization.channelRules`             | Autorisierungsmodus und Zulassungsliste pro Kanal-Nest.        |
| `channels.tlon.showModelSignature`                     | Hängt `_[Generated by <model>]_` an Antworten an.                      |
| `channels.tlon.responsePrefix`                         | Statisches Präfix, das ausgehenden Antworten vorangestellt wird. |
| `channels.tlon.accounts.<id>`                          | Zusätzliche benannte Konten (Einrichtungen mit mehreren Ships). |

## Hinweise

- Antworten in Gruppen benötigen eine @-Erwähnung (z. B. `~your-bot-ship`), sofern der Bot diesem Thread nicht bereits beigetreten ist.
- Antworten auf Threads werden im jeweiligen Thread veröffentlicht; außerdem werden dem Bot die letzten 10 Nachrichten aus dem Thread-Kontext
  für den Agenten vorangestellt.
- Rich Text (Fettdruck, Kursivschrift, Code, Überschriften, Listen) wird in das native Format von Tlon konvertiert.
- Wenn eine eingehende Nachricht um eine Zusammenfassung eines Kanals bittet (zum Beispiel „Diesen
  Kanal zusammenfassen“), wird anstelle des normalen Antwortablaufs eine integrierte Zusammenfassung des Verlaufs ausgelöst.

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) — DM-Authentifizierung und Kopplungsablauf
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungsbeschränkung
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Absicherung
