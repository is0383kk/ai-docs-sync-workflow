---
permalink: /security/formal-verification/
read_when:
    - Überprüfung formaler Garantien oder Einschränkungen des Sicherheitsmodells
    - TLA+/TLC-Sicherheitsmodellprüfungen reproduzieren oder aktualisieren
summary: Maschinell geprüfte Sicherheitsmodelle für die risikoreichsten Pfade von OpenClaw.
title: Formale Verifikation (Sicherheitsmodelle)
x-i18n:
    generated_at: "2026-07-26T18:04:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

Die formalen Sicherheitsmodelle von OpenClaw (derzeit TLA+/TLC) liefern ein maschinell geprüftes Argument dafür, dass bestimmte Pfade mit dem höchsten Risiko – Autorisierung, Sitzungsisolation, Tool-Zugriffskontrolle und Sicherheit bei Fehlkonfigurationen – unter ausdrücklich genannten Annahmen die vorgesehene Richtlinie durchsetzen.

> Hinweis: Einige ältere Links verweisen möglicherweise auf den früheren Projektnamen.

## Worum es sich handelt

Eine ausführbare, angreifergesteuerte Suite für Sicherheitsregressionen:

- Jede Behauptung verfügt über eine ausführbare Modellprüfung in einem endlichen Zustandsraum.
- Viele Behauptungen verfügen über ein zugehöriges negatives Modell, das für eine realistische Fehlerklasse eine Gegenbeispielspur erzeugt.

Dies ist **kein** Beweis dafür, dass OpenClaw in jeder Hinsicht sicher ist, und es verifiziert nicht die vollständige TypeScript-Implementierung.

## Speicherort der Modelle

Die Modelle werden in einem separaten Repository gepflegt: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models).

<Note>
Dieses Repository ist derzeit nicht erreichbar (GitHub gibt zum Zeitpunkt der Erstellung dieses Textes „Repository not found“ zurück). Falls es für Sie weiterhin nicht erreichbar ist, fragen Sie in den OpenClaw-Maintainer-Kanälen nach dem aktuellen Speicherort, bevor Sie davon ausgehen, dass die Modelle entfernt wurden.
</Note>

## Einschränkungen

- Dies sind Modelle, nicht die vollständige TypeScript-Implementierung – Abweichungen zwischen Modell und Code sind möglich.
- Die Ergebnisse sind durch den von TLC untersuchten Zustandsraum begrenzt. Grün bedeutet keine Sicherheit über die modellierten Annahmen und Grenzen hinaus.
- Einige Behauptungen beruhen auf ausdrücklichen Annahmen zur Umgebung (beispielsweise einer korrekten Bereitstellung und korrekten Konfigurationseingaben).

## Ergebnisse reproduzieren

Klonen Sie das Modell-Repository und führen Sie TLC aus:

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Java 11+ erforderlich (TLC wird auf der JVM ausgeführt).
# Das Repository enthält eine angeheftete tla2tools.jar und stellt bin/tlc sowie Make-Ziele bereit.

make <target>
```

Eine CI-Integration zurück in dieses Repository besteht noch nicht; eine zukünftige Iteration könnte in der CI ausgeführte Modelle mit öffentlichen Artefakten (Gegenbeispielspuren, Ausführungsprotokolle) oder einen gehosteten Workflow „Dieses Modell ausführen“ für kleine begrenzte Prüfungen hinzufügen.

## Behauptungen und Ziele

### Gateway-Exposition und Fehlkonfiguration eines offenen Gateways

**Behauptung:** Eine Bindung außerhalb der Loopback-Schnittstelle ohne Authentifizierung kann eine Remote-Kompromittierung ermöglichen und erhöht die Exposition; gemäß den Annahmen des Modells blockiert ein Token/Passwort nicht authentifizierte Angreifer.

| Ergebnis       | Ziele                                                            |
| -------------- | ---------------------------------------------------------------- |
| Grün           | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| Rot (erwartet) | `make gateway-exposure-v2-negative`                              |

Siehe auch `docs/gateway-exposure-matrix.md` im Modell-Repository.

### Node-Ausführungspipeline (Funktion mit dem höchsten Risiko)

**Behauptung:** `exec host=node` erfordert im Modell (a) eine Zulassungsliste für Node-Befehle sowie deklarierte Befehle und (b) eine direkte Genehmigung, sofern diese konfiguriert ist; Genehmigungen werden tokenisiert, um eine Wiederverwendung zu verhindern.

| Ergebnis       | Ziele                                                           |
| -------------- | --------------------------------------------------------------- |
| Grün           | `make nodes-pipeline`, `make approvals-token`                   |
| Rot (erwartet) | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### Kopplungsspeicher (DM-Zugriffskontrolle)

**Behauptung:** Kopplungsanfragen halten TTL und Obergrenzen für ausstehende Anfragen ein.

| Ergebnis       | Ziele                                                |
| -------------- | ---------------------------------------------------- |
| Grün           | `make pairing`, `make pairing-cap`                   |
| Rot (erwartet) | `make pairing-negative`, `make pairing-cap-negative` |

### Eingangs-Zugriffskontrolle (Erwähnungen und Umgehung durch Steuerbefehle)

**Behauptung:** In Gruppenkontexten, die eine Erwähnung erfordern, kann ein nicht autorisierter Steuerbefehl die Erwähnungs-Zugriffskontrolle nicht umgehen.

| Ergebnis       | Ziele                          |
| -------------- | ------------------------------ |
| Grün           | `make ingress-gating`          |
| Rot (erwartet) | `make ingress-gating-negative` |

### Routing und Isolation von Sitzungsschlüsseln

**Behauptung:** DMs verschiedener Gegenstellen werden nicht in derselben Sitzung zusammengeführt, sofern sie nicht ausdrücklich verknüpft oder entsprechend konfiguriert sind.

| Ergebnis       | Ziele                             |
| -------------- | --------------------------------- |
| Grün           | `make routing-isolation`          |
| Rot (erwartet) | `make routing-isolation-negative` |

## v1++-Modelle: Nebenläufigkeit, Wiederholungsversuche und Korrektheit von Spuren

Nachfolgemodelle, die die Realitätsnähe für Fehlerarten aus der Praxis erhöhen: nicht atomare Aktualisierungen, Wiederholungsversuche und Nachrichtenverteilung.

### Nebenläufigkeit und Idempotenz des Kopplungsspeichers

**Behauptung:** Der Kopplungsspeicher erzwingt `MaxPending` und Idempotenz auch bei verzahnten Abläufen – Prüfen und anschließendes Schreiben müssen atomar/gesperrt erfolgen, und eine Aktualisierung darf keine Duplikate erzeugen. Konkret: Gleichzeitige Anfragen dürfen `MaxPending` für einen Kanal nicht überschreiten, und wiederholte Anfragen/Aktualisierungen für dieselbe `(channel, sender)` erzeugen keine doppelten aktiven ausstehenden Zeilen.

| Ergebnis       | Ziele                                                                                                                                                                       |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Grün           | `make pairing-race` (atomare/gesperrte Prüfung der Obergrenze), `make pairing-idempotency`, `make pairing-refresh`, `make pairing-refresh-race`                                              |
| Rot (erwartet) | `make pairing-race-negative` (nicht atomares Wettlaufproblem zwischen Beginn und Commit bei der Obergrenze), `make pairing-idempotency-negative`, `make pairing-refresh-negative`, `make pairing-refresh-race-negative` |

### Korrelation und Idempotenz von Eingangsspuren

**Behauptung:** Die Aufnahme bewahrt die Spurenkorrelation über die Nachrichtenverteilung hinweg und ist bei Wiederholungsversuchen des Providers idempotent. Wenn aus einem externen Ereignis mehrere interne Nachrichten entstehen, behält jeder Teil dieselbe Spuren-/Ereignisidentität; Wiederholungsversuche führen nicht zu einer doppelten Verarbeitung; fehlen Ereignis-IDs des Providers, verwendet die Deduplizierung ersatzweise einen sicheren Schlüssel (beispielsweise die Spuren-ID), damit unterschiedliche Ereignisse nicht verworfen werden.

| Ergebnis       | Ziele                                                                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Grün           | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| Rot (erwartet) | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### Routing-Priorität von dmScope und identityLinks

**Behauptung:** Die Priorität von `dmScope` und Identitätsverknüpfungen verhalten sich deterministisch: Der standardmäßige Gültigkeitsbereich `main` verwendet für die DMs eines einzelnen Eigentümers eine gemeinsame fortlaufende Sitzung (der Standard für persönliche Agenten), während jeder konfigurierte isolierende Gültigkeitsbereich (`per-peer`, `per-channel-peer`, `per-account-channel-peer`) DM-Sitzungen strikt getrennt hält. Kanalspezifische `dmScope` haben Vorrang vor globalen Standardwerten; `identityLinks` führen Sitzungen nur innerhalb ausdrücklich verknüpfter Gruppen zusammen, nicht über voneinander unabhängige Gegenstellen hinweg. Bei Posteingängen mit mehreren Benutzern wird erwartet, dass ein isolierender Gültigkeitsbereich aktiviert wird (die Laufzeit-Sicherheitsprüfung empfiehlt dies, wenn sie DM-Datenverkehr von mehreren Benutzern erkennt).

| Ergebnis       | Ziele                                                                     |
| -------------- | ------------------------------------------------------------------------- |
| Grün           | `make routing-precedence`, `make routing-identitylinks`                   |
| Rot (erwartet) | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## Verwandte Themen

- [Bedrohungsmodell](/de/security/THREAT-MODEL-ATLAS)
- [Zum Bedrohungsmodell beitragen](/de/security/CONTRIBUTING-THREAT-MODEL)
- [Reaktion auf Sicherheitsvorfälle](/de/security/incident-response)
