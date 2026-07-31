---
read_when:
    - Sie möchten, dass OpenClaw Direktnachrichten über Nostr empfängt
    - Sie richten dezentrale Nachrichtenübermittlung ein
summary: Nostr-DM-Kanal über NIP-04-verschlüsselte Nachrichten
title: Nostr
x-i18n:
    generated_at: "2026-07-26T17:39:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31fa283f706036a37795ddad71602058ba94388a9cb01044927c4bb2d83ba4a8
    source_path: channels/nostr.md
    workflow: 16
---

Nostr ist ein herunterladbares Kanal-Plugin (`@openclaw/nostr`), mit dem OpenClaw über Nostr-Relays verschlüsselte Direktnachrichten gemäß NIP-04 empfangen und beantworten kann. Ein Konto pro Gateway; nur Direktnachrichten.

## Installation

```bash
openclaw plugins install @openclaw/nostr
```

Verwenden Sie die reine Paketspezifikation, um dem aktuellen offiziellen Release-Tag zu folgen. Fixieren Sie eine exakte Version nur, wenn Sie eine reproduzierbare Installation benötigen.

Aus einem lokalen Checkout (Entwicklungsabläufe):

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

Starten Sie das Gateway nach der Installation oder Aktivierung von Plugins neu. Das Onboarding (`openclaw onboard`) und `openclaw channels add` zeigen Nostr aus dem gemeinsamen Kanalkatalog an, sobald das Plugin installiert ist.

### Nicht interaktive Einrichtung

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

Verwenden Sie `--use-env`, um `NOSTR_PRIVATE_KEY` in der Umgebung zu belassen, statt den Schlüssel in der Konfiguration zu speichern (nur Standardkonto).

## Schnelleinrichtung

1. Generieren Sie bei Bedarf ein Nostr-Schlüsselpaar:

```bash
# Mit nak
nak key generate
```

2. Fügen Sie es der Konfiguration hinzu:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. Exportieren Sie den Schlüssel:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. Starten Sie das Gateway neu.

## Konfigurationsreferenz

| Schlüssel    | Typ      | Standardwert                                | Beschreibung                                             |
| ------------ | -------- | ------------------------------------------- | -------------------------------------------------------- |
| `privateKey` | string   | erforderlich                                | Privater Schlüssel im `nsec`- oder Hexadezimalformat; Secret-Referenzen sind zulässig |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | Relay-URLs (WebSocket)                                   |
| `dmPolicy`   | string   | `pairing`                                   | Zugriffsrichtlinie für Direktnachrichten                  |
| `allowFrom`  | string[] | `[]`                                        | Zulässige öffentliche Absenderschlüssel                  |
| `enabled`    | boolean  | `true`                                      | Kanal aktivieren/deaktivieren                            |
| `name`       | string   | -                                           | Anzeigename                                              |
| `profile`    | object   | -                                           | NIP-01-Profilmetadaten                                   |

## Profilmetadaten

Profildaten werden als NIP-01-`kind:0`-Ereignis veröffentlicht. Sie können sie über die Control UI (Channels -> Nostr -> Profile) verwalten oder direkt in der Konfiguration festlegen.

Beispiel:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "Persönlicher Assistent als Direktnachrichten-Bot",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

Hinweise:

- Profil-URLs müssen `https://` verwenden.
- Beim Import aus Relays werden Felder zusammengeführt und lokale Überschreibungen beibehalten.

## Zugriffskontrolle

### Richtlinien für Direktnachrichten

- **Kopplung** (Standard): Unbekannte Absender erhalten einen Kopplungscode.
- **Zulassungsliste**: Nur öffentliche Schlüssel in `allowFrom` können Direktnachrichten senden.
- **Offen**: öffentlich eingehende Direktnachrichten (erfordert `allowFrom: ["*"]`).
- **Deaktiviert**: Eingehende Direktnachrichten werden ignoriert.

Hinweise zur Durchsetzung:

- Signaturen eingehender Ereignisse werden vor der Prüfung der Absenderrichtlinie und der NIP-04-Entschlüsselung verifiziert, sodass gefälschte Ereignisse frühzeitig abgewiesen werden.
- Kopplungsantworten werden gesendet, ohne den ursprünglichen Inhalt der Direktnachricht zu entschlüsseln oder zu verarbeiten.
- Eingehende Direktnachrichten unterliegen global und pro Absender einer Ratenbegrenzung; übergroße Nutzdaten werden vor der Entschlüsselung verworfen.

### Beispiel für eine Zulassungsliste

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## Schlüsselformate

Akzeptierte Formate:

- **Privater Schlüssel:** `nsec...` oder Hexadezimalwert mit 64 Zeichen
- **Öffentliche Schlüssel (`allowFrom`):** `npub...` oder Hexadezimalwert

## Relays

Standardwerte: `relay.damus.io` und `nos.lol`.

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

Tipps:

- Verwenden Sie 2-3 Relays für Redundanz.
- Vermeiden Sie zu viele Relays (Latenz, Duplizierung).
- Kostenpflichtige Relays können die Zuverlässigkeit verbessern.
- Lokale Relays eignen sich für Tests (`ws://localhost:7777`).

## Protokollunterstützung

| NIP    | Status      | Beschreibung                              |
| ------ | ----------- | ----------------------------------------- |
| NIP-01 | Unterstützt | Grundlegendes Ereignisformat und Profilmetadaten |
| NIP-04 | Unterstützt | Verschlüsselte Direktnachrichten (`kind:4`) |
| NIP-17 | Geplant     | Verpackte Direktnachrichten               |
| NIP-44 | Geplant     | Versionierte Verschlüsselung              |

## Tests

### Lokales Relay

```bash
# strfry starten
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### Manueller Test

1. Notieren Sie den öffentlichen Schlüssel des Bots aus den Gateway-Protokollen oder `openclaw channels status` (Hexadezimalwert; konvertieren Sie ihn bei Bedarf in Ihrem Client in npub).
2. Öffnen Sie einen Nostr-Client (Amethyst, Damus usw.).
3. Senden Sie eine Direktnachricht an den öffentlichen Schlüssel des Bots.
4. Überprüfen Sie die Antwort.

## Fehlerbehebung

### Nachrichten werden nicht empfangen

- Überprüfen Sie, ob der private Schlüssel gültig ist.
- Stellen Sie sicher, dass die Relay-URLs erreichbar sind und `wss://` (oder lokal `ws://`) verwenden.
- Vergewissern Sie sich, dass `enabled` nicht `false` ist.
- Prüfen Sie die Gateway-Protokolle auf Relay-Verbindungsfehler.

### Antworten werden nicht gesendet

- Prüfen Sie, ob das Relay Schreibvorgänge akzeptiert.
- Überprüfen Sie die ausgehende Konnektivität.
- Achten Sie auf Ratenbegrenzungen des Relays.

### Doppelte Antworten

- Dies ist bei der Verwendung mehrerer Relays zu erwarten.
- Nachrichten werden anhand der Ereignis-ID dedupliziert; nur die erste Zustellung löst eine Antwort aus.

## Sicherheit

- Committen Sie niemals private Schlüssel.
- Verwenden Sie Umgebungsvariablen für Schlüssel.
- Erwägen Sie `allowlist` für produktiv eingesetzte Bots.
- Signaturen werden vor der Prüfung der Absenderrichtlinie verifiziert, und die Absenderrichtlinie wird vor der Entschlüsselung durchgesetzt. Dadurch werden gefälschte Ereignisse frühzeitig abgewiesen und unbekannte Absender können keine vollständige kryptografische Verarbeitung erzwingen.

## Einschränkungen (MVP)

- Nur Direktnachrichten (keine Gruppenchats).
- Keine Medienanhänge.
- Nur NIP-04 (NIP-17-Verpackung geplant).

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Kopplung](/de/channels/pairing) — Authentifizierung von Direktnachrichten und Kopplungsablauf
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungsbeschränkung
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Härtung
