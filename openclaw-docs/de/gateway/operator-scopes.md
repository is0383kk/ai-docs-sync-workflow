---
read_when:
    - Fehlersuche bei Fehlern aufgrund eines fehlenden Operator-Berechtigungsbereichs
    - Überprüfen von Genehmigungen für die Geräte- oder Node-Kopplung
    - Gateway-RPC-Methoden hinzufügen oder klassifizieren
summary: Operatorrollen, Geltungsbereiche und Prüfungen zum Genehmigungszeitpunkt für Gateway-Clients
title: Operatorbereiche
x-i18n:
    generated_at: "2026-07-26T17:49:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

Operator-Berechtigungsumfänge legen fest, was ein Gateway-Client nach erfolgreicher Authentifizierung tun darf.
Sie dienen als Schutzmechanismus der Steuerungsebene innerhalb einer einzelnen vertrauenswürdigen Gateway-Operatordomäne,
nicht zur Isolation feindlicher Mandanten. Für eine starke Trennung zwischen Personen,
Teams oder Maschinen betreiben Sie separate Gateways unter separaten Betriebssystembenutzern oder auf separaten Hosts.

Verwandte Themen: [Sicherheit](/de/gateway/security), [Gateway-Protokoll](/de/gateway/protocol),
[Gateway-Kopplung](/de/gateway/pairing), [Geräte-CLI](/de/cli/devices).

## Rollen

Jeder Gateway-WebSocket-Client stellt die Verbindung mit einer Rolle her:

- `operator`: Clients der Steuerungsebene wie CLI, Control UI, Automatisierung und
  vertrauenswürdige Hilfsprozesse.
- `node`: Funktions-Hosts (macOS, iOS, Android, headless), die
  Befehle über `node.invoke` bereitstellen.

Operator-RPC-Methoden erfordern die Rolle `operator`; von Nodes ausgehende Methoden
erfordern die Rolle `node`.

## Berechtigungsstufen

| Berechtigungsumfang     | Bedeutung                                                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | Schreibgeschützter Status, Listen, Katalog, Protokolle, Sitzungslesezugriffe und andere nicht verändernde Aufrufe.                                            |
| `operator.write`        | Verändernde Operatoraktionen: Nachrichten senden, Tools aufrufen, Sprech-/Spracheinstellungen aktualisieren, Node-Befehle weiterleiten. Erfüllt auch `operator.read`. |
| `operator.admin`        | Administrativer Zugriff. Erfüllt jeden `operator.*`-Berechtigungsumfang. Erforderlich für Konfigurationsänderungen, Aktualisierungen, native Hooks, reservierte Namensräume und Hochrisikogenehmigungen. |
| `operator.pairing`      | Verwaltung der Geräte- und Node-Kopplung: auflisten, genehmigen, ablehnen, entfernen, rotieren, widerrufen.                                                   |
| `operator.approvals`    | APIs für Ausführungs- und Plugin-Genehmigungen.                                                                                                                |
| `operator.questions`    | Interaktive Fragen auflisten, lesen, beantworten und abschließen.                                                                                              |
| `operator.talk.secrets` | Talk-Konfiguration einschließlich Geheimnissen lesen.                                                                                                          |

Unbekannte zukünftige `operator.*`-Berechtigungsumfänge erfordern eine exakte Übereinstimmung, sofern der Aufrufer
nicht bereits über `operator.admin` verfügt.

## Der Methoden-Berechtigungsumfang ist nur die erste Schranke

Jeder Gateway-RPC verfügt über einen Methoden-Berechtigungsumfang nach dem Prinzip der geringsten Rechte, der entscheidet, ob eine
Anfrage ihren Handler erreicht. Parameterabhängige Methoden leiten diesen Berechtigungsumfang vor
der Weiterleitung ab, sodass Autorisierungsfehler eine einheitliche strukturierte Antwort liefern:

- `agent` benötigt `operator.write` für gewöhnliche Durchläufe und `operator.admin` für
  Sitzungslebenszyklusbefehle vom Typ `/new` oder `/reset`.
- `node.invoke` benötigt `operator.write` für gewöhnliche Weiterleitungsbefehle und
  `operator.admin` für `browser.proxy`, `fs.listDir` und `terminal.upload`.
- `talk.config` benötigt `operator.read`; `includeSecrets: true` benötigt außerdem
  `operator.talk.secrets`.

Einige Handler führen anschließend strengere Prüfungen anhand des konkreten Objekts durch, das
genehmigt oder geändert wird:

- `device.pair.approve` ist mit `operator.pairing` erreichbar, aber bei der Genehmigung eines
  Operatorgeräts können nur Berechtigungsumfänge vergeben oder beibehalten werden, über die der Aufrufer bereits verfügt.
- `node.pair.approve` ist mit `operator.pairing` erreichbar und leitet anschließend zusätzliche
  Genehmigungsberechtigungen aus der deklarierten Befehlsliste des ausstehenden Nodes ab.
- `chat.send` ist eine Methode mit Schreibberechtigung, aber die Chatbefehle
  `/config set` und `/config unset` erfordern darüber hinaus `operator.admin`,
  unabhängig von der Chat-Sendeberechtigung des Aufrufers.

Dadurch können Operatoren mit geringerem Berechtigungsumfang risikoarme Kopplungsaktionen ausführen,
ohne dass sämtliche Kopplungsgenehmigungen ausschließlich Administratoren vorbehalten sind.

RPCs für Sitzungsänderungen werden anhand ihrer ausgehandelten Operator-Berechtigungsumfänge autorisiert,
unabhängig von `client.id` oder `client.mode` des verbundenen Clients. Die Clientidentität
kann weiterhin die Verbindungs- und Geräteauthentifizierungsrichtlinie beeinflussen, gewährt oder entzieht jedoch
keine Berechtigung zum Ändern von Sitzungen.

## Genehmigungen für die Gerätekopplung

Datensätze zur Gerätekopplung sind die dauerhafte Quelle genehmigter Rollen und Berechtigungsumfänge.
Ein bereits gekoppeltes Gerät erhält nicht stillschweigend umfassenderen Zugriff: Bei einer erneuten Verbindung,
die eine umfassendere Rolle oder umfassendere Berechtigungsumfänge anfordert, wird eine neue ausstehende
Upgrade-Anfrage erstellt.

Beim Genehmigen einer Geräteanfrage gilt:

- Eine Anfrage ohne Operatorrolle benötigt keine Genehmigung für einen Operator-Berechtigungsumfang.
- Eine Anfrage für eine Gerätefunktion ohne Operatorrolle (beispielsweise `node`) erfordert
  `operator.admin`, obwohl `device.pair.approve` selbst nur
  `operator.pairing` benötigt.
- Eine Anfrage für `operator.read`, `operator.write`, `operator.approvals`,
  `operator.questions`, `operator.pairing` oder `operator.talk.secrets` erfordert,
  dass der Aufrufer bereits über den betreffenden Berechtigungsumfang oder über `operator.admin` verfügt.
- Eine Anfrage für `operator.admin` erfordert `operator.admin`.
- Eine Reparaturanfrage ohne explizite Berechtigungsumfänge kann die Berechtigungsumfänge des bestehenden
  Operator-Tokens übernehmen; besitzt dieses Token Administratorberechtigungen, erfordert die Genehmigung dennoch
  `operator.admin`.

Sitzungen mit gemeinsamem Geheimnis und vertrauenswürdigem Proxy ohne Administratorberechtigungen können
Operatorgeräte-Anfragen nur innerhalb ihrer eigenen deklarierten Operator-Berechtigungsumfänge genehmigen; die Genehmigung
von Rollen ohne Operatorfunktion ist ausschließlich Administratoren vorbehalten, selbst wenn diese Sitzungen ansonsten
`operator.pairing` verwenden können.

Bei Sitzungen mit Tokens gekoppelter Geräte ist die Verwaltung auf das eigene Gerät beschränkt, sofern der Aufrufer
nicht über `operator.admin` verfügt: Ein Aufrufer ohne Administratorberechtigungen sieht nur seine eigenen Kopplungseinträge und
kann nur seinen eigenen Geräteeintrag genehmigen, ablehnen, rotieren, widerrufen oder entfernen.

## Genehmigungen für die Node-Kopplung

Ältere `node.pair.*`-Methoden verwenden einen separaten, vom Gateway verwalteten Speicher für Node-Kopplungen.
WS-Nodes verwenden stattdessen die Gerätekopplung (`role: node`), jedoch gilt dasselbe
Genehmigungsvokabular. Unter [Gateway-Kopplung](/de/gateway/pairing) erfahren Sie, wie die beiden
Speicher zusammenhängen.

`node.pair.approve` leitet zusätzliche erforderliche Berechtigungsumfänge aus der Befehlsliste
der ausstehenden Anfrage ab:

| Deklarierte Befehle                                                                                                  | Erforderliche Berechtigungsumfänge     |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| keine                                                                                                                | `operator.pairing`                    |
| gewöhnliche Node-Befehle                                                                                             | `operator.pairing` + `operator.write` |
| `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` oder `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

Das Genehmigen einer Node-Deklaration aktiviert keine Befehle, die einer separaten
Laufzeit-Zulassungslistenschranke unterliegen. Beispielsweise erfordert die Genehmigung eines Nodes, der
`computer.act` deklariert, eine Kopplungs- und Schreibberechtigung, erfasst jedoch nur die Schnittstelle.
Ein Administrator oder Eigentümer muss `computer.act` weiterhin aktivieren. Solange es
aktiviert bleibt, erfordert sein Aufruf über `node.invoke` eine Schreibberechtigung, jedoch nicht für jede
Aktion eine Administratorberechtigung.

Die Node-Kopplung stellt Identität und Vertrauen her; sie ersetzt nicht die eigene
Ausführungsgenehmigungsrichtlinie `system.run` eines Nodes.

## Authentifizierung mit gemeinsamem Geheimnis

Die Authentifizierung mit einem gemeinsam verwendeten Gateway-Token oder -Passwort wird als vertrauenswürdiger Operatorzugriff für
dieses Gateway behandelt. OpenAI-kompatible HTTP-Schnittstellen, `/tools/invoke` und HTTP-Endpunkte
für den Sitzungsverlauf stellen für die Bearer-Authentifizierung mit gemeinsamem Geheimnis den vollständigen standardmäßigen Operator-Berechtigungssatz wieder her,
selbst wenn ein Aufrufer engere deklarierte Berechtigungsumfänge sendet.

Identitätstragende Modi wie die Authentifizierung über einen vertrauenswürdigen Proxy oder `none` für privaten Eingang
können explizit deklarierte Berechtigungsumfänge weiterhin berücksichtigen. Verwenden Sie separate Gateways für eine echte
Trennung von Vertrauensgrenzen.
