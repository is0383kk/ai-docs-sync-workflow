---
read_when:
    - Sie benötigen eine dauerhafte Aufzeichnung der Gateway-Aktivitäten, ohne Inhalte zu speichern.
    - Sie entscheiden, ob die Überwachung des Nachrichtenlebenszyklus aktiviert werden soll
    - Sie müssen erklären, was Audit-Datensätze belegen und was nicht.
summary: Reine Metadaten-Prüfhistorie für Agentenläufe, Tool-Aktionen und optionale Nachrichtenlebenszyklen
title: Prüfverlauf
x-i18n:
    generated_at: "2026-07-26T18:26:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1005b214a674f0f888d759837bd627be458cefcf9ed61bda722499333361dc45
    source_path: gateway/audit.md
    workflow: 16
---

# Auditverlauf

Der Gateway führt ein begrenztes Audit-Register, das ausschließlich Metadaten enthält, in der gemeinsam genutzten OpenClaw-Zustandsdatenbank. Es beantwortet betriebliche Fragen wie „Welcher Agent wurde wann ausgeführt und wie wurde der Lauf beendet?“, „Welche Tool-Aktionen wurden während eines Laufs ausgeführt?“ sowie bei aktivierter Nachrichtenprüfung „Hat eine akzeptierte eingehende Nachricht die Weiterleitung erreicht?“ und „Hat eine ausgehende Nachricht einen endgültigen Zustellungsstatus erreicht?“.

Das Register speichert Identität, Reihenfolge, Herkunft, Aktion, Status und normalisierte Ergebniscodes. Es speichert niemals Prompts, Nachrichteninhalte, Tool-Argumente, Tool-Ergebnisse, Anhänge, Dateinamen, URLs, Befehlsausgaben oder unformatierten Fehlertext.

## Datensatzfamilien

Lauf- und Tool-Ereignisse werden immer aufgezeichnet, wenn die Auditierung aktiviert ist (Standardeinstellung). Ereignisse des Nachrichtenlebenszyklus sind optional und standardmäßig deaktiviert.

| Familie      | Aktionen                                                 | Standard |
| ------------ | -------------------------------------------------------- | -------- |
| Agent-Läufe  | `agent.run.started`, `agent.run.finished`                | ein      |
| Tool-Aktionen | `tool.action.started`, `tool.action.finished`            | ein      |
| Nachrichten  | `message.inbound.processed`, `message.outbound.finished` | aus      |

Jeder Datensatz enthält eine stabile Ereignis-ID, eine monoton steigende Registersequenz, einen Lebenszyklus-Zeitstempel, Akteur, Aktion, Status, `schemaVersion: 1` und `redaction: "metadata_only"`. Die vollständige Feldreferenz und die Abfragefilter finden Sie unter [Auditdatensätze](/de/cli/audit).

## Ereignisse des Nachrichtenlebenszyklus

Legen Sie mit [`audit.messages`](/de/gateway/configuration-reference#audit) fest, was aufgezeichnet wird, und starten Sie anschließend den Gateway neu:

- `off` (Standard): keine Nachrichtendatensätze.
- `direct`: nur Nachrichten in direkten Unterhaltungen.
- `all`: Direkt-, Gruppen- und Kanalnachrichten.

Nachrichtendatensätze entstehen an zwei maßgeblichen Grenzen:

- **Eingehende** Zeilen werden geschrieben, wenn eine akzeptierte Nachricht die zentrale Weiterleitung erreicht, einschließlich Duplikaten und endgültigen Verarbeitungsergebnissen.
- **Ausgehende** Zeilen werden geschrieben, wenn die gemeinsam genutzte dauerhafte Zustellung ein endgültiges Ergebnis erreicht: gesendet, unterdrückt, fehlgeschlagen oder ein explizites `unknown` für durch einen Absturz mehrdeutige Sendevorgänge. Ergebnisse der Warteschlangenwiederherstellung und der Dead-Letter-Verarbeitung sind eingeschlossen. Für jede ursprüngliche logische Antwortnutzlast wird eine endgültige Zeile erstellt; Aufteilung in Blöcke und Adapter-Fan-out werden in `resultCount` zusammengefasst.

### Klassifizierung der Unterhaltungsart

Der Modus `direct` bildet eine Datenschutzgrenze. Daher wird eine Nachricht nur dann als direkte Unterhaltung klassifiziert, wenn die Zielinformationen dies belegen: Der sendende Pfad hat die Art der Zielunterhaltung angegeben oder die Route der Zustellungssitzung benennt exakt den Kanal und den Peer, an den zugestellt wird. Schwächere Signale wie der Richtlinienstatus oder die ursprüngliche Unterhaltung können eine Nachricht als `group` klassifizieren und sie damit von der Erfassung durch `direct` ausschließen, aber niemals `direct` beanspruchen. Nachrichten, für die nicht nachgewiesen werden kann, dass sie direkt sind, werden als `unknown` klassifiziert und im Modus `direct` nicht aufgezeichnet. Kanäle, die keine Chattypen angeben, zeichnen daher im Modus `direct` möglicherweise weniger Zeilen auf als im Modus `all`.

## Datenschutzmodell

Nachrichtenzeilen speichern niemals unformatierte Plattformkennungen. Wenn eine Korrelation möglich ist, werden Konto-, Unterhaltungs-, Nachrichten- und Zielkennungen ausschließlich als installationslokale, schlüsselbasierte Pseudonyme (`hmac-sha256:v1:<keyId>:<digest>`) exportiert:

- Der HMAC-Schlüssel wird bei der ersten Verwendung generiert, ist für jede Kennungsart domänengetrennt und befindet sich in derselben Zustandsdatenbank wie das Register.
- Pseudonyme sind innerhalb einer Installation stabil, sodass Zeilen derselben Unterhaltung korreliert werden können, ohne die Plattformkennung offenzulegen.
- Dies ist **Korrelation, keine Anonymisierung**: Wer Lesezugriff auf die Zustandsdatenbank hat, besitzt auch Zugriff auf den Schlüssel und kann mögliche unformatierte Kennungen mit den Pseudonymen abgleichen. RPC- und CLI-Exporte enthalten den Schlüssel niemals.
- Wenn das Schlüsselmaterial fehlt oder beschädigt ist, während Nachrichtenzeilen aufbewahrt werden, schlägt der Gateway geschlossen fehl und verwirft neue Nachrichtendatensätze, anstatt den Schlüssel unbemerkt zu rotieren, was die Korrelation aufspalten würde.

Lauf- und Tool-Datensätze behalten `sessionKey` und `sessionId` zur Korrelation bei; kanonische Sitzungsschlüssel können selbst Plattformkonto- oder Peer-IDs enthalten. Nachrichtendatensätze lassen beide absichtlich weg.

Audit-Exporte bleiben auch ohne Inhalte vertrauliche betriebliche Metadaten: Zeitangaben, Kanäle, Ergebnisse und stabile Pseudonyme können Aktivitäten korrelieren. Schützen Sie Exporte mit denselben Zugriffskontrollen und Aufbewahrungsverfahren wie andere Betriebsdatensätze.

## Abdeckung und Beweisgrenzen

Das Register arbeitet nach bestem Bemühen und ist bewusst begrenzt. Betrachten Sie es als Nachweis dessen, was aufgezeichnet wurde, nicht als Beweis dessen, was geschehen ist:

- **Das Fehlen einer Zeile beweist nichts.** Vor der Annahme verworfene eingehende Nachrichten, Sendevorgänge aus CLI-Prozessen ohne laufenden Gateway-Recorder sowie Plugin-lokale oder direkte Sendepfade, die die gemeinsam genutzte dauerhafte Zustellung umgehen, hinterlassen keinen Datensatz.
- Schreibvorgänge werden über einen begrenzten Hintergrund-Worker ausgeführt; ein Worker-Ausfall oder eine überlastete Warteschlange führt zum Verwerfen von Datensätzen und protokolliert eine betriebliche Warnung.
- Durch einen Absturz mehrdeutige ausgehende Sendevorgänge werden als `unknown` aufgezeichnet, anstatt Ergebnisse zu erfinden.

Dieses Register unterstützt die Fehlerdiagnose und die betriebliche Prüfung. Es ist kein verlustfreies Compliance-Archiv. Falls Sie ein solches benötigen, verwenden Sie ein externes System, das durch [OpenTelemetry](/de/gateway/opentelemetry) oder Tools auf Kanalebene gespeist wird.

## Speicherung, Aufbewahrung und Migration

Datensätze befinden sich in der gemeinsam genutzten Zustandsdatenbank (`state/openclaw.sqlite`) und werden außerhalb des kritischen Zustellungspfads geschrieben. Abfragen geben niemals Datensätze zurück, die älter als 30 Tage sind, und das Register ist auf 100,000 Zeilen begrenzt. Abgelaufene Zeilen werden beim Start, bei der stündlichen Wartung und bei späteren Schreibvorgängen bereinigt. Die Aufbewahrungswartung wird auch bei deaktivierter Erfassung fortgesetzt.

Beim Upgrade von einem Gateway mit dem früheren, ausschließlich Lauf- und Tool-Daten enthaltenden Register wird das Schema beim Start automatisch migriert (oder über `openclaw doctor --fix`). Vorhandene Zeilen und ihre Registersequenzen bleiben erhalten.

## Abfragen

- CLI: [`openclaw audit`](/de/cli/audit) mit Filtern für Agent, Sitzung, Lauf, Art, Status, Richtung, Kanal, Zeitgrenzen und cursorbasierte Seitennavigation.
- Gateway-RPC: `audit.activity.list` (erfordert `operator.read`) gibt die versionierte V1-Union der Aktivitätsereignisse zurück; der ausgelieferte RPC `audit.list` bleibt für ältere Lauf-/Tool-Clients unverändert. Siehe [Gateway-Protokoll](/de/gateway/protocol#audit-ledger-rpc).

## Verwandte Themen

- [CLI für Auditdatensätze](/de/cli/audit)
- [Konfigurationsreferenz](/de/gateway/configuration-reference#audit)
- [Gateway-Protokoll](/de/gateway/protocol#audit-ledger-rpc)
- [OpenTelemetry](/de/gateway/opentelemetry)
