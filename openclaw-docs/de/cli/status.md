---
read_when:
    - Sie möchten eine schnelle Diagnose des Kanalzustands und der Empfänger der letzten Sitzungen.
    - Sie möchten einen einfügbaren „all“-Status für die Fehlerbehebung.
summary: CLI-Referenz für `openclaw status` (Diagnose, Prüfungen, Nutzungsmomentaufnahmen)
title: openclaw status
x-i18n:
    generated_at: "2026-07-26T18:22:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52e8076339216f11ddadf35e0ae8e5604322a47a5a9e2ee305468b2624d7cfde
    source_path: cli/status.md
    workflow: 16
---

Diagnose für Kanäle + Sitzungen.

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

| Flag                    | Beschreibung                                                                                                     |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `--all`                 | Vollständige Diagnose (schreibgeschützt, zum Einfügen geeignet). Umfasst Sicherheitsaudit, Plugin-Kompatibilität und Arbeitsspeicher-Vektorprüfungen. |
| `--deep`                | Führt Live-Prüfungen aus (WhatsApp Web + Telegram + Discord + Slack + Signal). Aktiviert außerdem das Sicherheitsaudit. |
| `--usage`               | Gibt normalisierte Provider-Nutzungszeitfenster als `X% left` aus.                                                          |
| `--json`                | Maschinenlesbare Ausgabe.                                                                                        |
| `--verbose` / `--debug` | Gibt vor dem Bericht zusätzlich die unaufbereitete Auflösung des Gateway-Ziels aus.                                                 |

Einfaches `openclaw status` verbleibt im schnellen schreibgeschützten Pfad und kennzeichnet den Arbeitsspeicher als
`not checked` statt als nicht verfügbar, wenn die Arbeitsspeicherprüfung übersprungen wird. Aufwendige
Sicherheitsaudits, Plugin-Kompatibilitätsprüfungen und Arbeitsspeicher-Vektorprüfungen bleiben
`openclaw status --all`, `openclaw status --deep`, `openclaw security audit`
und `openclaw memory status --deep` vorbehalten.

## Sitzungs- und Modellauflösung

- Die Ausgabe des Sitzungsstatus unterscheidet `Execution:` von `Runtime:`. `Execution`
  ist der Sandbox-Pfad (`direct`, `docker/*`), während `Runtime` angibt,
  ob die Sitzung `OpenClaw Default`, `OpenAI Codex`, ein CLI-
  Backend oder ein ACP-Backend wie `codex (acp/acpx)` verwendet. Unter
  [Agent-Laufzeitumgebungen](/de/concepts/agent-runtimes) finden Sie die Unterscheidung zwischen Provider, Modell und Laufzeitumgebung.
- Wenn der aktuelle Sitzungssnapshot lückenhaft ist, kann `/status` Token-
  und Cache-Zähler aus dem neuesten Transkript-Nutzungsprotokoll ergänzen. Vorhandene
  Live-Werte ungleich null haben weiterhin Vorrang vor Transkript-Ersatzwerten.
- Der Transkript-Rückgriff kann außerdem die Bezeichnung des aktiven Laufzeitmodells wiederherstellen, wenn
  sie im Live-Sitzungseintrag fehlt. Wenn dieses Transkriptmodell
  vom ausgewählten Modell abweicht, löst der Status das Kontextfenster anhand des
  wiederhergestellten Laufzeitmodells statt anhand des ausgewählten Modells auf.
- Bei der Berechnung der Prompt-Größe bevorzugt der Transkript-Rückgriff die größere
  promptbezogene Gesamtsumme, wenn Sitzungsmetadaten fehlen oder kleiner sind, damit
  Sitzungen mit benutzerdefinierten Providern nicht auf `0`-Tokenanzeigen reduziert werden.
- Wenn eine Sitzung an ein Modell gebunden ist, das vom konfigurierten
  Primärmodell abweicht, gibt der Status beide Werte, den Grund (`session override`) und
  den Hinweis `/model default` aus. Das konfigurierte Primärmodell gilt für neue oder
  nicht gebundene Sitzungen; bestehende gebundene Sitzungen behalten ihre Sitzungsauswahl,
  bis sie gelöscht wird.
- Die Ausgabe umfasst die Sitzungsspeicher jedes Agenten, wenn mehrere Agenten
  konfiguriert sind.

## Nutzung und Kontingent

- `--usage` gibt normalisierte Provider-Nutzungszeitfenster als `X% left` aus.
- Die Rohfelder `usage_percent` / `usagePercent` von MiniMax geben das verbleibende Kontingent an,
  daher invertiert OpenClaw sie vor der Anzeige; anzahlbasierte Felder haben Vorrang, wenn
  sie vorhanden sind. Bei `model_remains`-Antworten wird der Eintrag des Chatmodells bevorzugt, die
  Zeitfensterbezeichnung bei Bedarf aus Zeitstempeln abgeleitet und der Modellname in
  die Tarifbezeichnung aufgenommen.
- Fehler beim Aktualisieren der Modellpreise werden als optionale Preiswarnungen angezeigt.
  Sie bedeuten nicht, dass das Gateway oder die Kanäle fehlerhaft sind.

## Übersicht und Aktualisierungsstatus

- Die Übersicht enthält, sofern verfügbar, den Installations- und Laufzeitstatus des Gateway- und Node-Hostdienstes
  sowie eine kompakte Angabe zur Laufzeit des Gateway-Prozesses und des Hostsystems.
- Die Übersicht enthält den Aktualisierungskanal + Git-SHA (bei Quellcode-Checkouts).
- Aktualisierungsinformationen werden in der Übersicht angezeigt; wenn eine Aktualisierung verfügbar ist, gibt der Status
  einen Hinweis zur Ausführung von `openclaw update` aus (siehe [Aktualisierung](/de/install/updating)).

## Geheimnisse

- Wenn das laufende Gateway durch den Start, ein erneutes Laden oder das Schreiben einer Konfiguration über einen isolierten SecretRef-Eigentümer verfügt, enthält der Status `degradedSecretOwners` im JSON und eine Übersichtszeile **Beeinträchtigte Geheimnisse** in der menschenlesbaren Ausgabe. Jeder Eintrag nennt den Eigentümer, den Beeinträchtigungsstatus (`cold` oder `stale`), die Konfigurationspfade und den unkenntlich gemachten Grund. Inaktive Eigentümer sind nicht verfügbar; veraltete Eigentümer verwenden weiterhin die letzten bekanntermaßen gültigen Werte.
- Schreibgeschützte Statusoberflächen (`status`, `status --json`, `status --all`)
  lösen unterstützte SecretRefs für ihre jeweiligen Konfigurationspfade nach
  Möglichkeit auf.
- Wenn ein unterstützter Kanal-SecretRef konfiguriert, im
  aktuellen Befehlspfad jedoch nicht verfügbar ist, bleibt der Status schreibgeschützt und meldet eine beeinträchtigte Ausgabe,
  statt abzustürzen. Die menschenlesbare Ausgabe zeigt Warnungen wie „konfiguriertes Token
  in diesem Befehlspfad nicht verfügbar“, und die JSON-Ausgabe enthält
  `secretDiagnostics`.
- Wenn die befehlslokale SecretRef-Auflösung erfolgreich ist, bevorzugt der Status den
  aufgelösten Snapshot und entfernt vorübergehende Kanalmarkierungen vom Typ „Geheimnis nicht verfügbar“
  aus der endgültigen Ausgabe.
- `status --all` enthält eine Übersichtszeile für Geheimnisse und einen Diagnoseabschnitt,
  der Geheimnisdiagnosen zusammenfasst (zur besseren Lesbarkeit gekürzt), ohne
  die Berichterstellung zu stoppen.

## Arbeitsspeicher

`status --json --all` meldet Arbeitsspeicherdetails aus der aktiven Arbeitsspeicher-Plugin-
Laufzeitumgebung, die durch `plugins.slots.memory` ausgewählt wurde. Benutzerdefinierte Arbeitsspeicher-Plugins können das
integrierte `memory.search.enabled` deaktiviert lassen und dennoch
ihren eigenen Datei-, Chunk-, Vektor- und FTS-Status melden.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Doctor](/de/gateway/doctor)
