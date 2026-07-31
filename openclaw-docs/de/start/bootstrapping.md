---
read_when:
    - Verstehen, was beim ersten Agentenlauf geschieht
    - Erläuterung, wo sich Bootstrapping-Dateien befinden
    - Fehlerbehebung bei der Identitätseinrichtung während des Onboardings
sidebarTitle: Bootstrapping
summary: Ritual zur Agent-Initialisierung, das die Arbeitsbereichs- und Identitätsdateien anlegt
title: Agent-Bootstrapping
x-i18n:
    generated_at: "2026-07-26T18:47:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

Bootstrapping ist das Ritual beim ersten Start, das einen neuen Agent-Arbeitsbereich initialisiert und
den Agenten durch die Auswahl einer Identität führt. Es wird einmal ausgeführt, unmittelbar nach dem
Onboarding, beim ersten echten Durchlauf des Agenten.

## Was geschieht

Beim ersten Durchlauf mit einem brandneuen Arbeitsbereich (Standard: `~/.openclaw/workspace`)
führt OpenClaw Folgendes aus:

- Initialisiert `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` und `BOOTSTRAP.md`.
- Lässt den Agenten eine auf drei Schritte begrenzte Entstehungssequenz durchlaufen: Er fragt, wie Sie
  ihn nennen möchten, gibt eine kurze Zeile zu seinem Wesen und seiner Ausstrahlung wieder und fragt, ob Sie
  das minimal empfohlene Plugin-Set oder maximalen Komfort wünschen.
- Speichert die vereinbarte Identität doppelt: in `IDENTITY.md` und `SOUL.md` (was der
  Agent über sich selbst liest) sowie über `openclaw agents set-identity` (was Kanäle
  und die Benutzeroberfläche anzeigen).
- Liest die bereits während des Onboardings gespeicherten App-Empfehlungen, ohne erneut zu suchen.
  Offizielle Plugins verwenden `openclaw plugins install <id>`; Skills von ClawHub-Drittanbietern
  müssen weiterhin ausdrücklich aktiviert werden. Nachdem die Auswahl verarbeitet wurde, bestätigt der Agent
  das gespeicherte Angebot, sodass er nie wieder danach fragt.
- Löscht `BOOTSTRAP.md`, sobald der Arbeitsbereich konfiguriert erscheint, sodass das Ritual nur einmal ausgeführt wird.

Ein Arbeitsbereich gilt als konfiguriert, sobald `SOUL.md`, `IDENTITY.md` oder `USER.md`
von der jeweiligen Ausgangsvorlage abweicht oder ein Ordner namens `memory/` vorhanden ist.

<Note>
`BOOTSTRAP.md` deckt das vollständige Identitätsgespräch ab. Den Inhalt finden Sie unter
[BOOTSTRAP.md-Vorlage](/de/reference/templates/BOOTSTRAP).
</Note>

## Ausführungen mit eingebetteten und lokalen Modellen

Bei Ausführungen mit eingebetteten oder lokalen Modellen hält OpenClaw `BOOTSTRAP.md` aus dem
privilegierten Systemkontext heraus. Beim ersten primären interaktiven Durchlauf
übergibt OpenClaw den Dateiinhalt dennoch über die Benutzereingabe, sodass Modelle, die das
Tool `read` nicht zuverlässig aufrufen, das Ritual trotzdem abschließen können. Wenn der aktuelle
Durchlauf nicht sicher auf den Arbeitsbereich zugreifen kann, erhält der Agent statt einer allgemeinen Begrüßung
einen kurzen Hinweis zu einem eingeschränkten Bootstrapping.

## Bootstrapping überspringen

Um diesen Vorgang bei einem vorab initialisierten Arbeitsbereich zu überspringen, führen Sie Folgendes aus:

```bash
openclaw onboard --skip-bootstrap
```

## Ausführungsort

Das Bootstrapping wird immer auf dem Gateway-Host ausgeführt. Wenn sich die macOS-App mit einem
entfernten Gateway verbindet, befinden sich der Arbeitsbereich und seine Bootstrap-Dateien auf diesem entfernten
Rechner, nicht auf dem Mac.

<Note>
Wenn der Gateway auf einem anderen Rechner ausgeführt wird, bearbeiten Sie die Dateien des Arbeitsbereichs auf dem Gateway-
Host (zum Beispiel `user@gateway-host:~/.openclaw/workspace`).
</Note>

## Weiterführende Dokumentation

- Onboarding der macOS-App: [Onboarding](/de/start/onboarding)
- Struktur des Arbeitsbereichs: [Agent-Arbeitsbereich](/de/concepts/agent-workspace)
- Vorlageninhalt: [BOOTSTRAP.md-Vorlage](/de/reference/templates/BOOTSTRAP)
