---
read_when:
    - Genehmigungen für die Node-Kopplung ohne macOS-Benutzeroberfläche implementieren
    - CLI-Abläufe zum Genehmigen entfernter Nodes hinzufügen
    - Erweiterung des Gateway-Protokolls um die Node-Verwaltung
summary: 'Genehmigungen für Node-Fähigkeiten: So erhalten Nodes nach der Gerätekopplung Zugriff auf Befehle'
title: Node-Kopplung
x-i18n:
    generated_at: "2026-07-26T18:23:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 25e4016657379573ddb7e9027899afd8b97b16709da6e73ed44d4016b99e715a
    source_path: gateway/pairing.md
    workflow: 16
---

Die Node-Kopplung umfasst zwei Ebenen, die beide im Datensatz des gekoppelten Geräts in der
SQLite-Zustandsdatenbank des Gateways gespeichert werden:

- **Gerätekopplung** (Rolle `node`) steuert den `connect`-Handshake. Siehe
  [Automatische Gerätegenehmigung über vertrauenswürdige CIDRs](#trusted-cidr-device-auto-approval)
  unten und [Kanalkopplung](/de/channels/pairing).
- **Genehmigung von Node-Funktionen** (`node.pair.*`) steuert, welche deklarierten
  Funktionen/Befehle ein verbundener Node bereitstellen darf. Das Gateway ist die
  maßgebliche Quelle; Benutzeroberflächen (macOS-App, Control UI) sind Frontends, die ausstehende
  Anfragen genehmigen oder ablehnen.

Der frühere eigenständige Speicher für die Node-Kopplung (`nodes/paired.json` mit einem Token pro Node,
im Januar 2026 aus dem Verbindungspfad entfernt) ist nicht mehr vorhanden: Gateways übernehmen
beim Start einmalig alle verbleibenden Zeilen in die Gerätedatensätze und archivieren die
veralteten Dateien mit dem Suffix `.migrated`. Die Unterstützung für die veraltete TCP-Bridge wurde
entfernt.

## Funktionsweise der Funktionsgenehmigung

1. Ein Node stellt eine Verbindung zum Gateway-WS her (die Gerätekopplung steuert diesen Schritt).
2. Das Gateway vergleicht die deklarierte Funktions-/Befehlsoberfläche mit der
   genehmigten Oberfläche; neue oder erweiterte Oberflächen speichern eine **ausstehende Anfrage** im
   Gerätedatensatz und lösen `node.pair.requested` aus.
3. Sie genehmigen die Anfrage oder lehnen sie ab (CLI oder Benutzeroberfläche).
4. Bis zur Genehmigung bleiben Node-Befehle gefiltert; die Genehmigung gibt die deklarierte
   Oberfläche gemäß der normalen Befehlsrichtlinie frei.

Ausstehende Anfragen laufen automatisch **5 Minuten nach dem letzten
Wiederholungsversuch des Nodes** ab – ein Node, der aktiv versucht, die Verbindung wiederherzustellen, hält seine einzige ausstehende Anfrage aktiv,
anstatt bei jedem Versuch eine neue Anfrage (und Genehmigungsaufforderung) zu erzeugen.

## CLI-Ablauf (für Headless-Systeme geeignet)

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes status
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name "Living Room iPad"
```

`nodes status` zeigt gekoppelte/verbundene Nodes und deren Funktionen an.

## API-Oberfläche (Gateway-Protokoll)

Ereignisse:

- `node.pair.requested` – wird ausgelöst, wenn eine neue ausstehende Anfrage erstellt wird.
- `node.pair.resolved` – wird ausgelöst, wenn eine Anfrage genehmigt, abgelehnt oder
  abgelaufen ist.

Methoden:

- `node.pair.list` – listet ausstehende und gekoppelte Nodes auf (`operator.pairing`).
- `node.pair.approve` – genehmigt eine ausstehende Anfrage.
- `node.pair.reject` – lehnt eine ausstehende Anfrage ab.
- `node.pair.remove` – entfernt einen gekoppelten Node. Dadurch wird die Rolle `node` des Geräts
  im Speicher für gekoppelte Geräte widerrufen, die genehmigte Node-Oberfläche ebenfalls entfernt und
  die Sitzungen dieses Geräts mit Node-Rolle werden ungültig gemacht/getrennt. Ein Gerät mit **mehreren Rollen**
  (beispielsweise eines, das auch `operator` besitzt) behält seine Zeile und verliert nur
  die Rolle `node`; die Zeile eines reinen Node-Geräts wird gelöscht. Autorisierung:
  `operator.pairing` darf Node-Zeilen entfernen, die nicht zu Operatoren gehören; ein Aufrufer mit Geräte-Token,
  der auf einem Gerät mit mehreren Rollen seine **eigene** Node-Rolle widerruft, benötigt zusätzlich
  `operator.admin`.
- `node.rename` – benennt den für Operatoren sichtbaren Anzeigenamen eines gekoppelten Nodes um.

In 2026.7 entfernt: `node.pair.request` und `node.pair.verify`. Ausstehende
Anfragen werden während Node-Verbindungen vom Gateway selbst erstellt, und das
eigenständige Token pro Node, für das sie vorgesehen waren, existiert nicht mehr; für die Node-Authentifizierung wird das
Gerätekopplungstoken verwendet.

Hinweise:

- Erneute Verbindungen mit unveränderter Oberfläche verwenden die ausstehende Anfrage wieder; wiederholte
  Anfragen aktualisieren die gespeicherten Node-Metadaten und den neuesten auf der Zulassungsliste stehenden
  Snapshot der deklarierten Befehle für die Sichtbarkeit durch Operatoren.
- Operator-Berechtigungsstufen und Prüfungen zum Genehmigungszeitpunkt sind unter
  [Operator-Berechtigungen](/de/gateway/operator-scopes) zusammengefasst.
- `node.pair.approve` verwendet die deklarierten Befehle der ausstehenden Anfrage, um
  zusätzliche Genehmigungsberechtigungen durchzusetzen:
  - Anfrage ohne Befehle: `operator.pairing`
  - Anfrage mit gewöhnlichen Befehlen: `operator.pairing` + `operator.write`
  - administrativ sensible Anfrage, die `system.run`, `system.run.prepare`,
    `system.which`, `browser.proxy`, `fs.listDir` oder
    `system.execApprovals.get/set` enthält: `operator.pairing` + `operator.admin`

<Warning>
Die Genehmigung der Node-Kopplung zeichnet die vertrauenswürdige Funktionsoberfläche auf. Sie fixiert **nicht** die aktuelle Node-Befehlsoberfläche pro Node.

- Aktuelle Node-Befehle ergeben sich aus den Deklarationen des Nodes beim Verbindungsaufbau, gefiltert durch
  die globale Node-Befehlsrichtlinie des Gateways (`gateway.nodes.commands.allow` und
  `gateway.nodes.commands.deny`).
- Die `system.run`-Zulassungs- und Nachfragerichtlinie pro Node befindet sich auf dem Node in
  `exec.approvals.node.*`, nicht im Kopplungsdatensatz.

</Warning>

## Steuerung von Node-Befehlen (2026.3.31+)

<Warning>
**Inkompatible Änderung:** Ab `2026.3.31` sind Node-Befehle deaktiviert, bis die Node-Kopplung genehmigt wurde. Die Gerätekopplung allein reicht nicht mehr aus, um deklarierte Node-Befehle bereitzustellen.
</Warning>

Wenn ein Node zum ersten Mal eine Verbindung herstellt, wird die Kopplung automatisch angefordert.
Bis diese Anfrage genehmigt ist, werden alle ausstehenden Node-Befehle dieses Nodes
gefiltert und nicht ausgeführt. Nach der Genehmigung der Kopplung werden die deklarierten
Befehle des Nodes gemäß der normalen Befehlsrichtlinie verfügbar.

Das bedeutet:

- Nodes, die sich bisher allein auf die Gerätekopplung verlassen haben, um Befehle bereitzustellen, müssen
  nun zusätzlich die Node-Kopplung abschließen.
- Vor der Kopplungsgenehmigung in die Warteschlange gestellte Befehle werden verworfen und nicht zurückgestellt.

## Vertrauensgrenzen für Node-Ereignisse (2026.3.31+)

<Warning>
**Inkompatible Änderung:** Von Nodes ausgehende Ausführungen bleiben nun auf eine eingeschränkte vertrauenswürdige Oberfläche begrenzt.
</Warning>

Von Nodes ausgehende Zusammenfassungen und zugehörige Sitzungsereignisse sind auf die
vorgesehene vertrauenswürdige Oberfläche beschränkt. Benachrichtigungsgesteuerte oder von Nodes ausgelöste Abläufe, die
zuvor auf einen umfassenderen Zugriff auf Host- oder Sitzungstools angewiesen waren, müssen möglicherweise angepasst werden.
Diese Absicherung verhindert, dass Node-Ereignisse über die
Vertrauensgrenze des Nodes hinaus Zugriff auf Tools auf Host-Ebene erlangen.

Dauerhafte Aktualisierungen der Node-Präsenz folgen derselben Identitätsgrenze: Das Ereignis
`node.presence.alive` wird nur von authentifizierten Sitzungen von Node-Geräten
akzeptiert und aktualisiert Kopplungsmetadaten nur, wenn die Geräte-/Node-Identität
bereits gekoppelt ist. Ein selbst deklarierter Wert `client.id` reicht nicht aus, um
den Zuletzt-gesehen-Status zu schreiben.

## SSH-verifizierte automatische Gerätegenehmigung (Standard)

Die erstmalige Gerätekopplung für `role: node` von einer privaten/CGNAT-Adresse wird
automatisch genehmigt, wenn das Gateway den **Besitz des Rechners über SSH nachweisen** kann: Es
verbindet sich zurück zum Kopplungshost (`BatchMode`, `StrictHostKeyChecking=yes`),
führt dort `openclaw node identity --json` aus und genehmigt nur, wenn die entfernte
Geräte-ID und der öffentliche Schlüssel exakt mit der ausstehenden Anfrage übereinstimmen. Der Schlüsselabgleich
macht dies sicher: Erreichbarkeit allein führt niemals zur Genehmigung, sodass andere NAT-Nutzer,
andere Benutzer auf einem gemeinsam genutzten Host und LAN-Spoofing auf die normale
Aufforderung zurückfallen.

Standardmäßig aktiviert. Voraussetzungen für die Auslösung:

- Der Benutzer des Gateway-Prozesses (oder `sshVerify.user`) kann sich per SSH
  nicht interaktiv mit dem Node-Host verbinden (Schlüssel/Agent; Tailscale SSH funktioniert ebenfalls), und dem Hostschlüssel wird
  bereits vertraut.
- `openclaw` wird auf dem entfernten `PATH` für nicht interaktives `sh -lc` aufgelöst.
- Die verbindende IP-Adresse ist eine direkte (nicht über einen Proxy geleitete, keine Loopback-Adresse) private, ULA-,
  Link-Local- oder CGNAT-Adresse oder entspricht `sshVerify.cidrs`, wenn dies festgelegt ist.
- Dieselbe Mindestvoraussetzung wie bei der Genehmigung über vertrauenswürdige CIDRs: nur eine neue Node-Kopplung
  ohne Berechtigungen; Upgrades, Browser, Control UI und WebChat erfordern immer eine Aufforderung.

Während eine Prüfung ausgeführt wird, wird der Node-Client angewiesen, weitere Versuche zu unternehmen
(`wait_then_retry`), anstatt für die manuelle Genehmigung zu pausieren; schlägt die Prüfung
fehl, fällt der nächste Versuch auf den normalen Aufforderungsablauf zurück. Fehlgeschlagene Ziele
erhalten eine kurze Sperrfrist (5 Minuten nach einer Schlüsselabweichung).

Für genehmigte Geräte werden `approvedVia: "ssh-verified"` sowie ihre erste deklarierte
Funktionsoberfläche im selben Schritt genehmigt – der Schlüsselabgleich weist bereits nach,
dass der Node unter dem Konto des Operators auf einem ihm gehörenden Rechner ausgeführt wird, was
derselben Aussage entspricht, die eine manuelle Funktionsgenehmigung bestätigt. Spätere Erweiterungen der Oberfläche erfordern weiterhin
eine Aufforderung.

Absichern oder deaktivieren:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        // Vollständig deaktivieren:
        sshVerify: false,
        // ...oder Umfang/Parameter der Prüfung festlegen:
        // sshVerify: { user: "me", identity: "~/.ssh/probe", timeoutMs: 7000, cidrs: ["10.0.0.0/8"] },
      },
    },
  },
}
```

## Automatische Genehmigung (macOS-App)

Die macOS-App kann eine **stille Genehmigung** von Anfragen zur Node-Funktionsgenehmigung versuchen,
wenn:

- die Anfrage als `silent` markiert ist (das Gateway markiert die erste Funktionsoberfläche
  als still, wenn die Gerätekopplung nicht interaktiv genehmigt wurde), und
- die App eine SSH-Verbindung zum Gateway-Host mit demselben
  Benutzer verifizieren kann.

Wenn die stille Genehmigung fehlschlägt, fällt sie auf die normale Approve/Reject-Aufforderung zurück.

## Automatische Gerätegenehmigung über vertrauenswürdige CIDRs

Die WS-Gerätekopplung für `role: node` bleibt standardmäßig manuell. Für private Node-
Netzwerke, in denen das Gateway dem Netzwerkpfad bereits vertraut, können Operatoren diese Funktion
mit expliziten CIDRs oder exakten IP-Adressen aktivieren:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Sicherheitsgrenze:

- Deaktiviert, wenn `gateway.nodes.pairing.autoApproveCidrs` nicht festgelegt ist.
- Es gibt keinen pauschalen automatischen Genehmigungsmodus für LANs oder private Netzwerke; die SSH-verifizierte
  automatische Genehmigung (oben) erfordert einen kryptografischen Abgleich des Geräteschlüssels und niemals
  nur die Netzwerkzugehörigkeit.
- Nur eine neue Gerätekopplungsanfrage für `role: node` ohne angeforderte Berechtigungen ist
  berechtigt.
- Operator-, Browser-, Control-UI- und WebChat-Clients bleiben manuell.
- Upgrades von Rollen, Berechtigungen, Metadaten und öffentlichen Schlüsseln bleiben manuell.
- Loopback-Pfade auf demselben Host mit Headern eines vertrauenswürdigen Proxys sind nicht berechtigt, da dieser
  Pfad von lokalen Aufrufern gefälscht werden kann.

## Bereinigung nach Ersetzung stiller Kopplungen

Nicht interaktive Genehmigungen zeichnen ihre Herkunft in der Zeile des gekoppelten Geräts auf:
Genehmigungen durch lokale Richtlinien auf demselben Host als `silent`, Node-Genehmigungen über vertrauenswürdige CIDRs als
`trusted-cidr`, SSH-verifizierte Node-Genehmigungen als `ssh-verified`. Clients mit einem flüchtigen Zustandsverzeichnis (temporäre Home-Verzeichnisse,
Container, Sandboxes pro Ausführung) erzeugen bei jeder Ausführung ein neues Geräteschlüsselpaar, und jede
Ausführung koppelt sich still als völlig neues Gerät – ohne Bereinigung wächst die Liste der gekoppelten Geräte
bei jeder Ausführung um eine veraltete Zeile.

Wenn das Gateway eine **lokale** Gerätekopplung still genehmigt, setzt es
ältere mit `silent` genehmigte Datensätze außer Betrieb, die zum selben Client-Cluster gehören
(Übereinstimmung von `clientId`, `clientMode` und Anzeigename) und derzeit nicht
verbunden sind. Lokale Clients werden auf dem Gateway-Host selbst ausgeführt, sodass der Cluster-Schlüssel
nicht mit einem anderen Rechner übereinstimmen kann. Außer Betrieb gesetzte Zeilen verlieren ihre Tokens sofort;
jeder passende veraltete Node-Kopplungseintrag wird gelöscht und ein
Entfernungsereignis `node.pair.resolved` wird übertragen.

Grenzen:

- Nur Datensätze, deren letzte Genehmigung lokal auf demselben Host (`silent`) erfolgte, kommen
  sowohl als Auslöser als auch als Ziel infrage. Durch vertrauenswürdige CIDRs und SSH verifizierte Kopplungen
  erstrecken sich über mehrere Hosts, bei denen die Anzeigemetadaten keine Maschinenidentität darstellen. Daher werden sie
  niemals automatisch entfernt — verwenden Sie dafür die Bereinigung in der Control UI oder
  `openclaw nodes remove`.
- Vom Eigentümer genehmigte sowie per QR-/Einrichtungscode (Bootstrap) vorgenommene Kopplungen werden niemals
  automatisch entfernt. Datensätze, die genehmigt wurden, bevor Herkunftsinformationen verfügbar waren, bleiben geschützt,
  selbst nach einer späteren stillen erneuten Genehmigung derselben Geräte-ID.
- Derzeit verbundene Geräte werden übersprungen, sodass gleichzeitige lokale Sitzungen mit
  separaten Statusverzeichnissen ihre Token behalten, solange sie aktiv sind. Datensätze, die
  innerhalb der letzten Minute genehmigt wurden, werden ebenfalls übersprungen, damit gleichzeitige Kopplungs-Handshakes
  einander nicht aufheben können, bevor ihre Verbindungen registriert sind.
- Betroffene Clients sind konstruktionsbedingt lokal und koppeln sich daher bei
  ihrer nächsten Verbindung still erneut.

## Automatische Genehmigung bei Metadatenaktualisierungen

Wenn ein bereits gekoppeltes Gerät erneut eine Verbindung herstellt und nur nicht vertrauliche Metadaten
geändert wurden (beispielsweise der Anzeigename oder Hinweise zur Clientplattform), behandelt OpenClaw
dies als `metadata-upgrade`. Die stille automatische Genehmigung ist eng begrenzt: Sie gilt nur
für vertrauenswürdige lokale Wiederverbindungen außerhalb des Browsers, die bereits den Besitz
lokaler oder gemeinsam genutzter Zugangsdaten nachgewiesen haben, einschließlich Wiederverbindungen nativer Apps auf demselben Host nach
Änderungen der Betriebssystem-Versionsmetadaten. Browser-/Control-UI-Clients und entfernte Clients
verwenden weiterhin den expliziten Ablauf zur erneuten Genehmigung. Erweiterungen des Berechtigungsumfangs (von Lesen auf
Schreiben/Administration) und Änderungen des öffentlichen Schlüssels kommen **nicht** für die
automatische Genehmigung bei Metadatenaktualisierungen infrage; sie bleiben explizite Anfragen zur erneuten Genehmigung.

## Hilfsfunktionen für die QR-Kopplung

`/pair qr` rendert die Kopplungsnutzdaten als strukturierte Medien, sodass mobile Clients und
Browser-Clients sie direkt scannen können.

Beim Löschen eines Geräts werden außerdem alle veralteten ausstehenden Kopplungsanfragen für diese
Geräte-ID entfernt, sodass `nodes pending` nach einem Widerruf keine verwaisten Zeilen anzeigt.

## Lokalität und weitergeleitete Header

Die Gateway-Kopplung behandelt eine Verbindung nur dann als Loopback, wenn sowohl der ursprüngliche Socket
als auch alle Hinweise eines vorgeschalteten Proxys übereinstimmen. Wenn eine Anfrage über Loopback eingeht, aber
`Forwarded`, einen beliebigen `X-Forwarded-*`- oder einen `X-Real-IP`-Header enthält,
widerlegen diese weitergeleiteten Header die Annahme der Loopback-Lokalität, und der
Kopplungspfad erfordert eine explizite Genehmigung, anstatt die
Anfrage still als Verbindung auf demselben Host zu behandeln. Die entsprechende Regel für die
Operatorauthentifizierung finden Sie unter [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth).

## Speicherung (lokal, privat)

Der Kopplungsstatus befindet sich in den Datensätzen der gekoppelten Geräte in der gemeinsam genutzten SQLite-Statusdatenbank
im Statusverzeichnis des Gateways (standardmäßig `~/.openclaw`):

- `~/.openclaw/state/openclaw.sqlite` (gekoppelte Geräte mit Geräteauthentifizierung,
  genehmigte Node-Oberflächen, ausstehende Oberflächenanfragen, ausstehende Gerätekopplungsanfragen
  und Bootstrap-Token)

Wenn Sie `OPENCLAW_STATE_DIR` überschreiben, wird die Datenbank entsprechend verschoben. Gateways,
die von Versionen mit JSON-Speichern aktualisiert wurden, importieren diese beim Start und belassen
die Archive `devices/*.json.migrated` und `nodes/*.json.migrated`.

Sicherheitshinweise:

- Geräte-Token sind Geheimnisse; behandeln Sie die Statusdatenbank als vertraulich.
- Zum Rotieren eines Geräte-Tokens wird `openclaw devices rotate` /
  `device.token.rotate` verwendet.

## Transportverhalten

- Der Transport ist **zustandslos**; er speichert keine Mitgliedschaften.
- Wenn das Gateway offline oder die Kopplung deaktiviert ist, können Nodes nicht gekoppelt werden.
- Im Remote-Modus erfolgt die Kopplung mit dem Speicher des entfernten Gateways.

## Verwandte Themen

- [Kanalkopplung](/de/channels/pairing)
- [Nodes-CLI](/de/cli/nodes)
- [Geräte-CLI](/de/cli/devices)
