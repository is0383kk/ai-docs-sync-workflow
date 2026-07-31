---
read_when:
    - Sie möchten Anthropic-Modelle in OpenClaw verwenden
    - Sie möchten Claude-CLI- oder Claude-Desktop-Sitzungen auf gekoppelten Computern durchsuchen
summary: Anthropic Claude über API-Schlüssel oder die Claude CLI in OpenClaw verwenden
title: Anthropic
x-i18n:
    generated_at: "2026-07-26T18:32:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic entwickelt die Modellfamilie **Claude**. OpenClaw unterstützt zwei Authentifizierungswege:

- **API-Schlüssel** – direkter Zugriff auf die Anthropic-API mit nutzungsbasierter Abrechnung (`anthropic/*`-Modelle)
- **Claude CLI** – eine vorhandene Claude-Code-Anmeldung auf demselben Host wiederverwenden

## Nutzungs- und Kostenverfolgung

OpenClaw erkennt die verfügbaren Anthropic-Anmeldedaten und wählt die passende Nutzungsansicht aus:

- Claude-Abonnement-/Einrichtungsanmeldedaten zeigen Kontingentzeiträume und ein optionales Budget für zusätzliche Nutzung.
- `ANTHROPIC_ADMIN_KEY` oder `ANTHROPIC_ADMIN_API_KEY` zeigt in der Control UI unter **Nutzung** die vom Provider gemeldeten Organisationskosten und die Nutzung der Messages API für 30 Tage, einschließlich täglicher Ausgaben, Token-/Cache-Gesamtsummen, meistgenutzter Modelle und Kostenkategorien.
- In einem Anthropic-Provider-Profil gespeicherte `sk-ant-admin...`-Anmeldedaten werden automatisch als Admin-API-Schlüssel erkannt.

Der Kostenverlauf der Admin API stammt aus Anthropics [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api). Dabei handelt es sich um die tatsächliche Provider-Abrechnung, getrennt von den aus Sitzungen abgeleiteten geschätzten Kosten von OpenClaw.

<Warning>
Das Claude-CLI-Backend von OpenClaw führt die installierte Claude Code CLI im
nicht interaktiven Druckmodus (`claude -p`) aus. Anthropics aktuelle Dokumentation zu Claude Code
beschreibt diesen Modus als Agent-SDK-/programmatische Nutzung. Anthropics Support-Aktualisierung vom 15. Juni 2026
setzte die angekündigte separate Änderung der Agent-SDK-Abrechnung aus: Claude
Agent SDK, `claude -p` und die Nutzung durch Drittanbieter-Apps werden weiterhin auf die Nutzungslimits
des angemeldeten Abonnements angerechnet, und das zuvor angekündigte monatliche Agent-SDK-
Guthaben ist nicht verfügbar, während Anthropic diesen Plan überarbeitet.

Interaktives Claude Code wird weiterhin auf die Limits des angemeldeten Claude-Tarifs angerechnet.
Die Authentifizierung per API-Schlüssel wird direkt nach tatsächlicher Nutzung abgerechnet und hängt nicht von diesem Tarif ab.
Verwenden Sie für langlebige Gateway-Hosts, gemeinsam genutzte Automatisierung und vorhersehbare Produktions-
ausgaben einen Anthropic-API-Schlüssel.

Anthropics aktuelle Supportartikel können dieses Verhalten ohne eine
OpenClaw-Veröffentlichung ändern:

- [Claude-Code-CLI-Referenz](https://code.claude.com/docs/en/cli-usage)
- [Claude Agent SDK mit Ihrem Claude-Tarif verwenden](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [Claude Code mit Ihrem Pro- oder Max-Tarif verwenden](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [Claude Code mit Ihrem Team- oder Enterprise-Tarif verwenden](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [Claude-Code-Kosten verwalten](https://code.claude.com/docs/en/costs)

</Warning>

## Erste Schritte

<Tabs>
  <Tab title="API-Schlüssel">
    **Am besten geeignet für:** standardmäßigen API-Zugriff und nutzungsbasierte Abrechnung.

    <Steps>
      <Step title="API-Schlüssel abrufen">
        Erstellen Sie in der [Anthropic Console](https://console.anthropic.com/) einen API-Schlüssel.
      </Step>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard
        # auswählen: Anthropic-API-Schlüssel
        ```

        Alternativ können Sie den Schlüssel direkt übergeben:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="Verfügbarkeit des Modells überprüfen">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### Konfigurationsbeispiel

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **Am besten geeignet für:** die Wiederverwendung einer vorhandenen Claude-CLI-Anmeldung ohne separaten API-Schlüssel.

    <Steps>
      <Step title="Sicherstellen, dass Claude CLI installiert und angemeldet ist">
        Überprüfen Sie dies mit:

        ```bash
        claude --version
        ```
      </Step>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard
        # auswählen: Claude CLI
        ```

        OpenClaw erkennt die vorhandenen Claude-CLI-Anmeldedaten und verwendet sie wieder.
      </Step>
      <Step title="Verfügbarkeit des Modells überprüfen">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Details zur Einrichtung und Laufzeit des Claude-CLI-Backends finden Sie unter [CLI-Backends](/de/gateway/cli-backends).
    </Note>

    <Warning>
    Die Wiederverwendung von Claude CLI setzt voraus, dass der OpenClaw-Prozess auf demselben Host wie die
    Claude-CLI-Anmeldung ausgeführt wird. Bei Docker-Installationen kann ein Container-Home-Verzeichnis dauerhaft gespeichert und dort eine Anmeldung bei
    Claude Code durchgeführt werden; siehe
    [Claude-CLI-Backend in Docker](/de/install/docker#claude-cli-backend-in-docker).
    Andere Container-Installationen wie [Podman](/de/install/podman) binden das
    hostseitige `~/.claude` weder bei der Einrichtung noch zur Laufzeit ein; verwenden Sie dort einen Anthropic-API-Schlüssel oder wählen Sie
    einen Provider mit von OpenClaw verwaltetem OAuth wie
    [OpenAI Codex](/de/providers/openai).
    </Warning>

    ### Einrichtungstoken abrufen

    Führen Sie `claude setup-token` auf einem beliebigen Rechner aus, auf dem Claude Code installiert ist. Der Befehl gibt
    ein langlebiges Token aus, das mit `sk-ant-oat01-` beginnt.

    Fügen Sie das Token während des Onboardings in der macOS-App ein, indem Sie
    **Anthropic setup-token** unter **Connect with an API key or token** auswählen, oder verwenden Sie:

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### Konfigurationsbeispiel

    Bevorzugen Sie die kanonische Anthropic-Modellreferenz zusammen mit einer CLI-Laufzeitüberschreibung:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    Ältere `claude-cli/claude-opus-4-7`-Modellreferenzen funktionieren aus
    Kompatibilitätsgründen weiterhin, neue Konfigurationen sollten die Provider-/Modellauswahl jedoch als
    `anthropic/*` beibehalten und das Ausführungs-Backend in der Provider-/Modell-Laufzeitrichtlinie festlegen.

    ### Abrechnung und `claude -p`

    OpenClaw verwendet für Claude-CLI-
    Ausführungen den nicht interaktiven `claude -p`-Pfad von Claude Code. Anthropic behandelt diesen Pfad derzeit als Agent-SDK-/programmatische Nutzung:

    - Anthropics Support-Aktualisierung vom 15. Juni 2026 setzte den zuvor angekündigten
      separaten Agent-SDK-Guthabenplan aus.
    - Claude Agent SDK im Abonnementtarif, `claude -p` und die Nutzung durch Drittanbieter-Apps
      werden weiterhin auf die Nutzungslimits des angemeldeten Abonnements angerechnet.
    - Das zuvor angekündigte monatliche Agent-SDK-Guthaben ist nicht verfügbar, während
      Anthropic diesen Plan überarbeitet.
    - Anmeldungen über die Konsole bzw. per API-Schlüssel verwenden die nutzungsbasierte API-Abrechnung und erhalten
      das Agent-SDK-Guthaben des Abonnements nicht.

    Informationen zur Aussetzung finden Sie in Anthropics [Artikel zum Agent-SDK-Tarif
    ](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan) und Informationen zum Abonnementverhalten in den Artikeln zu Claude-Code-Tarifen für
    [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    sowie
    [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan).

    Anthropic kann das Abrechnungs- und Ratenbegrenzungsverhalten von Claude Code ohne eine
    OpenClaw-Veröffentlichung ändern. Prüfen Sie `claude auth status`, `/status` und
    Anthropics verlinkte Dokumentation, wenn eine vorhersehbare Abrechnung wichtig ist.

    <Tip>
    Verwenden Sie für gemeinsam genutzte Produktionsautomatisierung statt
    Claude CLI einen Anthropic-API-Schlüssel. OpenClaw unterstützt außerdem abonnementbasierte Optionen von
    [OpenAI Codex](/de/providers/openai), [Qwen Cloud](/de/providers/qwen),
    [MiniMax](/de/providers/minimax) und [Z.AI / GLM](/de/providers/zai).
    </Tip>

  </Tab>
</Tabs>

## Claude-Sitzungen auf mehreren Computern

Das gebündelte Anthropic-Plugin fügt der normalen Sitzungs-
seitenleiste eine Gruppe **Claude Code** hinzu. Zeilen werden im normalen Chat-Bereich geöffnet. Es erkennt nicht archivierte Claude-
Code-Sitzungen auf dem Gateway und auf verbundenen Node-Hosts:

- Claude-CLI-Sitzungen stammen aus gültigen Projektindexdatensätzen. Bei nicht indizierten
  Transkripten erkennt eine begrenzte Metadaten-Ausweichlogik gleichzeitig laufende interaktive Sitzungen ohne Sidechain
  (`cli`) und monitorlose Agent-SDK-CLI-Sitzungen (`sdk-cli`) unter
  `~/.claude/projects/`.
- Claude-Desktop-Sitzungen verwenden den Desktop-Titel, den Aktivitätszeitpunkt und den
  Archivstatus, wenn ihre Metadaten auf dieselbe Claude-Code-Sitzungs-ID verweisen.
- Eine reine CLI-Sitzung besitzt kein Archivierungskennzeichen und bleibt daher sichtbar, solange ihr
  Transkript vorhanden ist.

Für die Erkennung ist keine zusätzliche OpenClaw-Konfiguration erforderlich. Das Anthropic-Plugin
ist gebündelt und standardmäßig aktiviert; ein nativer macOS-Node stellt die schreibgeschützten
Claude-Sitzungsbefehle bereit, wenn das lokale Verzeichnis `~/.claude/projects/` vorhanden ist.
Genehmigen Sie das Upgrade der Node-Kopplung, wenn diese Befehle erstmals angezeigt werden.

Die Seitenleiste gruppiert Zeilen nach ihrem Gateway- oder gekoppelten Node-Host und zeigt die
neueste begrenzte Seite jedes Hosts an, sobald der jeweilige Computer antwortet. Sie gleicht die Daten erneut ab,
wenn sich die Host-Konnektivität ändert, die Seite wieder den Fokus erhält und höchstens alle
30 Sekunden, solange sie sichtbar ist. Dadurch erscheinen außerhalb von OpenClaw erstellte Claude-Sitzungen
ohne erneutes Laden. Bei einem geänderten Katalog erfolgt schneller ein weiterer Durchlauf. Verwenden Sie **Weitere
Sitzungen laden** unter einer Kataloggruppe, um für jeden Host mit
weiterem Verlauf die nächste Seite anzuhängen; angehängte Zeilen bleiben sichtbar und werden bei
Aktualisierungen erneut bis zur gleichen Tiefe abgerufen. Katalogclients verwenden `sessions.catalog.list`; beim Öffnen einer Zeile wird
`sessions.catalog.read` verwendet.

Bei der Terminal-Übernahme wird `claude` zunächst über den Anmelde-Shell-
PATH des Benutzers des besitzenden Hosts und erst danach über den PATH des Dienstes/Daemons aufgelöst. Dadurch bleiben von der App gestartete Sitzungen auf
dieselbe Claude CLI ausgerichtet, die dem Betreiber in einem normalen Terminal zur Verfügung steht.

Beim Auswählen einer Zeile wird zuerst die neueste Transkriptseite gelesen. **Ältere Transkript-
elemente laden** folgt einem opaken Byte-Cursor und liest einen weiteren begrenzten Abschnitt aus der
JSONL-Datei, statt den gesamten Verlauf zu laden. Normale Inhalte von Benutzern, Assistenten,
Schlussfolgerungen, Tool-Aufrufen und Tool-Ergebnissen bleiben erhalten. Ein einzelnes Element,
das die Sicherheitsobergrenze von Node/Gateway überschreitet, wird eindeutig als abgeschnitten gekennzeichnet.

Bei einer Gateway-lokalen `claude-cli`-Zeile ruft eine Eingabe im normalen Eingabefeld
`sessions.catalog.continue` auf. OpenClaw löst den lokalen Katalogdatensatz erneut auf,
erstellt eine modellgebundene native Sitzung oder verwendet sie wieder, importiert höchstens 200 sichtbare
Elemente oder 512 KiB und initialisiert die Claude-CLI-Bindung. Die erste Interaktion wird mit
`--fork-session` fortgesetzt; Claude weist der Abspaltung eine neue Sitzungs-ID zu, sodass spätere Interaktionen
die Abspaltung verwenden und die Quellsitzung unverändert bleibt.

Ein monitorloser Node-Host kann seine Claude-CLI-Zeilen ebenfalls fortsetzbar machen, indem
die nachstehende Node-lokale Einstellung aktiviert und der Node-Host neu gestartet wird:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Der Node stellt `agent.cli.claude.run.v1` nur bereit, wenn die Einstellung aktiviert ist
und die lokale ausführbare Datei `claude` aufgelöst werden kann. OpenClaw löst den Katalog-
datensatz auf diesem Node erneut auf, importiert denselben begrenzten Verlauf und bindet die übernommene
Sitzung an den Node und das vom Katalog gemeldete Arbeitsverzeichnis. Bei jeder Interaktion wird der
echte `claude -p`-Prozess des Nodes mit den Claude-Dateien und der Anmeldung dieses Nodes ausgeführt. Die
Richtlinie des Nodes für Ausführungsgenehmigungen gilt weiterhin; das Gateway kann die Zustimmung nicht erzwingen.

Die Node-Fortsetzung v1 ist nur einmalig möglich. Sie lässt die Gateway-Loopback-MCP-Konfiguration und
Argumente des Gateway-Skills-Plugins aus, initialisiert nicht erneut aus einem Gateway-Transkript und
lehnt Anhänge und Bilder ab. Claude-Desktop-Zeilen bleiben schreibgeschützt. Native
macOS-App-Nodes bleiben ebenfalls schreibgeschützt, bis die App den Ausführungsbefehl bereitstellt.

<Note>
Claude-Sitzungen auf gekoppelten Nodes bleiben schreibgeschützt, sofern der monitorlose Node nicht ausdrücklich
`agent.cli.claude.run.v1` bereitstellt. OpenClaw ändert niemals Claude-Desktop-
Metadaten und archiviert keine Claude-Sitzungen. Die Seite erfordert eine Betreiberverbindung
mit Schreibberechtigung, da sie authentifiziertes `node.invoke` verwendet; Auflisten und Lesen
bleiben selbst auf einem Node mit aktivierter Fortsetzung schreibgeschützt.
</Note>

Siehe [Nodes: Claude-Sitzungen und Transkripte](/de/nodes#claude-sessions-and-transcripts)
für den Node-Befehl und die Sicherheitsgrenze.

## Standardwerte für Thinking (Claude Opus 5, Sonnet 5, Mythos 5, Fable 5, 4.8 und 4.6)

`anthropic/claude-opus-5` verwendet standardmäßig adaptives Thinking mit dem Aufwand `high`.
Verwenden Sie `/think off`, um Thinking zu deaktivieren, oder `/think xhigh|max` für die
höheren nativen Aufwandsstufen des Modells. OpenClaw lässt bei Opus 5 manuelle Thinking-Budgets,
benutzerdefinierte Sampling-Parameter, Assistant-Prefills und Priority Tier aus, da
Anthropic diese Anfragefunktionen bei diesem Modell nicht unterstützt. Der Katalog
weist sein Kontextfenster mit 1.000.000 Token, sein Ausgabelimit von 128.000 Token, die
Bildeingabe und die Eingabe-/Ausgabepreise von `$5/$25` aus.

`anthropic/claude-sonnet-5` verwendet dieselben Standardwerte für adaptives Thinking und dieselben
Anfragebeschränkungen. Der Katalog verwendet bis zum 31. August 2026 die Einführungspreise
von Anthropic von `$2/$10` für Ein-/Ausgabe; die Standardpreise von
`$3/$15` gelten ab dem 1. September 2026.

`anthropic/claude-fable-5` verwendet immer adaptives Thinking und standardmäßig den Aufwand
`high`. Anthropic erlaubt bei diesem Modell keine Deaktivierung von Thinking, daher
werden `/think off` und `/think minimal` stattdessen dem Aufwand `low`
zugeordnet. OpenClaw lässt bei Fable-5-Anfragen außerdem benutzerdefinierte Temperaturwerte
aus, da Anthropic eine Temperaturüberschreibung bei jeder Anfrage mit aktiviertem Thinking
ablehnt.

`anthropic/claude-mythos-5` ist ein Modell mit eingeschränktem Zugriff und demselben Vertrag für
stets aktives adaptives Thinking. OpenClaw verwendet standardmäßig `high`, ordnet
`/think off` und `/think minimal` `low` zu und lässt vom Aufrufer
ausgewählte Sampling-Parameter aus. Der Katalog weist sein Kontextfenster mit 1.000.000 Token,
sein Ausgabelimit von 128.000 Token, die Bildeingabe und die Eingabe-/Ausgabepreise von
`$10/$50` aus.

Bei Claude Opus 4.8 bleibt Thinking in OpenClaw standardmäßig deaktiviert. Wenn Sie adaptives
Thinking explizit mit `/think high|xhigh|max` aktivieren, sendet OpenClaw die Aufwandswerte von
Anthropic für Opus 4.8; Claude-4.6-Modelle (Opus 4.6 und Sonnet 4.6) verwenden standardmäßig
`adaptive`.

Überschreiben Sie dies pro Nachricht mit `/think:<level>` oder in den Modellparametern:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
Zugehörige Anthropic-Dokumentation:
- [Adaptives Thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Erweitertes Thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## Fallback bei sicherheitsbedingter Ablehnung (Claude Fable 5)

<Warning>
Die Verwendung von Claude Fable 5 bedeutet zugleich die Verwendung von Claude Opus 4.8. Fable 5 wird mit
Sicherheitsklassifikatoren ausgeliefert, die eine Anfrage ablehnen können, und die von Anthropic vorgesehene
Wiederherstellung besteht darin, diese Anfrage von `claude-opus-4-8` beantworten zu lassen. OpenClaw aktiviert dies
bei direkten Anfragen mit API-Schlüssel automatisch, sodass einige Fable-Anfragen von Claude Opus 4.8 beantwortet
und entsprechend abgerechnet werden. Wenn Ihre Richtlinie oder Ihr Budget keine von Opus beantworteten Anfragen
zulässt, wählen Sie `anthropic/claude-fable-5` nicht aus.
</Warning>

### Warum dies existiert

Die Klassifikatoren von Fable 5 geben bei Anfragen in eingeschränkten
Domänen `stop_reason: "refusal"` zurück und erzeugen auch falsch positive Ergebnisse bei harmlosen,
angrenzenden Aufgaben (Sicherheitswerkzeuge, Biowissenschaften oder sogar die Aufforderung an das
Modell, seine unverarbeitete Schlussfolgerung wiederzugeben). Ohne Fallback endet die Anfrage mit
einem Fehler, obwohl ein anderes Claude-Modell sie problemlos beantworten würde – Anthropic weist
API-Integratoren in der eigenen Ablehnungsmeldung an, ein Fallback-Modell zu konfigurieren.

### Funktionsweise

1. Bei jeder direkten Anfrage mit API-Schlüssel an `anthropic/claude-fable-5` sendet OpenClaw
   die serverseitige Fallback-Einwilligung von Anthropic: den
   Beta-Header `server-side-fallback-2026-06-01` sowie
   `fallbacks: [{"model": "claude-opus-4-8"}]`. Claude Opus 4.8 ist das einzige
   Fallback-Ziel, das Anthropic für Fable 5 zulässt.
2. Nur eine Ablehnung durch den Sicherheitsklassifikator löst den Fallback aus. Ratenbegrenzungen,
   Überlastungen und Serverfehler verhalten sich exakt wie zuvor und durchlaufen den normalen
   [Modell-Failover](/de/concepts/model-failover) von OpenClaw.
3. Die Wiederherstellung erfolgt innerhalb desselben Aufrufs. Eine Ablehnung vor jeglicher Ausgabe ist
   abgesehen von der Latenz nicht erkennbar; die gesamte Antwort stammt von Opus 4.8. Bei einer
   Ablehnung während des Streams bleibt der partielle Text als Präfix erhalten, an das das Fallback-Modell
   anknüpft, während die Schlussfolgerung und Tool-Aufrufe des ablehnenden Modells gemäß den
   Wiedergaberegeln von Anthropic verworfen werden (sie dürfen weder zurückgegeben noch
   ausgeführt werden).
4. Wenn Claude Opus 4.8 ebenfalls ablehnt, wird die Ablehnung für die Anfrage wie vor Einführung
   dieser Funktion als Fehler ausgegeben.

Der Fallback erfolgt auf Ebene der Anthropic-API, daher muss `claude-opus-4-8` nicht
in Ihrer konfigurierten Modellliste oder Fallback-Kette enthalten sein – ein für Fable geeigneter
API-Schlüssel kann Opus immer verwenden.

### Beobachtbarkeit und Abrechnung

- Eine per Fallback beantwortete Anfrage zeichnet in der Assistant-Nachricht eine Diagnose
  `provider_fallback` auf, die `fromModel` und `toModel` nennt, und
  `responseModel` der Nachricht meldet `claude-opus-4-8`.
- Anthropic rechnet pro Versuch ab: Eine Ablehnung vor der Ausgabe ist kostenlos, und die
  Wiederherstellung wird zu den Tarifen von Claude Opus 4.8 abgerechnet (derzeit die Hälfte der
  Tarife von Fable 5). Die Kostenschätzung von OpenClaw pro Anfrage berechnet per Fallback
  beantwortete Anfragen entsprechend zu Opus-Tarifen.
- Bei einer Ablehnung während des Streams berechnet Anthropic zusätzlich den bereits gestreamten
  Fable-Teil; dieser Anteil wird in der versuchsbezogenen Nutzung der API ausgewiesen, aber nicht
  in die Kostenschätzung von OpenClaw pro Anfrage einbezogen.

### Geltungsbereich

Gilt für `anthropic/claude-fable-5` mit API-Schlüssel-Authentifizierung gegenüber
`api.anthropic.com`. OAuth (Wiederverwendung des Claude-CLI-Abonnements), Proxy-Basis-URLs,
Bedrock-, Vertex- und Foundry-Anfragen bleiben unverändert und geben Ablehnungen dort
weiterhin als Fehler aus.

Live verifiziert: Eine harmlose Aufforderung an Fable 5, seine unverarbeitete Gedankenkette
wiederzugeben, wird ohne Fallbacks mit `category: "reasoning_extraction"` abgelehnt, während dieselbe
Aufforderung über OpenClaw eine normale, von Opus beantwortete Antwort mit angehängter
Diagnose `provider_fallback` zurückgibt.

Das zugrunde liegende Verhalten wird im Anthropic-[Leitfaden zu Ablehnungen und
Fallbacks](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
beschrieben.

## Prompt-Caching

OpenClaw unterstützt die Prompt-Caching-Funktion von Anthropic bei der Authentifizierung mit API-Schlüssel.

| Wert                | Cache-Dauer | Beschreibung                                      |
| ------------------- | ----------- | ------------------------------------------------- |
| `"short"` (Standard) | 5 Minuten   | Wird bei API-Schlüssel-Authentifizierung automatisch angewendet |
| `"long"`            | 1 Stunde    | Erweiterter Cache                                 |
| `"none"`            | Kein Caching | Prompt-Caching deaktivieren                       |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Cache-Überschreibungen pro Agent">
    Verwenden Sie Parameter auf Modellebene als Ausgangsbasis und überschreiben Sie anschließend bestimmte Agenten über `agents.entries.*.params`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    Reihenfolge der Konfigurationszusammenführung:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params` (passendes `id`, Überschreibung nach Schlüssel)

    Dadurch kann ein Agent einen langlebigen Cache beibehalten, während ein anderer Agent mit demselben Modell das Caching für stoßartigen Datenverkehr mit geringer Wiederverwendung deaktiviert.

  </Accordion>

  <Accordion title="Hinweise zu Claude auf Bedrock">
    - Anthropic-Claude-Modelle auf Bedrock (`amazon-bedrock/*anthropic.claude*`) akzeptieren bei entsprechender Konfiguration die Durchleitung von `cacheRetention`.
    - Bedrock-Modelle, die nicht von Anthropic stammen, werden zur Laufzeit auf `cacheRetention: "none"` gesetzt.
    - Intelligente Standardwerte für API-Schlüssel belegen außerdem `cacheRetention: "short"` für Claude-auf-Bedrock-Referenzen vor, wenn kein expliziter Wert festgelegt ist.

  </Accordion>
</AccordionGroup>

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Schnellmodus">
    Der gemeinsame Schalter `/fast` von OpenClaw setzt das Feld `service_tier` von Anthropic für direkten API-Schlüssel-Datenverkehr an `api.anthropic.com`.

    | Befehl | Zugeordnet zu |
    |--------|---------------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - Gilt nur für direkte `api.anthropic.com`-Anfragen mit einem API-Schlüssel. OAuth-/Abonnementtoken-Anfragen und Proxy-Routen erhalten niemals ein Feld `service_tier`.
    - Explizite Parameter `serviceTier` oder `service_tier` überschreiben `/fast`, wenn beide gesetzt sind.
    - Claude Opus 5 und Sonnet 5 unterstützen Priority Tier nicht, daher lässt OpenClaw `service_tier` bei diesen Modellen aus.
    - Bei Konten ohne Priority-Tier-Kapazität kann `service_tier: "auto"` zu `standard` aufgelöst werden.

    </Note>

  </Accordion>

  <Accordion title="Medienverständnis (Bild und PDF)">
    Das gebündelte Anthropic-Plugin registriert das Verständnis von Bildern und PDFs. OpenClaw
    ermittelt die Medienfunktionen automatisch aus der konfigurierten Anthropic-Authentifizierung;
    keine zusätzliche Konfiguration ist erforderlich.

    | Eigenschaft             | Wert                  |
    | ----------------------- | --------------------- |
    | Standardmodell          | `claude-opus-5`       |
    | Unterstützte Eingabe    | Bilder, PDF-Dokumente |

    Wenn ein Bild oder eine PDF-Datei an eine Unterhaltung angehängt wird, leitet OpenClaw
    sie automatisch über den Anthropic-Provider für Medienverständnis weiter.

  </Accordion>

  <Accordion title="1M-Kontextfenster">
    Claude Opus 5, Sonnet 5, Mythos 5 und Fable 5 verfügen über ein exaktes
    Eingabefenster mit 1.000.000 Token und unterstützen bis zu 128.000 Ausgabetoken.
    Das 1M-Kontextfenster von Anthropic ist außerdem für Claude-4.x-Modelle mit adaptivem
    Thinking allgemein verfügbar: Opus 4.8,
    Opus 4.7, Opus 4.6 und Sonnet 4.6. OpenClaw dimensioniert diese Modelle
    automatisch; `params.context1m` ist nicht erforderlich:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    Ältere Konfigurationen können `params.context1m: true` beibehalten; für
    diese Modelle ist dies eine harmlose wirkungslose Operation, und OpenClaw sendet den eingestellten
    Beta-Header `context-1m-2025-08-07` grundsätzlich nicht mehr. Ältere
    `anthropicBeta`-Konfigurationseinträge mit diesem Wert werden bei der Auflösung der Anfrage-Header
    entfernt, und nicht unterstützte ältere Claude-Modelle verwenden weiterhin ihr normales Kontextfenster.

    `params.context1m: true` verhält sich beim Claude-CLI-Backend
    (`claude-cli/*`) genauso: Berechtigte, allgemein verfügbare Opus- und Sonnet-Modelle erhalten das
    1M-Fenster bereits automatisch, sodass der Parameter auch dort optional ist.

    <Warning>
    Erfordert Zugriff auf lange Kontexte für Ihre Anthropic-Anmeldedaten. Bei der Authentifizierung über OAuth-/Abonnementtoken bleiben die erforderlichen Anthropic-Beta-Header erhalten, OpenClaw entfernt jedoch den eingestellten 1M-Beta-Header, falls er noch in einer älteren Konfiguration enthalten ist.
    </Warning>

  </Accordion>

  <Accordion title="1M-Kontext für Claude Opus 5">
    `anthropic/claude-opus-5` und seine Variante `claude-cli` verfügen standardmäßig über ein
    1M-Kontextfenster; `params.context1m: true` ist nicht erforderlich.
  </Accordion>
</AccordionGroup>

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="401-Fehler / Token plötzlich ungültig">
    Die Anthropic-Token-Authentifizierung läuft ab und kann widerrufen werden. Verwenden Sie für neue Einrichtungen stattdessen einen Anthropic-API-Schlüssel.
  </Accordion>

  <Accordion title='Kein API-Schlüssel für Provider "anthropic" gefunden'>
    Die Anthropic-Authentifizierung gilt **pro Agent**; neue Agenten übernehmen die Schlüssel des Hauptagenten nicht. Führen Sie das Onboarding für diesen Agenten erneut durch (oder konfigurieren Sie einen API-Schlüssel auf dem Gateway-Host) und überprüfen Sie die Konfiguration anschließend mit `openclaw models status`.
  </Accordion>

  <Accordion title='Keine Anmeldedaten für das Profil "anthropic:default" gefunden'>
    Führen Sie `openclaw models status` aus, um zu sehen, welches Authentifizierungsprofil aktiv ist. Führen Sie das Onboarding erneut durch oder konfigurieren Sie einen API-Schlüssel für diesen Profilpfad.
  </Accordion>

  <Accordion title="Kein verfügbares Authentifizierungsprofil (alle in der Abklingzeit)">
    Prüfen Sie `openclaw models status --json` auf `auth.unusableProfiles`. Abklingzeiten aufgrund von Anthropic-Ratenbegrenzungen können modellspezifisch sein, sodass ein anderes Anthropic-Modell möglicherweise weiterhin verwendet werden kann. Fügen Sie ein weiteres Anthropic-Profil hinzu oder warten Sie das Ende der Abklingzeit ab.
  </Accordion>
</AccordionGroup>

<Note>
Weitere Hilfe: [Fehlerbehebung](/de/help/troubleshooting) und [häufig gestellte Fragen](/de/help/faq).
</Note>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern und Modellreferenzen sowie Konfiguration des Failover-Verhaltens.
  </Card>
  <Card title="CLI-Backends" href="/de/gateway/cli-backends" icon="terminal">
    Einrichtung des Claude-CLI-Backends und Details zur Laufzeit.
  </Card>
  <Card title="Prompt-Caching" href="/de/reference/prompt-caching" icon="database">
    Funktionsweise des Prompt-Cachings bei verschiedenen Providern.
  </Card>
  <Card title="OAuth und Authentifizierung" href="/de/gateway/authentication" icon="key">
    Details zur Authentifizierung und Regeln für die Wiederverwendung von Anmeldedaten.
  </Card>
</CardGroup>
