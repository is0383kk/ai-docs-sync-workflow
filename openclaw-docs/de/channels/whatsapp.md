---
read_when:
    - Arbeiten am Verhalten des WhatsApp-/Web-Kanals oder am Posteingangs-Routing
summary: Unterstützung für den WhatsApp-Kanal, Zugriffskontrollen, Zustellverhalten und Betrieb
title: WhatsApp
x-i18n:
    generated_at: "2026-07-26T17:40:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

Status: produktionsbereit über WhatsApp Web (Baileys). Der Gateway verwaltet die verknüpfte(n) Sitzung(en); es gibt keinen separaten Twilio-WhatsApp-Kanal.

## Installation

`openclaw onboard` und `openclaw channels add --channel whatsapp` fordern Sie zur Installation des Plugins auf, wenn Sie es zum ersten Mal auswählen; `openclaw channels login --channel whatsapp` bietet denselben Installationsablauf an, wenn das Plugin fehlt. Entwicklungs-Checkouts verwenden den lokalen Plugin-Pfad; stabile/Beta-Installationen installieren zuerst `@openclaw/whatsapp` aus ClawHub und greifen ersatzweise auf npm zurück. Die WhatsApp-Laufzeitumgebung wird außerhalb des zentralen OpenClaw-npm-Pakets ausgeliefert, daher verbleiben ihre Laufzeitabhängigkeiten beim externen Plugin. Manuelle Installation:

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

Verwenden Sie das reine npm-Paket (`@openclaw/whatsapp`) nur als Registry-Ausweichlösung; legen Sie eine exakte Version nur für eine reproduzierbare Installation fest.

<CardGroup cols={3}>
  <Card title="Kopplung" icon="link" href="/de/channels/pairing">
    Die standardmäßige DM-Richtlinie ist die Kopplung für unbekannte Absender.
  </Card>
  <Card title="Kanal-Fehlerbehebung" icon="wrench" href="/de/channels/troubleshooting">
    Kanalübergreifende Diagnose- und Reparaturleitfäden.
  </Card>
  <Card title="Gateway-Konfiguration" icon="settings" href="/de/gateway/configuration">
    Vollständige Muster und Beispiele für die Kanalkonfiguration.
  </Card>
</CardGroup>

## Schnelleinrichtung

<Steps>
  <Step title="Zugriffsrichtlinie konfigurieren">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="WhatsApp verknüpfen (QR-Code)">

```bash
openclaw channels login --channel whatsapp
```

    Die Anmeldung erfolgt ausschließlich per QR-Code. Stellen Sie auf entfernten oder monitorlosen Hosts vor Beginn der Anmeldung sicher, dass der aktuelle QR-Code zuverlässig an das Telefon übermittelt werden kann; im Terminal dargestellte QR-Codes, Screenshots oder Chat-Anhänge können während der Übermittlung ablaufen.

    Für ein bestimmtes Konto:

```bash
openclaw channels login --channel whatsapp --account work
```

    So binden Sie vor der Anmeldung ein vorhandenes/benutzerdefiniertes Authentifizierungsverzeichnis ein:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Gateway starten">

```bash
openclaw gateway
```

  </Step>

  <Step title="Erste DM-Zugriffsanfrage genehmigen (Kopplungsmodus)">

    Öffnen Sie **Settings → Channels → DM access requests**, suchen Sie das WhatsApp-Konto
    und genehmigen Sie den Absender. Wenn Sie die CLI bevorzugen:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    DM-Zugriffsanfragen laufen nach 1 Stunde ab; pro Konto sind maximal 3 ausstehende
    Anfragen zulässig. Diese Genehmigung ist vom WhatsApp-Anmelde-QR-Code getrennt, mit dem
    das Konto selbst verknüpft wird.

  </Step>
</Steps>

<Note>
Eine separate WhatsApp-Nummer wird empfohlen (Einrichtung und Metadaten sind dafür optimiert), Konfigurationen mit persönlicher Nummer bzw. Selbst-Chat werden jedoch vollständig unterstützt.
</Note>

## Bereitstellungsmuster

<AccordionGroup>
  <Accordion title="Dedizierte Nummer (empfohlen)">
    - separate WhatsApp-Identität für OpenClaw
    - klarere DM-Zulassungslisten und Routing-Grenzen
    - geringeres Risiko von Verwechslungen bei Selbst-Chats

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Ausweichlösung mit persönlicher Nummer">
    Das Onboarding unterstützt den Modus mit persönlicher Nummer und schreibt eine für Selbst-Chats geeignete Basiskonfiguration: `dmPolicy: "allowlist"`, `allowFrom` einschließlich Ihrer eigenen Nummer, `selfChatMode: true`. Die Laufzeitschutzmechanismen für Selbst-Chats richten sich nach der verknüpften eigenen Nummer sowie nach `allowFrom`.
  </Accordion>
</AccordionGroup>

## Laufzeitmodell

- Der Gateway verwaltet den WhatsApp-Socket und die Wiederverbindungsschleife.
- Ein Watchdog überwacht zwei Signale unabhängig voneinander: die rohe WhatsApp-Web-Transportaktivität und die Aktivität der Anwendungsnachrichten. Eine ruhige, aber verbundene Sitzung wird nicht neu gestartet, nur weil kürzlich keine Nachricht eingegangen ist; eine Wiederverbindung wird nur erzwungen, wenn für ein festes internes Zeitfenster (nicht vom Benutzer konfigurierbar) keine Transport-Frames mehr eingehen oder Anwendungsnachrichten länger als das Vierfache des normalen Nachrichten-Timeouts ausbleiben. Direkt nach einer Wiederverbindung für eine kürzlich aktive Sitzung verwendet dieses erste Zeitfenster statt des vierfachen Fensters den kürzeren normalen Nachrichten-Timeout. OpenClaw kann automatisch auf Offline-Nachrichten antworten, die Baileys früh während dieser Wiederverbindung zustellt, begrenzt durch die Lebensdauer der Deduplizierung eingehender Nachrichten-IDs; beim ersten Start bleibt die kurze Schutzfrist gegen veraltete Verlaufsnachrichten bestehen.
- Ausgehende Sendungen erfordern einen aktiven WhatsApp-Listener für das Zielkonto; andernfalls schlagen sie sofort fehl.
- Gruppensendungen fügen native Erwähnungsmetadaten für `@+<digits>`- und `@<digits>`-Token (in Text und Medienbeschriftungen) hinzu, wenn das Token mit aktuellen Teilnehmermetadaten übereinstimmt, einschließlich LID-basierter Gruppen.
- Status- und Broadcast-Chats (`@status`, `@broadcast`) werden ignoriert.
- Direktchats verwenden DM-Sitzungsregeln (`session.dmScope`; der Standardwert `main` führt DMs in der Hauptsitzung des Agenten zusammen). Gruppensitzungen werden pro JID isoliert (`agent:<agentId>:whatsapp:group:<jid>`).
- WhatsApp Channels/Newsletters können über ihre native `@newsletter`-JID explizite ausgehende Ziele sein und verwenden Kanal-Sitzungsmetadaten (`agent:<agentId>:whatsapp:channel:<jid>`) anstelle der DM-Semantik.
- Der WhatsApp-Web-Transport berücksichtigt die standardmäßigen Proxy-Umgebungsvariablen auf dem Gateway-Host (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, Varianten in Kleinschreibung). Bevorzugen Sie eine Proxy-Konfiguration auf Hostebene gegenüber kanalspezifischen Einstellungen.

## Aktuellen Anfragenden mit MeowCaller anrufen (experimentell)

Das Plugin kann `whatsapp_call` in von WhatsApp stammenden Agentendurchläufen bereitstellen. Es verwendet [MeowCaller](https://github.com/purpshell/meowcaller), um den aktuell autorisierten Anfragenden über WhatsApp anzurufen und nach der Annahme eine OpenClaw-TTS-Nachricht abzuspielen. Das Tool besitzt keinen Parameter für die Zielnummer, sodass ein Prompt den Anruf nicht umleiten kann. Standardmäßig deaktiviert.

<Warning>
MeowCaller ist experimentell, verfügt über keine mit einem Tag versehene Version und verwendet eine separat gekoppelte Sitzung eines verknüpften whatsmeow-Geräts – die Baileys-Anmeldedaten des Plugins können nicht wiederverwendet werden. Durch die Kopplung wird demselben WhatsApp-Konto ein weiteres verknüpftes Gerät hinzugefügt; scannen Sie mit der von OpenClaw verwendeten Identität. Der Modus mit persönlicher Nummer bzw. Selbst-Chat kann sich nicht selbst anrufen; verwenden Sie eine dedizierte OpenClaw-Nummer, um Ihre persönliche Nummer anzurufen.
</Warning>

<Steps>
  <Step title="Experimentelle Anrufe aktivieren">

    Fügen Sie `actions.calls: true` zur WhatsApp-Kanalkonfiguration hinzu und starten Sie den Gateway neu:

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    Wenn der Wert fehlt oder `false` lautet, stellt OpenClaw das Tool `whatsapp_call` nicht bereit.

  </Step>

  <Step title="Geprüfte MeowCaller-CLI installieren">

    Der Adapter erwartet eine ausführbare Datei `meowcaller` im `PATH` des Gateway-Hosts. Bis [MeowCaller-PR #7](https://github.com/purpshell/meowcaller/pull/7) zusammengeführt ist, erstellen Sie den geprüften Branch:

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    Stellen Sie sicher, dass `$HOME/.local/bin` im `PATH` des Gateway-Dienstes enthalten ist. Diese Revision besitzt explizite `pair`- und reine Sende-Befehle für `notify`; `notify` öffnet weder Mikrofon, Lautsprecher oder Videogerät noch eine Diagnoseaufzeichnung. Verwenden Sie nicht ersatzweise den Befehl `play` der vorgelagerten Beispiel-CLI.

  </Step>

  <Step title="Verknüpftes MeowCaller-Gerät koppeln">

    Bitten Sie den WhatsApp-Agenten, die Anrufeinrichtung zu prüfen (die Statusaktion `whatsapp_call` meldet das kontospezifische Statusverzeichnis und den Kopplungsbefehl). Für das Standardkonto:

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    Führen Sie dies interaktiv aus, scannen Sie den QR-Code über **WhatsApp > Linked devices** und warten Sie auf `MeowCaller linked device ready`. Halten Sie `wa-voip.db` geheim – dies ist die MeowCaller-Sitzung. Nicht standardmäßige Konten erhalten über die Statusaktion einen eigenen Speicherpfad; führen Sie unter Windows den entsprechenden PowerShell-Befehl aus.

  </Step>

  <Step title="TTS konfigurieren und über WhatsApp anrufen">

    Konfigurieren Sie einen telefoniefähigen [TTS-Provider](/de/tools/tts), starten Sie den Gateway neu und senden Sie anschließend eine Anfrage wie `Call me and say the build finished.` Das Tool ermittelt den Absender aus dem vertrauenswürdigen eingehenden Kontext, synthetisiert eine temporäre private WAV-Datei, führt MeowCaller für ein begrenztes Anruffenster aus und löscht anschließend die Audiodatei. OpenClaw übergibt den Speicher des Kontos explizit, wartet nach Annahme/Wiedergabe/Auflegen auf einen Exit-Status von null und behandelt einen Timeout oder einen Exit-Status ungleich null als fehlgeschlagenen Tool-Aufruf.

  </Step>
</Steps>

Einschränkungen: ausschließlich ausgehende Einzel-Audioanrufe, keine beliebigen Zielnummern, keine gemeinsame Authentifizierung mit der Chatverbindung, keine Selbstanrufe im Modus mit persönlicher Nummer bzw. Selbst-Chat, synthetisiertes Audio auf 60 Sekunden begrenzt, keine Empfangsbestätigung für die Hörbarkeit auf dem Mobilgerät über den Abschluss von Annahme/Wiedergabe/Auflegen durch MeowCaller hinaus, und OpenClaw beendet den Begleitprozess nach einem begrenzten Zeitfenster von 115–175 Sekunden (das die Verbindungs-, Annahme-, Wiedergabe- und Beendigungsphasen von MeowCaller abdeckt).

## Genehmigungsaufforderungen

WhatsApp kann Ausführungs- und Plugin-Genehmigungsaufforderungen als `👍`-/`👎`-Reaktionen darstellen, gesteuert durch die übergeordnete Konfiguration für die Weiterleitung von Genehmigungen:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` und `approvals.plugin` sind unabhängig voneinander; die Aktivierung von WhatsApp als Kanal verknüpft lediglich den Transport und sendet nichts, sofern die entsprechende Genehmigungsfamilie nicht aktiviert und dorthin weitergeleitet wird. Der Sitzungsmodus stellt native Emoji-Genehmigungen nur für Genehmigungen bereit, die aus WhatsApp stammen. Der Zielmodus verwendet die gemeinsame Weiterleitungspipeline für explizite Ziele und erzeugt keine separate Auffächerung auf Genehmiger-DMs.

WhatsApp-Genehmigungsreaktionen erfordern explizite Genehmiger in `allowFrom` (oder `"*"`). `defaultTo` legt gewöhnliche standardmäßige Nachrichtenziele fest, keine Genehmigerliste. Manuelle `/approve`-Befehle durchlaufen vor der Genehmigungsauflösung weiterhin den normalen WhatsApp-Autorisierungspfad für Absender.

## Reaktionen auf Fragen

Bei einer `ask_user`-Aufforderung mit einer einzelnen nicht geheimen Einfachauswahlfrage und einer bis vier Optionen zeigt WhatsApp `1️⃣` bis `4️⃣` neben den Optionsbezeichnungen an. Reagieren Sie auf die zugestellte Aufforderung mit der entsprechenden Zahl, um sie zu beantworten. OpenClaw ordnet die Zahl über den Gateway der kanonischen Option zu; veraltete oder doppelte Eingaben werden ignoriert. Aufforderungen mit mehreren Fragen, Mehrfachauswahl oder Freitext können weiterhin nur per Textantwort beantwortet werden. Die normalen WhatsApp-Zulassungsregeln für DMs/Gruppen autorisieren den reagierenden Absender.

## Plugin-Hooks und Datenschutz

Eingehende WhatsApp-Nachrichten können persönliche Inhalte, Telefonnummern, Gruppenkennungen, Absendernamen und Felder zur Sitzungskorrelation enthalten. WhatsApp sendet eingehende `message_received`-Hook-Nutzdaten nicht an Plugins, sofern Sie dies nicht ausdrücklich aktivieren:

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

Beschränken Sie die Aktivierung unter `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived` auf ein Konto. Aktivieren Sie dies nur für Plugins, denen Sie eingehende WhatsApp-Inhalte und -Kennungen anvertrauen.

## Zugriffskontrolle und Aktivierung

<Tabs>
  <Tab title="DM-Richtlinie">
    `channels.whatsapp.dmPolicy`:

    | Wert | Verhalten |
    | --- | --- |
    | `pairing` (Standard) | Unbekannte Absender beantragen eine Kopplung; der Eigentümer genehmigt sie |
    | `allowlist` | Nur Absender aus `allowFrom` werden zugelassen |
    | `open` | Erfordert, dass `allowFrom` den Wert `"*"` enthält |
    | `disabled` | Alle DMs blockieren |

    `allowFrom` akzeptiert Nummern im E.164-Format (intern normalisiert). Dies ist ausschließlich eine Zugriffskontrollliste für Absender von Direktnachrichten – sie beschränkt keine expliziten ausgehenden Sendungen an Gruppen-JIDs oder Kanal-JIDs von `@newsletter`.

    Überschreibung bei mehreren Konten: `channels.whatsapp.accounts.<id>.dmPolicy` (und `.allowFrom`) haben für dieses Konto Vorrang vor den Standardeinstellungen auf Kanalebene.

    Laufzeithinweise:

    - Kopplungen bleiben im Allow-Store des Kanals gespeichert und werden mit dem konfigurierten `allowFrom` zusammengeführt
    - geplante Automatisierungen und der Empfänger-Fallback für Heartbeat verwenden explizite Zustellungsziele oder den konfigurierten `allowFrom`; Genehmigungen von Direktnachrichten-Kopplungen sind keine impliziten Cron-/Heartbeat-Empfänger
    - wenn keine Positivliste konfiguriert ist, ist die verknüpfte eigene Nummer standardmäßig zulässig
    - OpenClaw koppelt ausgehende `fromMe`-Direktnachrichten niemals automatisch (Nachrichten, die Sie sich selbst vom verknüpften Gerät senden)

  </Tab>

  <Tab title="Gruppenrichtlinie und Positivlisten">
    Der Gruppenzugriff umfasst zwei Ebenen:

    1. **Positivliste für Gruppenmitgliedschaft** (`channels.whatsapp.groups`): Wenn `groups` weggelassen wird, kommen alle Gruppen infrage; ist es vorhanden, fungiert es als Gruppen-Positivliste (`"*"` lässt alle zu).
    2. **Richtlinie für Gruppenabsender** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`): `open` umgeht die Absender-Positivliste, `allowlist` erfordert eine Übereinstimmung mit `groupAllowFrom` (oder `*`), `disabled` blockiert alle eingehenden Gruppennachrichten.

    Wenn `groupAllowFrom` nicht festgelegt ist, greifen die Absenderprüfungen auf `allowFrom` zurück, sofern es Einträge enthält. Absender-Positivlisten werden vor der Aktivierung durch Erwähnung oder Antwort ausgewertet.

    Wenn überhaupt kein `channels.whatsapp`-Block vorhanden ist, greift die Laufzeit auf `groupPolicy: "allowlist"` zurück (mit einem Warnprotokolleintrag), selbst wenn `channels.defaults.groupPolicy` auf einen anderen Wert gesetzt ist.

    <Note>
    Die Auflösung der Gruppenmitgliedschaft verfügt über ein Sicherheitsnetz für Einzelkonten: Wenn nur ein WhatsApp-Konto konfiguriert ist und dessen `accounts.<id>.groups` ein explizit leeres Objekt (`{}`) ist, wird dies als „nicht festgelegt“ behandelt und es wird auf die `channels.whatsapp.groups`-Zuordnung auf Stammebene zurückgegriffen, anstatt unbemerkt jede Gruppe zu blockieren. Bei 2+ konfigurierten Konten bleibt eine explizit leere Kontozuordnung leer und greift nicht auf die Stammebene zurück – dadurch kann ein Konto absichtlich alle Gruppen deaktivieren, ohne gleichrangige Konten zu beeinflussen.
    </Note>

  </Tab>

  <Tab title="Erwähnungen und /activation">
    Gruppenantworten erfordern standardmäßig eine Erwähnung. Die Erkennung von Erwähnungen umfasst:

    - explizite WhatsApp-Erwähnungen der Bot-Identität
    - konfigurierte reguläre Ausdrücke für Erwähnungen (`agents.entries.*.groupChat.mentionPatterns`, Fallback `messages.groupChat.mentionPatterns`)
    - Transkripte eingehender Sprachnachrichten für autorisierte Gruppennachrichten
    - implizite Erkennung einer Antwort an den Bot (Absender der Antwort entspricht der Bot-Identität)

    Sicherheit: Ein Zitat oder eine Antwort erfüllt lediglich die Erwähnungsbedingung – es erteilt **keine** Absenderautorisierung. Mit `groupPolicy: "allowlist"` bleiben Absender, die nicht auf der Positivliste stehen, auch dann blockiert, wenn sie auf die Nachricht eines zulässigen Benutzers antworten.

    Aktivierungsbefehl auf Sitzungsebene: `/activation mention` oder `/activation always`. Dadurch wird der Sitzungsstatus aktualisiert (nicht die globale Konfiguration); der Befehl ist Eigentümern vorbehalten.

  </Tab>
</Tabs>

## Konfigurierte ACP-Bindungen

WhatsApp unterstützt dauerhafte ACP-Bindungen über `bindings[]` auf oberster Ebene:

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

Direktchats werden anhand von E.164-Nummern abgeglichen, Gruppen anhand von WhatsApp-Gruppen-JIDs. Gruppen-Positivlisten, Absenderrichtlinie und die Aktivierungsbedingung durch Erwähnung werden ausgeführt, bevor OpenClaw sicherstellt, dass die gebundene ACP-Sitzung vorhanden ist. Eine übereinstimmende Bindung übernimmt die Route – Broadcast-Gruppen verteilen diesen Durchlauf nicht an gewöhnliche WhatsApp-Sitzungen.

## Verhalten bei persönlicher Nummer und Selbstchat

Wenn die verknüpfte eigene Nummer auch in `allowFrom` vorhanden ist, werden Schutzmechanismen für Selbstchats aktiviert: Lesebestätigungen für Selbstchat-Durchläufe werden übersprungen, das automatische Auslösen durch Erwähnungs-JIDs, das Sie selbst benachrichtigen würde, wird ignoriert, und Antworten werden standardmäßig an `[{identity.name}]` (oder `[openclaw]`) gesendet, wenn `responsePrefix` für den Kanal oder das Konto nicht festgelegt ist.

## Nachrichtennormalisierung und Kontext

<AccordionGroup>
  <Accordion title="Eingehender Umschlag und Antwortkontext">
    Eingehende Nachrichten werden in den gemeinsamen eingehenden Umschlag eingebettet. Eine zitierte Antwort hängt den Kontext in dieser Form an:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Antwortmetadaten (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, Absender-JID/E.164) werden ausgefüllt, sofern verfügbar. Wenn das zitierte Ziel herunterladbare Medien enthält, speichert OpenClaw sie über den normalen Speicher für eingehende Medien und stellt `MediaPath`/`MediaType` bereit, damit der Agent sie direkt untersuchen kann, anstatt nur `<media:image>` zu sehen.

  </Accordion>

  <Accordion title="Medienplatzhalter und Extraktion von Standort-/Kontaktdaten">
    Nachrichten, die nur Medien enthalten, werden auf Platzhalter normalisiert: `<media:image>`, `<media:video>`, `<media:audio>`, `<media:document>`, `<media:sticker>`.

    Autorisierte Gruppensprachnachrichten werden vor der Erwähnungsbedingung transkribiert, wenn der Textkörper nur `<media:audio>` enthält, sodass das Aussprechen der Bot-Erwähnung in der Sprachnachricht die Antwort auslösen kann. Wenn das Transkript den Bot weiterhin nicht erwähnt, verbleibt es anstelle des unverarbeiteten Platzhalters im Verlauf ausstehender Gruppennachrichten.

    Standortangaben werden als knapper Koordinatentext dargestellt. Standortbezeichnungen/-kommentare und Kontakt-/vCard-Details werden als eingezäunte, nicht vertrauenswürdige Metadaten und nicht als eingebetteter Prompt-Text dargestellt.

  </Accordion>

  <Accordion title="Einfügung ausstehender Gruppenverläufe">
    Nicht verarbeitete Gruppennachrichten werden gepuffert und als Kontext eingefügt, sobald der Bot schließlich ausgelöst wird.

    - Standardlimit: `50`
    - Konfiguration: `channels.whatsapp.historyLimit`, Fallback `messages.groupChat.historyLimit`
    - `0` deaktiviert die Funktion

    Einfügungsmarkierungen: `[Chat messages since your last reply - for context]` und `[Current message - respond to this]`.

  </Accordion>

  <Accordion title="Lesebestätigungen">
    Für akzeptierte eingehende Nachrichten standardmäßig aktiviert. Global deaktivieren:

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    Kontospezifische Überschreibung: `channels.whatsapp.accounts.<id>.sendReadReceipts`. Selbstchat-Durchläufe überspringen Lesebestätigungen, auch wenn diese global aktiviert sind.

  </Accordion>
</AccordionGroup>

## Zustellung, Aufteilung und Medien

<AccordionGroup>
  <Accordion title="Textaufteilung">
    - Standardlimit für Abschnitte: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`; `newline` bevorzugt Absatzgrenzen (Leerzeilen) und greift anschließend auf eine längensichere Aufteilung zurück

  </Accordion>

  <Accordion title="Verhalten ausgehender Medien">
    - unterstützt Bild-, Video-, Audio- (PTT-Sprachnachricht) und Dokument-Nutzlasten
    - Audio wird als Baileys-Nutzlast `audio` mit `ptt: true` gesendet und als Push-to-Talk-Sprachnachricht dargestellt; `audioAsVoice` bleibt in Antwort-Nutzlasten erhalten, sodass die Ausgabe von TTS-Sprachnachrichten unabhängig vom Quellformat des Providers diesen Pfad verwendet
    - natives Ogg-/Opus-Audio wird als `audio/ogg; codecs=opus` gesendet; alle anderen Formate (einschließlich MP3-/WebM-Ausgaben von Microsoft Edge TTS) werden vor der PTT-Zustellung mit `ffmpeg` in Ogg/Opus mit 48 kHz und einem Kanal transkodiert
    - `/tts latest` sendet die neueste Assistentenantwort als einzelne Sprachnachricht und unterdrückt wiederholte Sendungen derselben Antwort; `/tts chat on|off|default` steuert automatisches TTS für den aktuellen Chat
    - `gifPlayback: true` bei Videosendungen aktiviert die Wiedergabe als animiertes GIF
    - `forceDocument`/`asDocument` leitet ausgehende Bilder, GIFs und Videos über die Baileys-Dokument-Nutzlast, um die Medienkomprimierung von WhatsApp zu vermeiden und den ermittelten Dateinamen sowie MIME-Typ beizubehalten
    - Beschriftungen werden auf das erste Medienelement einer Antwort mit mehreren Medien angewendet, ausgenommen PTT-Sprachnachrichten: Das Audio wird zuerst ohne Beschriftung gesendet, anschließend wird die Beschriftung als separate Textnachricht gesendet (WhatsApp-Clients stellen Beschriftungen von Sprachnachrichten nicht einheitlich dar)
    - die Medienquelle kann HTTP(S), `file://` oder ein lokaler Pfad sein

  </Accordion>

  <Accordion title="Mediengrößenlimits und Fallback-Verhalten">
    - Speicherlimit für eingehende und Sendelimit für ausgehende Medien: `channels.whatsapp.mediaMaxMb` (Standard: `50`)
    - kontospezifische Überschreibung: `channels.whatsapp.accounts.<id>.mediaMaxMb`
    - Bilder werden automatisch optimiert (Größenänderung/Qualitätsabstufung), um die Limits einzuhalten, sofern `forceDocument`/`asDocument` keine Zustellung als Dokument anfordert
    - wenn das Senden von Medien fehlschlägt, sendet der Fallback für das erste Element eine Textwarnung, anstatt die Antwort unbemerkt zu verwerfen

  </Accordion>
</AccordionGroup>

## Antwortzitate

`channels.whatsapp.replyToMode` steuert das native Zitieren von Antworten (ausgehende Antworten zitieren sichtbar die eingehende Nachricht):

| Wert              | Verhalten                                                       |
| ----------------- | -------------------------------------------------------------- |
| `"off"` (Standard) | Nie zitieren; als einfache Nachricht senden                    |
| `"first"`         | Nur den ersten Abschnitt der ausgehenden Antwort zitieren      |
| `"all"`           | Jeden Abschnitt der ausgehenden Antwort zitieren               |
| `"batched"`       | In Warteschlangen gebündelte Antworten zitieren; sofortige Antworten nicht zitieren |

Kontospezifische Überschreibung: `channels.whatsapp.accounts.<id>.replyToMode`.

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## Reaktionsstufe

`channels.whatsapp.reactionLevel` steuert, wie umfassend der Agent Emoji-Reaktionen verwendet:

| Stufe                 | Bestätigungsreaktionen | Vom Agenten initiierte Reaktionen |
| --------------------- | ------------- | -------------------------- |
| `"off"`               | Nein          | Nein                       |
| `"ack"`               | Ja            | Nein                       |
| `"minimal"` (Standard) | Ja            | Ja, zurückhaltende Vorgabe |
| `"extensive"`         | Ja            | Ja, empfohlene Vorgabe     |

Kontospezifische Überschreibung: `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## Bestätigungsreaktionen

`channels.whatsapp.ackReaction` sendet beim Eingang sofort eine Reaktion, die durch `reactionLevel` eingeschränkt wird (unterdrückt, wenn `"off"`):

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

Hinweise: Wird unmittelbar nach Annahme der eingehenden Nachricht gesendet (vor der Antwort); wenn `ackReaction` ohne `emoji` vorhanden ist, verwendet WhatsApp das Identitäts-Emoji des zugewiesenen Agenten mit „👀“ als Fallback (`ackReaction` weglassen oder `emoji: ""` festlegen, um keine Bestätigung zu senden); Fehler werden protokolliert, blockieren jedoch nicht die Zustellung der Antwort; im Gruppenmodus `mentions` erfolgt eine Reaktion nur bei durch Erwähnung ausgelösten Durchläufen, während die Gruppenaktivierung `always` diese Prüfung umgeht; WhatsApp verwendet ausschließlich `channels.whatsapp.ackReaction` (das veraltete `messages.ackReaction` gilt hier nicht).

## Lebenszyklus-Statusreaktionen

Legen Sie `messages.statusReactions.enabled: true` fest, damit WhatsApp während eines Durchlaufs die Bestätigungsreaktion ersetzt, anstatt ein statisches Empfangs-Emoji beizubehalten. Dabei werden Zustände wie „in Warteschlange“, „denkt nach“, „Werkzeugaktivität“, Compaction, „abgeschlossen“ und „Fehler“ durchlaufen:

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

Hinweise: `channels.whatsapp.ackReaction` steuert weiterhin die Berechtigung für Direktnachrichten und Gruppen; der Warteschlangenzustand verwendet dasselbe effektive Emoji wie einfache Bestätigungsreaktionen; WhatsApp verfügt pro Nachricht über einen Reaktionsplatz für den Bot, daher ersetzen Lebenszyklusaktualisierungen die aktuelle Reaktion direkt und stellen die Bestätigung nach dem abschließenden Erfolgs- oder Fehlerzustand wieder her.

## Mehrere Konten und Anmeldedaten

<AccordionGroup>
  <Accordion title="Kontoauswahl und Standardeinstellungen">
    Konto-IDs stammen aus `channels.whatsapp.accounts`. Für die Auswahl des Standardkontos wird `default` verwendet, falls vorhanden, andernfalls die erste konfigurierte Konto-ID (alphabetisch sortiert). Konto-IDs werden intern für die Suche normalisiert.
  </Accordion>

  <Accordion title="Anmeldedatenpfade und Abwärtskompatibilität">
    - aktueller Authentifizierungspfad: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (Sicherung: `creds.json.bak`)
    - die veraltete Standardauthentifizierung in `~/.openclaw/credentials/` wird für Abläufe mit dem Standardkonto weiterhin erkannt/migriert

  </Accordion>

  <Accordion title="Abmeldeverhalten">
    `openclaw channels logout --channel whatsapp [--account <id>]` löscht den WhatsApp-Authentifizierungsstatus für dieses Konto. Wenn ein Gateway erreichbar ist, wird bei der Abmeldung zuerst der aktive Listener für dieses Konto beendet, sodass die verknüpfte Sitzung bereits vor dem nächsten Neustart keine Nachrichten mehr empfängt. `openclaw channels remove --channel whatsapp` beendet den aktiven Listener ebenfalls, bevor die Kontokonfiguration deaktiviert oder gelöscht wird.

    In veralteten Authentifizierungsverzeichnissen bleibt `oauth.json` erhalten, während die Baileys-Authentifizierungsdateien entfernt werden.

  </Accordion>
</AccordionGroup>

## Tools, Aktionen und Konfigurationsänderungen

- Die Unterstützung für Agent-Tools umfasst die WhatsApp-Reaktionsaktion (`react`).
- Aktionsfreigaben: `channels.whatsapp.actions.reactions`, `channels.whatsapp.actions.polls` (vorhandene Aktionen verwenden standardmäßig `true`), `channels.whatsapp.actions.calls` (Standardwert `false`, siehe MeowCaller oben).
- Vom Kanal initiierte Konfigurationsänderungen sind standardmäßig aktiviert; deaktivieren Sie sie über `channels.whatsapp.configWrites: false`.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Nicht verknüpft (QR erforderlich)">
    Symptom: Der Kanalstatus meldet, dass keine Verknüpfung besteht.

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="Verknüpft, aber getrennt / Schleife bei der Wiederherstellung der Verbindung">
    Symptom: Ein verknüpftes Konto weist wiederholte Verbindungsabbrüche oder Wiederverbindungsversuche auf.

    Inaktive Konten können über das normale Nachrichten-Timeout hinaus verbunden bleiben; der Watchdog startet nur neu, wenn die WhatsApp-Web-Transportaktivität aussetzt, der Socket geschlossen wird oder die Aktivität auf Anwendungsebene über das längere Sicherheitszeitfenster hinaus ausbleibt (siehe Laufzeitmodell oben).

    Behebung:

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    Wenn die Schleife nach der Behebung von Hostverbindung und Zeitsteuerung weiterhin besteht, sichern Sie das Authentifizierungsverzeichnis des Kontos und verknüpfen Sie es erneut:

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    Wenn `~/.openclaw/logs/whatsapp-health.log` `Gateway inactive` meldet, aber sowohl `openclaw gateway status` als auch `openclaw channels status --probe` einen fehlerfreien Zustand anzeigen, führen Sie `openclaw doctor` aus. Unter Linux warnt doctor vor veralteten crontab-Einträgen, die das nicht mehr verwendete Skript `~/.openclaw/bin/ensure-whatsapp.sh` aufrufen; entfernen Sie diese Einträge mit `crontab -e` — Cron verfügt möglicherweise nicht über die systemd-User-Bus-Umgebung, wodurch das alte Skript den Zustand des Gateways falsch melden kann.

  </Accordion>

  <Accordion title="Zeitüberschreitung bei der QR-Anmeldung hinter einem Proxy">
    Symptom: `openclaw channels login --channel whatsapp` schlägt mit `status=408 Request Time-out` oder einer Trennung des TLS-Sockets fehl, bevor ein verwendbarer QR-Code angezeigt wird.

    Die WhatsApp-Web-Anmeldung verwendet die standardmäßige Proxy-Umgebung des Gateway-Hosts (`HTTPS_PROXY`, `HTTP_PROXY`, Varianten in Kleinbuchstaben, `NO_PROXY`). Stellen Sie sicher, dass der Gateway-Prozess die Proxy-Umgebung übernimmt und dass `NO_PROXY` nicht auf `mmg.whatsapp.net` zutrifft.

  </Accordion>

  <Accordion title="Kein aktiver Listener beim Senden">
    Ausgehende Sendevorgänge schlagen sofort fehl, wenn für das Zielkonto kein aktiver Gateway-Listener vorhanden ist. Vergewissern Sie sich, dass das Gateway ausgeführt wird und das Konto verknüpft ist.
  </Accordion>

  <Accordion title="Antwort erscheint im Transkript, aber nicht in WhatsApp">
    Transkriptzeilen zeichnen auf, was der Agent erzeugt hat; die Zustellung über WhatsApp wird separat überprüft. OpenClaw betrachtet eine automatische Antwort erst dann als gesendet, wenn Baileys für mindestens einen sichtbaren Text- oder Medienversand eine ID der ausgehenden Nachricht zurückgibt.

    Bestätigungsreaktionen sind unabhängige Empfangsbestätigungen vor der Antwort — eine erfolgreiche Reaktion belegt nicht, dass die nachfolgende Text-/Medienantwort akzeptiert wurde. Prüfen Sie die Gateway-Protokolle auf `auto-reply delivery failed` oder `auto-reply was not accepted by WhatsApp provider`.

  </Accordion>

  <Accordion title="Gruppennachrichten werden unerwartet ignoriert">
    Prüfen Sie in dieser Reihenfolge: `groupPolicy`, `groupAllowFrom`/`allowFrom`, Einträge in der Positivliste `groups`, Erwähnungsfilterung (`requireMention` + Erwähnungsmuster) und doppelte Schlüssel in `openclaw.json` (spätere JSON5-Einträge überschreiben frühere — behalten Sie pro Gültigkeitsbereich nur einen Eintrag `groupPolicy` bei).

    Wenn `channels.whatsapp.groups` vorhanden ist, kann WhatsApp weiterhin Nachrichten aus anderen Gruppen erkennen, OpenClaw verwirft sie jedoch vor dem Sitzungsrouting. Fügen Sie die Gruppen-JID zu `channels.whatsapp.groups` oder `groups["*"]` hinzu, um alle Gruppen zuzulassen, während die Absenderautorisierung weiterhin über `groupPolicy`/`groupAllowFrom` gesteuert wird.

  </Accordion>

  <Accordion title="Bun-Laufzeitwarnung">
    OpenClaw-Gateways erfordern Node. Bun stellt die vom kanonischen Zustandsspeicher verwendete API `node:sqlite` nicht bereit, und doctor migriert veraltete Bun-Dienste zu Node.
  </Accordion>
</AccordionGroup>

## System-Prompts

WhatsApp unterstützt Telegram-ähnliche System-Prompts für Gruppen und Direktchats über die Zuordnungen `groups` und `direct`.

Auflösung für Gruppennachrichten: Zunächst wird die effektive Zuordnung `groups` bestimmt — wenn das Konto überhaupt einen eigenen Schlüssel `groups` definiert, ersetzt dieser die Stammzuordnung `groups` vollständig (keine tiefe Zusammenführung). Anschließend erfolgt die Prompt-Suche in dieser einzelnen resultierenden Zuordnung:

1. **Gruppenspezifischer Prompt** (`groups["<groupId>"].systemPrompt`): wird verwendet, wenn der Gruppeneintrag vorhanden **und** sein Schlüssel `systemPrompt` definiert ist. Eine leere Zeichenfolge (`""`) unterdrückt den Platzhalter und wendet keinen Prompt an.
2. **Gruppen-Platzhalter-Prompt** (`groups["*"].systemPrompt`): wird verwendet, wenn der spezifische Gruppeneintrag fehlt oder ohne Schlüssel `systemPrompt` vorhanden ist.

Die Auflösung für Direktnachrichten folgt demselben Muster anhand der Zuordnung `direct` und `direct["*"]`.

<Note>
`dms` bleibt der einfache Bucket für individuelle DM-Verlaufsüberschreibungen (`dms.<id>.historyLimit`). Prompt-Überschreibungen befinden sich unter `direct`.
</Note>

<Note>
Dieses Verhalten, bei dem das Konto die Stammzuordnung für die Prompt-Auflösung ersetzt, ist eine einfache flache Überschreibung: Jeder kontospezifische Schlüssel `groups`/`direct`, einschließlich eines ausdrücklich leeren Objekts, ersetzt die Stammzuordnung. Es unterscheidet sich von der oben beschriebenen Prüfung der Gruppenzugehörigkeits-Positivliste, die für eine versehentlich leere `groups: {}` ein Sicherheitsnetz bei nur einem Konto bietet.
</Note>

**Unterschied zu Telegram:** Telegram unterdrückt die Stammzuordnung `groups` für jedes Konto in einer Einrichtung mit mehreren Konten (selbst für Konten ohne eigene `groups`), damit ein Bot keine Gruppennachrichten für Gruppen empfängt, denen er nicht angehört. WhatsApp wendet diesen Schutz nicht an — die Stammzuordnungen `groups`/`direct` werden unabhängig von der Anzahl der Konten von jedem Konto ohne eigene Überschreibung übernommen. Definieren Sie in einer WhatsApp-Einrichtung mit mehreren Konten die vollständige Zuordnung ausdrücklich unter jedem Konto, wenn Sie kontospezifische Prompts wünschen.

Wichtiges Verhalten:

- `channels.whatsapp.groups` ist sowohl eine gruppenspezifische Konfigurationszuordnung als auch die Positivliste für Gruppen auf Chat-Ebene. Sowohl im Stamm- als auch im Kontogültigkeitsbereich bedeutet `groups["*"]`, dass für diesen Gültigkeitsbereich „alle Gruppen zugelassen sind“.
- Fügen Sie den Platzhalter `systemPrompt` nur hinzu, wenn in diesem Gültigkeitsbereich bereits alle Gruppen zugelassen werden sollen. Damit nur eine festgelegte Gruppe von Gruppen-IDs zulässig bleibt, wiederholen Sie den Prompt für jeden ausdrücklich in der Positivliste aufgeführten Eintrag, anstatt `groups["*"]` zu verwenden.
- Gruppenzulassung und Absenderautorisierung sind separate Prüfungen. `groups["*"]` erweitert die Auswahl der Gruppen, die die Gruppenverarbeitung erreichen; dadurch werden nicht alle Absender in diesen Gruppen autorisiert — dies wird weiterhin durch `groupPolicy`/`groupAllowFrom` gesteuert.
- `channels.whatsapp.direct` hat für DMs keine entsprechende Nebenwirkung: `direct["*"]` stellt lediglich eine Standardkonfiguration bereit, nachdem eine DM bereits durch `dmPolicy` zusammen mit `allowFrom` oder Regeln des Kopplungsspeichers zugelassen wurde.

Beispiel:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // Nur verwenden, wenn im Stammgültigkeitsbereich alle Gruppen zugelassen werden sollen.
        // Gilt für alle Konten, die keine eigene groups-Zuordnung definieren.
        "*": { systemPrompt: "Standard-Prompt für alle Gruppen." },
      },
      direct: {
        // Gilt für alle Konten, die keine eigene direct-Zuordnung definieren.
        "*": { systemPrompt: "Standard-Prompt für alle Direktchats." },
      },
      accounts: {
        work: {
          groups: {
            // Dieses Konto definiert eigene groups, daher werden die groups des Stammbereichs
            // vollständig ersetzt. Um einen Platzhalter beizubehalten, definieren Sie "*" auch hier ausdrücklich.
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "Konzentrieren Sie sich auf das Projektmanagement.",
            },
            // Nur verwenden, wenn in diesem Konto alle Gruppen zugelassen werden sollen.
            "*": { systemPrompt: "Standard-Prompt für Arbeitsgruppen." },
          },
          direct: {
            // Dieses Konto definiert eine eigene direct-Zuordnung, daher werden die direct-Einträge
            // des Stammbereichs vollständig ersetzt. Um einen Platzhalter beizubehalten, definieren Sie "*" auch hier ausdrücklich.
            "+15551234567": { systemPrompt: "Prompt für einen bestimmten geschäftlichen Direktchat." },
            "*": { systemPrompt: "Standard-Prompt für geschäftliche Direktchats." },
          },
        },
      },
    },
  },
}
```

## Verweise zur Konfigurationsreferenz

Primäre Referenz: [Konfigurationsreferenz – WhatsApp](/de/gateway/config-channels#whatsapp)

| Bereich          | Felder                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Zugriff          | `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`                                             |
| Zustellung       | `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`      |
| Mehrere Konten   | `accounts.<id>.enabled`, `accounts.<id>.authDir` und weitere kontospezifische Überschreibungen                              |
| Betrieb          | `configWrites`, `enabled`                                                                                      |
| Eingangs-Batching | `messages.inbound.debounceMs`, `messages.inbound.byChannel.whatsapp`                                           |
| Sitzungsverhalten | `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`                                   |
| Prompts          | `groups.<id>.systemPrompt`, `groups["*"].systemPrompt`, `direct.<id>.systemPrompt`, `direct["*"].systemPrompt` |

## Verwandte Themen

- [Kopplung](/de/channels/pairing)
- [Gruppen](/de/channels/groups)
- [Sicherheit](/de/gateway/security)
- [Kanalrouting](/de/channels/channel-routing)
- [Multi-Agent-Routing](/de/concepts/multi-agent)
- [Fehlerbehebung](/de/channels/troubleshooting)
