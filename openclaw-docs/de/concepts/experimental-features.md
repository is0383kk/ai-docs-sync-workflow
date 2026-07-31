---
read_when:
    - Sie sehen einen `.experimental`-Konfigurationsschlüssel und möchten wissen, ob er stabil ist
    - Sie möchten Vorschaufunktionen der Runtime ausprobieren, ohne sie mit den normalen Standardeinstellungen zu verwechseln
    - Sie möchten die derzeit dokumentierten experimentellen Flags an einer zentralen Stelle finden
summary: Was experimentelle Flags in OpenClaw bedeuten und welche derzeit dokumentiert sind
title: Experimentelle Funktionen
x-i18n:
    generated_at: "2026-07-26T17:45:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

Experimentelle Funktionen sind Vorschaufunktionen hinter expliziten Flags. Sie benötigen mehr Praxiserfahrung, bevor sie eine stabile Standardeinstellung oder einen langfristigen Vertrag erhalten.

- Standardmäßig deaktiviert, sofern eine Dokumentation keine eng begrenzte automatische Einrichtungsregel beschreibt.
- Form und Verhalten können sich schneller ändern als stabile Konfigurationen.
- Bevorzugen Sie einen stabilen Pfad, wenn bereits einer vorhanden ist.
- Führen Sie eine breite Einführung erst nach Tests in einer kleineren Umgebung durch.

## Derzeit dokumentierte Flags

| Oberfläche                | Schlüssel                                                                                     | Verwenden, wenn                                                                                                                    | Weitere Informationen                                                                  |
| ------------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Lokale Modelllaufzeit     | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | Ein kleineres oder strengeres lokales Backend mit der vollständigen standardmäßigen Tool-Oberfläche von OpenClaw nicht zurechtkommt | [Lokale Modelle](/de/gateway/local-models)                                                 |
| Codex-Harness             | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | Sie den nativen Codex-App-Server 0.132.0 oder neuer auf einen durch eine OpenClaw-Sandbox gestützten Exec-Server ausrichten möchten, statt den Codemodus zu deaktivieren | [Codex-Harness-Referenz](/de/plugins/codex-harness-reference#sandboxed-native-execution) |
| Strukturiertes Planungstool | `tools.experimental.planTool`                                                                 | Sie das strukturierte Tool `update_plan` für die Nachverfolgung mehrstufiger Arbeiten in kompatiblen Laufzeiten und Benutzeroberflächen verfügbar machen möchten | [Gateway-Konfigurationsreferenz](/de/gateway/config-tools#toolsexperimental)             |
| Codemodus                 | `tools.codeMode.enabled`                                                                      | Sie kompakten, codegesteuerten Zugriff auf einen verborgenen OpenClaw-Toolkatalog wünschen                                         | [Codemodus](/de/tools/code-mode)                                                           |
| Schwarm                   | `tools.swarm.enabled`                                                                         | Sie möchten, dass Codemodus-Skripte begrenzte Gruppen von Unteragenten parallel orchestrieren                                     | [Schwarm](/de/tools/swarm)                                                                 |

## Labs der Control UI

Öffnen Sie **Settings → Agents & Tools → Labs**, um Experimente zu verwalten, die über einen
Schalter in der Control UI verfügen. Beim Aktivieren oder Deaktivieren eines Experiments wird die kanonische Gateway-
Konfiguration sofort angepasst; die Seite zeigt nur dann einen Hinweis zum Neustart an, wenn eine Funktion
einen solchen erfordert.

Codemodus und Schwarm sind die derzeit ausgelieferten Labs-Einträge. Beide Schalter
schreiben vorhandene validierte Konfigurationsschlüssel und werden normalerweise für zukünftige Agenten-
Ausführungen wirksam, ohne dass der Gateway neu gestartet werden muss.

## Schlanker Modus für lokale Modelle

`agents.defaults.experimental.localModelLean: true` entfernt bei jedem Durchlauf umfangreiche optionale Tools aus der direkten Oberfläche des Agenten: `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` und `pdf`. Explizit zugelassene oder für die Zustellung erforderliche Tools bleiben verfügbar, obwohl die Tool-Suche sie möglicherweise katalogisiert, statt sie direkt verfügbar zu machen. Im schlanken Modus verwenden Plugin-/MCP-/Client-Kataloge außerdem standardmäßig die strukturierte Tool-Suche (`tool_search`, `tool_describe`, `tool_call`), wenn `tools.toolSearch` noch nicht festgelegt ist. Verwenden Sie `agents.entries.*.experimental.localModelLean`, um dies auf einen Agenten zu beschränken.

Während des Onboardings legt eine verifizierte Inferenzroute über `ollama` oder `lmstudio` automatisch `agents.defaults.experimental.localModelLean: true` fest, wenn dieser Wert nicht vorhanden ist. OpenClaw zeichnet auf, dass die Einstellung aus dem Onboarding stammt, sodass eine später verifizierte nicht lokale Route nur die automatische Einstellung aufhebt. Eine explizit konfigurierte Einstellung `true` oder `false` bleibt erhalten. Andere selbst gehostete und OpenAI-kompatible Provider werden nicht aus Modellnamen oder URLs abgeleitet.

Wenn Sie die Tool-Suche bereits global abstimmen, lässt OpenClaw diese Konfiguration unverändert. Legen Sie `tools.toolSearch: false` fest, um die standardmäßige Tool-Suche des schlanken Modus zu deaktivieren.

Im strukturierten Modus `tools` bleibt bei schlanken Ausführungen `exec` direkt neben den Steuerelementen der Tool-Suche sichtbar, damit auf Programmierung abgestimmte lokale Modelle weiterhin ihren vertrauten Shell-Pfad auswählen können. Dadurch ändert sich nur die Sichtbarkeit im Schema: Die normale Tool-Richtlinie, das Sandboxing und Genehmigungen für Ausführungen gelten weiterhin. Die expliziten Modi `code` und `directory` behalten ihr normales Compaction-Verhalten bei.

### Warum diese Tools

Diese Tools verfügen über die umfangreichsten Beschreibungen, die breitesten Parameterstrukturen oder die höchste Wahrscheinlichkeit, ein kleines Modell vom normalen Programmier- und Konversationspfad abzulenken. Bei einem Backend mit kleinem Kontext oder strengerer OpenAI-Kompatibilität macht dies den Unterschied zwischen:

- Tool-Schemas, die in den Prompt passen, oder solchen, die den Konversationsverlauf verdrängen.
- Der Auswahl des richtigen Tools durch das Modell oder fehlerhaften Tool-Aufrufen aufgrund zu vieler ähnlicher Schemas.
- Dem Verbleib des Chat-Completions-Adapters innerhalb der Grenzwerte für strukturierte Ausgaben oder einem 400-Fehler wegen der Größe der Tool-Aufruf-Nutzlast.

Durch ihre Entfernung wird nur die direkte Tool-Liste verkürzt. Dem Modell stehen weiterhin `read`, `write`, `edit`, `exec`, `apply_patch`, Bildverständnis, Websuche/-abruf (sofern konfiguriert), Speicher sowie Sitzungs-/Agenten-Tools zur Verfügung. Zusätzliche Kataloge bleiben über die Tool-Suche erreichbar, sofern Sie nicht `tools.toolSearch: false` festlegen; durch explizite Tool-Zulassungen kann ein schlanker Agent wieder in einen reduzierten Arbeitsablauf aufgenommen werden.

### Wann der Modus aktiviert werden sollte

Aktivieren Sie den schlanken Modus, sobald Sie nachgewiesen haben, dass das Modell mit dem Gateway kommunizieren kann, vollständige Agentendurchläufe sich jedoch fehlerhaft verhalten:

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` ist erfolgreich.
2. Ein normaler Agentendurchlauf schlägt aufgrund fehlerhafter Tool-Aufrufe oder übergroßer Prompts fehl oder weil das Modell seine Tools ignoriert.
3. Das Umschalten von `localModelLean: true` behebt den Fehler.

### Wann der Modus deaktiviert bleiben sollte

Wenn Ihr Backend die vollständige Standardlaufzeit problemlos verarbeitet, lassen Sie diese Option deaktiviert. Sie ist eine Problemumgehung für lokale Stacks, die eine kleinere Tool-Oberfläche benötigen, und keine Standardeinstellung für gehostete Modelle oder gut ausgestattete lokale Systeme.

Der schlanke Modus ersetzt weder `tools.profile`, `tools.allow`/`tools.deny` noch den Ausweichmechanismus `compat.supportsTools: false` des Modells. Für eine dauerhaft schmalere Tool-Oberfläche eines bestimmten Agenten sollten Sie diese stabilen Optionen bevorzugen.

### Aktivieren

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

Nur für einen Agenten:

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

Starten Sie den Gateway nach dem Ändern des Flags neu. Die schlanke Filterung entfernt `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` und `pdf`, sofern Sie diese nicht explizit mit `tools.allow` oder `tools.alsoAllow` beibehalten; die Tool-Suche kann beibehaltene Tools weiterhin katalogisieren, statt sie direkt verfügbar zu machen.

## Experimentell bedeutet nicht verborgen

Eine experimentelle Funktion sollte in der Dokumentation und im Konfigurationspfad selbst klar als solche gekennzeichnet sein und nicht hinter einer stabil wirkenden Standardoption verborgen werden.

## Verwandte Themen

- [Funktionen](/de/concepts/features)
- [Release-Kanäle](/de/install/development-channels)
