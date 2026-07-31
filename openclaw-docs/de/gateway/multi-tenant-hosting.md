---
doc-schema-version: 1
read_when:
    - Sie hosten OpenClaw für mehrere Benutzer oder Organisationen
    - Sie müssen eine Isolationsgrenze für Mandanten-Workloads auswählen
summary: Mehrere Mandanten-Vertrauensdomänen als jeweils eine isolierte OpenClaw-Gateway-Zelle pro Mandant hosten
title: Mandantenfähiges Hosting
x-i18n:
    generated_at: "2026-07-26T17:49:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 383d32331b45d40db6fb4ff8242dd9a3cf8898a3ccab19f0372cd06bbd83fc05
    source_path: gateway/multi-tenant-hosting.md
    workflow: 16
---

# Multi-Tenant-Hosting

Das Standardsicherheitsmodell von OpenClaw sieht eine vertrauenswürdige Betreibergrenze pro Gateway vor, keine Isolation nicht vertrauenswürdiger Mandanten innerhalb eines gemeinsam genutzten Gateways. Das Hosting von Benutzern oder Organisationen, die keine gemeinsame Vertrauensgrenze haben, erfordert daher für jeden Mandanten eine separate vollständige OpenClaw-Instanz.

`openclaw fleet` bezeichnet jede isolierte Instanz als **Zelle**. Eine Zelle ist ein vollständiges Gateway in einem gehärteten Container mit eigenem Zustand, eigenen Anmeldedaten, eigenem Arbeitsbereich, eigenen Kanalkonten, eigenem Token und einem Host-Port, der ausschließlich an die Loopback-Schnittstelle gebunden ist.

Fleet ist **experimentell**: Seine Befehle, Flags und sein Containerprofil können sich zwischen Releases ohne Übergangsfrist ändern.

Fleet wurde auf Linux- und macOS-Hosts getestet. Windows-Hosts sind derzeit nicht getestet.

## Warum jeder Mandant eine Zelle benötigt

Ein authentifizierter Betreiber innerhalb eines Gateways hat eine vertrauenswürdige Kontroll­ebenenrolle. Sitzungs-IDs wählen das Routing aus; sie autorisieren keinen Mandanten gegenüber einem anderen. Agent-Sandboxing kann die Auswirkungen nicht vertrauenswürdiger Inhalte und der Werkzeugausführung reduzieren, macht ein gemeinsam genutztes Gateway jedoch nicht zu einer Autorisierungsgrenze zwischen Mandanten.

Verwenden Sie eine Zelle pro Mandant, damit jede Vertrauensdomäne über einen separaten Gateway-Prozess, Container, persistenten Zustandsbaum und eigene Gateway-Anmeldedaten verfügt. Dies entspricht dem [Gateway-Sicherheitsmodell](/de/gateway/security): Bringen Sie Benutzer, die einander nicht vertrauen, nicht im selben OpenClaw-Prozess oder unter demselben Betriebssystembenutzer unter.

## Architektur

Die Fleet-CLI ist eine hostseitige Lebenszyklusverwaltung. Sie erfasst Zellen in der OpenClaw-Zustandsdatenbank und weist eine lokale Docker- oder Podman-Laufzeitumgebung an, deren Container zu erstellen, zu untersuchen, zu starten, zu stoppen, zu ersetzen und zu entfernen. Remote-Laufzeitendpunkte werden nicht unterstützt, da die Bind-Pfade und Loopback-URLs von Fleet zum lokalen Host gehören. Fleet leitet keine Mandantennachrichten weiter und fügt keinen gemeinsamen Datenpfad auf Anwendungsebene zwischen Zellen hinzu.

Jede Zelle führt das offizielle `ghcr.io/openclaw/openclaw`-Image in einem eigenen benutzerdefinierten Bridge-Netzwerk aus. Separate Bridges verhindern direkten Datenverkehr zwischen den Container-IPs verschiedener Zellen, während der ausgehende NAT-Zugriff für Provider und Kanäle erhalten bleibt. Ausgehender Datenverkehr ist standardmäßig uneingeschränkt. Podman-Zellen können `--network internal` verwenden, um ausgehenden Datenverkehr zu blockieren und gleichzeitig den veröffentlichten Loopback-Port des Gateways beizubehalten. Interne Docker-Netzwerke unterbrechen diesen veröffentlichten Port, weshalb Fleet diese Kombination ablehnt; setzen Sie die Docker-Richtlinie für ausgehenden Datenverkehr stattdessen mit Host-Firewall-Regeln wie der `DOCKER-USER`-Kette durch. Das Gateway der Zelle lauscht im Container auf Port `18789`, während die Laufzeitumgebung ihn auf dem Host ausschließlich unter `127.0.0.1:<allocated-port>` veröffentlicht. Wenn Remotezugriff erforderlich ist, kann ein Betreiber einen genehmigten Reverse-Proxy, SSH-Tunnel oder ein Tailnet vor diesen Loopback-Endpunkt schalten.

Der persistente Gateway-Zustand stammt aus `<state-dir>/fleet/cells/<tenant>/` und wird unter `/home/node/.openclaw` eingebunden. Die Verschlüsselungsschlüssel für Authentifizierungsprofile stammen aus dem separaten Host-Pfad `<state-dir>/fleet/auth-profile-secrets/<tenant>/` und werden unter `/home/node/.config/openclaw` eingebunden, entsprechend dem offiziellen [Docker-Layout für Speicherung und Persistenz](/de/install/docker#storage-and-persistence). Der Schlüssel befindet sich nicht unterhalb der regulären Zustandseinbindung. Die Kanalkonten jedes Mandanten enden innerhalb der Zelle, der sie gehören; Fleet stellt weder ein gemeinsam genutztes Kanalkonto noch einen Router für eingehende Nachrichten bereit.

Das offizielle Image verwendet standardmäßig den Nicht-Root-Benutzer `node` mit der UID 1000. Fleet verwendet hostkompatible Benutzerzuordnungen, damit private Bind-Mounts beschreibbar bleiben: Podman verwendet `keep-id`, Docker im Root-Modus verwendet die aufrufende Nicht-Root-Identität und Docker im Rootless-Modus ordnet Container-Root dem nicht privilegierten Daemon-Benutzer zu. Docker und Podman wenden eine private `:Z`-Neukennzeichnung an, wenn SELinux auf dem Host aktiv ist. Das Containerprofil vermeidet privilegierte Host-Funktionen und ist für den Rootless-Betrieb geeignet; der Rootless-Betrieb ist jedoch eine Entscheidung und Voraussetzung der Host-Laufzeitumgebung und wird von Fleet nicht automatisch aktiviert.

## Vertrauensgrenze

Multi-Tenancy schützt Mandanten voreinander. Jeder Mandant vertraut dem Fleet-Betreiber und dem Host. Der Schutz vor einem kompromittierten Host ist kein Ziel.

Das bedeutet, dass ein Hostadministrator die Containerkonfiguration und -umgebung untersuchen, eingebundene Zellendaten lesen, Images ersetzen oder Container betreten kann. Gateway-Tokens und mit `--env` übergebene Werte sind für einen Administrator über die Inspektionsfunktionen von Docker oder Podman sichtbar. Verwenden Sie daher entsprechende Host-Kontrollen, Richtlinien für administrativen Zugriff, Überwachung, Sicherungen und einen genehmigten Secret-Manager.

Die Baseline verhindert eine versehentliche Netzwerkfreigabe über Wildcards und entfernt gängige Primitive zur Rechteausweitung in Containern, macht einen nicht vertrauenswürdigen Host jedoch nicht sicher.

## Isolationsstufen

Wählen Sie die Grenze, die zu den von Ihnen gehosteten Mandanten passt:

1. **Gehärtete Container-Baseline.** Fleet entfernt alle Linux-Capabilities, aktiviert `no-new-privileges`, wendet Grenzwerte für PID, Arbeitsspeicher, CPU und optional den beschreibbaren Datenträger-Layer an, verwendet separate persistente Einbindungen und Netzwerke pro Zelle und veröffentlicht ausschließlich an die Loopback-Schnittstelle des Hosts. Bridge-Netzwerke lassen ausgehenden Datenverkehr uneingeschränkt; verwenden Sie Podman `--network internal` oder eine Docker-Host-Firewall-Richtlinie, wenn eine Zelle keine ausgehenden Verbindungen initiieren darf. Dies ist das Standardprofil für Mandanten, die dem Betreiber und dem Host vertrauen.
2. **Stärkere Container- oder VM-Isolation.** Konfigurieren Sie Docker oder Podman für Arbeitslasten mit höherem Risiko so, dass eine stärkere OCI-Isolationslaufzeit wie gVisor oder Kata Containers verwendet wird, oder platzieren Sie Zellen in MicroVMs. Dies ist eine Laufzeit- oder Infrastrukturkonfiguration; die Option `--runtime docker|podman` von Fleet wählt die Container-CLI aus, nicht das OCI-Isolations-Backend. Siehe Dockers [alternative Containerlaufzeiten](https://docs.docker.com/engine/daemon/alternative-runtimes/) und den [Leitfaden zur Docker-VM-Laufzeit](/de/install/docker-vm-runtime).
3. **Separate Maschinen für nicht vertrauenswürdige Mandanten.** Bringen Sie einander nicht vertrauende Mandanten nicht im selben OpenClaw-Prozess oder unter demselben Betriebssystembenutzer unter. Wenn Mandanten nicht demselben Hostbetreiber vertrauen oder eine stärkere administrative Grenze benötigen, verwenden Sie separate VMs oder physische Hosts mit separater Laufzeitverwaltung.

Keine Stufe dieser Einteilung ändert das Vertrauensmodell der OpenClaw-Anwendung: Ein Gateway bleibt eine vertrauenswürdige Betreiberdomäne.

## Schnellstart

Erstellen Sie eine Zelle. Der Befehl gibt ein generiertes Gateway-Token einmalig aus; speichern Sie es daher sofort:

```bash
openclaw fleet create acme
```

Öffnen Sie die angegebene `http://127.0.0.1:<port>`-URL auf dem Fleet-Host, authentifizieren Sie sich mit dem Token dieses Mandanten und konfigurieren Sie die Provider-Anmeldedaten und Kanalkonten innerhalb der Zelle.

Prüfen Sie den Containerzustand und die Erreichbarkeit des Gateways:

```bash
openclaw fleet status acme
```

Führen Sie ein Upgrade durch, wobei Host-Port, eingebundene Daten, Ressourcenprofil, benutzerdefinierte Umgebung und Gateway-Token erhalten bleiben:

```bash
openclaw fleet upgrade acme
```

Entfernen Sie den Container und den Registrierungseintrag, wobei die Mandantendaten erhalten bleiben:

```bash
openclaw fleet rm acme --force
```

Um auch die persistenten Mandantendaten zu löschen, fügen Sie `--purge-data` hinzu. Die vollständige Löschung erfordert `--force`, ist unumkehrbar und führt vor dem Löschen eine Einschlussprüfung des aufgelösten Pfads durch:

```bash
openclaw fleet rm acme --purge-data --force
```

Alle Befehle und Optionen finden Sie in der [CLI-Referenz zu `openclaw fleet`](/de/cli/fleet).

## Aktueller Umfang

Fleet stellt die folgenden Funktionen nicht bereit:

- Gemeinsam genutzte Kanalkonten oder einen gemeinsamen Ingress-Router
- Reduzierte Host-Prozesse pro Mandant anstelle vollständiger OpenClaw-Instanzen
- Remote-Zellenhosts, die von einer zentralen Verwaltung gesteuert werden
- Ein Self-Service-Portal für Mandanten, eine Abrechnungsebene oder eine Benutzeroberfläche für delegierte Administration

Diese Funktionen erfordern ausdrückliche Verträge für Identität, Routing, Autorisierung und Fehlerdomänen. Nähern Sie sie nicht durch die gemeinsame Nutzung eines Gateways oder seiner Anmeldedaten durch mehrere Mandanten an. Fleet ist eine Lebenszyklusverwaltung für einen einzelnen Host; Multi-Maschinen-Fleets mit Identitätsverwaltung erfordern eine separate Kontrollebene.

## Verwandte Themen

- [`openclaw fleet`](/de/cli/fleet)
- [Gateway-Sicherheit](/de/gateway/security)
- [Mehrere Gateways](/de/gateway/multiple-gateways)
- [Docker](/de/install/docker)
- [Podman](/de/install/podman)
