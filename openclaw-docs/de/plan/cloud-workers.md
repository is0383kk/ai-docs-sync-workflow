---
read_when:
    - Entwurf oder Implementierung der Bereitstellung von Cloud-Workern, des Worker-Modus oder der Sitzungsübergabe
    - Änderungen an environments.*, dem Worker-Protokoll, der Transkriptaufnahme oder den RPCs des Inferenz-Proxys
    - Überprüfung der Sicherheitslage bei der Remote-Ausführung von Agenten
summary: Führen Sie Agentensitzungen auf kurzlebigen, per SSH erreichbaren Maschinen mit über den Gateway weitergeleiteter Inferenz und Live-Streaming in der Seitenleiste aus.
title: Plan für Cloud-Worker
x-i18n:
    generated_at: "2026-07-26T19:03:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 134c3f6e486837607225d95d12a3153525b14237b362b9f9957313d9bc379dc4
    source_path: plan/cloud-workers.md
    workflow: 16
---

## Status

Vorschlag, Revision 3. Nicht implementiert. Ausrichtung 2026-07 vereinbart; Revision 2 berücksichtigte die Ergebnisse der adversarialen Prüfung (dediziertes Worker-Protokoll, Zustandsautomaten für Platzierung/Umgebung, Git-bewusste eingehende Synchronisierung, unidirektionale Übergabe in v1, Sicherheitsformulierung zu kontrolliertem ausgehendem Datenverkehr). Revision 3 legt das Zuständigkeitsmodell für die Synchronisierung fest (der Worker erstellt Commits, das Gateway übernimmt und veröffentlicht sie), fügt einen einfachen Synchronisierungsmodus ohne Git hinzu, korrigiert die Worker-Ausführung auf vollständig innerhalb der Box, verlagert die Internetrichtlinie in die Bereitstellungsphase und nimmt die Agent-Weiterleitung wieder in Meilenstein 3 auf.

## Problem

OpenClaw-Agent-Sitzungen führen ihre Schleife, Tools und Inferenz innerhalb des Gateway-Prozesses auf einem einzelnen Rechner aus. Die Rechenleistung ist durch diesen Rechner begrenzt, lang laufende Aufgaben belegen ihn, und parallele Arbeiten konkurrieren um seine Ressourcen. Gehostete Produkte (Cursor-Cloud-Agenten, Claude Code im Web, Codex Cloud) lösen dies mit kurzlebigen Cloud-Sandboxes pro Aufgabe, erfordern jedoch die Infrastruktur und das Vertrauen eines Anbieters.

Betreiber, die bereits über zusätzliche Rechner verfügen (oder diese günstig mieten können), haben keine Möglichkeit anzugeben: Führen Sie diese Sitzung dort aus, zeigen Sie sie wie jede andere Sitzung in meiner Seitenleiste an und verwerfen Sie den Rechner anschließend.

## Ziele

- Eine vollständige Agent-Sitzung (Schleife + Tools) auf einem kurzlebigen Remote-Rechner („Cloud-Worker“) ausführen, während die Sitzung in der Control UI exakt wie eine lokale Sitzung angezeigt und gestreamt wird.
- Keine dauerhaft vorhandenen Zugangsdaten auf dem Worker (keine Provider-Authentifizierung, keine Forge-Token) und kein direkter ausgehender Netzwerkzugriff; die Box benötigt lediglich einen erreichbaren sshd.
- Bereitstellen, synchronisieren, ausführen, Ergebnisse erfassen, zerstören – vollständig automatisiert und Provider-austauschbar (erster Provider: Lease-CLIs im Crabbox-Stil).
- Laufende Arbeit an einer Turn-Grenze vom Gateway an einen Worker weiterleiten, ohne Transkript, Sitzungsidentität oder – wenn die Anfragebytes gleichwertig bleiben – die Provider-Cache-Affinität zu verlieren; Ergebnisse sicher zurückholen.
- Sowohl Menschen (UI) als auch Agenten (Tool) können Arbeit an einen Cloud-Worker weiterleiten.
- Tagelang laufende Sitzungen unterstützen; die Lebensdauer ist eine Richtlinie und keine hartcodierte Obergrenze.

## Nicht-Ziele (v1)

- Keine externen Coding-Harnesses (Claude Code, Codex CLI) auf Workern. Worker-Sitzungen führen ausschließlich den eingebetteten Runner von OpenClaw aus. Harness-Unterstützung ist eine optionale v2-Funktion, da Harnesses ihre eigene Inferenz mit eigenen Zugangsdaten ausführen.
- Keine Best-of-N-/parallele Auffächerung von Versuchen.
- Keine VPN-/Tailnet-Abhängigkeit. Als Transport dient ausschließlich SSH.
- Keine neue Sandbox-Laufzeit. Der Worker-Rechner bildet die Isolationsgrenze; später kann zusätzlich eine betriebssystembasierte Sandbox innerhalb der Box eingesetzt werden.
- Keine symmetrische Live-Migration in v1: Die Weiterleitung erfolgt von lokal → Worker; Worker → lokal erfordert eine angehaltene Sitzung sowie eine abgeschlossene Workspace-Abstimmung. Eine spätere bidirektionale Live-Übergabe baut auf demselben Barrierenmechanismus auf.
- Kein JSON-Nebenzustand auf dem Gateway; Umgebungs-, Platzierungs-, Cursor- und Berechtigungszustand werden in SQLite gespeichert.

## Vorbilder (was wir übernehmen, was wir umkehren)

- Cursor-Cloud-Agenten: Die Agent-Schleife läuft in ihrer Cloud; die VM ist ein Ziel für die Tool-Ausführung; ein nur erweiterbarer Konversationsspeicher wird an alle Clients gestreamt; ein Snapshot nach der Installation ermöglicht einen Warmstart; selbst gehostete Worker sind ausschließlich ausgehend verbundene Worker-Prozesse. Wir übernehmen das Modell, bei dem „die maßgebliche Konversationsquelle beim Orchestrator verbleibt“, sowie das Streaming-Modell; die Platzierung der Schleife kehren wir um (siehe Entscheidung unten).
- Codex Cloud: zweiphasige Laufzeit – vernetzte Einrichtungsphase, anschließend Offline-Agent-Phase mit entfernten Geheimnissen; Container-Zustands-Cache für schnelle Folgeaufgaben. Wir übernehmen die Phasentrennung als Konzept für unseren ausgehenden Datenverkehr und die Cache-Idee für vorgewärmte Images in v2.
- Claude Code im Web: VM pro Sitzung; Git-Proxy zur Isolierung von Zugangsdaten (echte Token gelangen nie in die Sandbox, Push ist auf den Sitzungs-Branch beschränkt); Dateisystem-Snapshot nach der Einrichtung; Teleport-Übergabe = gepushter Branch + wiedergegebener Verlauf. Wir übernehmen die Isolierung der Zugangsdaten und den Übergaberahmen, die ausgehende Synchronisierung erfolgt jedoch per rsync vom Gateway, sodass veränderte Arbeitsverzeichnisse funktionieren und sich kein Forge-Token in der Nähe der Box befindet.
- Copilot-Coding-Agent: standardmäßig verweigerter ausgehender Datenverkehr mit Positivliste für Paketregistrys. Unser Standard im laufenden Betrieb ist strenger (überhaupt kein direkter ausgehender Datenverkehr), da Inferenz und Websuche über den SSH-Tunnel erfolgen – siehe jedoch Sicherheit dazu, warum dies „kontrollierter ausgehender Datenverkehr“ und nicht „kein ausgehender Datenverkehr“ ist.

## Architekturentscheidung: Schleife auf dem Worker, Inferenz über das Gateway

Drei Platzierungen wurden erwogen:

1. Die Schleife verbleibt auf dem Gateway, der Worker führt Tools aus (Cursor-Modell). Sicherste Fehlerdomäne (Transkript, Inferenz, Genehmigungen und Wiederherstellung nach Neustarts bleiben vollständig lokal) und von Prüfern bevorzugter erster Meilenstein. Als Produktarchitektur verworfen: Die Nicht-Ausführungs-Tools von OpenClaw sind prozessinterne Dateisystemoperationen, sodass jedes Lesen/Bearbeiten/Durchsuchen von Dateien zu einem Netzwerk-Roundtrip oder einer umfangreichen Umgestaltung der Tool-Oberfläche in grobe Workspace-RPCs wird; das Laufzeitverhalten ist kommunikationsintensiv und latenzgebunden. Wir übernehmen dieses Prinzip dort, wo es bereits umgesetzt ist (Auslagerung der Ausführung auf Nodes), bauen jedoch keine Ebene zur Remote-Ausführung von Tools.
2. Schleife und Inferenz befinden sich beide auf dem Worker. Einfachste Fehlerdomäne, aber Modellzugangsdaten (einschließlich OAuth-Profile) müssten an kurzlebige Rechner übertragen werden, das Gateway verliert die Kontrolle über Richtlinien, Routing und Auditing, und die Migration ändert die Identität, die den Provider aufruft, wodurch Provider-Caches ungültig werden.
3. Schleife + Tools auf dem Worker, Modellaufrufe über einen Proxy durch das Gateway. Ausgewählt. Ein Roundtrip pro Modell-Turn statt pro Tool-Aufruf; Tools laufen neben dem Code; das Gateway bleibt alleiniger Eigentümer der Authentifizierungsprofile, des Provider-Routings und der Richtlinien; der Worker enthält keine Geheimnisse.

Der Preis von Option 3 ist eine synchrone Abhängigkeit vom Gateway während jedes Modell-Turns. Daher sind die Regeln für ihre Ausfallsicherheit Bestandteil der Entscheidung und kein nachträglicher Zusatz:

- Ein Ausfall des Gateways während eines Turns lässt den aktiven Provider-Aufruf fehlschlagen. Der Turn wird als fehlgeschlagen markiert und nach der erneuten Verbindung als neuer Turn wiederholt; ein laufender Provider-Stream wird nicht transparent wiedergegeben (Risiko doppelter Abrechnung/doppelter Tool-Aufrufe).
- Jede Worker↔Gateway-Operation enthält eine dauerhafte Identität (siehe Worker-Protokoll), sodass Verbindungen nach einer Wiederherstellung fortgesetzt werden oder zwischengespeicherte Endergebnisse abrufen, statt hängen zu bleiben.
- Das Gateway ist eine kapazitätsverwaltete Komponente: Begrenzungen für gleichzeitige Worker, Flusssteuerung und Lastabwurf gehören zum Umfang von v1 (siehe Kapazität).

Da das Gateway sowohl das Transkript speichert als auch den gesamten Provider-Datenverkehr initiiert, ist die Sitzung ortsunabhängig: Das Verschieben der Schleife zwischen Gateway und Worker ändert weder auf der Provider-Seite noch im UI-Datenpfad etwas. Dadurch werden die Weiterleitung und das Zurückholen kostengünstig.

## Komponenten

### 1. Umgebungszustandsautomat + Provider-Vertrag

`environments.*` ist im Gateway-Protokoll derzeit eine reine Statusprojektion. Den dauerhaften Kern bilden ein von SQLite verwalteter Umgebungsdatensatz und ein Zustandsautomat, die vor den RPC-Strukturen entworfen werden:

`requested → provisioning → bootstrapping → ready → (attached|idle) → draining → destroying → destroyed | failed | orphaned`

- Die Bereitstellung ist absturzsicher: Der Absichtsdatensatz wird vor dem Provider-Aufruf mit einer deterministischen Vorgangs-ID gespeichert, sodass ein Neustart des Gateways ein laufendes Lease übernehmen kann, statt eine doppelte Bereitstellung durchzuführen oder einen kostenpflichtigen Rechner zu verwaisen.
- Die Abstimmung nach Neustarts und ein Bereiniger für verwaiste Ressourcen (Provider `inspect` im Vergleich zu lokalen Datensätzen) sind Anforderungen für v1 und keine bloßen Härtungsmaßnahmen.

Provider-Vertrag (durch ein Plugin implementiert; keine Providernamen oder Richtlinien im Kern):

```ts
type WorkerProvider = {
  id: string;
  provision(profile: WorkerProfile, opId: string): Promise<WorkerLease>; // → SSH-Host/-Port/-Benutzer/-Schlüsselmaterial
  inspect(lease: { leaseId: string; profile: WorkerProfile }): Promise<LeaseStatus>; // Übernahme/Zustand/Bereinigung verwaister Ressourcen
  renew?(leaseId: string): Promise<void>; // langlebige Sitzungen im Verhältnis zu Provider-TTLs
  destroy(lease: { leaseId: string; profile: WorkerProfile }): Promise<void>; // idempotent, kehrt erst nach Nachweis des Abbaus zurück
};
```

RPCs: `environments.create`, `environments.destroy`, erweitertes `environments.list/status` (Provider, Lease-ID, Zustand, Alter, Leerlaufzeit, angehängte Sitzungen). Erste Provider: ein Wrapper für eine Lease-CLI im Crabbox-Stil (Produktpfad) und ein Provider für statische SSH-Hosts, der als ausschließlich für die Entwicklung vorgesehen gekennzeichnet ist – ein Worker auf einem gemeinsam genutzten Host kann fremde Hostdaten lesen, daher dienen statische Hosts der Funktionsentwicklung und sind nicht die Standardkonfiguration.

### 2. Worker-Bootstrap: OpenClaw auf der Box installieren

Kein eigens entwickeltes Worker-Artefakt und keine Abhängigkeit von der Verfügbarkeit von npm:

- Kanonische Installation für alle Modi: ein vom Gateway erzeugtes Worker-Bundle mit Inhalts-Hash (die eigene Build-Ausgabe des Gateways, als Tarball gepackt), das über SSH übertragen und auf der Box installiert wird. Dadurch werden Entwicklungs-Builds und noch nicht veröffentlichte Commits konstruktionsbedingt abgedeckt.
- `npm i -g openclaw@<exact gateway version>` ist eine Optimierung, wenn auf dem Gateway eine veröffentlichte Version ausgeführt wird; niemals `latest`.
- Der Bootstrap ist idempotent; bei einem vorgewärmten Lease mit übereinstimmendem Bundle-Hash wird die Installation übersprungen. Unvorbereitete Rechner benötigen möglicherweise eine vernetzte Toolchain-Phase (Node-Laufzeit) – diese ist Teil der Einrichtungsphase und wird anschließend geschlossen.
- Der Handshake überprüft den Worker-Build-Hash, den Satz unterstützter Protokollfunktionen und die Laufzeitkompatibilität. Die vorhandenen Versions-/Protokollprüfungen des Gateways reichen hierfür nicht aus (über SSH getunnelte Nodes sind von der Zurückweisung bei nicht exakt übereinstimmender Version ausgenommen), daher führt die Worker-Zulassung eine eigene Prüfung auf exakte Build-Übereinstimmung durch.

Der Worker-Modus (`openclaw worker`) ist ein Einstiegspunkt und keine Abspaltung: Verbindungsverarbeitung plus eingebetteter Agent-Runner, wobei Sitzungspersistenz und Modellaufrufe durch Gateway-RPCs gestützt werden. Er darf keine Gateway-Oberflächen starten: keine Kanäle, kein automatischer Plugin-Start über den Tool-Satz der Sitzung hinaus, kurzlebiges Zustandsverzeichnis, keine lokalen Authentifizierungsprofile.

### 3. Transport: alles über SSH

Das Gateway verwaltet die Konnektivität; der Worker benötigt lediglich sshd:

- Das Gateway öffnet eine SSH-Verbindung zum Worker (Zugangsdaten aus dem Provider-Lease, Hostschlüssel aus der Bereitstellungsausgabe fest hinterlegt – kein `StrictHostKeyChecking=no`) und richtet einen Reverse-Tunnel ein, der einen lokalen Worker-Socket an den WS-Endpunkt des Gateways weiterleitet.
- Steuerungs-/Modellverkehr und Workspace-Übertragung verwenden separate SSH-Verbindungen mit demselben fest hinterlegten Vertrauensmaterial, damit rsync Token-Streams nicht durch Head-of-Line-Blocking verzögern kann.
- Der Lebenszyklus des Tunnels (Keepalive, erneute Verbindung mit Backoff) wird von der Umgebungslaufzeit auf dem Gateway verwaltet. Eine kurze Tunnelunterbrechung ist auf Sitzungsebene unsichtbar: Der dauerhafte Protokollzustand (unten) ermöglicht es dem Worker, sich erneut anzuhängen und fortzufahren.

### 4. Worker-Protokoll (dediziert; nicht das Node-Protokoll)

Eine adversariale Prüfung der aktuellen Node-Schnittstellen schloss eine einfache Wiederverwendung aus: Ausstehende Node-Aufrufe sind prozesslokale Promises, die mit der Verbindung enden, Node-Idempotenzschlüssel werden zwar geparst, aber nicht dedupliziert, und – entscheidend – ein verbundener Node kann gewöhnliche Node-Ereignisse senden (einschließlich Anforderungen für Agent-Ausführungen), sodass „Node-Typ + Fähigkeitsobergrenze“ keine Sicherheitsgrenze für eingehende Daten darstellt. Worker erhalten daher eine authentifizierte `worker`-Rolle mit einer geschlossenen, versionierten Positivliste für RPCs/Ereignisse; Worker-Verbindungen können keine älteren Node-Ereignishandler erreichen.

Identität und Zugangsdaten: Bei der Bereitstellung werden kurzlebige Worker-Zugangsdaten erstellt, die an die Umgebungs-ID, den Worker-Schlüssel, den Bundle-Hash, die einzige zulässige Sitzung, den zulässigen RPC-Satz und ein Ablaufdatum gebunden sind. Die durch SSH verifizierte Kopplung gilt weiterhin (wir haben die Box bereitgestellt und besitzen den Schlüssel), die Autorisierung erfolgt jedoch anhand der erstellten Zugangsdaten und nicht anhand der deklarierten Node-Oberfläche.

Dauerhafte Operationssemantik (Struktur aus der vorhandenen ACP-Laufzeit und ihrem Ereignis-Ledger übernommen – stabile Handles, Serialisierung pro Sitzung, dauerhafte Wiedergabe von `(session, seq)`):

- Jeder Vorgang ist `(sessionId, lifecycleRevision, runId, ownerEpoch, streamKind, seq)` zugeordnet.
- Besitzepochen grenzen veraltete Worker aus: Ein Ersatz-Worker erhöht die Epoche; verspätete Ergebnisse aus der alten Epoche werden deterministisch abgelehnt.
- Mindestens-einmal-Zustellung mit persistierten ACK-Cursorn und zwischengespeicherten Endergebnissen in SQLite; die Deduplizierung ist deterministisch. Keine Zusicherungen einer Genau-einmal-Zustellung.
- Explizite Frames für Abbruch, Schließen, Fortsetzen und Endergebnisse; Credit-/Fenster-basierte Flusssteuerung für Streams.
- Die Aushandlung von Protokollfunktionen ist von der allgemeinen Node-Protokollversion unabhängig.

### 5. RPCs des Sitzungs-Backends

Zwei unterschiedliche Verträge – die aktuelle Codebasis trennt dauerhafte Transkriptmutationen (im Besitz des Sitzungsmanagers, JSONL-Baum mit Eltern-/Blattzustand) von prozesslokalen Live-Ereignissen (Streaming-Deltas, Tool-Lebenszyklus, Genehmigungen), und das Worker-Protokoll muss diese Trennung beibehalten:

- Dauerhafte Transkript-Commits: Der Worker übermittelt semantische Anhänge-Batches mit `runEpoch` + Compare-and-Swap des Basisblatts; der Gateway-Sitzungsmanager erzeugt Eintrags-IDs und Eltern-IDs. Der Worker darf niemals vertrauenswürdige Transkriptzeilen, Eintrags-IDs, Eltern-IDs oder fremde Sitzungs-IDs bereitstellen.
- Wiederholbare Live-Ereignisse: eine typisierte Ereignis-Union mit Worker-Sequenznummern, Gateway-ACKs, begrenzter Aufbewahrung und Ausgrenzung verspäteter Ereignisse, die den bestehenden Agent-Ereignis-Fan-out speist, sodass Chatansicht, Tool-Zeilen und Ungelesen-/Statuslogik sich identisch zu lokalen Sitzungen verhalten.

Inferenz-Proxy: Das Ereignisvokabular des bestehenden Runtime-Proxy-Stream-Clients (`src/agents/runtime/proxy.ts`) wird wiederverwendet, aber die Vertrauensgrenze wird verschoben. Der Worker sendet nur Sitzungs-/Ausführungsidentität, eine genehmigte Modellreferenz, Kontext und eingeschränkte Generierungsoptionen; der Gateway löst Provider, Endpunkt, Authentifizierung, Header, Routing und Kostenrichtlinie aus seinem eigenen Katalog auf. Ein vom Worker bereitgestelltes Modellobjekt (z. B. ein angreifergesteuertes `baseUrl`) wird abgelehnt. Größenlimits für Anfragen, Abbruch, Audit und Wiederholung von Endergebnissen gelten ebenfalls. Im Gateway ausgeführte Tools (Websuche) werden auf dem Gateway ausgeführt und geben Ergebnisse über denselben Kanal zurück.

### 6. Workspace-Synchronisierung

Der Synchronisierungsanker ist ein Gateway-lokaler Workspace mit exklusivem Platzierungsbesitz: bei Git-Workspaces ein dedizierter verwalteter Worktree (die bestehenden Metadaten verwalteter Worktrees – Branch, Basis, Snapshot-Besitz – bilden die Grundlage); bei Nicht-Git-Workspaces ein dem Gateway gehörendes Zielverzeichnis. Niemals der Live-Checkout des Benutzers. Der exklusive Besitz, solange die Sitzung remote platziert ist, macht die eingehende Synchronisierung konstruktionsbedingt konfliktfrei.

Aufteilung des Besitzes – Commit vs. Veröffentlichung:

- Der Worker-seitige Agent erstellt in seiner Kopie wie üblich Commits (`git commit` ist ein lokaler Vorgang ohne Zugangsdaten; die Autorenidentität wird aus der Gateway-Konfiguration projiziert). Diese Commits sind inaktive Objekte, bis der Gateway sie übernimmt.
- Der Gateway erledigt alles, was Vertrauen erfordert: Überprüfung, dass eingehende Commits auf der aufgezeichneten Basis aufbauen, Fast-Forward des lokalen Worktrees, Push, PR-Erstellung und optionale Signierung/Neusignierung – alles mit Gateway-lokalen Zugangsdaten. Der Worker besitzt niemals Git- oder Forge-Zugangsdaten und greift niemals auf ein Remote-Repository zu.

Zwei Synchronisierungsmodi, ausgewählt danach, ob der Workspace ein Git-Repository ist:

- Git-Modus. Ausgehend: Der Worktree wird per rsync (einschließlich nicht committeter und zulässiger nicht verfolgter Dateien; Ein-/Ausschlüsse im Crabbox-Stil, `.worktreeinclude` wird beachtet) über die SSH-Identität des Tunnels übertragen und als unveränderliches Basismanifest (Inhaltshashes + Basis-Commit) aufgezeichnet. Eingehend: Neue Commits werden als Git-Bundle oder temporäre Referenz auf Grundlage der aufgezeichneten Basis zurückgegeben; nicht verfolgte Artefakte werden über ein explizites Manifest mit Prüfungen von Größe, Typ und Symlink-Begrenzung zurückgegeben. Bei der Übernahme wird die Abstammung von der Basis überprüft und bei Divergenz angehalten – nichts überschreibt stillschweigend eine der beiden Seiten. Löschungen, Umbenennungen, Submodule und Symlink-Ausbrüche werden durch die Manifestregeln und nicht durch rsync-Heuristiken behandelt.
- Einfacher Modus (kein Git – z. B. beim Erstellen eines Projekts von Grund auf auf der Box). Ausgehend gelten dieselbe rsync-Übertragung und dasselbe Basismanifest. Eingehend erfolgt eine anhand des Manifests ermittelte differenzielle Spiegelung zurück in das dem Gateway gehörende Zielverzeichnis, einschließlich Weitergabe von Löschungen. Der Modus ist aus demselben Grund wie der Git-Modus sicher: Exklusiver Besitz bedeutet, dass keine gleichzeitigen lokalen Bearbeitungen vorhanden sind, mit denen Konflikte entstehen könnten; das Basismanifest erkennt dennoch unerwartete lokale Abweichungen und hält an, statt Dateien zu überschreiben.

Checkpoints schützen tagelange Sitzungen vor Lease-Verlust: regelmäßige eingehende Checkpoints (Sitzungs-Branch-Commits im Git-Modus, Manifest-Snapshots im einfachen Modus); der Rhythmus ist eine Profilrichtlinie (standardmäßig rundenbasiert).

### 7. Platzierungs-Zustandsautomat, Sitzungen und Benutzeroberfläche

Die Runtime-Platzierung ist ein SQLite-eigener, an die Sitzung gebundener Zustandsautomat und kein Paar loser Zeilenfelder:

`local → requested → provisioning → syncing → starting → active(worker) → draining → reconciling → local | reclaimed | failed`

Er persistiert Umgebungs-ID, Übergangsgeneration, aktive Besitzerepoche, Workspace-Basismanifest, Worker-Bundle-Hash und letzte ACK-Cursor. Die Rundenzulassung beansprucht die Platzierung atomar, bevor eine der Schleifen eine Runde startet. Dadurch kann eine lokale Nachricht, die anhand eines veralteten Snapshots zugelassen wurde, niemals mit einer Worker-Runde konkurrieren – zu jedem Zeitpunkt besitzt genau eine Schleife die Sitzung.

Benutzeroberfläche:

- Eine Worker-Sitzung ist eine gewöhnliche Sitzungszeile mit Platzierungsmetadaten. Sie befindet sich im normalen Speicher, wird über `sessions.list` aufgelistet und über bestehende Abonnements gestreamt – Seitenleiste und Chat benötigen keinen neuen Datenpfad, sondern nur eine Darstellung: ein Worker-Badge und den Platzierungs-/Umgebungsstatus (`provisioning / syncing / running / idle / reconciling / reclaimed`).
- Erstellungs-UX: Die Sitzungs-Zielleiste (Neugestaltung der Sitzungsseitenleiste) erhält neben Gateway und Node ein Cloud-Worker-Ziel. Erfordert ein konfiguriertes Provider-Profil; die Funktion ist unsichtbar, bis sie konfiguriert wurde.
- Agent-Weiterleitung: Ein Sitzungstool ermöglicht es einem Agenten, Arbeit an einen Cloud-Worker zu übergeben, wie es ein Mensch tut (Worker-gestützte Untersitzung im Stil eines Unteragenten). Wird im selben Meilenstein wie die menschliche Weiterleitung ausgeliefert und durch dieselbe Opt-in-Provider-Konfiguration gesteuert. Rekursion ist strukturell begrenzt (Worker-Sitzungen können in v1 nicht selbst Worker beauftragen); die Ausgabenkontrolle erfolgt über umgebungsbezogene Abrechnung/Audits und nicht über Kontingentmechanismen.

## Weiterleitung und Übergabe

v1 ist bewusst asymmetrisch:

- Lokal → Worker (Weiterleitung): Die nachstehende Migrationsbarriere passieren, einen Worker bereitstellen oder wiederverwenden, synchronisieren, die Platzierung umschalten; die nächste Runde wird remote ausgeführt.
- Worker → lokal (Zurückholen): Die Sitzung anhalten (den Worker über dieselbe Barriere leeren), den eingehenden Abgleich abschließen und die Platzierung auf lokal umschalten. Keine Live-Migration.
- Symmetrische Live-Übergabe (Verschieben einer aktiv arbeitenden Sitzung in beide Richtungen ohne Anhalten) verwendet dieselben Barrieren- und Abgleichsmechanismen und wird ausgeliefert, nachdem Fehlerinjektionstests die Barriere nachgewiesen haben.

Migrationsbarriere („Rundengrenze“ allein reicht nicht aus – Genehmigungen, Hintergrundprozesse und Transkriptzusammenführungen nach Freigabe einer Sperre können sie überlappen):

1. Zulassung neuer Runden anhalten (Platzierungsanspruch).
2. Aktive Ausführungen abbrechen oder leeren.
3. Ausstehende Ausführungsgenehmigungen und Ausführungsberechtigungen widerrufen.
4. Seitliche Transkriptschreibvorgänge und ACKs für Live-Ereignisse leeren.
5. Untergeordnete Worker-Prozesse beenden.
6. Den alten Besitzer durch Erhöhen der Besitzerepoche ausgrenzen.
7. Workspace abgleichen (eingehend, konfliktbewusst).
8. Den neuen Besitzer aktivieren.

Cache-Affinität: Da Provider-Anfragen bei beiden Platzierungen vom Gateway ausgehen, bleibt die Cache-Affinität erhalten, wenn die serialisierte Provider-Anfrage äquivalent bleibt – gleiche Tool-Reihenfolge, Systemanweisungen, Provider-Wrapper und Cache-Metadaten (die auf der Gateway-Seite verbleiben). Dies ist eine testbare Eigenschaft und keine Annahme: Byte-Äquivalenztests zwischen lokaler und Worker-Platzierung für jeden unterstützten Provider-Transport sind Teil des Meilensteins, der die Worker-Schleife einführt.

## Sicherheitsmodell

Präzise formuliert: Der Worker hat keinen direkten ausgehenden Netzwerkzugriff und keine dauerhaft hinterlegten Provider-/Forge-Zugangsdaten. Er hat nicht „keinen ausgehenden Zugriff“ – Inferenz und vom Gateway ausgeführte Tools sind kontrollierte ausgehende Kanäle (ein durch Prompt-Injection manipulierter Worker kann weiterhin Workspace-Bytes in Modellkontext oder Websuchanfragen einfügen). Dementsprechend:

- Abrechnung kontrollierter ausgehender Daten: umgebungsbezogenes Audit und für Betreiber sichtbare Abrechnung beim Inferenz-Proxy und bei Gateway-Tools. Raten-/Byte-Limits existieren als Protokollflusssteuerung (Kapazität), nicht als Ausgabenkontingentmechanismen.
- Der eingehende Zugriff des Workers auf den Gateway ist auf die geschlossene Positivliste des Worker-Protokolls beschränkt; Transkriptschreibvorgänge sind strukturell eingeschränkt (vom Gateway erzeugte IDs, eine einzige gebundene Sitzung).
- Worker-Ausführungen besitzen innerhalb der Box vollständige Berechtigungen. Die Box ist entbehrlich und frei von Zugangsdaten, sodass eine Genehmigung pro Befehl Reibung erzeugt, ohne etwas zu schützen; die geschützte Grenze bilden der eingehende Abgleich und das Audit. Ausführungen durchlaufen niemals den Node-Genehmigungspfad des Gateways.
- Die Internetrichtlinie ist eine Provider-Entscheidung zum Bereitstellungszeitpunkt: Das Umgebungsprofil entscheidet bei der Erstellung der Box (Firewall/Sicherheitsgruppe/Netzwerk ohne ausgehenden Zugriff), optional mit einer vernetzten Einrichtungsphase, die der Provider vor der Agent-Phase schließt. Der Core implementiert keinen Netzwerk-Umschalter zur Laufzeit.
- Box-Hygiene zum Bereitstellungszeitpunkt: Cloud-Metadaten-Endpunkt blockiert oder nachweislich nicht vorhanden, kein Instanzprofil, kein übernommener SSH-Agent, kein Docker-Socket, saubere Umgebung/Home-Verzeichnis. SSH-Hostschlüssel werden anhand der Bereitstellungsausgabe fest hinterlegt.
- Genehmigungen und Richtlinien für alles Gateway-seitige (Push, PR, Provider-Aufrufe) werden weiterhin auf dem Gateway ausgeführt.

Auswirkungsradius einer kompromittierten Worker-Sitzung: die synchronisierte Workspace-Kopie sowie das, was die auditierten Proxy-Kanäle erlauben – keine Zugangsdaten, kein direkter Netzwerkzugriff, keine Gateway-Oberfläche außerhalb der Positivliste.

## Kapazität

Der Gateway leitet jeden Prompt und Token-Stream für N Worker weiter, daher definiert v1 ein Kapazitätsmodell, statt es erst im Produktionsbetrieb zu entdecken: Limits für gleichzeitige Worker pro Gateway, Credit-Fenster pro Stream (die aktuelle Ereignis-Stream-Warteschlange ist unbegrenzt, und die Obergrenze des Node-Socket-Puffers erzwingt das Schließen langsamer Verbraucher – beides ist unverändert ungeeignet), begrenzte Zwischenspeicherung auf Festplatte für Lastspitzen sowie Lastabwurf mit sichtbaren Rückstauzuständen in der Benutzeroberfläche. Die Workspace-Übertragung bleibt auf ihrem eigenen SSH-Kanal.

## Lebenszyklus

- Automatisches Anhalten bei Inaktivität und TTL sind Richtlinien des Provider-Profils und keine festen Konstanten. Die Standardwerte sind großzügig und bieten explizites Keep-alive; tagelange Arbeit wird als primärer Anwendungsfall unterstützt (Provider `renew` ist für Lease-basierte Backends vorhanden); eine Sitzung mit einer laufenden Runde oder kürzlich erfolgter Aktivität wird niemals zurückgefordert.
- Bei Ausfall oder Rückforderung des Workers: Die Platzierung wechselt zu `reclaimed`, die Sitzungszeile bleibt bestehen, und die nächste Nachricht stellt einen neuen Worker bereit und synchronisiert erneut vom letzten Checkpoint. Die Unterhaltung geht niemals verloren (Gateway-seitiger Speicher); Workspace-Änderungen seit dem letzten Checkpoint gehen verloren, und die Benutzeroberfläche weist darauf hin.
- Wiederverwendung warmer Leases ab dem ersten Tag (bei Providern, die dies unterstützen); ein Image-Snapshot nach dem Bootstrap ist der Schnellstartpfad für v2.

## Konfigurationsoberfläche

Minimal und optional: ein Provider-Profilblock (Provider-ID, Zugangsdaten-/CLI-Referenz, Synchronisierungsregeln, Lebensdauerrichtlinie, Budgets, optionale Einrichtungsphase) sowie eine Platzierungsauswahl pro Sitzung. Keine neuen Umgebungsvariablen. Nicht konfigurierte Installationen zeigen nichts an.

## Meilensteine

Die Implementierung wird als kleine, unabhängig zusammenführbare PRs eingespielt; jeder nachstehende Meilenstein ist eine PR-Serie und keine einzelne Änderung.

1. Grundlagen: Umgebungs-Zustandsautomat + Provider-Vertrag + Provider nach crabbox-Schema (statisches SSH als Entwicklungs-Testumgebung), Bootstrap des Worker-Bundles + Zulassungs-Handshake, SSH-Tunnel + Hostschlüssel-Pinning, Snapshot des verwalteten Worktrees + ausgehende Synchronisierung (Git- + Klartextmodi). Bereinigung verwaister Ressourcen + Übernahme nach Neustart.
2. Worker-Protokoll + Worker-Schleife: authentifizierte Worker-Rolle, dauerhafte Vorgänge/Epochen/ACK-Cursor, Transkript-Commit + Verträge für Live-Ereignisse, Inferenz-Proxy mit vom Gateway aufgelösten Modellen, Flusssteuerung. Ein Provider, ausschließlich manuelles Dispatching neuer Sitzungen, keine Übergabe. Fehlerinjektionstests (Tunnelunterbrechung, Gateway-Neustart, Worker-Ausfall) sind Voraussetzung für den Abschluss.
3. Dispatching + Rückübertragung + Agenten-Dispatching: Migrationsbarriere, mit der Zielleiste der UI verbundener Platzierungs-Zustandsautomat, eingehende Abstimmung + Checkpoints, Audit pro Umgebung, Kapazitätsgrenzen, Agenten-Dispatching-Tool (Worker-Sitzungen können nicht rekursiv dispatchen). Tests zur Byte-Äquivalenz des Prompt-Caches.
4. Symmetrische Live-Übergabe nach dem Fehlerinjektionsnachweis für Meilenstein 3.

Später: ACP-Testumgebungen auf Workern als umgebungsspezifische optionale Zugangsdaten-Hydratisierung; Schnellstart per Snapshot/Warm-Image; Fan-out (N Leases, derselbe Prompt); Betriebssystem-Sandboxing innerhalb der Box; umfassendere Artefakterfassung über das Artefaktschema.

## Offene Fragen

- Verfügbarkeit von Plugins/Skills auf Workern: Im Repository enthaltene Skills werden ohne Zusatzaufwand mit dem Workspace synchronisiert; für im Gateway konfigurierte Agenten-Skills/-Plugins ist eine ausdrückliche Entscheidung zur Synchronisierung oder zum Ausschluss erforderlich (das Tool-/Plugin-Manifest ist in jedem Fall Teil des Zulassungs-Handshakes).
- Standardmäßiger Checkpoint-Takt: rundenbasiert oder zeitbasiert für sehr kommunikationsintensive Sitzungen.
- Zusammenspiel von Umgebungsprofilen und Multi-Agent-Routing (agentenspezifische Standardprofile oder ausschließlich sitzungsspezifische Auswahl).
