---
read_when:
    - Musik oder Audio über den Agenten generieren
    - Konfigurieren von Providern und Modellen für die Musikgenerierung
    - Die Parameter des Tools music_generate verstehen
sidebarTitle: Music generation
summary: Musik über music_generate in ComfyUI-, fal-, Google-Lyria-, MiniMax- und OpenRouter-Workflows generieren
title: Musikgenerierung
x-i18n:
    generated_at: "2026-07-26T18:13:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

Das Tool `music_generate` erstellt Musik oder Audio über die gemeinsame
Funktion zur Musikgenerierung, die von ComfyUI, fal, Google, MiniMax und
OpenRouter unterstützt wird.

<Note>
`music_generate` wird nur angezeigt, wenn mindestens ein Provider für die Musikgenerierung
verfügbar ist: eine explizite `agents.defaults.mediaModels.music`-Konfiguration oder ein
für die Authentifizierung konfigurierter Provider (beispielsweise mit festgelegtem API-Schlüssel).
</Note>

Bei sitzungsgebundenen Agent-Ausführungen startet `music_generate` als Hintergrundaufgabe,
verfolgt den Fortschritt im Aufgabenprotokoll und aktiviert anschließend den Agent, sobald der Titel
bereit ist, damit dieser die Person benachrichtigen und das fertige Audio anhängen kann. Der Abschluss-Agent
folgt dem Vertrag der Sitzung für sichtbare Antworten: eine automatische abschließende Antwort,
wenn dies konfiguriert ist, oder `message(action="send")`, wenn die Sitzung das
Nachrichten-Tool erfordert. Wenn die anfragende Sitzung inaktiv ist oder ihre Aktivierung fehlschlägt und
das generierte Audio weiterhin in der Antwort fehlt, sendet OpenClaw einen
idempotenten direkten Fallback, der nur das fehlende Audio enthält.

## Schnellstart

<Tabs>
  <Tab title="Über gemeinsamen Provider">
    <Steps>
      <Step title="Authentifizierung konfigurieren">
        Legen Sie für mindestens einen Provider einen API-Schlüssel fest – beispielsweise
        `GEMINI_API_KEY` oder `MINIMAX_API_KEY`.
      </Step>
      <Step title="Standardmodell auswählen (optional)">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Agent auffordern">
        _„Erstellen Sie einen schwungvollen Synthpop-Titel über eine nächtliche Fahrt durch eine
        neonbeleuchtete Stadt.“_

        Der Agent ruft `music_generate` automatisch auf. Eine Aufnahme in die
        Tool-Zulassungsliste ist nicht erforderlich.
      </Step>
    </Steps>

    Ohne eine sitzungsgebundene Agent-Ausführung (in direkten/lokalen Kontexten) wird das Tool
    direkt ausgeführt und gibt den endgültigen Medienpfad im selben Tool-Ergebnis zurück.

  </Tab>
  <Tab title="ComfyUI-Workflow">
    <Steps>
      <Step title="Workflow konfigurieren">
        Konfigurieren Sie `plugins.entries.comfy.config.music` mit einer Workflow-
        JSON-Datei sowie Prompt- und Ausgabeknoten.
      </Step>
      <Step title="Cloud-Authentifizierung (optional)">
        Legen Sie für Comfy Cloud `COMFY_API_KEY` oder `COMFY_CLOUD_API_KEY` fest.
      </Step>
      <Step title="Tool aufrufen">
        ```text
        /tool music_generate prompt="Warme Ambient-Synthesizer-Schleife mit weicher Tonbandtextur"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

Beispiel-Prompts:

```text
Erstellen Sie einen cineastischen Klaviertitel mit sanften Streichern und ohne Gesang.
```

```text
Erstellen Sie eine energiegeladene Chiptune-Schleife über den Start einer Rakete bei Sonnenaufgang.
```

Verwenden Sie `action: "list"`, um verfügbare Provider/Modelle zu prüfen, und
`action: "status"`, um die aktive sitzungsgebundene Musikaufgabe zu prüfen:

```text
/tool music_generate action=list
/tool music_generate action=status
```

Beispiel für direkte Generierung:

```text
/tool music_generate prompt="Verträumter Lo-Fi-Hip-Hop mit Vinyltextur und sanftem Regen" instrumental=true
```

## Unterstützte Provider

| Provider   | Standardmodell                | Referenzeingaben | Unterstützte Steuerelemente                            | Authentifizierung                       |
| ---------- | ---------------------------- | ---------------- | ----------------------------------------------------- | -------------------------------------- |
| ComfyUI    | `workflow`                   | Bis zu 1 Bild    | Durch den Workflow definierte Musik oder Audio         | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | Keine            | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` oder `FAL_API_KEY`            |
| Google     | `lyria-3-clip-preview`       | Bis zu 10 Bilder | `lyrics`, `instrumental`, `format`                    | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax    | `music-2.6`                  | Keine            | `lyrics`, `instrumental`, `format` (nur mp3)          | `MINIMAX_API_KEY` oder MiniMax OAuth     |
| OpenRouter | `google/lyria-3-pro-preview` | Bis zu 1 Bild    | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY`                   |

MiniMax registriert zwei Provider-IDs, die dieselben Modelle verwenden: `minimax` für
die Authentifizierung per API-Schlüssel und `minimax-portal` für OAuth. Modellreferenzen folgen dem Authentifizierungsweg
(`minimax/music-2.6` gegenüber `minimax-portal/music-2.6`); siehe
[MiniMax](/de/providers/minimax#music-generation).

fal bietet neben seinem standardmäßigen, von MiniMax unterstützten Modell auch
`fal-ai/ace-step/prompt-to-audio` (wav, keine Liedtexte, kein
Schalter für reine Instrumentalmusik) und `fal-ai/stable-audio-25/text-to-audio` (wav,
nur Prompt) an. Googles Standardmodell
`lyria-3-clip-preview` gibt ausschließlich mp3 aus; `lyria-3-pro-preview` unterstützt auch
wav. MiniMax bietet außerdem `music-2.6-free`, `music-cover` und
`music-cover-free` an. OpenRouter bietet außerdem `google/lyria-3-clip-preview` an.

### Funktionsmatrix

Der explizite Modusvertrag, der von `music_generate`, Vertragstests und dem
gemeinsamen Live-Durchlauf verwendet wird:

| Provider   | `generate` | `edit` | Bearbeitungsgrenze | Gemeinsame Live-Testläufe                                                |
| ---------- | :--------: | :----: | ------------------ | ------------------------------------------------------------------------ |
| ComfyUI    |     ✓      |   ✓    | 1 Bild             | Nicht im gemeinsamen Durchlauf; abgedeckt durch `extensions/comfy/comfy.live.test.ts` |
| fal        |     ✓      |   —    | Keine              | `generate`                                                       |
| Google     |     ✓      |   ✓    | 10 Bilder          | `generate`, `edit`                                   |
| MiniMax    |     ✓      |   —    | Keine              | `generate`                                                       |
| OpenRouter |     ✓      |   ✓    | 1 Bild             | `generate`, `edit`                                   |

## Tool-Parameter

<ParamField path="prompt" type="string" required>
  Prompt für die Musikgenerierung. Für `action: "generate"` erforderlich.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` gibt die aktuelle Sitzungsaufgabe zurück; `"list"` prüft Provider.
</ParamField>
<ParamField path="model" type="string">
  Überschreibung des Providers/Modells (z. B. `google/lyria-3-pro-preview`,
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  Optionale Liedtexte, wenn der Provider die explizite Eingabe von Liedtexten unterstützt.
</ParamField>
<ParamField path="instrumental" type="boolean">
  Fordert eine rein instrumentale Ausgabe an, wenn der Provider dies unterstützt.
</ParamField>
<ParamField path="image" type="string">
  Pfad oder URL eines einzelnen Referenzbilds.
</ParamField>
<ParamField path="images" type="string[]">
  Mehrere Referenzbilder (bis zu 10 bei unterstützenden Providern).
</ParamField>
<ParamField path="durationSeconds" type="number">
  Zieldauer in Sekunden, wenn der Provider Dauerhinweise unterstützt.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  Hinweis zum Ausgabeformat, wenn der Provider dies unterstützt.
</ParamField>
<ParamField path="filename" type="string">Hinweis zum Ausgabedateinamen.</ParamField>

<Note>
Nicht alle Provider unterstützen alle Parameter. OpenClaw validiert dennoch feste
Grenzen wie die Anzahl der Eingaben vor der Übermittlung. Wenn ein Provider
eine Dauer unterstützt, aber ein niedrigeres Maximum als der angeforderte Wert verwendet, begrenzt OpenClaw
die Dauer auf den nächstgelegenen unterstützten Wert. Tatsächlich nicht unterstützte optionale Hinweise
werden mit einer Warnung ignoriert, wenn der ausgewählte Provider oder das Modell sie nicht
berücksichtigen kann. Tool-Ergebnisse melden die angewendeten Einstellungen; `details.normalization`
erfasst jede Zuordnung vom angeforderten zum angewendeten Wert.
</Note>

Zeitüberschreitungen für Provider-Anfragen sind ausschließlich eine Betreiberkonfiguration. OpenClaw verwendet
`agents.defaults.mediaModels.music.timeoutMs`, wenn dies konfiguriert ist, erhöht
Werte unter 120000ms auf 120000ms und verwendet andernfalls standardmäßig 300000ms
für Provider-Anfragen.

## Asynchrones Verhalten

Sitzungsgebundene Musikgenerierung wird als Hintergrundaufgabe ausgeführt:

- **Hintergrundaufgabe:** `music_generate` erstellt eine Hintergrundaufgabe, gibt sofort eine
  Antwort mit Start-/Aufgabeninformationen zurück und veröffentlicht den fertigen Titel später in
  einer nachfolgenden Agent-Nachricht.
- **Vermeidung von Duplikaten:** Solange eine Aufgabe `queued` oder `running` ist, geben spätere
  `music_generate`-Aufrufe in derselben Sitzung den Aufgabenstatus zurück, anstatt
  eine weitere Generierung zu starten. Verwenden Sie `action: "status"`, um dies ausdrücklich zu prüfen.
  Eine kürzlich abgeschlossene, übereinstimmende Anfrage wird ebenfalls 2 Minuten lang dedupliziert.
- **Statusabfrage:** `openclaw tasks list` oder `openclaw tasks show <taskId>`
  prüft den Status von Aufgaben in der Warteschlange, in Ausführung und mit Endstatus.
- **Aktivierung bei Abschluss:** OpenClaw fügt ein internes Abschlussereignis wieder
  in dieselbe Sitzung ein, damit das Modell selbst die benutzerseitige Folgemeldung
  verfassen kann.
- **Prompt-Hinweis:** Spätere Benutzer-/manuelle Durchläufe in derselben Sitzung erhalten einen kurzen
  Laufzeithinweis, wenn bereits eine Musikaufgabe läuft, damit das Modell
  `music_generate` nicht unbedacht erneut aufruft.
- **Fallback ohne Sitzung:** Direkte/lokale Kontexte ohne echte Agent-
  Sitzung werden direkt ausgeführt und geben das endgültige Audioergebnis im selben Durchlauf zurück.

### Aufgabenlebenszyklus

Die Musikaufgabe stellt dieselben Zustände wie die allgemeine Aufgabenregistrierung bereit (siehe
[Hintergrundaufgaben](/de/automation/tasks#task-lifecycle) für den vollständigen Zustandsautomaten
einschließlich `timed_out`, `cancelled` und `lost`). Die meisten Musikausführungen
durchlaufen:

| Zustand     | Bedeutung                                                                                      |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | Aufgabe erstellt; wartet darauf, dass der Provider sie annimmt.                                |
| `running`   | Der Provider verarbeitet sie (üblicherweise 30 Sekunden bis 3 Minuten, abhängig von Provider und Dauer). |
| `succeeded` | Titel bereit; der Agent wird aktiviert und veröffentlicht ihn in der Unterhaltung.             |
| `failed`    | Provider-Fehler oder Zeitüberschreitung; der Agent wird mit Fehlerdetails aktiviert.            |

Prüfen Sie den Status über die CLI:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## Konfiguration

### Modellauswahl

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### Reihenfolge der Provider-Auswahl

OpenClaw versucht die Provider in dieser Reihenfolge:

1. `model`-Parameter aus dem Tool-Aufruf (falls der Agent einen angibt).
2. `musicGenerationModel.primary` aus der Konfiguration.
3. `musicGenerationModel.fallbacks` der Reihe nach.
4. Automatische Erkennung ausschließlich anhand authentifizierungsgestützter Provider-Standardwerte:
   - zuerst der aktuelle Standard-Provider für Textmodelle, sofern er auch
     Musikgenerierung anbietet;
   - die übrigen registrierten Provider für Musikgenerierung, alphabetisch nach
     Provider-ID.

Wenn ein Provider fehlschlägt, wird automatisch der nächste Kandidat versucht. Wenn alle
fehlschlagen, enthält der Fehler Details zu jedem Versuch.

Der automatische Fallback zwischen authentifizierten Providern ist immer aktiviert. Ein aufrufbezogener
`model` bleibt maßgeblich.

## Hinweise zu Providern

<AccordionGroup>
  <Accordion title="ComfyUI">
    Workflow-gesteuert und abhängig vom konfigurierten Graphen sowie der Node-Zuordnung
    für Eingabeaufforderungs-/Ausgabefelder. Das gebündelte `comfy`-Plugin bindet sich über die
    Registry der Musikgenerierungs-Provider in das gemeinsame
    `music_generate`-Tool ein.
  </Accordion>
  <Accordion title="fal">
    Verwendet fal-Modellendpunkte über den gemeinsamen Authentifizierungspfad des Providers. Der
    gebündelte Provider verwendet standardmäßig `fal-ai/minimax-music/v2.6` und stellt außerdem
    `fal-ai/ace-step/prompt-to-audio` und
    `fal-ai/stable-audio-25/text-to-audio` für Anfragen zur Audioerzeugung aus Eingabeaufforderungen bereit.
    Liedtexte und Instrumentalmodus sind ausschließlich für MiniMax-Modelle verfügbar; die anderen beiden
    Modelle unterstützen nur Eingabeaufforderungen.
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    Verwendet die Stapelgenerierung von Lyria 3. Der aktuelle gebündelte Ablauf unterstützt
    Eingabeaufforderungen, optionalen Liedtext und optionale Referenzbilder. Das
    Standardmodell `lyria-3-clip-preview` gibt ausschließlich mp3 aus; das
    Modell `lyria-3-pro-preview` unterstützt außerdem wav.
  </Accordion>
  <Accordion title="MiniMax">
    Verwendet den Stapel-Endpunkt `music_generation`. Unterstützt Eingabeaufforderungen, optionale
    Liedtexte, Instrumentalmodus und mp3-Ausgabe entweder über die
    API-Schlüssel-Authentifizierung `minimax` oder `minimax-portal` OAuth. Stellt außerdem die Modelle
    `music-2.6-free`, `music-cover` und `music-cover-free` bereit.
  </Accordion>
  <Accordion title="OpenRouter">
    Verwendet die Audioausgabe der OpenRouter-Chatvervollständigung mit aktiviertem Streaming. Der
    gebündelte Provider verwendet standardmäßig `google/lyria-3-pro-preview` und stellt außerdem
    `openrouter/google/lyria-3-clip-preview` bereit.
  </Accordion>
</AccordionGroup>

## Den richtigen Pfad wählen

- **Gemeinsamer Provider-gestützter Pfad**, wenn Sie Modellauswahl, Provider-
  Failover und den integrierten asynchronen Aufgaben-/Statusablauf benötigen.
- **Plugin-Pfad (ComfyUI)**, wenn Sie einen benutzerdefinierten Workflow-Graphen oder einen
  Provider benötigen, der nicht zur gemeinsamen gebündelten Musikfunktion gehört.

Wenn Sie ComfyUI-spezifisches Verhalten debuggen, lesen Sie
[ComfyUI](/de/providers/comfy). Wenn Sie das Verhalten gemeinsamer Provider debuggen,
beginnen Sie mit [fal](/de/providers/fal), [Google (Gemini)](/de/providers/google),
[MiniMax](/de/providers/minimax) oder [OpenRouter](/de/providers/openrouter).

## Funktionsmodi der Provider

Der gemeinsame Vertrag für die Musikgenerierung unterstützt explizite Modusdeklarationen:

- `generate` für die Generierung nur aus Eingabeaufforderungen.
- `edit`, wenn die Anfrage ein oder mehrere Referenzbilder enthält.

Neue Provider-Implementierungen sollten explizite Modusblöcke bevorzugen:

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

Veraltete flache Felder wie `maxInputImages`, `supportsLyrics` und
`supportsFormat` reichen **nicht** aus, um Bearbeitungsunterstützung auszuweisen. Provider
sollten `generate` und `edit` explizit deklarieren, damit Live-Tests, Vertragstests
und das gemeinsame `music_generate`-Tool die Modusunterstützung
deterministisch validieren können.

## Live-Tests

Optionale Live-Testabdeckung für die gemeinsam gebündelten Provider (fal, Google, MiniMax,
OpenRouter):

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Entsprechender Repository-Wrapper, der dieselbe Testdatei ausführt:

```bash
pnpm test:live:media:music
```

Diese Live-Datei verwendet standardmäßig bereits exportierte Umgebungsvariablen des Providers vor gespeicherten
Authentifizierungsprofilen und führt sowohl die Abdeckung für `generate` als auch für deklarierte `edit` aus, wenn
der Provider den Bearbeitungsmodus aktiviert. Aktuelle Abdeckung:

- `google`: `generate` plus `edit`
- `fal`: nur `generate`
- `minimax`: nur `generate`
- `openrouter`: `generate` plus `edit`
- `comfy`: separate Comfy-Live-Testabdeckung, nicht Teil des gemeinsamen Provider-Durchlaufs

Optionale Live-Testabdeckung für den gebündelten ComfyUI-Musikpfad:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Die Comfy-Live-Datei deckt außerdem Comfy-Workflows für Bilder und Videos ab, wenn diese
Abschnitte konfiguriert sind.

## Verwandte Themen

- [Hintergrundaufgaben](/de/automation/tasks) — Aufgabenverfolgung für abgekoppelte `music_generate`-Ausführungen
- [ComfyUI](/de/providers/comfy)
- [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults) — `musicGenerationModel`-Konfiguration
- [Google (Gemini)](/de/providers/google)
- [MiniMax](/de/providers/minimax)
- [Modelle](/de/concepts/models) — Modellkonfiguration und Failover
- [Übersicht der Tools](/de/tools)
