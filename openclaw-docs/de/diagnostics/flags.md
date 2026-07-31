---
read_when:
    - Sie benötigen gezielte Debug-Protokolle, ohne die globalen Protokollierungsstufen zu erhöhen
    - Sie müssen subsystem­spezifische Protokolle für den Support erfassen
summary: Diagnose-Flags für gezielte Debug-Protokolle
title: Diagnose-Flags
x-i18n:
    generated_at: "2026-07-26T17:46:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

Diagnose-Flags aktivieren zusätzliche Protokollierung für ein Subsystem, ohne
`logging.level` global zu erhöhen. Ein Flag hat keine Wirkung, sofern es nicht von einem Subsystem geprüft wird.

## Funktionsweise

- Bei Flags wird die Groß-/Kleinschreibung nicht berücksichtigt. Sie werden aus `diagnostics.flags` in der
  Konfiguration sowie der Umgebungsüberschreibung `OPENCLAW_DIAGNOSTICS` aufgelöst, dedupliziert und in Kleinbuchstaben umgewandelt.
- `name.*` entspricht `name` selbst und allem unter `name.` (beispielsweise
  entspricht `telegram.*` dem Wert `telegram.http`).
- `*` oder `all` aktiviert jedes Flag.
- Starten Sie das Gateway nach einer Änderung von `diagnostics.flags` in der Konfiguration neu; die Änderung wird nicht
  zur Laufzeit neu geladen.

## Bekannte Flags

| Flag                  | Aktiviert                                                 |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Protokollierung von HTTP-Fehlern der Telegram Bot API     |
| `brave.http`          | Protokollierung von Anfragen, Antworten und Cache-Vorgängen bei Brave Search |
| `profiler`            | Profiler für die Antwortphase und Codex-App-Server-Profiler (beide) |
| `reply.profiler`      | Nur den Profiler für die Antwortphase                     |
| `codex.profiler`      | Nur den Codex-App-Server-Profiler                          |
| `health`              | Debugdetails zu Gateway-Zustandsprüfung, Konto und Bindung |
| `ingress.timing`      | Zeitmessungen für das Laden von Sitzungen, die Modellauswahl und den Modellkatalog |
| `plugin.load-profile` | Zeitmessungen für das synchrone Laden von Plugin-Modulen   |
| `timeline`            | Strukturiertes JSONL-Zeitleistenartefakt (siehe unten)     |

## Über die Konfiguration aktivieren

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Mehrere Flags:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## Umgebungsüberschreibung (einmalig)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

Werte werden an Kommas oder Leerraum getrennt. Sonderwerte:

| Wert                        | Wirkung                                  |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | Alle Flags deaktivieren und dabei auch die Konfiguration überschreiben |
| `1`, `true`, `all`, `*`     | Jedes Flag aktivieren                    |

`OPENCLAW_DIAGNOSTICS=0` deaktiviert für diesen
Prozess Flags sowohl aus der Umgebung als auch aus der Konfiguration. Dies ist nützlich, um ein in der Konfiguration aktiv gebliebenes Profiler-Flag vorübergehend zu unterdrücken,
ohne die Datei zu bearbeiten.

## Profiler-Flags

Profiler-Flags steuern leichtgewichtige Zeitmessungsabschnitte; im deaktivierten Zustand verursachen sie keinen Mehraufwand.

Alle durch Profiler-Flags gesteuerten Abschnitte für einen Gateway-Lauf aktivieren:

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

Nur Profiler-Abschnitte für die Antwortverteilung aktivieren:

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

Nur Profiler-Abschnitte für Start, Werkzeuge und Threads des Codex-App-Servers aktivieren:

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler` aktiviert sowohl den Antwort-Profiler als auch den Codex-Profiler; verwenden Sie die
bereichsspezifischen Flag-Namen, um nur einen davon zu aktivieren.

Alternativ in der Konfiguration festlegen:

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

Starten Sie das Gateway nach einer Änderung der Konfigurations-Flags neu. Um ein Profiler-Flag zu deaktivieren,
entfernen Sie es aus `diagnostics.flags` und führen Sie einen Neustart durch, oder starten Sie den Prozess mit
`OPENCLAW_DIAGNOSTICS=0`, um für diesen Lauf jedes Diagnose-Flag zu überschreiben.

## Zeitleistenartefakte

Das Flag `timeline` (Alias: `diagnostics.timeline`) schreibt strukturierte Zeitmessungsereignisse für Start
und Laufzeit als JSONL für externe QA-Testumgebungen:

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

Alternativ in der Konfiguration aktivieren:

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

Der Ausgabepfad stammt immer aus `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH`, selbst
wenn das Flag selbst in der Konfiguration festgelegt ist; für den Pfad gibt es keinen Konfigurationsschlüssel.
Wenn `timeline` nur über die Konfiguration aktiviert ist, fehlen die frühesten Abschnitte zum Laden der Konfiguration,
da OpenClaw die Konfiguration zu diesem Zeitpunkt noch nicht gelesen hat; nachfolgende Startabschnitte
werden normal erfasst.

`OPENCLAW_DIAGNOSTICS=1`, `=all` und `=*` aktivieren ebenfalls die Zeitleiste, da sie
jedes Flag aktivieren. Verwenden Sie vorzugsweise das bereichsspezifische Flag `timeline`, wenn Sie nur das
JSONL-Artefakt und nicht jedes andere Diagnose-Flag benötigen.

Für Ereignisschleifen-Verzögerungsmessungen in der Zeitleiste ist zusätzlich zu
`timeline` eine weitere explizite Aktivierung erforderlich: Legen Sie zusätzlich zur Aktivierung der Zeitleiste `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` (oder `on`/`true`/`yes`) fest.

Zeitleisteneinträge verwenden die `openclaw.diagnostics.v1`-Hülle und können
Prozess-IDs, Phasennamen, Abschnittsnamen, Zeitdauern, Plugin-IDs, Anzahlen von Abhängigkeiten,
Ereignisschleifen-Verzögerungsmessungen, Namen von Provider-Vorgängen, den Beendigungsstatus von Unterprozessen
sowie Namen und Meldungen von Startfehlern enthalten. Behandeln Sie Zeitleistendateien als lokale
Diagnoseartefakte; prüfen Sie sie, bevor Sie sie außerhalb Ihres Rechners weitergeben.

## Speicherort der Protokolle

Flags schreiben Protokolle in die standardmäßige Diagnoseprotokolldatei. Standardmäßig:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Benannte Profile verwenden `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`; beispielsweise
verwendet `--dev` den Wert `openclaw-dev-YYYY-MM-DD.log`.

Wenn Sie `logging.file` festlegen, verwenden Sie stattdessen diesen Pfad. Protokolle liegen im JSONL-Format vor (ein JSON-
Objekt pro Zeile). Die Schwärzung wird weiterhin gemäß `logging.redactSensitive` angewendet.
Unter [Protokollierung](/de/logging) finden Sie das vollständige Modell zur Auflösung von Protokollpfaden, Rotation und
Schwärzung.

## Protokolle extrahieren

Die neueste Protokolldatei des aktiven Profils lesen:

```bash
openclaw logs --plain
# Beispiel für ein benanntes Profil:
openclaw --profile work logs --plain
```

Nach HTTP-Diagnosen für Telegram filtern:

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

Nach HTTP-Diagnosen für Brave Search filtern:

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

Oder während der Reproduktion fortlaufend ausgeben:

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

Verwenden Sie für entfernte Gateways stattdessen `openclaw logs --follow` (siehe
[/cli/logs](/de/cli/logs)).

## Hinweise

- Wenn `logging.level` höher als `warn` eingestellt ist, können durch Flags gesteuerte Protokolle
  unterdrückt werden. Der Standardwert `info` ist geeignet.
- `brave.http` protokolliert URLs und Abfrageparameter von Brave-Search-Anfragen, Status und Zeitmessung
  der Antworten sowie Treffer-, Fehltreffer- und Schreibereignisse des Caches. Der API-Schlüssel
  (der als Anfrage-Header gesendet wird) und Antwortinhalte werden nicht protokolliert, Suchanfragen können jedoch
  vertraulich sein.
- Flags können bedenkenlos aktiviert bleiben; sie beeinflussen nur das Protokollvolumen des
  jeweiligen Subsystems.
- Verwenden Sie [/logging](/de/logging), um Protokollziele, Protokollstufen und Schwärzung zu ändern.

## Verwandte Themen

- [Gateway-Diagnose](/de/gateway/diagnostics)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
