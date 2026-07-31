---
read_when:
    - Sie möchten OpenClaw über Twilio mit SMS verbinden
    - Sie müssen einen SMS-Webhook oder eine Zulassungsliste einrichten
summary: Einrichtung des Twilio-SMS-Kanals, Zugriffskontrollen und Webhook-Konfiguration
title: SMS
x-i18n:
    generated_at: "2026-07-26T17:40:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99a76b2f2d66858f8eb699939084104e620af9bc024053bbe1c1d7350530bff0
    source_path: channels/sms.md
    workflow: 16
---

OpenClaw empfängt und sendet SMS über eine Twilio-Telefonnummer oder einen Messaging Service. Der Gateway registriert eine eingehende Webhook-Route (standardmäßig `/webhooks/sms`), validiert standardmäßig Twilio-Anfragesignaturen und sendet Antworten über die Messages API von Twilio zurück.

Status: offizielles Plugin, separat installiert. Nur Text: keine MMS/Medien, nur Direktnachrichten.

<CardGroup cols={3}>
  <Card title="Kopplung" icon="link" href="/de/channels/pairing">
    Die standardmäßige DM-Richtlinie für SMS ist die Kopplung.
  </Card>
  <Card title="Gateway-Sicherheit" icon="shield" href="/de/gateway/security">
    Prüfen Sie die Webhook-Exposition und die Zugriffskontrollen für Absender.
  </Card>
  <Card title="Fehlerbehebung für Kanäle" icon="wrench" href="/de/channels/troubleshooting">
    Kanalübergreifende Diagnose- und Reparaturleitfäden.
  </Card>
</CardGroup>

## Voraussetzungen

Sie benötigen:

- Das offizielle SMS-Plugin, installiert mit `openclaw plugins install @openclaw/sms`.
- Ein Twilio-Konto mit einer SMS-fähigen Telefonnummer oder einem Twilio Messaging Service.
- Die Twilio Account SID und das Auth Token.
- Eine öffentliche HTTPS-URL, über die Ihr OpenClaw Gateway erreichbar ist.
- Eine Auswahl für die Absenderrichtlinie: `pairing` (Standard) für die private Nutzung, `allowlist` für vorab genehmigte Telefonnummern oder `open` nur für einen absichtlich öffentlichen SMS-Zugriff.

Eine Twilio-Nummer kann sowohl SMS als auch [Sprachanrufe](/de/plugins/voice-call) unterstützen, wenn sie über beide Funktionen verfügt. Der SMS-Webhook und der Sprach-Webhook werden in Twilio separat konfiguriert und verwenden separate Gateway-Pfade; diese Seite behandelt ausschließlich den SMS-Webhook.

## Schnelleinrichtung

<Steps>
  <Step title="Plugin installieren">
    ```bash
    openclaw plugins install @openclaw/sms
    ```
  </Step>
  <Step title="Twilio-Absender erstellen oder auswählen">
    Öffnen Sie in Twilio **Phone Numbers > Manage > Active numbers** und wählen Sie eine SMS-fähige Nummer aus. Speichern Sie:

    - Account SID, zum Beispiel `ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
    - Auth Token
    - Absendertelefonnummer, zum Beispiel `+15551234567`

    Wenn Sie anstelle einer festen Absendernummer einen Messaging Service verwenden, speichern Sie die Messaging Service SID, zum Beispiel `MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

  </Step>

  <Step title="SMS-Kanal konfigurieren">

Speichern Sie Folgendes als `sms.patch.json5` und ändern Sie die Platzhalter:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Wenden Sie die Konfiguration an:

```bash
openclaw config patch --file ./sms.patch.json5 --dry-run
openclaw config patch --file ./sms.patch.json5
```

  </Step>

  <Step title="Twilio auf den Gateway-Webhook verweisen">
    Öffnen Sie in den Einstellungen der Twilio-Telefonnummer **Messaging** und setzen Sie **A message comes in** auf:

```text
https://gateway.example.com/webhooks/sms
```

    Verwenden Sie HTTP `POST`. Der standardmäßige lokale Pfad ist `/webhooks/sms`; ändern Sie `channels.sms.webhookPath`, wenn Sie eine andere Route benötigen.

  </Step>

  <Step title="Exakten SMS-Webhook-Pfad verfügbar machen">
    Ihre öffentliche URL muss den SMS-Pfad an den Gateway-Prozess weiterleiten (Standardport `18789`). Wenn Sie Tailscale Funnel für lokale Tests verwenden, geben Sie `/webhooks/sms` ausdrücklich frei:

```bash
tailscale funnel --bg --set-path /webhooks/sms http://127.0.0.1:<gateway-port>/webhooks/sms
tailscale funnel status
```

    Sprachanrufe und SMS verwenden separate Webhook-Pfade. Wenn dieselbe Twilio-Nummer beide verarbeitet, lassen Sie beide Routen in Twilio und in Ihrem Tunnel konfiguriert.

  </Step>

  <Step title="Gateway starten und ersten Absender genehmigen">

```bash
openclaw gateway
```

Senden Sie eine SMS an die Twilio-Nummer. Die erste Nachricht erstellt eine Kopplungsanfrage. Genehmigen Sie sie:

```bash
openclaw pairing list sms
openclaw pairing approve sms <CODE>
```

    Kopplungscodes laufen nach 1 Stunde ab.

  </Step>
</Steps>

## Konfigurationsbeispiele

Alle Schlüssel befinden sich unter `channels.sms` (und je Konto unter `channels.sms.accounts.<id>`):

| Schlüssel                                | Standard        | Zweck                                                               |
| ---------------------------------------- | --------------- | ------------------------------------------------------------------- |
| `enabled`                               | `true`          | Kanal oder Konto aktivieren beziehungsweise deaktivieren.           |
| `accountSid`                            | —               | Twilio Account SID (`AC...`).                                       |
| `authToken`                             | —               | Twilio Auth Token; Klartextzeichenfolge oder SecretRef.             |
| `fromNumber`                            | —               | Absendernummer im E.164-Format.                                     |
| `messagingServiceSid`                   | —               | Messaging Service SID (`MG...`), die verwendet wird, wenn kein `fromNumber` aufgelöst wird. |
| `defaultTo`                             | —               | Standardziel, wenn ein Sendeablauf kein ausdrückliches Ziel angibt. |
| `webhookPath`                           | `/webhooks/sms` | Gateway-HTTP-Pfad für eingehende Twilio-Webhooks.                   |
| `publicWebhookUrl`                      | —               | In Twilio konfigurierte öffentliche URL; für die Signaturvalidierung erforderlich. |
| `dangerouslyDisableSignatureValidation` | `false`         | `X-Twilio-Signature`-Prüfungen überspringen; nur für lokale Tunneltests.        |
| `dmPolicy`                              | `"pairing"`     | `pairing`, `allowlist`, `open` oder `disabled`.                      |
| `allowFrom`                             | `[]`            | Zulässige Absendernummern im E.164-Format oder `"*"` mit `dmPolicy: "open"`.  |
| `textChunkLimit`                        | `1500`          | Maximale Zeichenanzahl pro ausgehendem SMS-Abschnitt.                |
| `accounts`, `defaultAccount`            | —               | Zuordnung mehrerer Konten und ID des Standardkontos.                |

### Konfigurationsdatei

Verwenden Sie die Einrichtung per Konfigurationsdatei, wenn die Kanaldefinition zusammen mit der Gateway-Konfiguration übertragen werden soll:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

### Umgebungsvariablen

Umgebungsvariablen gelten nur für das Standardkonto; Konfigurationswerte haben Vorrang vor Umgebungswerten.

| Variable                                        | Entspricht                                         |
| ----------------------------------------------- | -------------------------------------------------- |
| `TWILIO_ACCOUNT_SID`                            | `accountSid`                                       |
| `TWILIO_AUTH_TOKEN`                             | `authToken`                                        |
| `TWILIO_PHONE_NUMBER` (Alias `TWILIO_SMS_FROM`) | `fromNumber`                                       |
| `TWILIO_MESSAGING_SERVICE_SID`                  | `messagingServiceSid`                              |
| `SMS_PUBLIC_WEBHOOK_URL`                        | `publicWebhookUrl`                                 |
| `SMS_WEBHOOK_PATH`                              | `webhookPath`                                      |
| `SMS_ALLOWED_USERS`                             | `allowFrom` (durch Kommas getrennt)               |
| `SMS_TEXT_CHUNK_LIMIT`                          | `textChunkLimit`                                   |
| `SMS_DANGEROUSLY_DISABLE_SIGNATURE_VALIDATION`  | `dangerouslyDisableSignatureValidation` (`"true"`) |

```bash
export TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export TWILIO_AUTH_TOKEN="<twilio-auth-token>"
export TWILIO_PHONE_NUMBER="+15551234567"
export SMS_PUBLIC_WEBHOOK_URL="https://gateway.example.com/webhooks/sms"
```

Aktivieren Sie anschließend den Kanal in der Konfiguration:

```json5
{
  channels: {
    sms: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

### SecretRef-Auth-Token

`authToken` kann eine SecretRef (`source: "env" | "file" | "exec"`) sein. Verwenden Sie diese Option, wenn der Gateway das Twilio Auth Token über die Secrets-Laufzeit von OpenClaw auflösen soll, anstatt es als Klartextkonfiguration zu speichern:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: { source: "env", provider: "default", id: "TWILIO_AUTH_TOKEN" },
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Die referenzierte Umgebungsvariable oder der Secret-Provider muss für die Gateway-Laufzeit sichtbar sein. Starten Sie verwaltete Gateway-Prozesse nach Änderungen an den Umgebungsvariablen des Hosts neu.

### Messaging-Service-Absender

Verwenden Sie `messagingServiceSid` anstelle von `fromNumber`, wenn Twilio den Absender über einen Messaging Service auswählen soll:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      messagingServiceSid: "MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

Wenn nach der Auflösung von Konfiguration und Umgebungsvariablen sowohl `fromNumber` als auch `messagingServiceSid` vorhanden sind, wird `fromNumber` verwendet.

### Standardziel für ausgehende Nachrichten

Legen Sie `defaultTo` fest, wenn Automatisierungen oder vom Agenten initiierte Zustellungen ein Standardziel verwenden sollen, falls ein Sendeablauf kein ausdrückliches Ziel angibt:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      defaultTo: "+15557654321",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
    },
  },
}
```

## Zugriffskontrolle

`channels.sms.dmPolicy` steuert den direkten SMS-Zugriff:

- `pairing` (Standard): Unbekannte Absender erhalten einen Kopplungscode; genehmigen Sie ihn mit `openclaw pairing approve sms <CODE>`.
- `allowlist`: Nur Absender in `allowFrom` werden verarbeitet. Ein leeres `allowFrom` weist jeden Absender ab (der Gateway protokolliert beim Start eine Warnung).
- `open`: Die Konfigurationsvalidierung erfordert, dass `allowFrom` den Wert `"*"` enthält. Ohne den Platzhalter können nur aufgeführte Nummern chatten.
- `disabled`: Alle eingehenden DMs werden verworfen.

Einträge in `allowFrom` sollten Telefonnummern im E.164-Format sein, beispielsweise `+15551234567`. Die Präfixe `sms:` und `twilio-sms:` werden akzeptiert und normalisiert. Bevorzugen Sie für einen privaten Assistenten `dmPolicy: "allowlist"` mit ausdrücklich angegebenen Telefonnummern:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "allowlist",
      allowFrom: ["+15557654321"],
    },
  },
}
```

## SMS senden

Wenn der SMS-Kanal ausgewählt ist, akzeptieren Ziele reine E.164-Nummern oder das Präfix `sms:`:

```bash
openclaw message send --channel sms --target sms:+15551234567 --message "hello"
```

Bei impliziter Kanalauswahl wählt das Präfix `twilio-sms:` diesen Kanal aus, ohne das Dienstpräfix `sms:` zu übernehmen, das iMessage zur Auswahl der SMS-Zustellung über den Mobilfunkanbieter für seine eigenen Ziele verwendet:

```bash
openclaw message send --target twilio-sms:+15551234567 --message "hello"
```

Die CLI erfordert ein ausdrückliches `--target`. `defaultTo` ist für Automatisierungen und vom Agenten initiierte Zustellpfade vorgesehen, bei denen das Ziel aus der Kanalkonfiguration aufgelöst werden kann.

Antworten des Agenten auf eingehende SMS-Unterhaltungen werden automatisch über den konfigurierten Twilio-Absender an den Absender zurückgesendet.

Die SMS-Ausgabe besteht aus reinem Text. OpenClaw entfernt Markdown, vereinfacht mit Code-Fences umschlossene Codeblöcke, schreibt Links als `label (url)` um und teilt lange Antworten vor dem Versand über Twilio in Abschnitte mit höchstens `textChunkLimit` Zeichen (Standardwert: 1500) auf.

## Einrichtung überprüfen

Nach dem Start des Gateway:

1. Vergewissern Sie sich, dass das Gateway-Protokoll die SMS-Webhook-Route anzeigt.
2. Führen Sie eine Twilio-seitige Prüfung aus (überprüft die konfigurierte Twilio-Webhook-URL/-Methode und aktuelle Fehler bei eingehenden Nachrichten):

```bash
openclaw channels capabilities --channel sms
openclaw channels status --channel sms --probe --json
```

3. Senden Sie von Ihrem Telefon eine SMS an die Twilio-Nummer.
4. Führen Sie `openclaw pairing list sms` aus.
5. Genehmigen Sie den Kopplungscode mit `openclaw pairing approve sms <CODE>`.
6. Senden Sie eine weitere SMS und vergewissern Sie sich, dass der Agent antwortet.

Verwenden Sie für Tests, die nur ausgehende Nachrichten betreffen:

```bash
openclaw message send --channel sms --target sms:+15557654321 --message "OpenClaw SMS test"
```

### Ende-zu-Ende-Test über macOS iMessage/SMS

Auf einem Mac, der über Nachrichten Mobilfunk-SMS senden kann, können Sie `imsg` verwenden, um die Absenderseite zu steuern, ohne Ihr Telefon zu verwenden:

```bash
imsg send --to "+15551234567" --service sms --text "OpenClaw SMS E2E $(date -u +%Y%m%dT%H%M%SZ)" --json
openclaw pairing list sms
openclaw pairing approve sms <CODE>
imsg send --to "+15551234567" --service sms --text "reply exactly SMS pong" --json
```

Die erste Nachricht sollte eine Kopplungsanfrage erstellen. Die zweite Nachricht sollte die Antwort des Agenten über Twilio empfangen.

## Webhook-Sicherheit

Standardmäßig validiert OpenClaw `X-Twilio-Signature` mithilfe von `publicWebhookUrl` und `authToken`. Der Endpunktteil von `publicWebhookUrl` muss Byte für Byte mit der in Twilio konfigurierten URL übereinstimmen, einschließlich Schema, Host, Pfad und Abfragezeichenfolge. OpenClaw schließt Twilio-[Verbindungsüberschreibungs](https://www.twilio.com/docs/usage/webhooks/webhooks-connection-overrides)-Fragmente (`#...`) von der Signaturberechnung aus, wie von Twilio vorgeschrieben.

Die Webhook-Route erzwingt unabhängig von der Signaturvalidierung außerdem:

- Nur `POST`.
- Budget für fehlgeschlagene Anfragen von 300 Anfragen pro Minute und SMS-Konto, Webhook-Route sowie aufgelöster Clientadresse. Alle Anfragen werden auf dieses Budget angerechnet, HTTP 429 wird jedoch erst angewendet, nachdem das Parsen des Anfragetexts, die Twilio-Validierung oder der Abgleich der AccountSid einer Anfrage fehlgeschlagen ist.
- Ratenbegrenzung für weiterleitbare Rückrufe von 30 akzeptierten Rückrufen pro Minute und SMS-Konto, Webhook-Route sowie aufgelöster Clientadresse, nachdem diese Prüfungen erfolgreich waren (darüber HTTP 429). Wenn die Signaturvalidierung deaktiviert ist, stellt dieses Limit von 30 pro Minute die Obergrenze für nicht authentifizierte Weiterleitungen dar.
- Clientadressen werden anhand der gemeinsamen Regeln des Gateway für vertrauenswürdige Proxys aufgelöst. Wenn `gateway.trustedProxies` den Reverse-Proxy enthält, der Twilio-Rückrufe weiterleitet, verwendet OpenClaw für diese Begrenzungen die weitergeleitete Clientadresse; andernfalls wird auf die direkte Socketadresse zurückgegriffen.
- `AccountSid` in den Nutzdaten muss mit dem konfigurierten `accountSid` übereinstimmen (andernfalls HTTP 403).
- Wiederholte `MessageSid`-Werte werden 10 Minuten lang dedupliziert.
- Der Wiederholungscache jedes SMS-Kontos bewahrt bis zu 10.000 aktive Nachrichten-SIDs auf. Wenn alle Plätze aktiv sind, werden neue Webhooks für dieses Konto mit HTTP 429 und einem `Retry-After`-Header abgelehnt, bis der älteste Platz abläuft.
- Anfragetexte über 32 KB werden abgelehnt.

Twilio wiederholt HTTP 429 standardmäßig nicht und dokumentiert keine Unterstützung für `Retry-After`. Die Verbindungsüberschreibungen `#rp=4xx` und `#rp=all` aktivieren Wiederholungsversuche bei 4xx-Antworten, Twilio begrenzt die gesamte Wiederholungstransaktion jedoch auf 15 Sekunden. Daher können die Wiederholungsversuche abgeschlossen sein, bevor ein Platz im Wiederholungscache abläuft. Konfigurieren Sie eine Fallback-URL, wenn ein anderer Handler fehlgeschlagene Zustellungen empfangen muss; behandeln Sie eine 429-Antwort als sicher ablehnende Zurückweisung und nicht als zuverlässigen Rückstau.

Nur für lokale Tunneltests können Sie Folgendes festlegen:

```json5
{
  channels: {
    sms: {
      dangerouslyDisableSignatureValidation: true,
    },
  },
}
```

Verwenden Sie auf einem öffentlichen Gateway keine deaktivierte Signaturvalidierung.

## Konfiguration mehrerer Konten

Verwenden Sie `accounts`, wenn Sie mehr als eine Twilio-Nummer betreiben:

```json5
{
  channels: {
    sms: {
      accounts: {
        support: {
          enabled: true,
          accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          authToken: "twilio-auth-token",
          fromNumber: "+15551234567",
          publicWebhookUrl: "https://gateway.example.com/webhooks/sms/support",
          webhookPath: "/webhooks/sms/support",
          dmPolicy: "allowlist",
          allowFrom: ["+15557654321"],
        },
      },
    },
  },
}
```

Jedes Konto muss einen eigenen `webhookPath` verwenden; das Gateway verweigert die Registrierung einer Webhook-Route, deren Pfad bereits einem anderen Konto zugeordnet ist. Die Umgebungs-Fallbacks `TWILIO_*`/`SMS_*` gelten nur für das Standardkonto; legen Sie `defaultAccount` fest, um das Standardkonto zu ändern.

## Fehlerbehebung

### Twilio gibt 403 zurück oder OpenClaw lehnt den Webhook ab

Prüfen Sie, ob `publicWebhookUrl` exakt mit der in Twilio konfigurierten URL übereinstimmt, einschließlich Schema, Host, Pfad und Abfragezeichenfolge. Twilio signiert die öffentliche URL-Zeichenfolge, sodass Proxy-Umschreibungen und alternative Hostnamen die Signaturvalidierung beeinträchtigen können.

Eine 403-Antwort mit `Invalid account` bedeutet, dass `AccountSid` in den eingehenden Nutzdaten nicht mit dem konfigurierten `accountSid` übereinstimmt; prüfen Sie, ob der Webhook auf das Konto verweist, dem die Nummer gehört.

### Es erscheint keine Kopplungsanfrage

Prüfen Sie die **Messaging**-Webhook-URL und -Methode der Twilio-Nummer. Sie muss auf die SMS-Webhook-URL verweisen und `POST` verwenden. Vergewissern Sie sich außerdem, dass das Gateway über das öffentliche Internet oder Ihren Tunnel erreichbar ist.

Wenn das Twilio-Nachrichtenprotokoll den Fehler `11200` anzeigt, hat Twilio die eingehende SMS akzeptiert, konnte Ihren Webhook jedoch nicht erreichen. Prüfen Sie Folgendes:

- Twilio **Messaging > A message comes in** verweist auf `publicWebhookUrl`.
- Die Methode lautet `POST`.
- Der Tunnel oder Reverse-Proxy stellt exakt `webhookPath` bereit; führen Sie für Tailscale Funnel `tailscale funnel status` aus und vergewissern Sie sich, dass `/webhooks/sms` aufgeführt ist.
- `publicWebhookUrl` verwendet dasselbe Schema, denselben Host, Pfad und dieselbe Abfragezeichenfolge, die Twilio sendet, damit die Signaturvalidierung die signierte URL reproduzieren kann.

`openclaw channels status --channel sms --probe` zeigt sowohl nicht übereinstimmende Twilio-Webhook-Einstellungen als auch aktuelle `11200`-Fehler an.

### Ausgehende Sendungen schlagen fehl

Vergewissern Sie sich, dass `accountSid`, `authToken` und entweder `fromNumber` oder `messagingServiceSid` aufgelöst werden. Wenn Sie ein Twilio-Testkonto verwenden, muss die Zielnummer möglicherweise in Twilio verifiziert werden, bevor ausgehende SMS gesendet werden können.

### Nachrichten treffen ein, aber der Agent antwortet nicht

Prüfen Sie `dmPolicy` und `allowFrom`. Bei der standardmäßigen `pairing`-Richtlinie muss der Absender genehmigt sein, bevor normale Agenteninteraktionen verarbeitet werden.
