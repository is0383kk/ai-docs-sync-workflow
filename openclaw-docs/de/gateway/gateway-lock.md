---
read_when:
    - Gateway-Prozess ausführen oder debuggen
    - Untersuchung der Erzwingung einer einzelnen Instanz
summary: 'Gateway-Singleton-Schutz: Dateisperre plus WebSocket-/HTTP-Bindung'
title: Gateway-Sperre
x-i18n:
    generated_at: "2026-07-26T17:48:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f5ac6d42c437b481c68a23a0aa4c00aeac9131acd76f3516ce3e949f325e265b
    source_path: gateway/gateway-lock.md
    workflow: 16
---

## Warum

- Nur ein Gateway-Prozess sollte ein Zustandsverzeichnis besitzen; führen Sie zusätzliche Gateways mit isolierten Profilen, Zustandsverzeichnissen, Konfigurationen und Ports aus.
- Abstürze/SIGKILL überstehen, ohne veraltete Sperrdateien zu hinterlassen.
- Schnell mit einer klaren Fehlermeldung abbrechen, wenn ein anderes Gateway den Port bereits belegt.

## Drei Ebenen

Beim Start wird der Besitz in drei Schritten in dieser Reihenfolge erzwungen:

1. Die **Sperre für den Zustandsbesitz** erwirbt eine Sperre, die durch das kanonische Zustandsverzeichnis bestimmt wird. Jedes Gateway ist daran beteiligt, einschließlich Gateways, die mit `OPENCLAW_ALLOW_MULTI_GATEWAY=1` gestartet wurden, sodass destruktive SQLite-Wartungsarbeiten nicht mit einem aktiven Besitzer in Konflikt geraten können.
2. Die **Konfigurationssperre** erwirbt die bisherige konfigurationsspezifische Sperre und zeichnet den Laufzeit-Port auf. Der Multi-Gateway-Modus überspringt dieses Konfigurations-Singleton, behält jedoch die Sperre für den Zustandsbesitz bei.
3. Die **Socket-Bindung** bindet den HTTP/WebSocket-Listener (standardmäßig `ws://127.0.0.1:18789`) als exklusiven TCP-Listener.

Jede Ebene kann unabhängig fehlschlagen und löst ihren eigenen `GatewayLockError` aus.

### Zustands- und Konfigurationssperren

- Die Gültigkeit der Sperre ergibt sich aus der aufgezeichneten PID, der plattformspezifischen Prozessstartidentität, sofern verfügbar, und der Gateway-Prozessidentität. Ein verifizierter Besitzer bleibt während des Starts maßgeblich, bevor sein Port mit dem Lauschen beginnt.
- Ein dedizierter SQLite-Koordinator serialisiert die Metadatenprüfung, die Rückgewinnung veralteter Besitzer und das Ersetzen von Sperren. Seine exklusive Transaktion wird automatisch freigegeben, wenn der besitzende Prozess abstürzt.
- Wenn eine Sperrdatei fehlt oder der aufgezeichnete Besitzerprozess nicht mehr ausgeführt wird, gewinnt der Startvorgang die Sperre zurück und wird fortgesetzt.
- Wenn eine der beiden Sperren aktiv gehalten wird, wiederholt der Startvorgang den Versuch bis zu 5 Sekunden lang (Standardwert), bevor er abbricht:

  ```text
  GatewayLockError("Gateway wird bereits ausgeführt (PID <pid>); Zeitüberschreitung der Sperre nach <ms> ms")
  ```

### Socket-Bindung

- Unter `EADDRINUSE` wiederholt der Startvorgang den Bindungsversuch bis zu 20-mal in Intervallen von 500 ms (insgesamt ungefähr 10 Sekunden), um ein `TIME_WAIT`-Zeitfenster nach einem kürzlich beendeten Prozess zu überbrücken.
- Wenn der Port nach den Wiederholungsversuchen weiterhin verwendet wird:

  ```text
  GatewayLockError("Eine andere Gateway-Instanz lauscht bereits auf ws://127.0.0.1:<port>")
  ```

- Andere Bindungsfehler:

  ```text
  GatewayLockError("Gateway-Socket konnte nicht an ws://127.0.0.1:<port> gebunden werden: <cause>")
  ```

Beim Herunterfahren schließt das Gateway den HTTP/WebSocket-Server und entfernt seine Zustands-
und Konfigurationssperrdateien.

## Betriebshinweise

- Wenn der Port durch einen anderen Prozess belegt ist, der kein Gateway ist, wird derselbe Fehler ausgegeben; geben Sie den Port frei oder wählen Sie mit `openclaw gateway --port <port>` einen anderen aus.
- `OPENCLAW_ALLOW_MULTI_GATEWAY=1` erlaubt mehrere Konfigurations-/Laufzeitinstanzen, jedoch keinen gemeinsam genutzten veränderlichen Zustand. Jede Instanz benötigt weiterhin ein eindeutiges `OPENCLAW_STATE_DIR`.
- Unter einer Dienstverwaltung prüft ein neuer Gateway-Prozess, bei dem einer der obigen Fehler auftritt, zunächst `/healthz` am vorhandenen Prozess. Wenn dieser Prozess fehlerfrei arbeitet, überlässt der neue Prozess ihm die Kontrolle, statt fehlzuschlagen. Unter systemd wird er mit dem Code `78` beendet; das `RestartPreventExitStatus=78` der Unit verhindert, dass `Restart=always` bei einem Sperr- oder `EADDRINUSE`-Konflikt eine Schleife durchläuft. Wenn der vorhandene Prozess nie fehlerfrei arbeitet, ist die Wiederholung der Zustandsprüfung zeitlich begrenzt, und der Start schlägt anschließend mit dem obigen Sperrfehler fehl, statt endlos eine Schleife zu durchlaufen.
- Die macOS-App verwendet vor dem Starten des Gateways eine eigene einfache PID-Sicherung; die oben beschriebene Dateisperre und Socket-Bindung bilden die tatsächliche Laufzeitdurchsetzung.

## Verwandte Themen

- [Mehrere Gateways](/de/gateway/multiple-gateways) – mehrere Instanzen mit eindeutigen Ports ausführen
- [Fehlerbehebung](/de/gateway/troubleshooting) – Diagnose von `EADDRINUSE` und Portkonflikten
