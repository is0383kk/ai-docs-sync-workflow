---
read_when:
    - Node ist verbunden, aber Kamera-/Canvas-/Bildschirm-/Exec-Tools schlagen fehl
    - Sie benötigen das mentale Modell für Node-Kopplung im Vergleich zu Genehmigungen
summary: Fehlerbehebung bei Node-Kopplung, Anforderungen an den Vordergrund, Berechtigungen und Tool-Fehlern
title: Node-Fehlerbehebung
x-i18n:
    generated_at: "2026-07-26T17:55:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

Verwenden Sie diese Seite, wenn ein Node im Status sichtbar ist, die Node-Tools jedoch fehlschlagen.

## Befehlsabfolge

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Führen Sie anschließend Node-spezifische Prüfungen aus:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Anzeichen für einen fehlerfreien Zustand:

- Der Node ist verbunden und für die Rolle `node` gekoppelt.
- `nodes describe` enthält die aufgerufene Fähigkeit.
- Die Ausführungsgenehmigungen zeigen den erwarteten Modus bzw. die erwartete Zulassungsliste.

## Anforderungen an den Vordergrundbetrieb

`canvas.*`, `camera.*` und `screen.*` funktionieren auf iOS-/Android-Nodes nur im Vordergrund.

Schnelle Prüfung und Fehlerbehebung:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

Wenn `NODE_BACKGROUND_UNAVAILABLE` angezeigt wird, bringen Sie die Node-App in den Vordergrund und versuchen Sie es erneut.

## Berechtigungsmatrix

| Fähigkeit                    | iOS                                            | Android                                        | macOS-Node-App                              | Typischer Fehlercode                          |
| ---------------------------- | ---------------------------------------------- | ---------------------------------------------- | ------------------------------------------- | --------------------------------------------- |
| `camera.snap`, `camera.clip` | Kamera (+ Mikrofon für Audio in Clips)         | Kamera (+ Mikrofon für Audio in Clips)         | Kamera (+ Mikrofon für Audio in Clips)      | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | Bildschirmaufnahme (+ optionales Mikrofon)     | Aufforderung zur Bildschirmaufnahme (+ optionales Mikrofon) | Bildschirmaufnahme                          | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | Nicht verfügbar                                | Nicht verfügbar                                | Bedienungshilfen + Bildschirmaufnahme       | `COMPUTER_DISABLED`, `ACCESSIBILITY_REQUIRED` |
| `location.get`               | Beim Verwenden oder Immer (modusabhängig)      | Standortzugriff im Vorder-/Hintergrund je nach Modus | Standortberechtigung                         | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | Nicht verfügbar (Pfad des Node-Hosts)          | Nicht verfügbar (Pfad des Node-Hosts)          | Ausführungsgenehmigungen erforderlich       | `SYSTEM_RUN_DENIED`                           |

## Kopplung und Genehmigungen

Drei separate Kontrollstufen bestimmen, ob ein Node-Befehl erfolgreich ausgeführt wird:

1. **Gerätekopplung**: Kann sich dieser Node mit dem Gateway verbinden?
2. **Gateway-Richtlinie für Node-Befehle**: Ist die RPC-Befehls-ID durch `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` und die Plattformstandards erlaubt?
3. **Ausführungsgenehmigungen**: Darf dieser Node einen bestimmten Shell-Befehl lokal ausführen?

Die Node-Kopplung ist eine Identitäts-/Vertrauensprüfung und keine Genehmigungsoberfläche für einzelne Befehle. Für `system.run` befindet sich die Node-spezifische Richtlinie in der Datei mit den Ausführungsgenehmigungen dieses Nodes (`openclaw approvals get --node ...`) und nicht im Kopplungsdatensatz des Gateways.

Schnelle Prüfungen:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- Kopplung fehlt: Genehmigen Sie zuerst das Node-Gerät.
- In `nodes describe` fehlt ein Befehl: Prüfen Sie die Gateway-Richtlinie für Node-Befehle und ob der Node diesen Befehl beim Verbindungsaufbau tatsächlich deklariert hat.
- Die Kopplung funktioniert, aber `system.run` schlägt fehl: Korrigieren Sie die Ausführungsgenehmigungen/Zulassungsliste auf diesem Node.

Bei genehmigungsabhängigen `host=node`-Ausführungen bindet das Gateway die Ausführung außerdem an den vorbereiteten kanonischen `systemRunPlan`. Wenn ein späterer Aufrufer den Befehl, das Arbeitsverzeichnis oder die Sitzungsmetadaten ändert, bevor die genehmigte Ausführung weitergeleitet wird, lehnt das Gateway die Ausführung wegen einer Abweichung von der Genehmigung ab, anstatt der bearbeiteten Nutzlast zu vertrauen.

## Häufige Node-Fehlercodes

| Code                                   | Bedeutung                                                                                                                                                                                 |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | Die App befindet sich im Hintergrund; bringen Sie sie in den Vordergrund.                                                                                                                  |
| `CAMERA_DISABLED`                      | Der Kameraschalter ist in den Node-Einstellungen deaktiviert.                                                                                                                              |
| `*_PERMISSION_REQUIRED`                | Die Betriebssystemberechtigung fehlt oder wurde verweigert.                                                                                                                                |
| `LOCATION_DISABLED`                    | Der Standortmodus ist deaktiviert.                                                                                                                                                         |
| `LOCATION_PERMISSION_REQUIRED`         | Der angeforderte Standortmodus wurde nicht genehmigt.                                                                                                                                      |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | Die App befindet sich im Hintergrund, es liegt jedoch nur die Berechtigung „Beim Verwenden“ vor.                                                                                           |
| `COMPUTER_DISABLED`                    | Aktivieren Sie **Allow Computer Control** in der macOS-App und genehmigen Sie anschließend die Aktualisierung der Kopplung.                                                                |
| `ACCESSIBILITY_REQUIRED`               | Gewähren Sie dem aktuellen OpenClaw-App-Bundle in den macOS-Systemeinstellungen Zugriff auf die Bedienungshilfen.                                                                          |
| `SYSTEM_RUN_DENIED: approval required` | Die Ausführungsanfrage erfordert eine ausdrückliche Genehmigung.                                                                                                                           |
| `SYSTEM_RUN_DENIED: allowlist miss`    | Der Befehl wird durch den Zulassungslistenmodus blockiert. Auf Windows-Node-Hosts werden Shell-Wrapper-Formen wie `cmd.exe /c ...` im Zulassungslistenmodus als nicht in der Zulassungsliste enthalten behandelt, sofern sie nicht über den Nachfrageablauf genehmigt wurden. |

## Schnelle Wiederherstellungsschleife

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

Falls das Problem weiterhin besteht:

- Genehmigen Sie die Gerätekopplung erneut.
- Öffnen Sie die Node-App erneut im Vordergrund.
- Erteilen Sie die Betriebssystemberechtigungen erneut.
- Erstellen Sie die Richtlinie für Ausführungsgenehmigungen neu oder passen Sie sie an.

Prüfen Sie für die Computersteuerung außerdem, ob ein bildverarbeitungsfähiger Agent das Tool `computer` bereitstellt, ob `screen.snapshot` mit der Berechtigung zur Bildschirmaufnahme erfolgreich ausgeführt wird und ob `/phone status` die gewünschte temporäre oder dauerhafte Gateway-Autorisierung anzeigt. Ein Eintrag `gateway.nodes.commands.deny` überschreibt immer `gateway.nodes.commands.allow`.

## Verwandte Themen

- [Node-Übersicht](/de/nodes)
- [Kamera-Nodes](/de/nodes/camera)
- [Standortbefehl](/de/nodes/location-command)
- [Computernutzung](/de/nodes/computer-use)
- [Ausführungsgenehmigungen](/de/tools/exec-approvals)
- [Gateway-Kopplung](/de/gateway/pairing)
- [Gateway-Fehlerbehebung](/de/gateway/troubleshooting)
- [Kanal-Fehlerbehebung](/de/channels/troubleshooting)
