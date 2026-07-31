---
read_when:
    - Sie möchten OpenAI-Modelle in OpenClaw verwenden
    - Sie möchten die Codex-Abonnementauthentifizierung anstelle von API-Schlüsseln verwenden
    - Sie benötigen ein strikteres Ausführungsverhalten für GPT-5-Agenten
summary: OpenAI über API-Schlüssel oder ein Codex-Abonnement in OpenClaw verwenden
title: OpenAI
x-i18n:
    generated_at: "2026-07-26T18:44:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw verwendet eine Provider-ID, `openai`, sowohl für die direkte Authentifizierung per API-Schlüssel als auch für die
ChatGPT/Codex-Abonnementauthentifizierung. `openai/*` ist die kanonische Modellroute.
Bei eingebetteten Agent-Durchläufen, für die keine Laufzeitrichtlinie oder `auto` festgelegt ist, bestimmen die Routendaten
von OpenAI, ob OpenClaw implizit die gebündelte Codex-App-Server-Laufzeit auswählen darf.
Das Präfix `openai/*` allein wählt keine Laufzeit aus.

- **Agent-Modelle** – `openai/*` über die durch die explizite
  `agentRuntime`-Konfiguration oder die implizite Routenrichtlinie von OpenAI ausgewählte Laufzeit. Melden Sie sich für die Nutzung eines ChatGPT/Codex-Abonnements
  mit der Codex-Authentifizierung an oder konfigurieren Sie ein Authentifizierungsprofil mit API-Schlüssel,
  wenn Sie eine schlüsselbasierte Abrechnung wünschen.
- **OpenAI-APIs ohne Agent** – direkter Zugriff auf die OpenAI Platform mit nutzungsabhängiger Abrechnung
  über `OPENAI_API_KEY` oder ein `openai`-Authentifizierungsprofil mit API-Schlüssel.
- **Legacy-Konfiguration** – Referenzen auf `codex/*` und `openai-codex/*` werden durch
  `openclaw doctor --fix` zu `openai/*` sowie dem modellspezifischen `agentRuntime.id: "codex"`
  repariert.

OpenAI unterstützt ausdrücklich die OAuth-Nutzung von Abonnements in externen Tools und
Workflows wie OpenClaw.

## Nutzungs- und Kostenverfolgung

OpenClaw behandelt Abonnementkontingente und die Abrechnung der Platform API getrennt:

- ChatGPT/Codex OAuth zeigt den Abonnementtarif, die Kontingentzeiträume und das Guthaben an.
- `OPENAI_ADMIN_KEY` zeigt in der Control UI unter **Nutzung** 30 Tage der vom Provider gemeldeten Organisationskosten und Completions-Nutzung an, einschließlich täglicher Ausgaben, Anfragen-/Token-Gesamtsummen, meistgenutzter Modelle und Kostenkategorien.
- `OPENAI_PROJECT_ID` beschränkt den Verlauf der Admin API optional auf ein Projekt.
- OpenClaw sendet niemals `OPENAI_API_KEY` oder ein `openai`-Inferenzprofil an Organisations-APIs; diese Anmeldedaten können zu benutzerdefinierten, Azure- oder agentenlokalen Endpunkten gehören.

Ein expliziter Admin-Schlüssel hat Vorrang vor OAuth. Der vom Provider gemeldete Verlauf wird nicht mit den aus OpenClaw-Sitzungen abgeleiteten geschätzten Kosten zusammengeführt; er kann API-Aktivitäten anderer Clients und anbieterseitige Abrechnungsanpassungen enthalten.

Die Dokumentation zum [API-Nutzungs-Dashboard](https://help.openai.com/en/articles/10478918) von OpenAI beschreibt die Anforderungen hinsichtlich Organisationseigentümerschaft und expliziter Berechtigungen für das Usage Dashboard für den Zugriff auf Nutzungsdaten.

Provider, Modell, Laufzeit und Kanal sind separate Ebenen. Wenn diese Bezeichnungen
durcheinandergeraten, lesen Sie [Agent-Laufzeiten](/de/concepts/agent-runtimes), bevor Sie
die Konfiguration ändern.

## Schnellauswahl

| Ziel                                              | Verwenden                                                           | Hinweise                                                             |
| ------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------- |
| ChatGPT/Codex-Abonnement, native Codex-Laufzeit   | `openai/gpt-5.6-sol`                                                   | Neue Abonnementeinrichtung; mit Codex-Authentifizierung anmelden.    |
| Direkte API-Schlüssel-Abrechnung für Agent-Durchläufe | `openai/gpt-5.6` plus ein geordnetes Authentifizierungsprofil mit API-Schlüssel | Neue API-Schlüssel-Einrichtung; die reine Direkt-API-ID wird als Sol aufgelöst. |
| Eine genaue GPT-5.6-Stufe auswählen               | `openai/gpt-5.6-sol`, `-terra` oder `-luna`      | Prüfen Sie mit `models list`, welche Stufen für dieses Konto verfügbar sind. |
| Konto ohne Zugriff auf GPT-5.6                    | `openai/gpt-5.5`                                                   | Explizite Wiederherstellungsoption; OpenClaw führt kein stilles Downgrade durch. |
| Direkte API-Schlüssel-Abrechnung, explizite OpenClaw-Laufzeit | `openai/gpt-5.6` plus Provider/Modell `agentRuntime.id: "openclaw"` | Wählen Sie ein normales `openai`-Authentifizierungsprofil mit API-Schlüssel. |
| Neuester Alias des ChatGPT-Instant-Modells        | `openai/chat-latest`                                                   | Nur direkter API-Schlüssel; veränderlicher Alias, nicht der stabile Standardwert. |
| Bilderzeugung oder -bearbeitung                    | `openai/gpt-image-2`                                                   | Funktioniert mit `OPENAI_API_KEY` oder Codex OAuth.                |
| Bilder mit transparentem Hintergrund              | `openai/gpt-image-1.5`                                                   | Setzen Sie `outputFormat` auf `png` oder `webp` und `background=transparent`. |

## Zuordnung der Bezeichnungen

| Angezeigter Name                         | Ebene              | Bedeutung                                                                                |
| ---------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| `openai`                      | Provider-Präfix    | Kanonische OpenAI-Modellroute; die Routendaten bestimmen die implizite Laufzeit.         |
| `codex`-Plugin               | Plugin             | Gebündeltes Plugin, das die native Codex-App-Server-Laufzeit und die `/codex`-Chat-Steuerelemente bereitstellt. |
| Provider/Modell `agentRuntime.id: codex`      | Agent-Laufzeit     | Erzwingt das native Codex-App-Server-Harness für passende eingebettete Durchläufe.        |
| `/codex ...`                      | Chat-Befehlssatz   | Bindet/steuert Codex-App-Server-Threads aus einer Unterhaltung heraus.                    |
| `runtime: "acp", agentId: "codex"`                      | ACP-Sitzungsroute  | Expliziter Ausweichpfad, der Codex über ACP/acpx ausführt.                                |

## Implizite Agent-Laufzeit

Wenn die Provider/Modell-Richtlinie `agentRuntime` nicht festgelegt oder auf `auto` gesetzt ist, wählt die
providereigene Routenrichtlinie von OpenAI die implizite Laufzeit anhand des effektiven
Endpunkts und Adapters aus:

| Effektive Routendaten                                                                                                                                                  | Implizite Laufzeit    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Exakter offizieller Platform-HTTPS-Endpunkt mit `openai-responses` oder exakter offizieller ChatGPT-HTTPS-Endpunkt mit `openai-chatgpt-responses`; keine explizit angegebene Anfrageüberschreibung | Codex kann ausgewählt werden |
| Explizit angegebener `openai-completions`-Adapter                                                                                                                        | OpenClaw              |
| Benutzerdefinierter Endpunkt                                                                                                                                           | OpenClaw              |
| Expliziter exakter offizieller Endpunkt mit HTTP                                                                                                                       | Abgelehnt             |
| Route mit einer explizit angegebenen Provider/Modell-Anfrageüberschreibung                                                                                             | OpenClaw              |

Eine explizite, vom Standard abweichende Provider/Modell-Einstellung `agentRuntime.id` bleibt maßgeblich.
Beispielsweise hält `agentRuntime.id: "openclaw"` eine ansonsten für Codex geeignete
Route auf OpenClaw, während `agentRuntime.id: "codex"` Codex voraussetzt und
geschlossen fehlschlägt, wenn die effektive Route nicht als Codex-kompatibel deklariert ist.
Die Laufzeitauswahl ändert weder den Anmeldedatentyp noch die Abrechnung: Die Authentifizierung
per Platform-API-Schlüssel und die ChatGPT/Codex-Abonnementauthentifizierung bleiben getrennt.

`openclaw doctor --fix` migriert Legacy-Modellreferenzen auf `codex/*` und `openai-codex/*`,
Legacy-IDs von Codex-Authentifizierungsprofilen und Legacy-Einträge der Codex-Authentifizierungsreihenfolge zur
kanonischen Route `openai`. Migrierte Modellreferenzen erhalten das modellspezifische
`agentRuntime.id: "codex"`; verwenden Sie `auth.order.openai` für neue Konfigurationen der Authentifizierungsreihenfolge.

<Note>
Bei einer neuen OpenAI-Einrichtung wird nur dann ein primäres GPT-5.6-Modell festgelegt, wenn kein primäres Modell
konfiguriert ist. Das Hinzufügen oder Aktualisieren der OpenAI-Authentifizierung behält eine vorhandene explizite
Auswahl einschließlich `openai/gpt-5.5` bei, sofern Sie nicht ausdrücklich
`models auth login --set-default` oder `models set` verwenden. Verwenden Sie ein Authentifizierungsprofil mit API-Schlüssel
nur, wenn Sie für ein Agent-Modell eine Authentifizierung per API-Schlüssel wünschen.
</Note>

## Eingeschränkte Vorschau von GPT-5.6

OpenClaw erkennt die exakten Modell-IDs `openai/gpt-5.6-sol`,
`openai/gpt-5.6-terra` und `openai/gpt-5.6-luna`. Alle drei bieten im aktuellen Katalog
`xhigh`- und `max`-Reasoning. OpenAI beschreibt Sol als
Flaggschiff-Stufe, Terra als ausgewogene Stufe und Luna als schnelle,
kostengünstigere Stufe. Siehe die
[Ankündigung zur Einführung von GPT-5.6](https://openai.com/index/previewing-gpt-5-6-sol/)
und den [Zugriffsleitfaden](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna).

Bei direkter OpenAI-Authentifizierung per API-Schlüssel ist die reine ID `openai/gpt-5.6` ein Alias für
Sol und der Standardwert bei einer neuen Einrichtung. Der native Codex-Katalog wendet
diesen Direkt-API-Alias nicht clientseitig an; abhängig vom Workspace-Zugriff kann er
die exakten Sol-, Terra- und Luna-IDs anzeigen. Eine neue ChatGPT/Codex-OAuth-Einrichtung verwendet daher
`openai/gpt-5.6-sol`. Prüfen Sie das aktuelle Konto mit:

```bash
openclaw models list --provider openai
```

Der Zugriff der API-Organisation und des Codex-Workspace kann unterschiedlich sein. Wenn GPT-5.6 nicht
verfügbar ist, wählen Sie GPT-5.5 explizit aus:

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw zeigt den vorgelagerten Zugriffsfehler an und ersetzt eine
GPT-5.6-Auswahl nicht stillschweigend durch GPT-5.5.

<Note>
Geeignete exakte offizielle HTTPS-Routen können das gebündelte Codex-App-Server-Plugin
auswählen, wenn keine Laufzeitrichtlinie festgelegt oder diese auf `auto` gesetzt ist; explizit angegebene Completions-Routen,
benutzerdefinierte Endpunkte und Überschreibungen des Anfragetransports verbleiben auf OpenClaw. Offizielle
Klartext-HTTP-Endpunkte werden abgelehnt. Eine explizite Provider/Modell-Laufzeitkonfiguration bleibt
maßgeblich. Führen Sie `openclaw doctor --fix` aus, um veraltete Legacy-Codex-Modellreferenzen,
`codex-cli/*`-Referenzen oder alte Laufzeit-Sitzungsbindungen zu reparieren, die nicht durch eine
explizite Laufzeitkonfiguration festgelegt wurden.
</Note>

## Funktionsumfang von OpenClaw

| OpenAI-Funktion                    | OpenClaw-Oberfläche                                                                            | Status                                                                     |
| --------------------------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Chat / Responses                  | `openai/<model>`-Modell-Provider                                                             | Ja                                                                         |
| Codex-Abonnementmodelle           | `openai/<model>` mit OpenAI OAuth                                                            | Ja                                                                         |
| Veraltete Codex-Modellreferenzen  | alte Codex-Modellreferenzen, `codex-cli/<model>`                                                | Durch doctor zu `openai/<model>` repariert                               |
| Codex-App-Server-Harness          | Codex-kompatible HTTPS-Route mit nicht gesetzter Runtime/`auto` oder explizitem `agentRuntime.id: codex` | Ja                                                            |
| Serverseitige Websuche            | Natives OpenAI-Responses-Tool                                                                  | Ja, wenn die Websuche aktiviert und kein anderer Provider festgelegt ist   |
| Bilder                            | `image_generate`                                                                             | Ja                                                                         |
| Videos                            | `video_generate`                                                                             | Ja                                                                         |
| Text-to-Speech                    | `tts.provider: "openai"` / `tts`                                                       | Ja                                                                         |
| Batch-Speech-to-Text              | `tools.media.audio` / Medienverständnis                                                         | Ja                                                                         |
| Streaming-Speech-to-Text          | Voice Call `streaming.provider: "openai"`                                                                  | Ja                                                                         |
| Echtzeit-Sprache                  | Voice Call `realtime.provider: "openai"` / Control UI Talk `talk.realtime.provider: "openai"`                            | Ja (OpenAI-Platform-API-Schlüssel)                                         |
| Einbettungen                      | Provider für Speichereinbettungen                                                              | Ja                                                                         |

<Note>
OpenAI-Echtzeit-Sprache läuft über die öffentliche **OpenAI Platform Realtime
API** und erfordert einen Platform-API-Schlüssel. Codex-OAuth-Token authentifizieren
stattdessen das ChatGPT-Codex-Backend; sie sind nicht mit Platform-API-
Schlüsseln für die öffentlichen Realtime-Endpunkte austauschbar.

Wenn die Authentifizierung per API-Schlüssel eine fehlende Abrechnung meldet, laden Sie unter
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
Platform-Guthaben für die Organisation auf, der Ihre Echtzeit-Anmeldedaten
zugeordnet sind. Echtzeit-Sprache akzeptiert das durch
`openclaw onboard --auth-choice openai-api-key` erstellte API-Schlüssel-Authentifizierungsprofil `openai`, einen über
`talk.realtime.providers.openai.apiKey` für Control UI Talk festgelegten Platform-API-Schlüssel oder
`plugins.entries.voice-call.config.realtime.providers.openai.apiKey` für Voice
Call oder die Umgebungsvariable `OPENAI_API_KEY`.

In Control UI Video Talk empfängt OpenAI WebRTC den Kamerakontext bei Bedarf:
Wenn das Modell `describe_view` aufruft, sendet der Browser ein begrenztes JPEG über
den Echtzeit-Datenkanal. OpenClaw fügt der OpenAI-Sitzung keine kontinuierliche
Kameraspur hinzu.
</Note>

## Speichereinbettungen

OpenClaw kann OpenAI oder einen OpenAI-kompatiblen Einbettungsendpunkt für die
`memory_search`-Indizierung und Abfrageeinbettungen verwenden:

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

Legen Sie für OpenAI-kompatible Endpunkte, die asymmetrische Einbettungsbezeichnungen erfordern,
`queryInputType` und `documentInputType` unter `memory.search` fest. OpenClaw
leitet diese als providerspezifische `input_type`-Anfragefelder weiter:
Abfrageeinbettungen verwenden `queryInputType`; indizierte Speicherabschnitte und die Batch-Indizierung verwenden
`documentInputType`. Das vollständige Beispiel finden Sie in der
[Referenz zur Speicherkonfiguration](/de/reference/memory-config#provider-specific-config).

## Erste Schritte

<Tabs>
  <Tab title="API-Schlüssel (OpenAI Platform)">
    **Am besten geeignet für:** direkten API-Zugriff und nutzungsbasierte Abrechnung.

    <Steps>
      <Step title="API-Schlüssel abrufen">
        Erstellen oder kopieren Sie einen API-Schlüssel im [OpenAI-Platform-Dashboard](https://platform.openai.com/api-keys).
      </Step>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Oder übergeben Sie den Schlüssel direkt:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="Verfügbarkeit des Modells überprüfen">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### Routenzusammenfassung

    | Modellreferenz     | Runtime-Richtlinie oder Routenfakten                              | Route                       | Authentifizierung                       |
    | ------------------ | ----------------------------------------------------------------- | --------------------------- | --------------------------------------- |
    | `openai/gpt-5.6` | nicht gesetzt/`auto`, exakte offizielle native HTTPS-Route, keine Anfrageüberschreibung | Codex kann ausgewählt werden | Geordnetes API-Schlüssel-Authentifizierungsprofil |
    | `openai/gpt-5.6` | Provider/Modell `agentRuntime.id: "openclaw"`                                | Eingebettete OpenClaw-Runtime | Ausgewähltes `openai`-API-Schlüsselprofil |
    | `openai/gpt-5.5` | expliziter Provider/explizites Modell `agentRuntime.id`          | Ausgewählte Agent-Runtime   | Ausgewähltes OpenAI-API-Schlüsselprofil |
    | `openai/*` | erstellte Completions, benutzerdefinierte Route oder Anfrageüberschreibung | Eingebettete OpenClaw-Runtime | Anmeldedatentyp bleibt unverändert |
    | `openai/*` | offizieller Klartext-HTTP-Endpunkt                                | Abgelehnt                   | Anmeldedaten werden nicht gesendet      |

    <Note>
    Wenn die Runtime nicht gesetzt ist oder `auto` verwendet wird, darf nur eine geeignete
    exakte offizielle native HTTPS-Route implizit den Codex-App-Server-Harness auswählen.
    Erstellen Sie für die Authentifizierung per API-Schlüssel bei einem Agent-Modell ein
    `openai`-API-Schlüssel-Authentifizierungsprofil und ordnen Sie es mit
    `auth.order.openai`; `OPENAI_API_KEY` bleibt der direkte Fallback für
    OpenAI-API-Oberflächen ohne Agent. Führen Sie `openclaw doctor --fix` aus, um ältere
    veraltete Codex-Einträge der Authentifizierungsreihenfolge zu migrieren.
    </Note>

    ### Konfigurationsbeispiel

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    Die einfache direkte API-ID `gpt-5.6` wird der Sol-Stufe zugeordnet. Falls diese API-
    Organisation GPT-5.6 nicht bereitstellt, legen Sie das primäre Modell explizit auf
    `openai/gpt-5.5` fest.

    Um das aktuelle Instant-Modell von ChatGPT über die OpenAI API auszuprobieren, legen Sie das Modell
    auf `openai/chat-latest` fest:

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` ist ein dynamischer Alias. Eine neue Einrichtung mit OpenAI-API-Schlüssel verwendet stattdessen
    `openai/gpt-5.6`, dessen einfache direkte API-ID der Sol-Stufe zugeordnet wird. Vorhandene
    explizite primäre Modelle, einschließlich `openai/gpt-5.5`, bleiben unverändert. Der
    Alias `chat-latest` akzeptiert nur die Textausführlichkeit `medium`; OpenClaw erzwingt
    für dieses Modell bei jeder anderen angeforderten Ausführlichkeit `medium`.

    <Warning>
    OpenClaw stellt `gpt-5.3-codex-spark` **nicht** über die direkte Route mit OpenAI-
    API-Schlüssel bereit. Es ist nur über Katalogeinträge des Codex-Abonnements
    verfügbar, wenn es für Ihr angemeldetes Konto freigeschaltet ist.
    </Warning>

  </Tab>

  <Tab title="Codex-Abonnement">
    **Am besten geeignet für:** die Verwendung Ihres ChatGPT-/Codex-Abonnements mit nativer Codex-
    App-Server-Ausführung anstelle eines separaten API-Schlüssels. Codex Cloud erfordert
    die Anmeldung bei ChatGPT.

    <Steps>
      <Step title="Codex OAuth ausführen">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        Oder führen Sie OAuth direkt aus:

        ```bash
        openclaw models auth login --provider openai
        ```

        Fügen Sie bei Headless-Einrichtungen oder Einrichtungen, bei denen Callbacks nicht möglich sind, `--device-code` hinzu, um sich
        mithilfe eines ChatGPT-Gerätecode-Ablaufs anstelle des Browser-
        Callbacks auf localhost anzumelden:

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="Kanonische OpenAI-Modellroute verwenden">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        Für diese exakte offizielle native HTTPS-Route ist keine Runtime-Konfiguration
        erforderlich. Sie kann die Codex-App-Server-Runtime automatisch auswählen, und
        OpenClaw installiert oder repariert das mitgelieferte Codex-Plugin, wenn diese Runtime
        ausgewählt wird.
      </Step>
      <Step title="Verfügbarkeit der Codex-Authentifizierung überprüfen">
        ```bash
        openclaw models list --provider openai
        ```

        Senden Sie nach dem Start des Gateways `/codex status` oder `/codex models`
        im Chat, um die native App-Server-Runtime zu überprüfen.
      </Step>
    </Steps>

    ### Routenzusammenfassung

    | Modellreferenz                   | Runtime-Richtlinie oder Routenfakten                              | Route                                                       | Authentifizierung                                      |
    | -------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
    | `openai/gpt-5.6-sol`               | nicht gesetzt/`auto`, exakte offizielle native HTTPS-Route, keine Anfrageüberschreibung | Codex kann ausgewählt werden | Codex-Anmeldung oder ein geordnetes `openai`-Authentifizierungsprofil |
    | `openai/gpt-5.6-terra`               | nicht gesetzt/`auto`, exakte offizielle native HTTPS-Route, keine Anfrageüberschreibung | Codex kann ausgewählt werden | Codex-Anmeldung, wenn der Katalog Terra bereitstellt   |
    | `openai/gpt-5.6-luna`               | nicht gesetzt/`auto`, exakte offizielle native HTTPS-Route, keine Anfrageüberschreibung | Codex kann ausgewählt werden | Codex-Anmeldung, wenn der Katalog Luna bereitstellt    |
    | `openai/gpt-5.6-sol`               | Provider/Modell `agentRuntime.id: "openclaw"`                                | Eingebettete OpenClaw-Runtime, interner Codex-Authentifizierungstransport | Ausgewähltes `openai`-OAuth-Profil |
    | `openai/gpt-5.5`               | expliziter Provider/explizites Modell `agentRuntime.id`          | Ausgewählte Agent-Runtime                                  | Ausgewähltes OpenAI-Authentifizierungsprofil           |
    | `openai/*`               | erstellte Completions, benutzerdefinierte Route oder Anfrageüberschreibung | Eingebettete OpenClaw-Runtime                       | Anforderung an Anmeldedaten bleibt routenspezifisch    |
    | `openai/*`               | offizieller Klartext-HTTP-Endpunkt                                | Abgelehnt                                                  | Anmeldedaten werden nicht gesendet                     |
    | Veraltete Codex-GPT-5.5-Referenz | durch doctor repariert                                            | In `openai/gpt-5.5` umgeschrieben                        | Migriertes OpenAI-OAuth-Profil                         |
    | `codex-cli/gpt-5.5`               | durch doctor repariert                                            | In `openai/gpt-5.5` umgeschrieben                        | Codex-App-Server-Authentifizierung                     |

    <Warning>
    Bei einer neuen, abonnementgestützten Einrichtung wird exakt `openai/gpt-5.6-sol` verwendet; der
    native Codex-Katalog kann außerdem exakte Terra- oder Luna-Referenzen bereitstellen. Wenn das
    Konto GPT-5.6 nicht bereitstellt, wählen Sie ausdrücklich `openai/gpt-5.5`. Ältere
    Codex-GPT-Referenzen sind veraltete OpenClaw-Routen, nicht der native Codex-Laufzeitpfad;
    führen Sie `openclaw doctor --fix` aus, um sie zu migrieren, ohne eine
    bestehende explizite GPT-5.5-Auswahl zu aktualisieren. `gpt-5.3-codex-spark` bleibt auf
    Konten beschränkt, deren Codex-Abonnementkatalog es aufführt; direkte OpenAI-
    API-Schlüssel- und Azure-Referenzen dafür bleiben unterdrückt.
    </Warning>

    <Note>
    Neue Konfigurationen sollten die Authentifizierungsreihenfolge für OpenAI-Agenten unter `auth.order.openai` ablegen;
    Doctor migriert ältere Einträge der veralteten Codex-Authentifizierungsreihenfolge.
    </Note>

    ### Konfigurationsbeispiel

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    Behalten Sie bei einem API-Schlüssel als Ausweichoption das ausgewählte Modell unter `openai/*` bei und legen Sie
    die Authentifizierungsreihenfolge unter `openai` ab. OpenClaw versucht zuerst das Abonnement und anschließend
    den API-Schlüssel, während es weiterhin das Codex-Harness verwendet:

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    Das Onboarding importiert kein OAuth-Material mehr aus `~/.codex`. Melden Sie sich mit
    Browser-OAuth (Standard) oder dem oben beschriebenen Gerätecode-Ablauf an; OpenClaw verwaltet die
    resultierenden Anmeldedaten in seinem eigenen Agent-Authentifizierungsspeicher.
    </Note>

    ### Codex-OAuth-Routing prüfen und wiederherstellen

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    Fügen Sie für einen bestimmten Agenten `--agent <id>` hinzu:

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    Wenn eine ältere Konfiguration noch veraltete Codex-GPT-Referenzen oder eine überholte OpenAI-
    Laufzeitsitzungsbindung ohne explizite Laufzeitkonfiguration enthält, reparieren Sie sie:

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    Wenn `models auth list --provider openai` kein verwendbares Profil anzeigt, melden Sie sich
    erneut an:

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    Verwenden Sie `--profile-id` für mehrere Codex-OAuth-Anmeldungen im selben Agenten und
    steuern Sie sie anschließend über die Authentifizierungsreihenfolge oder `/model ...@<profileId>`:

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    Führen Sie `openclaw doctor --fix` aus, um ältere veraltete OpenAI-Codex-Präfix-
    Profil-IDs und Reihenfolgeneinträge zu migrieren, bevor Sie sich auf die Profilreihenfolge verlassen.

    ### Statusanzeige

    Im Chat zeigt `/status`, welche Modelllaufzeit für die aktuelle
    Sitzung aktiv ist. Das gebündelte Codex-App-Server-Harness wird als
    `Runtime: OpenAI Codex` angezeigt, wenn eine geeignete implizite Route oder eine explizite
    Provider-/Modelllaufzeitrichtlinie es auswählt.

    ### Doctor-Warnung

    Wenn veraltete Codex-Modellreferenzen oder überholte OpenAI-Laufzeitbindungen in der Konfiguration
    oder im Sitzungsstatus verbleiben, schreibt `openclaw doctor --fix` sie mit
    der Codex-Laufzeit in `openai/*` um, sofern OpenClaw nicht explizit konfiguriert ist.

    ### Standardwerte für Kontextfenster und optionale Aktivierung langer Kontexte

    OpenClaw behandelt die native Modellkapazität und das aktive Laufzeitbudget als
    separate Werte:

    - `contextWindow` deklariert das gesamte Modellfenster des Providers.
    - `contextTokens` begrenzt, wie viel von diesem Fenster OpenClaw für aktive Eingaben verwendet.

    ChatGPT-/Codex-OAuth folgt dem aktuellen Codex-Kontokatalog. Der aktuelle
    Katalog weist für GPT-5.6 üblicherweise ein aktives Fenster von `272000` Token aus.
    Direkte GPT-5.5- und GPT-5.6-Modelle mit API-Schlüssel verwenden ebenfalls standardmäßig `272000`
    `contextTokens`, obwohl die Platform API ein größeres natives
    Fenster bereitstellt. Dadurch bleiben das normale Latenz-, Qualitäts- und Kostenprofil
    über alle Authentifizierungsmodi hinweg konsistent. Ein konfigurierter Wert für `agents.defaults.contextTokens` kann
    dieses Budget weiter reduzieren, aber ein Modell nicht über seine konfigurierte
    Obergrenze `contextTokens` hinaus anheben.

    Für direkte GPT-5.5- und GPT-5.6-Nutzung mit API-Schlüssel dokumentiert OpenAI ein Provider-Fenster von `1050000`
    Token und maximal `128000` Ausgabe-Token. Wenn die
    vollständige Ausgabekapazität reserviert wird, verbleiben `922000` Token für Eingaben. Dies ist ein abgeleitetes
    Betriebsbudget und kein separat vom Provider veröffentlichtes Eingabelimit. Siehe den
    offiziellen [Modellvergleich](https://developers.openai.com/api/docs/models/compare)
    und die [GPT-5.5-Modellseite](https://developers.openai.com/api/docs/models/gpt-5.5).
    Das folgende Beispiel aktiviert diese Kapazität für ein Terra-Modell und weist
    OpenAI an, bei `700000` aktiven Token eine Compaction durchzuführen:

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    `agentRuntime.id: "openclaw"` ist in diesem Beispiel beabsichtigt. Es belegt, dass der
    eingebettete OpenClaw-Responses-Pfad die oben angegebenen Modellmetadaten und serverseitigen
    Compaction-Einstellungen verwendet. Ein Thread des nativen Codex-Harness verwaltet sein Kontextbudget
    stattdessen in der Codex-Konfiguration; siehe
    [Langer Kontext im Codex-Harness](/de/plugins/codex-harness#direct-api-long-context).

    <Warning>
    OpenAI berechnet höhere Preise für lange Kontexte, sobald eine GPT-5.5- oder GPT-5.6-
    Anfrage `272000` Eingabe-Token überschreitet: Die gesamte qualifizierte Anfrage wird
    mit dem 2-Fachen Eingabe- und dem 1,5-Fachen Ausgabepreis abgerechnet. Große Prompts werden über
    mehrere Durchläufe hinweg erneut gesendet oder komprimiert, sodass eine optional aktivierte Sitzung erheblich mehr
    als die Standardeinstellung kosten kann, selbst wenn die sichtbare Antwort kurz ist. Siehe
    [OpenAI-API-Preise](https://developers.openai.com/api/docs/pricing). Die API
    bleibt die maßgebliche Quelle für Kontozugriff, tatsächliche Limits und Abrechnung.
    </Warning>

    ### Katalogwiederherstellung

    OpenClaw verwendet vorgelagerte Codex-Katalogmetadaten für `gpt-5.5`, wenn diese
    vorhanden sind. Wenn die Live-Codex-Erkennung die Zeile `gpt-5.5` auslässt, obwohl das Konto
    authentifiziert ist, erzeugt OpenClaw diese OAuth-Modellzeile, damit Cron-,
    Sub-Agent- und konfigurierte Standardmodellläufe nicht mit
    `Unknown model` fehlschlagen.

  </Tab>
</Tabs>

## Authentifizierung des nativen Codex-App-Servers

Das native Codex-App-Server-Harness verwendet `openai/*`-Modellreferenzen, wenn eine geeignete
exakte offizielle HTTPS-Route es implizit auswählt oder wenn Provider-/Modell-
`agentRuntime.id: "codex"` es explizit auswählt. Seine Authentifizierung bleibt
kontobasiert. OpenClaw wählt die Authentifizierung in dieser Reihenfolge aus:

1. Geordnete OpenAI-Authentifizierungsprofile für den Agenten, vorzugsweise unter
   `auth.order.openai`. Führen Sie `openclaw doctor --fix` aus, um ältere veraltete
   Codex-Authentifizierungsprofil-IDs und die Authentifizierungsreihenfolge zu migrieren.
2. Das bestehende Konto des App-Servers, etwa eine lokale ChatGPT-
   Anmeldung der Codex CLI. Für das standardmäßige isolierte Agent-Home bindet OpenClaw dieses native
   CLI-Konto über dessen Anmelde-RPC in den App-Server ein; die Konfiguration, Plugins
   und der Thread-Speicher der CLI werden nicht gemeinsam verwendet.
3. Nur für lokale stdio-App-Server-Starts und nur, wenn der App-Server
   kein Konto meldet: `CODEX_API_KEY`, danach `OPENAI_API_KEY`.

Eine lokale Anmeldung mit einem ChatGPT-/Codex-Abonnement wird nicht allein deshalb ersetzt, weil der
Gateway-Prozess außerdem `OPENAI_API_KEY` für direkte OpenAI-Modelle oder
Einbettungen enthält. Die Ausweichoption über den API-Schlüssel aus der Umgebung gilt nur für den lokalen stdio-Pfad ohne Konto;
sie wird niemals über WebSocket-App-Server-Verbindungen gesendet. Wenn ein
abonnementartiges Codex-Profil ausgewählt ist, hält OpenClaw außerdem
`CODEX_API_KEY` und `OPENAI_API_KEY` aus dem erzeugten stdio-App-Server-Unterprozess
heraus und sendet die ausgewählten Anmeldedaten stattdessen über den Anmelde-RPC des App-Servers.

Wenn dieses Abonnementprofil durch ein Codex-Nutzungslimit blockiert wird, markiert OpenClaw
das Profil bis zur von Codex angegebenen Rücksetzzeit als blockiert und ermöglicht der
Authentifizierungsreihenfolge, zum nächsten `openai:*`-Profil zu wechseln, ohne das ausgewählte
Modell zu ändern oder das Codex-Harness zu verlassen. Nach Ablauf der Rücksetzzeit kann das
Abonnementprofil wieder verwendet werden.

## Bilderzeugung

Das gebündelte Plugin `openai` registriert die Bilderzeugung über das
Werkzeug `image_generate`. Es unterstützt sowohl die Bilderzeugung mit OpenAI-API-Schlüssel als auch
mit Codex-OAuth über dieselbe Modellreferenz `openai/gpt-image-2`.

| Funktion                  | OpenAI-API-Schlüssel               | Codex-OAuth                          |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| Modellreferenz            | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| Authentifizierung         | `OPENAI_API_KEY`                   | OpenAI-Codex-OAuth-Anmeldung          |
| Transport                 | OpenAI Images API                  | Codex-Responses-Backend               |
| Max. Bilder pro Anfrage   | 4                                  | 4                                    |
| Bearbeitungsmodus         | Aktiviert (bis zu 5 Referenzbilder)| Aktiviert (bis zu 5 Referenzbilder)   |
| Größenüberschreibungen    | Unterstützt, einschließlich 2K-/4K-Größen | Unterstützt, einschließlich 2K-/4K-Größen |
| Seitenverhältnis/Auflösung | Nicht an OpenAI Images API weitergeleitet | Wenn sicher, einer unterstützten Größe zugeordnet |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
Unter [Bilderzeugung](/de/tools/image-generation) finden Sie gemeinsame Werkzeugparameter,
die Provider-Auswahl und das Failover-Verhalten.
</Note>

`gpt-image-2` ist der Standard für die Text-zu-Bild-Erzeugung und Bildbearbeitung mit OpenAI.
`gpt-image-1.5`, `gpt-image-1` und `gpt-image-1-mini` können weiterhin
als explizite Modellüberschreibungen verwendet werden. Verwenden Sie `openai/gpt-image-1.5` für
PNG-/WebP-Ausgaben mit transparentem Hintergrund; die aktuelle `gpt-image-2`-API lehnt
`background: "transparent"` ab.

Rufen Sie für eine Anfrage mit transparentem Hintergrund `image_generate` mit
`model: "openai/gpt-image-1.5"`, `outputFormat: "png"` oder `"webp"` sowie
`background: "transparent"` auf; die ältere Provider-Option `openai.background` wird
weiterhin akzeptiert. OpenClaw schützt außerdem die öffentlichen OpenAI- und OpenAI-Codex-OAuth-
Routen, indem standardmäßige transparente `openai/gpt-image-2`-Anfragen in
`gpt-image-1.5` umgeschrieben werden; Azure- und benutzerdefinierte OpenAI-kompatible Endpunkte behalten ihre
konfigurierten Bereitstellungs-/Modellnamen bei.

Dieselbe Einstellung ist für Headless-CLI-Läufe verfügbar:

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "Ein einfacher roter Kreisaufkleber auf transparentem Hintergrund" \
  --json
```

Verwenden Sie dieselben Flags `--output-format` und `--background` mit
`openclaw infer image edit`, wenn Sie mit einer Eingabedatei beginnen.
`--openai-background` bleibt als OpenAI-spezifischer Alias verfügbar. Verwenden Sie
`--quality low|medium|high|auto`, um Qualität und Kosten von OpenAI Images zu steuern.
Verwenden Sie `--openai-moderation low|auto`, um den Moderationshinweis von OpenAI entweder aus
`image generate` oder `image edit` zu übergeben.

Für ChatGPT-/Codex-OAuth-Installationen ist dieselbe `openai/gpt-image-2`-Referenz beizubehalten. Wenn
ein `openai`-OAuth-Profil konfiguriert ist, löst OpenClaw das gespeicherte OAuth-
Zugriffstoken auf und sendet Bildanfragen über das Codex-Responses-Backend; es
versucht nicht zuerst `OPENAI_API_KEY` und greift auch nicht stillschweigend auf einen API-Schlüssel zurück.
Konfigurieren Sie stattdessen `models.providers.openai` ausdrücklich mit einem API-Schlüssel, einer benutzerdefinierten Basis-
URL oder einem Azure-Endpunkt, wenn Sie die direkte Route über die OpenAI Images API
verwenden möchten. Wenn sich dieser benutzerdefinierte Bildendpunkt unter einer vertrauenswürdigen LAN-/privaten Adresse befindet,
legen Sie außerdem `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` fest; OpenClaw
blockiert private/interne OpenAI-kompatible Bildendpunkte, sofern diese
explizite Zustimmung nicht vorliegt.

Generieren:

```
/tool image_generate model=openai/gpt-image-2 prompt="Ein professionelles Launch-Poster für OpenClaw unter macOS" size=3840x2160 count=1
```

Ein transparentes PNG generieren:

```
/tool image_generate model=openai/gpt-image-1.5 prompt="Ein einfacher roter kreisförmiger Aufkleber auf transparentem Hintergrund" outputFormat=png background=transparent
```

Bearbeiten:

```
/tool image_generate model=openai/gpt-image-2 prompt="Die Form des Objekts beibehalten und das Material in lichtdurchlässiges Glas ändern" image=/path/to/reference.png size=1024x1536
```

## Videogenerierung

Das gebündelte `openai`-Plugin registriert die Videogenerierung über das
Werkzeug `video_generate`.

| Funktion              | Wert                                                                               |
| --------------------- | ---------------------------------------------------------------------------------- |
| Standardmodell        | `openai/sora-2`                                                                 |
| Modi                  | Text-zu-Video, Bild-zu-Video, Bearbeitung eines einzelnen Videos                   |
| Referenzeingaben      | 1 Bild oder 1 Video                                                                |
| Größenüberschreibungen | Für Text-zu-Video und Bild-zu-Video unterstützt                                   |
| Seitenverhältnis      | Wird in die nächstgelegene unterstützte Größe umgewandelt und nicht unverändert weitergeleitet |
| Andere Überschreibungen | `resolution`, `audio`, `watermark` werden nicht unterstützt, verworfen und erzeugen eine Werkzeugwarnung |

OpenAI-Anfragen für Bild-zu-Video verwenden `POST /v1/videos` mit einem Bild-
`input_reference`. Bearbeitungen eines einzelnen Videos verwenden `POST /v1/videos/edits` mit dem
hochgeladenen Video im Feld `video`.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
Unter [Videogenerierung](/de/tools/video-generation) finden Sie Informationen zu gemeinsamen Werkzeugparametern,
zur Provider-Auswahl und zum Failover-Verhalten.

Der OpenAI-Provider deklariert `supportsSize`, jedoch nicht `supportsAspectRatio` oder
`supportsResolution`. Die gemeinsame Normalisierungsschicht von OpenClaw wandelt ein
angefordertes `aspectRatio` in die am besten passende OpenAI-`size` um, bevor die
Anfrage den Provider erreicht, sodass Anfragen mit Seitenverhältnis im Allgemeinen weiterhin funktionieren.
Für `resolution` gibt es keinen Größen-Fallback; der Wert wird verworfen und dem Aufrufer als
`Ignored unsupported overrides for openai/<model>: resolution=<value>` gemeldet.
</Note>

## GPT-5-Prompt-Beitrag

OpenClaw fügt für Modelle der GPT-5-Familie beim Provider
`openai` einen gemeinsamen GPT-5-Prompt-Beitrag hinzu (einschließlich älterer Codex-Referenzen vor der Reparatur, die zu
`openai/*` normalisiert werden). Andere Provider, die ebenfalls Modell-IDs der GPT-5-Familie bereitstellen,
wie OpenRouter- oder opencode-Routen, erhalten dieses Overlay nicht; die Einschränkung erfolgt anhand
der Provider-ID `openai`, nicht allein anhand der Modell-ID. Ältere GPT-4.x-Modelle
erhalten es nie.

Der native Codex-App-Server-Harness erhält weder den Verhaltensvertrag für Persona/Werkzeug-
disziplin noch das freundliche Overlay für den Interaktionsstil über
Entwickleranweisungen; der native Codex behält das Codex-eigene Verhalten für Basis, Modell und
Projektdokumentation bei, und OpenClaw deaktiviert die integrierte Persönlichkeit von Codex für
native Threads, damit die Persönlichkeitsdateien des Agenten-Arbeitsbereichs maßgeblich bleiben.
OpenClaw stellt nativen Codex-Threads ausschließlich Laufzeitkontext bereit: Kanal-
zustellung, dynamische OpenClaw-Werkzeuge, ACP-Delegierung, Arbeitsbereichskontext und
OpenClaw Skills. Der Heartbeat-Hinweistext aus demselben Beitrag ist die
einzige Ausnahme: Native Codex-Heartbeat-Durchläufe erhalten ihn, wobei er als separate
Anweisungen zur Zusammenarbeit und nicht über den gemeinsamen Hook für Prompt-Beiträge
eingefügt wird.

Der GPT-5-Beitrag fügt übereinstimmenden, von OpenClaw zusammengestellten Prompts einen markierten Verhaltensvertrag für die
Beständigkeit der Persona, Ausführungssicherheit, Werkzeugdisziplin, Ausgabeform, Abschluss-
prüfungen und Verifizierung hinzu. Kanalspezifisches Antwort- und Verhalten bei stillen Nachrichten verbleibt im gemeinsamen OpenClaw-System-
Prompt und in der Richtlinie für ausgehende Zustellung. Die Ebene für den freundlichen Interaktionsstil ist
separat und konfigurierbar.

| Wert                       | Wirkung                                             |
| -------------------------- | --------------------------------------------------- |
| `"friendly"` (Standard) | Aktiviert die Ebene für den freundlichen Interaktionsstil |
| `"on"`         | Alias für `"friendly"`                        |
| `"off"`         | Deaktiviert nur die Ebene für den freundlichen Stil |

<Tabs>
  <Tab title="Konfiguration">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
Bei der Laufzeit wird die Groß-/Kleinschreibung der Werte nicht berücksichtigt, daher deaktivieren sowohl `"Off"` als auch `"off"` die
Ebene für den freundlichen Stil.
</Tip>

<Note>
Das ältere `plugins.entries.openai.config.personality` wird weiterhin als
Kompatibilitäts-Fallback gelesen, wenn die gemeinsame Einstellung
`agents.defaults.promptOverlays.gpt5.personality` nicht festgelegt ist.
</Note>

## Stimme und Sprache

<AccordionGroup>
  <Accordion title="Sprachsynthese (TTS)">
    Das gebündelte `openai`-Plugin registriert die Sprachsynthese für die
    `tts`-Oberfläche.

    | Einstellung  | Konfigurationspfad                                      | Standard                            |
    | ------------ | ------------------------------------------------------- | ----------------------------------- |
    | Modell       | `tts.providers.openai.model`                                      | `gpt-4o-mini-tts`                  |
    | Stimme       | `tts.providers.openai.speakerVoice`                                      | `coral`                  |
    | Geschwindigkeit | `tts.providers.openai.speed`                                   | (nicht festgelegt)                  |
    | Anweisungen  | `tts.providers.openai.instructions`                                      | (nicht festgelegt, nur `gpt-4o-mini-tts`) |
    | Format       | `tts.providers.openai.responseFormat`                                      | `opus` für Sprachnachrichten, `mp3` für Dateien |
    | API-Schlüssel | `tts.providers.openai.apiKey`                                     | Greift auf `OPENAI_API_KEY` zurück |
    | Basis-URL    | `tts.providers.openai.baseUrl`                                      | `https://api.openai.com/v1`                  |
    | Zusätzlicher Body | `tts.providers.openai.extraBody` / `extra_body`           | (nicht festgelegt)                  |

    Verfügbare Modelle: `gpt-4o-mini-tts`, `tts-1`, `tts-1-hd`. Verfügbare Stimmen:
    `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `fable`, `juniper`,
    `marin`, `onyx`, `nova`, `sage`, `shimmer`, `verse`.

    `extraBody` wird nach den von OpenClaw
    generierten Feldern mit dem JSON der `/audio/speech`-Anfrage zusammengeführt. Verwenden Sie es daher für OpenAI-kompatible Endpunkte, die
    zusätzliche Schlüssel wie `lang` benötigen. Prototypschlüssel werden ignoriert.

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    Legen Sie `OPENAI_TTS_BASE_URL` fest, um die TTS-Basis-URL zu überschreiben, ohne den
    Endpunkt der Chat-API zu beeinflussen. OpenAI TTS und Realtime Voice werden beide
    über einen API-Schlüssel der OpenAI Platform konfiguriert; reine OAuth-Installationen können weiterhin
    Codex-gestützte Chatmodelle verwenden, jedoch keine Live-Sprachantworten von OpenAI.
    </Note>

  </Accordion>

  <Accordion title="Sprache-zu-Text">
    Das gebündelte `openai`-Plugin registriert die Batch-Sprache-zu-Text-Verarbeitung über
    die Transkriptionsoberfläche für das Medienverständnis von OpenClaw.

    - Standardmodell: `gpt-4o-transcribe`
    - Endpunkt: OpenAI REST `/v1/audio/transcriptions`
    - Eingabepfad: mehrteiliger Audiodatei-Upload
    - Wird überall verwendet, wo die Transkription eingehender Audiodaten `tools.media.audio` liest,
      einschließlich Segmenten aus Discord-Sprachkanälen und Audioanhängen von Kanälen

    So erzwingen Sie OpenAI für die Transkription eingehender Audiodaten:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    Sprach- und Prompt-Hinweise werden an OpenAI weitergeleitet, wenn sie durch die
    gemeinsame Audiomedienkonfiguration oder eine Transkriptionsanfrage pro Aufruf bereitgestellt werden.

  </Accordion>

  <Accordion title="Echtzeittranskription">
    Das gebündelte `openai`-Plugin registriert die Echtzeittranskription für das
    Voice-Call-Plugin.

    | Einstellung      | Konfigurationspfad                                                | Standard |
    | ---------------- | ----------------------------------------------------------------- | -------- |
    | Modell           | `plugins.entries.voice-call.config.streaming.providers.openai.model`                                                | `gpt-4o-transcribe` |
    | Sprache          | `...openai.language`                                                | (nicht festgelegt) |
    | Prompt           | `...openai.prompt`                                                | (nicht festgelegt) |
    | Stilledauer      | `...openai.silenceDurationMs`                                                | `800` |
    | VAD-Schwellenwert | `...openai.vadThreshold`                                               | `0.5` |
    | Authentifizierung | `...openai.apiKey`, `OPENAI_API_KEY` oder API-Schlüsselprofil `openai` | API-Schlüssel der Platform erforderlich |

    <Note>
    Verwendet eine WebSocket-Verbindung zu `wss://api.openai.com/v1/realtime` mit
    G.711-μ-law-Audio (`g711_ulaw` / `audio/pcmu`). Bei einem `openai`-API-Schlüsselprofil
    erstellt der Gateway vor dem Öffnen des WebSockets ein kurzlebiges Client-
    Secret für die Realtime-Transkription. Dieser Streaming-Provider ist für den Echtzeittranskriptionspfad von Voice
    Call vorgesehen; Discord Voice zeichnet derzeit kurze
    Segmente auf und verwendet stattdessen den Batch-Transkriptionspfad
    `tools.media.audio`.
    </Note>

  </Accordion>

  <Accordion title="Echtzeitstimme">
    Das gebündelte `openai`-Plugin registriert Echtzeitstimme für das Voice-Call-
    Plugin.

    | Einstellung                               | Konfigurationspfad                                                              | Standard             |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ---------------------- |
    | Modell                                  | `plugins.entries.voice-call.config.realtime.providers.openai.model`     | `gpt-realtime-2.1`  |
    | Stimme                                  | `...openai.voice`                                                       | `alloy`             |
    | Temperatur (Azure-Bereitstellungs-Bridge)  | `...openai.temperature`                                                 | `0.8`               |
    | VAD-Schwellenwert                          | `...openai.vadThreshold`                                                | `0.5`                |
    | Stilledauer                       | `...openai.silenceDurationMs`                                           | `500`                |
    | Präfix-Padding                         | `...openai.prefixPaddingMs`                                             | `300`                |
    | Reasoning-Aufwand                       | `...openai.reasoningEffort`                                             | (nicht gesetzt)              |
    | Authentifizierung                                   | `openai` API-Schlüsselprofil, `...openai.apiKey` oder `OPENAI_API_KEY` | OpenAI-Platform-API-Schlüssel erforderlich |

    Verfügbare integrierte Realtime-Stimmen für `gpt-realtime-2.1`: `alloy`, `ash`,
    `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`.
    OpenAI empfiehlt `marin` und `cedar` für die beste Realtime-Qualität. Dies
    ist ein separater Satz gegenüber den obigen Text-to-Speech-Stimmen; eine reine TTS-Stimme
    wie `fable`, `nova` oder `onyx` ist für Realtime-Sitzungen nicht gültig.
    Setzen Sie das Modell explizit auf `gpt-realtime-2.1-mini`, wenn Sie die
    kleinere, kostengünstigere Realtime-2.1-Variante bevorzugen.

    <Note>
    **GPT-Live (demnächst verfügbar).** Die Vollduplexmodelle `gpt-live-1` und
    `gpt-live-1-mini` von OpenAI ersetzten im Juli 2026 den ChatGPT-Sprachmodus; die
    Entwickler-API wird schrittweise für Organisationen mit frühem Zugriff eingeführt. OpenClaw
    erkennt die Modellfamilie, führt sie aber noch nicht aus: GPT-Live-Sitzungen sind
    ausschließlich WebRTC-basiert, steuern ihren Sprecherwechsel selbst (kein VAD) und delegieren Agentenarbeit
    über ein Übergabeereignisprotokoll, das die Realtime-Transporte von OpenClaw
    noch nicht implementieren. Die Konfiguration eines `gpt-live-*`-Modells wird sicher abgelehnt und
    gibt Hinweise sowohl zur WebSocket-Bridge als auch zu Talk-Browsersitzungen, anstatt
    Audio ohne Agentenzugriff unbemerkt zu verbinden. Der API-Zugriff ist während des frühen Zugriffs
    außerdem pro OpenAI-Organisation beschränkt. Behalten Sie `gpt-realtime-2.1` (den
    Standard) bei, bis die GPT-Live-Unterstützung verfügbar ist.
    </Note>

    <Note>
    Serverseitige OpenAI-Realtime-Bridges verwenden die GA-Realtime-WebSocket-Sitzungsstruktur,
    die `session.temperature` nicht akzeptiert. Azure-OpenAI-
    Bereitstellungen bleiben über `azureEndpoint` und `azureDeployment` verfügbar und
    behalten die bereitstellungskompatible Sitzungsstruktur (einschließlich `temperature`) bei.
    Unterstützt bidirektionale Tool-Aufrufe und G.711-µ-Law-Audio.
    </Note>

    <Note>
    Die Realtime-Stimme wird beim Erstellen der Sitzung ausgewählt. OpenAI erlaubt es, die meisten
    Sitzungsfelder später zu ändern, die Stimme kann jedoch nicht mehr geändert werden, nachdem das
    Modell in dieser Sitzung Audio ausgegeben hat. OpenClaw stellt derzeit die
    IDs der integrierten Realtime-Stimmen als Zeichenfolgen bereit.
    </Note>

    <Note>
    Control UI Talk verwendet OpenAI-Realtime-Browsersitzungen mit einem vom Gateway
    ausgestellten kurzlebigen Client-Secret und einem direkten WebRTC-SDP-Austausch des Browsers
    mit der OpenAI Realtime API. Das Gateway stellt dieses Client-Secret mit
    den ausgewählten `openai`-Anmeldedaten aus. Konfigurierte Schlüssel, API-Schlüsselprofile und
    `OPENAI_API_KEY` haben Vorrang; ein `openai`-OAuth-Profil oder eine externe
    Codex-Anmeldung dient als Fallback. Gateway-Relay und serverseitige Realtime-
    WebSocket-Bridges für Sprachanrufe verwenden dieselbe Reihenfolge der Anmeldedaten für native OpenAI-Endpunkte.
    Eine Live-Verifizierung für Maintainer ist verfügbar mit
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`;
    die OpenAI-Abschnitte verifizieren sowohl die serverseitige WebSocket-Bridge als auch den
    WebRTC-SDP-Austausch des Browsers, ohne Secrets zu protokollieren.
    Übergeben Sie `--openai-only`, um diese beiden Abschnitte ohne Google-Anmeldedaten auszuführen.
    </Note>

  </Accordion>
</AccordionGroup>

## Azure-OpenAI-Endpunkte

Der gebündelte Provider `openai` kann für die Bildgenerierung eine Azure-OpenAI-Ressource
verwenden, indem die Basis-URL überschrieben wird. Im Bildgenerierungspfad erkennt OpenClaw
Azure-Hostnamen in `models.providers.openai.baseUrl` und wechselt automatisch zur
Azure-Anfragestruktur.

<Note>
Realtime-Sprache verwendet einen separaten Konfigurationspfad
(`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`)
und wird von `models.providers.openai.baseUrl` nicht beeinflusst. Die zugehörigen Azure-
Einstellungen finden Sie im Akkordeon **Realtime-Sprache** unter [Sprache und Sprachausgabe](#voice-and-speech).
</Note>

Verwenden Sie Azure OpenAI, wenn:

- Sie bereits über ein Azure-OpenAI-Abonnement, ein Kontingent oder eine Unternehmensvereinbarung
  verfügen
- Sie regionale Datenresidenz oder von Azure bereitgestellte Compliance-Kontrollen benötigen
- Sie den Datenverkehr innerhalb eines bestehenden Azure-Mandanten halten möchten

### Konfiguration

Richten Sie für die Azure-Bildgenerierung über den gebündelten Provider `openai`
`models.providers.openai.baseUrl` auf Ihre Azure-Ressource und setzen Sie `apiKey` auf
den Azure-OpenAI-Schlüssel (nicht auf einen OpenAI-Platform-Schlüssel):

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw erkennt diese Azure-Hostsuffixe für die Azure-Bildgenerierungsroute:

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

Bei Bildgenerierungsanfragen an einen erkannten Azure-Host führt OpenClaw Folgendes aus:

- Sendet den Header `api-key` anstelle von `Authorization: Bearer`
- Verwendet bereitstellungsbezogene Pfade (`/openai/deployments/{deployment}/...`)
- Hängt `?api-version=...` an jede Anfrage an
- Verwendet für Azure-Bildgenerierungsaufrufe ein standardmäßiges Anfrage-Timeout von 600s.
  Aufrufspezifische `timeoutMs`-Werte überschreiben diesen Standard weiterhin.

Andere Basis-URLs (öffentliches OpenAI, OpenAI-kompatible Proxys) verwenden weiterhin die
standardmäßige OpenAI-Bildanfragestruktur.

<Note>
Das Azure-Routing für den Bildgenerierungspfad des Providers `openai` erfordert
OpenClaw 2026.4.22 oder höher. Frühere Versionen behandeln jede benutzerdefinierte
`openai.baseUrl` wie den öffentlichen OpenAI-Endpunkt und schlagen bei Azure-Bildbereitstellungen
fehl.
</Note>

### API-Version

Setzen Sie `AZURE_OPENAI_API_VERSION`, um eine bestimmte Azure-Vorschau- oder GA-Version
für den Azure-Bildgenerierungspfad festzulegen:

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

Wenn die Variable nicht gesetzt ist, lautet der Standard `2024-12-01-preview`.

### Modellnamen sind Bereitstellungsnamen

Azure OpenAI bindet Modelle an Bereitstellungen. Bei Azure-Bildgenerierungsanfragen,
die über den gebündelten Provider `openai` geroutet werden, muss das Feld `model` in OpenClaw
dem **Azure-Bereitstellungsnamen** entsprechen, den Sie im Azure-Portal konfiguriert haben, und nicht
der öffentlichen OpenAI-Modell-ID.

Wenn Sie eine Bereitstellung namens `gpt-image-2-prod` erstellen, die `gpt-image-2` bereitstellt:

```
/tool image_generate model=openai/gpt-image-2-prod prompt="Ein übersichtliches Poster" size=1024x1024 count=1
```

Dieselbe Regel für Bereitstellungsnamen gilt für jeden Bildgenerierungsaufruf, der
über den gebündelten Provider `openai` geroutet wird.

### Regionale Verfügbarkeit

Die Azure-Bildgenerierung ist derzeit nur in einer Teilmenge der Regionen verfügbar
(beispielsweise `eastus2`, `swedencentral`, `polandcentral`, `westus3`,
`uaenorth`). Prüfen Sie vor dem Erstellen einer Bereitstellung die aktuelle Regionsliste von Microsoft
und stellen Sie sicher, dass das jeweilige Modell in Ihrer Region angeboten wird.

### Parameterunterschiede

Azure OpenAI und das öffentliche OpenAI akzeptieren nicht immer dieselben Bildparameter.
Azure lehnt möglicherweise Optionen ab, die das öffentliche OpenAI zulässt (beispielsweise bestimmte
`background`-Werte für `gpt-image-2`), oder stellt sie nur für bestimmte Modellversionen
bereit. Diese Unterschiede stammen von Azure und dem zugrunde liegenden Modell, nicht von
OpenClaw. Wenn eine Azure-Anfrage mit einem Validierungsfehler fehlschlägt, prüfen Sie im
Azure-Portal den Parametersatz, der von Ihrer konkreten Bereitstellung und API-Version
unterstützt wird.

<Note>
Azure OpenAI verwendet nativen Transport und Kompatibilitätsverhalten, erhält jedoch nicht
die verborgenen Attributions-Header von OpenClaw – siehe das Akkordeon **Native und OpenAI-kompatible
Routen** unter [Erweiterte Konfiguration](#advanced-configuration).

Verwenden Sie für Chat- oder Responses-Datenverkehr auf Azure (über die Bildgenerierung hinaus) den
Onboarding-Ablauf oder eine dedizierte Azure-Provider-Konfiguration; `openai.baseUrl` allein
übernimmt nicht die Azure-API-/Authentifizierungsstruktur. Es gibt einen separaten
Provider `azure-openai-responses/*`; siehe das Akkordeon zur serverseitigen Compaction
weiter unten.
</Note>

## Erweiterte Konfiguration

Die folgenden modellspezifischen `params`-Beispiele gestalten die eingebettete Provider-Anfrage
von OpenClaw. Ihre Konfiguration gilt als bewusst festgelegtes Anfrageverhalten, sodass eine ansonsten geeignete
`auto`-Route bei OpenClaw verbleibt, anstatt Codex implizit auszuwählen. Das native
Codex-App-Server-Harness verwaltet seinen eigenen Transport und seine eigenen Anfrageeinstellungen; ein explizites
`agentRuntime.id: "codex"` wird sicher abgelehnt, wenn die effektive Route nicht als
Codex-kompatibel deklariert ist.

<AccordionGroup>
  <Accordion title="Transport (WebSocket oder SSE)">
    OpenClaw verwendet für `openai/*` bevorzugt WebSocket mit SSE-Fallback (`"auto"`).

    Im Modus `"auto"` führt OpenClaw Folgendes aus:
    - Wiederholt einen frühen WebSocket-Fehler einmal, bevor auf SSE zurückgegriffen wird
    - Markiert WebSocket nach einem Fehler für 60 Sekunden als beeinträchtigt und verwendet während
      der Abkühlphase SSE
    - Fügt stabile Sitzungs- und Durchgangsidentitäts-Header für Wiederholungsversuche und
      Neuverbindungen hinzu
    - Normalisiert Nutzungszähler (`input_tokens` / `prompt_tokens`) über
      Transportvarianten hinweg

    | Wert                | Verhalten                          |
    | ---------------------- | ------------------------------------ |
    | `"auto"` (Standard)   | Zuerst WebSocket, SSE als Fallback     |
    | `"sse"`              | Ausschließlich SSE erzwingen                    |
    | `"websocket"`        | Ausschließlich WebSocket erzwingen              |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    Zugehörige OpenAI-Dokumentation:
    - [Realtime API mit WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
    - [Streaming-API-Antworten (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="Schnellmodus">
    OpenClaw stellt einen gemeinsamen Schnellmodus-Schalter für `openai/*` bereit:

    - **Chat/Oberfläche:** `/fast status|auto|on|off`
    - **Konfiguration:** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    Wenn aktiviert, ordnet OpenClaw den Schnellmodus der priorisierten OpenAI-Verarbeitung
    (`service_tier = "priority"`) zu. Bestehende `service_tier`-Werte bleiben
    erhalten, und der Schnellmodus schreibt `reasoning` oder
    `text.verbosity` nicht um. `fastMode: "auto"` startet neue Modellaufrufe bis zum
    automatischen Grenzwert im Schnellmodus und startet spätere Wiederholungs-, Fallback-, Tool-Ergebnis- oder
    Fortsetzungsaufrufe danach ohne Schnellmodus. Der Grenzwert beträgt standardmäßig 60 Sekunden;
    setzen Sie `params.fastAutoOnSeconds` für das aktive Modell, um ihn zu ändern.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    Sitzungsüberschreibungen haben Vorrang vor der Konfiguration. Wenn die Sitzungsüberschreibung in der
    Sitzungsoberfläche gelöscht wird, verwendet die Sitzung wieder den konfigurierten Standard.
    </Note>

  </Accordion>

  <Accordion title="Prioritätsverarbeitung (service_tier)">
    Die API von OpenAI stellt die Prioritätsverarbeitung über `service_tier` bereit. Legen Sie sie pro
    Modell in OpenClaw fest:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    Unterstützte Werte: `auto`, `default`, `flex`, `priority`.

    <Warning>
    `serviceTier` wird nur an native OpenAI-Endpunkte
    (`api.openai.com`) und native Codex-Endpunkte (`chatgpt.com/backend-api`) weitergeleitet.
    Wenn Sie einen der beiden Provider über einen Proxy leiten, lässt OpenClaw
    `service_tier` unverändert.
    </Warning>

  </Accordion>

  <Accordion title="Serverseitige Compaction (Responses API)">
    Für direkte OpenAI-Responses-Modelle (`openai/*` auf `api.openai.com`) aktiviert der
    OpenClaw-Stream-Wrapper des OpenAI-Plugins automatisch die serverseitige
    Compaction:

    - Erzwingt `store: true` (sofern die Modellkompatibilität nicht `supportsStore: false` festlegt)
    - Fügt `context_management: [{ type: "compaction", compact_threshold: ... }]` ein
    - Standardwert für `compact_threshold`: 70 % von `contextWindow` (oder `80000`, wenn
      nicht verfügbar)

    Dies gilt für den integrierten OpenClaw-Laufzeitpfad und für Hooks des OpenAI-Providers,
    die von eingebetteten Ausführungen verwendet werden. Das native Codex-App-Server-Harness verwaltet
    seinen eigenen Kontext über Codex und ist von dieser Einstellung nicht betroffen.

    <Tabs>
      <Tab title="Explizit aktivieren">
        Nützlich für kompatible Endpunkte wie Azure OpenAI Responses:

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Benutzerdefinierter Schwellenwert">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Deaktivieren">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` steuert nur das Einfügen von `context_management`.
    Direkte OpenAI-Responses-Modelle erzwingen weiterhin `store: true`, sofern die Kompatibilität
    nicht `supportsStore: false` festlegt.
    </Note>

  </Accordion>

  <Accordion title="Strikter agentischer GPT-Modus">
    Bei GPT-5-Familienmodellen des Providers `openai`, die über die eingebettete
    Laufzeit von OpenClaw ausgeführt werden, verwendet OpenClaw bereits standardmäßig einen strengeren Ausführungsvertrag namens
    `strict-agentic`. Er wird automatisch aktiviert, wenn der aufgelöste Provider
    `openai` ist und die Modell-ID der GPT-5-Familie entspricht, sofern die Konfiguration
    ihn nicht ausdrücklich deaktiviert:

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    Das explizite Festlegen von `"strict-agentic"` hat auf einem unterstützten Pfad keine Wirkung (es
    ist bereits der Standard) und ist bei nicht unterstützten Provider-Modell-Paaren wirkungslos.

    Wenn `strict-agentic` aktiv ist, führt OpenClaw Folgendes aus:
    - Aktiviert für umfangreiche Aufgaben automatisch `update_plan`
    - Wiederholt strukturell leere oder ausschließlich aus Schlussfolgerungen bestehende Durchläufe mit einer Fortsetzung,
      die eine sichtbare Antwort erzeugt
    - Verwendet explizite Planereignisse des Harnesses, wenn das ausgewählte Harness
      diese bereitstellt

    OpenClaw klassifiziert den Text des Assistenten nicht, um zu entscheiden, ob es sich bei einem Durchlauf um einen
    Plan, eine Fortschrittsmeldung oder eine endgültige Antwort handelt.

    <Note>
    Dieser Vertrag befindet sich vollständig im eingebetteten Agent-Runner von OpenClaw. Er gilt
    nicht für das native Codex-App-Server-Harness, das sein eigenes
    Durchlauf- und Planverhalten verwaltet; bei nativen Codex-Ausführungen ist die Auswahl des Harnesses wichtiger als die
    Einstellung des Ausführungsvertrags.
    </Note>

  </Accordion>

  <Accordion title="Native und OpenAI-kompatible Routen">
    OpenClaw behandelt direkte Endpunkte von OpenAI, Codex und Azure OpenAI
    anders als generische OpenAI-kompatible `/v1`-Proxys:

    **Native Routen** (`openai/*`, Azure OpenAI):
    - Behält `reasoning: { effort: "none" }` nur für Modelle bei, die den
      OpenAI-Aufwand `none` unterstützen
    - Lässt deaktiviertes Reasoning bei Modellen oder Proxys weg, die
      `reasoning.effort: "none"` ablehnen
    - Verwendet für Tool-Schemas standardmäßig den strikten Modus
    - Fügt verborgene Zuordnungs-Header nur bei verifizierten nativen Hosts hinzu (Azure
      OpenAI erhält diese Header nicht, obwohl es sich um eine native Route handelt)
    - Behält die ausschließlich für OpenAI geltende Anfrageformung bei (`service_tier`, `store`,
      Reasoning-Kompatibilität, Hinweise zum Prompt-Cache)

    **Proxy-/kompatible Routen:**
    - Verwenden ein weniger striktes Kompatibilitätsverhalten
    - Entfernen bei nicht nativen `openai-completions`-Payloads `store` aus Completions
    - Akzeptieren die Durchleitung von erweitertem `params.extra_body`-/`params.extraBody`-JSON
      für OpenAI-kompatible Completions-Proxys
    - Akzeptieren `params.chat_template_kwargs` für OpenAI-kompatible Completions-
      Proxys wie vLLM
    - Erzwingen weder strikte Tool-Schemas noch ausschließlich nativen Routen vorbehaltene Header

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="Bilderzeugung" href="/de/tools/image-generation" icon="image">
    Gemeinsame Parameter für Bild-Tools und Auswahl des Providers.
  </Card>
  <Card title="Videoerzeugung" href="/de/tools/video-generation" icon="video">
    Gemeinsame Parameter für Video-Tools und Auswahl des Providers.
  </Card>
  <Card title="OAuth und Authentifizierung" href="/de/gateway/authentication" icon="key">
    Details zur Authentifizierung und Regeln für die Wiederverwendung von Anmeldedaten.
  </Card>
</CardGroup>
