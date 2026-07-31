---
read_when:
    - Vom Gateway gestartete Cloud-Worker betreiben oder debuggen
    - Worker-Zulassung, Sitzungszuweisung oder lokale Tool-Isolierung überprüfen
summary: Interne Betreiberreferenz für die eingeschränkte Cloud-Worker-Laufzeit
title: Worker
x-i18n:
    generated_at: "2026-07-26T17:44:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c4749e2abaf4fca00d903114b0661454d67207547fe17711dc5315656e0cd14
    source_path: cli/worker.md
    workflow: 16
---

# `openclaw worker`

`openclaw worker` ist der eingeschränkte Laufzeit-Einstiegspunkt, den ein Cloud-Worker-
Orchestrator innerhalb einer vorbereiteten Worker-Umgebung startet. Er ist kein
Allzweckbefehl für die manuelle Worker-Registrierung.

Das Gateway installiert das passende OpenClaw-Bundle und öffnet den per Hostschlüssel
fixierten Reverse-SSH-Tunnel. Der Worker-Launcher startet diesen Befehl mit einer
vorbereiteten Zuweisung. Der Befehl stellt die Verbindung über den durch den Tunnel
weitergeleiteten lokalen Socket her und wird mit der dedizierten Rolle `worker` zugelassen.

## Startvertrag

Der Befehl liest genau einen größenbeschränkten JSON-Start-Umschlag von der Standardeingabe.
Der Umschlag enthält den Speicherort des lokalen Sockets, die ausgestellte Worker-
Zugangsinformation, die Bundle- und Protokollidentität, die Owner-Epoche, die einzelne
zugewiesene Sitzung und Ausführung sowie die exakten Namen der Worker-lokalen Tools,
die für diese Ausführung autorisiert sind. Das Gateway ermittelt diesen endgültigen
Toolsatz vor der Übergabe anhand der aktuellen Richtlinie; die Rohkonfiguration und
die Identität des eingeplanten Owners gelangen niemals in den Worker-Umschlag.
Die Zugangsinformation wird niemals über Befehlszeilenargumente akzeptiert, und diese
Seite enthält absichtlich weder ein Beispiel für Zugangsinformationen noch für einen
manuell erstellten Umschlag.

Die Zulassung schlägt nach dem Fail-Closed-Prinzip fehl, wenn der Umschlag ungültig ist,
die Zugangsinformation abgelehnt wird, die Bundle- oder Protokollfunktionen nicht
übereinstimmen oder die Sitzung und die Owner-Epoche nicht mehr aktuell sind. Fehlende,
doppelte oder unbekannte Toolnamen machen den Umschlag ebenfalls ungültig. Betreiber
sollten Worker über den Cloud-Worker-Orchestrator starten, statt diesen Einstiegspunkt
direkt aufzurufen.

## Laufzeitgrenze

Der Prozess führt die normale eingebettete Agentenschleife mit einem eingeschränkten
Backend aus:

- Die Coding-Tools `read`, `write`, `edit`, `apply_patch`, `exec` und `process`
  werden lokal im Worker-Arbeitsbereich ausgeführt, wenn sie in der vom Gateway
  ausgestellten Ausführungsberechtigung enthalten sind. Bei einer leeren Berechtigung
  wird das Modell ohne Tools ausgeführt.
- Modellaufrufe verwenden den Inferenz-Proxy des Gateways. Es wird kein lokales
  Modellauthentifizierungsprofil geladen.
- Transkriptschreibvorgänge verwenden den RPC für Transkript-Commits des Gateways.
- Streaming- und Tool-Lebenszyklusaktualisierungen verwenden den Live-Event-RPC des Gateways.
- Nur die zugewiesene Sitzung und Ausführung werden akzeptiert.

Der Worker-Modus startet weder Kanäle noch Gateway-HTTP-Oberflächen oder den
automatischen Plugin-Start über den zugewiesenen Sitzungstoolsatz hinaus. Er verwendet
ein temporäres Zustandsverzeichnis und verfügt über keine dauerhaften Provider- oder
Forge-Zugangsinformationen.

Die Weiterleitung von Sitzungen zwischen Workern ist in diesem Modus nicht verfügbar.
Platzierung und Weiterleitung bleiben Eigentum des Gateways: Ein Betreiber kann eine
vorhandene lokale Sitzung mit verwaltetem Arbeitsbaum über das Gateway weiterleiten,
während ein Worker-Prozess weder sich selbst noch einen anderen Worker weiterleiten kann.

Die vorbereitete Zuweisung enthält den Transkriptkontext, das akzeptierte Basisblatt,
die Commit-Sequenz und den Live-Event-Cursor. Bei einer erneuten Tunnelverbindung wird
der Prozess mit derselben Zugangsinformation und Owner-Epoche erneut zugelassen, behält
die akzeptierte Transkriptbasis bei, spielt das noch nicht bestätigte Ende seiner
Live-Events erneut ab und bindet eine laufende Inferenzausführung mit derselben Identität
wieder an. Die abschließende Inferenznachricht ist maßgeblich, falls gestreamte Deltas
verpasst wurden. Eine ablösende Owner-Epoche grenzt den Prozess aus und bewirkt ein
ordnungsgemäßes Beenden.

Eine `stale-base-leaf`-Transkriptablehnung stoppt die aktuelle Ausführung endgültig.
Der Worker-Modus versucht die abgelehnte Sequenz nicht erneut mit einem anderen Blatt,
sodass kein doppelter Commit erzeugt wird; ein noch nicht committetes, im Arbeitsspeicher
befindliches Ende dieser Ausführung geht verloren. Der Neustart liegt in der
Verantwortung des Platzierungs-Owners von Meilenstein 3, der eine neue Zuweisung anhand
des maßgeblichen Transkripts und Commit-Ledgers des Gateways erstellen muss. Ebenso
beendet ein Neustart des Gateway-Prozesses eine ausstehende Inferenzausführung mit einem
Providerfehler; nur die erneute Verbindung eines Tunnels oder Worker-WebSockets kann
sich wieder an einen aktiven Inferenzstream desselben Prozesses anbinden.

Weitere Informationen zur geschlossenen Worker-RPC-Oberfläche finden Sie unter
[Gateway-Protokoll](/de/gateway/protocol#worker-role-and-closed-protocol) und zum Architektur-
und Sicherheitsmodell unter [Plan für Cloud-Worker](/de/plan/cloud-workers).
