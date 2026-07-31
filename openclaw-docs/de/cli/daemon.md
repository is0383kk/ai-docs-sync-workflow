---
read_when:
    - Sie verwenden weiterhin `openclaw daemon ...` in Skripten
    - Sie benötigen Befehle für den Dienstlebenszyklus (installieren/starten/stoppen/neu starten/Status anzeigen)
summary: CLI-Referenz für `openclaw daemon` (veralteter Alias für die Verwaltung des Gateway-Dienstes)
title: Daemon
x-i18n:
    generated_at: "2026-07-26T17:41:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Veralteter Alias für die Gateway-Dienstverwaltung. `openclaw daemon ...` wird denselben Befehlen zur Dienststeuerung zugeordnet wie `openclaw gateway ...`. Verwenden Sie für aktuelle Dokumentation und Beispiele vorzugsweise [`openclaw gateway`](/de/cli/gateway).

## Verwendung

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Unterbefehle und Optionen

| Unterbefehl  | Optionen                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (nur launchd: KeepAlive/RunAtLoad bis zum nächsten Start dauerhaft unterdrücken) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: Zeigt den Installationsstatus des Dienstes (launchd/systemd/schtasks) an und prüft den Zustand des Gateways.
- `install`: Installiert den Dienst; `--force` installiert eine vorhandene Installation neu bzw. überschreibt sie.
- `restart --safe`: Weist das laufende Gateway an, aktive Aufgaben vorab zu prüfen und einen einzigen zusammengefassten Neustart zu planen, nachdem die Aufgaben abgeschlossen sind, begrenzt auf 5 Minuten. Nach Ablauf dieses Zeitbudgets wird der Neustart dennoch erzwungen. Ein einfaches `restart` verwendet direkt den Dienstmanager; `--force` bewirkt die sofortige Außerkraftsetzung.
- `restart --safe --skip-deferral`: Umgeht die Verzögerungssperre für aktive Aufgaben, sodass das Gateway auch bei gemeldeten Blockaden sofort neu startet. Erfordert `--safe`.

## Hinweise

- `status` löst konfigurierte SecretRefs für die Authentifizierung der Prüfung nach Möglichkeit auf. Wenn eine erforderliche SecretRef nicht aufgelöst ist, meldet `status --json` `rpc.authWarning`; übergeben Sie `--token`/`--password` explizit oder lösen Sie zuerst die Quelle des Secrets auf. Warnungen zu nicht aufgelöster Authentifizierung werden unterdrückt, sobald die Prüfung ansonsten erfolgreich ist.
- `status --deep` fügt eine nach bestem Bemühen ausgeführte systemweite Suche nach anderen Gateway-ähnlichen Diensten hinzu (gibt Hinweise zur Bereinigung aus; die Empfehlung bleibt ein Gateway pro Rechner) und führt die Konfigurationsvalidierung im Plugin-bewussten Modus aus. Dabei werden Warnungen aus Plugin-Manifesten angezeigt, die der schnelle Standardpfad überspringt.
- Bei Linux-Installationen mit systemd prüfen Kontrolldurchläufe auf Token-Abweichungen sowohl die Unit-Quellen `Environment=` als auch `EnvironmentFile=`.
- Kontrolldurchläufe auf Token-Abweichungen lösen `gateway.auth.token`-SecretRefs anhand der zusammengeführten Laufzeitumgebung auf (zuerst die Umgebung des Dienstbefehls, dann die Prozessumgebung). Wenn die Token-Authentifizierung nicht tatsächlich aktiv ist (`gateway.auth.mode` von `password`/`none`/`trusted-proxy` oder nicht gesetzt, wobei das Passwort Vorrang erhalten kann), wird die Auflösung des Konfigurationstokens übersprungen.
- `install` überprüft, ob ein über SecretRef verwalteter `gateway.auth.token` auflösbar ist, speichert den aufgelösten Wert jedoch niemals in den Umgebungsmetadaten des Dienstes. Wenn er nicht aufgelöst werden kann, schlägt die Installation sicher geschlossen fehl.
- Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind und `gateway.auth.mode` nicht gesetzt ist, blockiert `install`, bis Sie den Modus explizit festlegen.
- Unter macOS beschränkt `install` den Zugriff auf LaunchAgent-plist-Dateien sowie auf die generierte Umgebungsdatei und den Wrapper auf den Eigentümer (Modus `0600`/`0700`), statt Secrets in `EnvironmentVariables` einzubetten.
- Beim Betrieb mehrerer Gateways auf einem Host müssen Ports, Konfiguration/Zustand und Arbeitsbereiche voneinander isoliert werden. Siehe [Mehrere Gateways](/de/gateway#multiple-gateways-same-host).

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Gateway-Betriebshandbuch](/de/gateway)
