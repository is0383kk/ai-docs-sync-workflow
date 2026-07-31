---
read_when:
    - Überprüfung der Sicherheitslage oder von Bedrohungsszenarien
    - Arbeiten an Sicherheitsfunktionen oder Audit-Antworten
summary: OpenClaw-Bedrohungsmodell, abgebildet auf das MITRE-ATLAS-Framework
title: Bedrohungsmodell (MITRE ATLAS)
x-i18n:
    generated_at: "2026-07-26T18:06:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c88ffdef850bd2afaf835baab2555304c914a0be1df6b6b9109e0f55d1448392
    source_path: security/THREAT-MODEL-ATLAS.md
    workflow: 16
---

**Version:** 1.0-Entwurf | **Framework:** [MITRE ATLAS](https://atlas.mitre.org/) (Bedrohungslandschaft durch Angriffe auf KI-Systeme) + Datenflussdiagramme

Dieses Bedrohungsmodell dokumentiert von Angreifern ausgehende Bedrohungen für die KI-Agentenplattform OpenClaw und den Skills-Marktplatz ClawHub. Es ist ein fortlaufend aktualisiertes Dokument, das von der OpenClaw-Community gepflegt wird. Unter [Mitwirkung am Bedrohungsmodell](/de/security/CONTRIBUTING-THREAT-MODEL) erfahren Sie, wie Sie neue Bedrohungen melden, Angriffsketten vorschlagen oder Gegenmaßnahmen empfehlen können.

**Wichtige ATLAS-Ressourcen:** [Techniken](https://atlas.mitre.org/techniques/) | [Taktiken](https://atlas.mitre.org/tactics/) | [Fallstudien](https://atlas.mitre.org/studies/) | [ATLAS GitHub](https://github.com/mitre-atlas/atlas-data) | [Mitwirkung an ATLAS](https://atlas.mitre.org/resources/contribute)

---

## 1. Geltungsbereich

| Komponente                    | Enthalten | Hinweise                                                  |
| ----------------------------- | --------- | --------------------------------------------------------- |
| OpenClaw-Agentenlaufzeit      | Ja        | Zentrale Agentenausführung, Tool-Aufrufe, Sitzungen        |
| Gateway                       | Ja        | Authentifizierung, Routing, Kanalintegration               |
| Kanalintegrationen            | Ja        | WhatsApp, Telegram, Discord, Signal, Slack usw.            |
| ClawHub-Marktplatz            | Ja        | Veröffentlichung, Moderation und Verteilung von Skills     |
| MCP-Server                    | Ja        | Externe Tool-Provider                                      |
| Benutzergeräte                | Teilweise | Mobile Apps, Desktop-Clients                               |

Nicht abgedeckte Meldungen und Muster für Fehlalarme (öffentliche Erreichbarkeit über das Internet, ausschließlich auf Prompt Injection beruhende Angriffsketten ohne Umgehung einer Grenze, gegenseitig nicht vertrauende Betreiber auf demselben Gateway-Host und weitere) sind in [`SECURITY.md`](https://github.com/openclaw/openclaw/blob/main/SECURITY.md) aufgeführt; für den aktuellen Geltungsbereich von Schwachstellenmeldungen ist diese Datei maßgeblich, nicht diese Seite.

## 2. Systemarchitektur

### 2.1 Vertrauensgrenzen

```text
┌─────────────────────────────────────────────────────────────────┐
│                    NICHT VERTRAUENSWÜRDIGE ZONE                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  WhatsApp   │  │  Telegram   │  │   Discord   │  ...         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
└─────────┼────────────────┼────────────────┼──────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 VERTRAUENSGRENZE 1: Kanalzugriff                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      GATEWAY                              │   │
│  │  • Gerätekopplung (1 Std. TTL für DM-Kopplung /           │   │
│  │    5 Min. TTL für Node-Kopplung)                          │   │
│  │  • AllowFrom-/Positivlistenvalidierung                    │   │
│  │  • Token-/Passwort-/Tailscale-Authentifizierung           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 VERTRAUENSGRENZE 2: Sitzungsisolierung           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   AGENTENSITZUNGEN                        │   │
│  │  • Sitzungsschlüssel = agent:channel:peer                 │   │
│  │  • Tool-Richtlinien je Agent                              │   │
│  │  • Transkriptprotokollierung                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 VERTRAUENSGRENZE 3: Tool-Ausführung              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  AUSFÜHRUNGS-SANDBOX                      │   │
│  │  • Docker-Sandbox (Standard) oder Host                    │   │
│  │    (Ausführungsgenehmigungen)                             │   │
│  │  • Node-Fernausführung                                    │   │
│  │  • SSRF-Schutz (DNS-Pinning + IP-Blockierung)             │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 VERTRAUENSGRENZE 4: Externe Inhalte              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              ABGERUFENE URLs / E-MAILS / WEBHOOKS        │   │
│  │  • Kapselung externer Inhalte                             │   │
│  │    (XML-Tags mit zufälliger Begrenzung)                   │   │
│  │  • Einfügung von Sicherheitshinweisen                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 VERTRAUENSGRENZE 5: Lieferkette                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLAWHUB                              │   │
│  │  • Veröffentlichung von Skills                            │   │
│  │    (SemVer, SKILL.md erforderlich)                        │   │
│  │  • Moderationsprüfung mit statischen Mustern und          │   │
│  │    AST-naher Analyse                                      │   │
│  │  • LLM-basierte agentische Risikoprüfung +                │   │
│  │    VirusTotal-Prüfung                                     │   │
│  │  • Überprüfung des Alters von GitHub-Konten (14 Tage)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Datenflüsse

| Fluss | Quelle  | Ziel     | Daten                       | Schutz                   |
| ----- | ------- | -------- | --------------------------- | ------------------------ |
| F1    | Kanal   | Gateway  | Benutzernachrichten          | TLS, AllowFrom           |
| F2    | Gateway | Agent    | Weitergeleitete Nachrichten | Sitzungsisolierung       |
| F3    | Agent   | Tools    | Tool-Aufrufe                | Richtliniendurchsetzung  |
| F4    | Agent   | Extern   | `web_fetch`-Anfragen | SSRF-Blockierung         |
| F5    | ClawHub | Agent    | Skill-Code                  | Moderation, Prüfung      |
| F6    | Agent   | Kanal    | Antworten                   | Ausgabefilterung         |

---

## 3. Bedrohungsanalyse nach ATLAS-Taktik

### 3.1 Aufklärung (AML.TA0002)

#### T-RECON-001: Erkennung von Agentenendpunkten

| Attribut                    | Wert                                                                    |
| --------------------------- | ----------------------------------------------------------------------- |
| **ATLAS-ID**                | AML.T0006 – Aktives Scannen                                             |
| **Beschreibung**            | Angreifer suchen nach erreichbaren OpenClaw-Gateway-Endpunkten          |
| **Angriffsvektor**          | Netzwerkscans, Shodan-Abfragen, DNS-Aufzählung                           |
| **Betroffene Komponenten**  | Gateway, erreichbare API-Endpunkte                                      |
| **Aktuelle Gegenmaßnahmen** | Tailscale-Authentifizierungsoption, standardmäßige Bindung an Loopback   |
| **Restrisiko**              | Mittel – öffentliche Gateways können entdeckt werden                    |
| **Empfehlungen**            | Sichere Bereitstellung dokumentieren, Ratenbegrenzung für Endpunkte zur Erkennung hinzufügen |

#### T-RECON-002: Sondierung von Kanalintegrationen

| Attribut                    | Wert                                                                   |
| --------------------------- | ---------------------------------------------------------------------- |
| **ATLAS-ID**                | AML.T0006 – Aktives Scannen                                            |
| **Beschreibung**            | Angreifer sondieren Nachrichtenkanäle, um KI-verwaltete Konten zu identifizieren |
| **Angriffsvektor**          | Senden von Testnachrichten, Beobachten von Antwortmustern               |
| **Betroffene Komponenten**  | Alle Kanalintegrationen                                                 |
| **Aktuelle Gegenmaßnahmen** | Keine spezifischen                                                     |
| **Restrisiko**              | Niedrig – begrenzter Nutzen allein durch die Erkennung                  |
| **Empfehlungen**            | Randomisierung der Antwortzeiten erwägen                               |

---

### 3.2 Erstzugriff (AML.TA0004)

#### T-ACCESS-001: Abfangen des Kopplungscodes

| Attribut                | Wert                                                                                                                    |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0040 - Zugriff auf die Inferenz-API eines KI-Modells                                                               |
| **Beschreibung**        | Ein Angreifer fängt während des Kopplungszeitfensters einen Kopplungscode ab (1 Std. bei DM/allgemeiner Kopplung, 5 Min. bei Node-Kopplung) |
| **Angriffsvektor**      | Ausspähen über die Schulter, Netzwerk-Sniffing, Social Engineering                                                      |
| **Betroffene Komponenten** | Gerätekopplungssystem                                                                                                |
| **Aktuelle Schutzmaßnahmen** | 1 Std. TTL (DM/allgemeine Kopplung), 5 Min. TTL (Node-Kopplung); Codes werden über den bestehenden Kanal gesendet   |
| **Restrisiko**          | Mittel – Kopplungszeitfenster ausnutzbar                                                                                 |
| **Empfehlungen**        | Kopplungszeitfenster verkürzen, Bestätigungsschritt hinzufügen                                                          |

#### T-ACCESS-002: AllowFrom-Spoofing

| Attribut                | Wert                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0040 - Zugriff auf die Inferenz-API eines KI-Modells                            |
| **Beschreibung**        | Ein Angreifer täuscht in einem Kanal die Identität eines zulässigen Absenders vor    |
| **Angriffsvektor**      | Kanalabhängig – Vortäuschen von Telefonnummern, Identitätsvortäuschung per Benutzername |
| **Betroffene Komponenten** | Kanalbezogene AllowFrom-Validierung                                                |
| **Aktuelle Schutzmaßnahmen** | Kanalspezifische Identitätsprüfung                                               |
| **Restrisiko**          | Mittel – einige Kanäle bleiben anfällig für Spoofing                                  |
| **Empfehlungen**        | Kanalspezifische Risiken dokumentieren und, wo möglich, kryptografische Verifizierung hinzufügen |

#### T-ACCESS-003: Token-Diebstahl

| Attribut                | Wert                                                                    |
| ----------------------- | ----------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0040 - Zugriff auf die Inferenz-API eines KI-Modells               |
| **Beschreibung**        | Ein Angreifer stiehlt Authentifizierungstoken aus Konfigurations-/Anmeldedatendateien |
| **Angriffsvektor**      | Schadsoftware, unbefugter Gerätezugriff, Offenlegung von Konfigurationssicherungen |
| **Betroffene Komponenten** | Speicherung von Kanal-/Provider-Anmeldedaten, Konfigurationsspeicherung |
| **Aktuelle Schutzmaßnahmen** | Dateiberechtigungen                                                  |
| **Restrisiko**          | Hoch – Token werden im Klartext auf dem Datenträger gespeichert         |
| **Empfehlungen**        | Verschlüsselung ruhender Token implementieren, Token-Rotation hinzufügen |

---

### 3.3 Ausführung (AML.TA0005)

#### T-EXEC-001: Direkte Prompt-Injection

| Attribut                | Wert                                                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0051.000 - LLM-Prompt-Injection: Direkt                                                                                                     |
| **Beschreibung**        | Ein Angreifer sendet präparierte Prompts, um das Verhalten des Agenten zu manipulieren                                                           |
| **Angriffsvektor**      | Kanalnachrichten mit gegnerischen Anweisungen                                                                                                    |
| **Betroffene Komponenten** | Agenten-LLM, alle Eingabeschnittstellen                                                                                                       |
| **Aktuelle Schutzmaßnahmen** | Mustererkennung, Einbettung externer Inhalte; ohne Umgehung einer Schutzgrenze bei Schwachstellenmeldungen als außerhalb des Geltungsbereichs behandelt (siehe `SECURITY.md`) |
| **Restrisiko**          | Kritisch – nur Erkennung, keine Blockierung; ausgefeilte Angriffe umgehen sie                                                                    |
| **Empfehlungen**        | Ausgabevalidierung und Benutzerbestätigung für sensible Aktionen zusätzlich zur bestehenden Erkennung                                            |

#### T-EXEC-002: Indirekte Prompt-Injection

| Attribut                | Wert                                                                                                                        |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0051.001 - LLM-Prompt-Injection: Indirekt                                                                               |
| **Beschreibung**        | Ein Angreifer bettet schädliche Anweisungen in abgerufene Inhalte ein                                                        |
| **Angriffsvektor**      | Schädliche URLs, manipulierte E-Mails, kompromittierte Webhooks                                                              |
| **Betroffene Komponenten** | `web_fetch`, E-Mail-Erfassung, externe Datenquellen                                                                |
| **Aktuelle Schutzmaßnahmen** | Einbettung von Inhalten mit zufälligen Begrenzungsmarkierungen im XML-Stil, Normalisierung von Homoglyphen/Sondertoken und ein Sicherheitshinweis |
| **Restrisiko**          | Hoch – das LLM kann die Anweisungen der Einbettung dennoch ignorieren                                                        |
| **Empfehlungen**        | Getrennte Ausführungskontexte für eingebettete Inhalte                                                                       |

#### T-EXEC-003: Einschleusung von Werkzeugargumenten

| Attribut                | Wert                                                                  |
| ----------------------- | --------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0051.000 - LLM-Prompt-Injection: Direkt                          |
| **Beschreibung**        | Ein Angreifer manipuliert durch Prompt-Injection Werkzeugargumente    |
| **Angriffsvektor**      | Präparierte Prompts, die Werte von Werkzeugparametern beeinflussen    |
| **Betroffene Komponenten** | Alle Werkzeugaufrufe                                               |
| **Aktuelle Schutzmaßnahmen** | Ausführungsgenehmigungen für gefährliche Befehle                  |
| **Restrisiko**          | Hoch – beruht auf dem Urteilsvermögen des Benutzers                   |
| **Empfehlungen**        | Argumentvalidierung, parametrisierte Werkzeugaufrufe                   |

#### T-EXEC-004: Umgehung der Ausführungsgenehmigung

| Attribut                | Wert                                                                                                                                                                                   |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0043 - Präparieren gegnerischer Daten                                                                                                                                             |
| **Beschreibung**        | Ein Angreifer erstellt Befehle, die die Zulassungsliste für Genehmigungen umgehen                                                                                                      |
| **Angriffsvektor**      | Befehlsverschleierung, Ausnutzung von Aliasen, Pfadmanipulation                                                                                                                         |
| **Betroffene Komponenten** | `src/infra/exec-approvals*.ts`, Befehlszulassungsliste                                                                                                                                          |
| **Aktuelle Schutzmaßnahmen** | Zulassungsliste + Nachfragemodus sowie Befehlsnormalisierung (Entpacken von Dispatch-Wrappern, Erkennung eingebetteter Auswertung, Analyse von Shell-Befehlsketten)                  |
| **Restrisiko**          | Hoch – die Normalisierung erschwert die Umgehung durch Verschleierung, verhindert sie jedoch nicht; Feststellungen, die ausschließlich die Parität zwischen Ausführungspfaden betreffen, gelten als Härtung und nicht als Schwachstellen (siehe `SECURITY.md`) |
| **Empfehlungen**        | Abdeckung der Befehlsnormalisierung gegen neue Verschleierungstechniken kontinuierlich erweitern                                                                                        |

---

### 3.4 Persistenz (AML.TA0006)

#### T-PERSIST-001: Installation eines schädlichen Skills

| Attribut                | Wert                                                                                                                          |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0010.001 - Kompromittierung der Lieferkette: KI-Software                                                                  |
| **Beschreibung**        | Ein Angreifer veröffentlicht einen schädlichen Skill auf ClawHub                                                               |
| **Angriffsvektor**      | Konto erstellen, Skill mit verstecktem schädlichem Code veröffentlichen                                                        |
| **Betroffene Komponenten** | ClawHub, Laden von Skills, Agentenausführung                                                                                |
| **Aktuelle Schutzmaßnahmen** | Prüfung des Alters des GitHub-Kontos, statische musterbasierte/AST-nahe Scans, LLM-basierte agentische Risikoprüfung, VirusTotal-Scans |
| **Restrisiko**          | Hoch – Erkennungsebenen sind vorhanden, Skills werden jedoch weiterhin mit Agentenberechtigungen und ohne Ausführungs-Sandboxing ausgeführt |
| **Empfehlungen**        | Ausführungs-Sandboxing für Skills, erweiterte Community-Prüfung                                                                 |

#### T-PERSIST-002: Manipulation eines Skill-Updates

| Attribut                | Wert                                                                          |
| ----------------------- | ----------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0010.001 - Kompromittierung der Lieferkette: KI-Software                 |
| **Beschreibung**        | Ein Angreifer kompromittiert einen beliebten Skill und verteilt ein schädliches Update |
| **Angriffsvektor**      | Kontokompromittierung, Social Engineering des Skill-Eigentümers               |
| **Betroffene Komponenten** | ClawHub-Versionierung, automatische Aktualisierungsabläufe                 |
| **Aktuelle Schutzmaßnahmen** | Versions-Fingerprinting, erneute Moderation/Prüfung neuer Versionen       |
| **Restrisiko**          | Hoch – automatische Updates können schädliche Versionen abrufen, bevor die Prüfung abgeschlossen ist |
| **Empfehlungen**        | Signierung von Updates, Rollback-Funktion, Versionsfixierung                  |

#### T-PERSIST-003: Manipulation der Agentenkonfiguration

| Attribut                | Wert                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0010.002 - Kompromittierung der Lieferkette: Daten                              |
| **Beschreibung**        | Angreifer verändert die Agentenkonfiguration, um den Zugriff dauerhaft aufrechtzuerhalten |
| **Angriffsvektor**      | Änderung der Konfigurationsdatei, Einschleusen von Einstellungen                     |
| **Betroffene Komponenten** | Agentenkonfiguration, Tool-Richtlinien                                             |
| **Aktuelle Schutzmaßnahmen** | Dateiberechtigungen                                                             |
| **Restrisiko**          | Mittel – erfordert lokalen Zugriff                                                   |
| **Empfehlungen**        | Überprüfung der Konfigurationsintegrität, Audit-Protokollierung von Konfigurationsänderungen |

---

### 3.5 Umgehung von Schutzmaßnahmen (AML.TA0007)

#### T-EVADE-001: Umgehung von Moderationsmustern

| Attribut                | Wert                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0043 - Erstellen adversarieller Daten                                                 |
| **Beschreibung**        | Angreifer erstellt Skill-Inhalte, um die Moderationsprüfungen von ClawHub zu umgehen       |
| **Angriffsvektor**      | Unicode-Homoglyphen, Kodierungstricks, dynamisches Laden                                   |
| **Betroffene Komponenten** | Moderations-/Scan-Pipeline von ClawHub                                                  |
| **Aktuelle Schutzmaßnahmen** | Statische Musterregeln, AST-nahe Codeprüfung, LLM-basierte Prüfung agentischer Risiken, VirusTotal |
| **Restrisiko**          | Mittel – neuartige Verschleierungen können mehrschichtige Heuristiken weiterhin umgehen    |
| **Empfehlungen**        | Den Korpus aus Mustern und Verhaltensmerkmalen bei neu entdeckten Umgehungen kontinuierlich erweitern |

#### T-EVADE-002: Ausbruch aus der Inhaltskapselung

| Attribut                | Wert                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0043 - Erstellen adversarieller Daten                                                                   |
| **Beschreibung**        | Angreifer erstellt Inhalte, die aus dem Kontext der Kapselung externer Inhalte ausbrechen                    |
| **Angriffsvektor**      | Tag-Manipulation, Kontextverwirrung, Überschreiben von Anweisungen                                           |
| **Betroffene Komponenten** | Kapselung externer Inhalte                                                                                |
| **Aktuelle Schutzmaßnahmen** | XML-artige Markierungen mit zufälligen Begrenzungen und Sicherheitshinweis sowie Erkennung vorgetäuschter Markierungen durch Homoglyphen und Leerraumvarianten |
| **Restrisiko**          | Mittel – neuartige Ausbruchsmethoden werden regelmäßig entdeckt                                              |
| **Empfehlungen**        | Ausgabeseitige Validierung zusätzlich zur eingabeseitigen Kapselung                                          |

---

### 3.6 Erkundung (AML.TA0008)

#### T-DISC-001: Auflistung von Tools

| Attribut                | Wert                                                         |
| ----------------------- | ------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0040 - Zugriff auf die Inferenz-API des KI-Modells       |
| **Beschreibung**        | Angreifer ermittelt durch Prompts die verfügbaren Tools       |
| **Angriffsvektor**      | Anfragen nach dem Muster „Welche Tools haben Sie?“            |
| **Betroffene Komponenten** | Tool-Registry des Agenten                                  |
| **Aktuelle Schutzmaßnahmen** | Keine spezifischen                                        |
| **Restrisiko**          | Niedrig – Tools sind im Allgemeinen dokumentiert              |
| **Empfehlungen**        | Kontrollen für die Sichtbarkeit von Tools erwägen             |

#### T-DISC-002: Extraktion von Sitzungsdaten

| Attribut                | Wert                                                          |
| ----------------------- | ------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0040 - Zugriff auf die Inferenz-API des KI-Modells        |
| **Beschreibung**        | Angreifer extrahiert sensible Daten aus dem Sitzungskontext    |
| **Angriffsvektor**      | Anfragen wie „Was haben wir besprochen?“, Untersuchung des Kontexts |
| **Betroffene Komponenten** | Sitzungsprotokolle, Kontextfenster                          |
| **Aktuelle Schutzmaßnahmen** | Sitzungsisolierung pro Absender (Schlüssel `agent:channel:peer`) |
| **Restrisiko**          | Mittel – sitzungsinterne Daten sind absichtlich zugänglich     |
| **Empfehlungen**        | Schwärzung sensibler Daten im Kontext                          |

---

### 3.7 Erfassung und Exfiltration (AML.TA0009, AML.TA0010)

#### T-EXFIL-001: Datendiebstahl über web_fetch

| Attribut                | Wert                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0009 - Erfassung                                                                      |
| **Beschreibung**        | Angreifer exfiltriert Daten, indem er den Agenten anweist, sie an eine externe URL zu senden |
| **Angriffsvektor**      | Prompt-Injection, die den Agenten veranlasst, Daten per POST an einen Angreiferserver zu senden |
| **Betroffene Komponenten** | Tool `web_fetch`                                                                |
| **Aktuelle Schutzmaßnahmen** | SSRF-Blockierung für interne/private Netzwerke (DNS-Pinning und IP-Blockierung)        |
| **Restrisiko**          | Hoch – beliebige externe URLs bleiben zulässig                                              |
| **Empfehlungen**        | URL-Zulassungsliste, Berücksichtigung der Datenklassifizierung                              |

#### T-EXFIL-002: Unbefugtes Senden von Nachrichten

| Attribut                | Wert                                                                     |
| ----------------------- | ------------------------------------------------------------------------ |
| **ATLAS-ID**            | AML.T0009 - Erfassung                                                    |
| **Beschreibung**        | Angreifer veranlasst den Agenten, Nachrichten mit sensiblen Daten zu senden |
| **Angriffsvektor**      | Prompt-Injection, die den Agenten veranlasst, dem Angreifer eine Nachricht zu senden |
| **Betroffene Komponenten** | Nachrichten-Tool, Kanalintegrationen                                 |
| **Aktuelle Schutzmaßnahmen** | Kontrollmechanismus für ausgehende Nachrichten                     |
| **Restrisiko**          | Mittel – der Kontrollmechanismus kann möglicherweise umgangen werden     |
| **Empfehlungen**        | Explizite Bestätigung bei neuen Empfängern                               |

#### T-EXFIL-003: Abgreifen von Anmeldedaten

| Attribut                | Wert                                                                                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0009 - Erfassung                                                                                                                                   |
| **Beschreibung**        | Bösartiger Skill greift Anmeldedaten aus dem Agentenkontext ab                                                                                          |
| **Angriffsvektor**      | Skill-Code liest Umgebungsvariablen und Konfigurationsdateien                                                                                           |
| **Betroffene Komponenten** | Skill-Ausführungsumgebung                                                                                                                            |
| **Aktuelle Schutzmaßnahmen** | ClawHub-Prüfung auf Anmeldedatenmuster (hartcodierte Geheimnisse, Zugriff auf Anmeldedaten-Umgebungsvariablen in Verbindung mit Netzwerkübertragungen); keine Ausführungs-Sandbox für Skills zur Laufzeit |
| **Restrisiko**          | Kritisch – Skills werden mit den Berechtigungen des Agenten ausgeführt                                                                                  |
| **Empfehlungen**        | Sandbox-Ausführung für Skills, Isolierung von Anmeldedaten                                                                                              |

---

### 3.8 Auswirkungen (AML.TA0011)

#### T-IMPACT-001: Unbefugte Befehlsausführung

| Attribut                | Wert                                                                                                      |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0031 - Beeinträchtigung der Integrität des KI-Modells                                                |
| **Beschreibung**        | Angreifer führt beliebige Befehle auf dem System des Benutzers aus                                       |
| **Angriffsvektor**      | Prompt-Injection in Kombination mit der Umgehung der Ausführungsgenehmigung                               |
| **Betroffene Komponenten** | Bash-Tool, Befehlsausführung                                                                          |
| **Aktuelle Schutzmaßnahmen** | Ausführungsgenehmigungen, Docker-Sandbox-Option (standardmäßiges Laufzeit-Backend)                   |
| **Restrisiko**          | Kritisch – Ausführung auf dem Host ist möglich, wenn die Sandbox deaktiviert ist                          |
| **Empfehlungen**        | Benutzerführung für Genehmigungen verbessern; Bereitstellungen ohne Sandbox bleiben eine bewusste Betreiberentscheidung und werden entsprechend dokumentiert |

#### T-IMPACT-002: Ressourcenerschöpfung (DoS)

| Attribut                | Wert                                                  |
| ----------------------- | ----------------------------------------------------- |
| **ATLAS-ID**            | AML.T0031 - Beeinträchtigung der Integrität des KI-Modells |
| **Beschreibung**        | Angreifer erschöpft API-Guthaben oder Rechenressourcen |
| **Angriffsvektor**      | Automatisierte Nachrichtenflut, kostspielige Tool-Aufrufe |
| **Betroffene Komponenten** | Gateway, Agentensitzungen, API-Provider             |
| **Aktuelle Schutzmaßnahmen** | Keine                                              |
| **Restrisiko**          | Hoch – keine Ratenbegrenzung pro Absender              |
| **Empfehlungen**        | Ratenbegrenzungen pro Absender, Kostenbudgets          |

#### T-IMPACT-003: Rufschädigung

| Attribut                | Wert                                                              |
| ----------------------- | ----------------------------------------------------------------- |
| **ATLAS-ID**            | AML.T0031 - Beeinträchtigung der Integrität des KI-Modells         |
| **Beschreibung**        | Angreifer veranlasst den Agenten, schädliche/anstößige Inhalte zu senden |
| **Angriffsvektor**      | Prompt-Injection, die unangemessene Antworten verursacht           |
| **Betroffene Komponenten** | Ausgabegenerierung, Kanalnachrichten                            |
| **Aktuelle Schutzmaßnahmen** | Inhaltsrichtlinien des LLM-Providers                          |
| **Restrisiko**          | Mittel – Provider-Filter sind nicht vollkommen zuverlässig         |
| **Empfehlungen**        | Ausgabefilterschicht, Benutzerkontrollen                            |

---

## 4. ClawHub-Lieferkettenanalyse

### 4.1 Aktuelle Sicherheitskontrollen

| Kontrolle                      | Implementierung                                                                       | Wirksamkeit                                                                    |
| ------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Alter des GitHub-Kontos        | `requireGitHubAccountAge()` (mindestens 14 Tage)                                               | Mittel – erhöht die Hürde für neue Angreifer                                   |
| Pfadbereinigung                | `sanitizePath()`                                                                    | Hoch – verhindert Path Traversal                                               |
| Dateitypvalidierung            | `isTextFile()`                                                                    | Mittel – nur Textdateien werden gescannt, dennoch ausnutzbar                   |
| Größenbeschränkungen           | insgesamt 50MB pro Bundle (`MAX_PUBLISH_TOTAL_BYTES`)                                        | Hoch – verhindert Ressourcenerschöpfung                                        |
| Erforderliche SKILL.md         | Obligatorische Readme-Datei bei der Veröffentlichung                                  | Geringer Sicherheitswert – dient nur zur Information                           |
| Statisches + AST-nahes Scannen | Muster-Engine für exec, Exfiltration, Abgriff von Anmeldedaten, Verschleierung und mehr | Mittel bis hoch – deckt viele bekannte Missbrauchsmuster ab, bleibt musterbasiert |
| LLM-basierte agentische Risikoprüfung | Durch einen Sicherheitsprompt gesteuertes Urteil bei der Veröffentlichung       | Mittel bis hoch – erkennt Verhalten, das statische Muster nicht erfassen       |
| VirusTotal-Scanning            | In Veröffentlichungs-/Neuscan-Abläufe für Skills und Pakete eingebunden, durch API-Schlüssel des Betreibers geschützt | Hoch, wenn aktiviert – Erkennung durch statische Engines |
| Moderationsstatus              | Feld `moderationStatus`                                                               | Mittel – manuelle Prüfung möglich                                              |

### 4.2 Einschränkungen der Moderation

Das statische Scanning von ClawHub prüft den Codeinhalt von Skills direkt (nicht nur Slug/Metadaten/Frontmatter) und deckt gefährliche exec-Aufrufe, dynamische Codeausführung, den Abgriff von Anmeldedaten, Exfiltrationsmuster, verschleierte Payloads und mehr ab. Bekannte Lücken:

- Die musterbasierte Erkennung kann weiterhin durch hinreichend neuartige Verschleierung umgangen werden.
- LLM-basierte Prüfung und VirusTotal-Scanning hängen davon ab, dass betreiberseitige API-Schlüssel bzw. die entsprechende Konfiguration aktiviert sind.
- Keine Sandbox für die Laufzeitausführung isoliert einen Skill nach der Installation von den eigenen Berechtigungen des Agenten.

### 4.3 Abzeichen

Skills und Pakete tragen von Moderatoren zugewiesene Abzeichen: `highlighted`, `official`, `deprecated`, `redactionApproved` (nur Skills). Meldungen aus der Community (`skillReports`) und Audit-Protokollierung (`auditLogs`) unterstützen die Moderationsabläufe.

---

## 5. Risikomatrix

### 5.1 Wahrscheinlichkeit und Auswirkung

| Bedrohungs-ID | Wahrscheinlichkeit | Auswirkung | Risikostufe  | Priorität |
| ------------- | ------------------ | ---------- | ------------ | --------- |
| T-EXEC-001    | Hoch               | Kritisch   | **Kritisch** | P0        |
| T-PERSIST-001 | Hoch               | Kritisch   | **Kritisch** | P0        |
| T-EXFIL-003   | Mittel             | Kritisch   | **Kritisch** | P0        |
| T-IMPACT-001  | Mittel             | Kritisch   | **Hoch**     | P1        |
| T-EXEC-002    | Hoch               | Hoch       | **Hoch**     | P1        |
| T-EXEC-004    | Mittel             | Hoch       | **Hoch**     | P1        |
| T-ACCESS-003  | Mittel             | Hoch       | **Hoch**     | P1        |
| T-EXFIL-001   | Mittel             | Hoch       | **Hoch**     | P1        |
| T-IMPACT-002  | Hoch               | Mittel     | **Hoch**     | P1        |
| T-EVADE-001   | Hoch               | Mittel     | **Mittel**   | P2        |
| T-ACCESS-001  | Niedrig            | Hoch       | **Mittel**   | P2        |
| T-ACCESS-002  | Niedrig            | Hoch       | **Mittel**   | P2        |
| T-PERSIST-002 | Niedrig            | Hoch       | **Mittel**   | P2        |

### 5.2 Angriffsketten kritischer Pfade

**Kette 1: Skill-basierter Datendiebstahl**

```text
T-PERSIST-001 → T-EVADE-001 → T-EXFIL-003
(Bösartigen Skill veröffentlichen) → (Moderation umgehen) → (Anmeldedaten abgreifen)
```

**Kette 2: Prompt Injection bis RCE**

```text
T-EXEC-001 → T-EXEC-004 → T-IMPACT-001
(Prompt einschleusen) → (exec-Genehmigung umgehen) → (Befehle ausführen)
```

**Kette 3: Indirekte Injection über abgerufene Inhalte**

```text
T-EXEC-002 → T-EXFIL-001 → Externe Exfiltration
(URL-Inhalt manipulieren) → (Agent ruft Inhalt ab und folgt den Anweisungen) → (Daten werden an den Angreifer gesendet)
```

---

## 6. Zusammenfassung der Empfehlungen

### 6.1 Sofort (P0)

| ID    | Empfehlung                                                   | Behandelt                  |
| ----- | ------------------------------------------------------------ | -------------------------- |
| R-002 | Sandbox für die Skill-Ausführung implementieren              | T-PERSIST-001, T-EXFIL-003 |
| R-003 | Ausgabevalidierung für sensible Aktionen hinzufügen          | T-EXEC-001, T-EXEC-002     |

### 6.2 Kurzfristig (P1)

| ID    | Empfehlung                                                                  | Behandelt    |
| ----- | --------------------------------------------------------------------------- | ------------ |
| R-004 | Ratenbegrenzung pro Absender implementieren                                 | T-IMPACT-002 |
| R-005 | Verschlüsselung ruhender Tokens hinzufügen                                  | T-ACCESS-003 |
| R-006 | UX für exec-Genehmigungen verbessern und Befehlsnormalisierung weiter ausbauen | T-EXEC-004 |
| R-007 | URL-Zulassungsliste für `web_fetch` implementieren                   | T-EXFIL-001  |

### 6.3 Mittelfristig (P2)

| ID    | Empfehlung                                                          | Behandelt     |
| ----- | ------------------------------------------------------------------- | ------------- |
| R-008 | Wo möglich kryptografische Kanalverifizierung hinzufügen           | T-ACCESS-002  |
| R-009 | Integritätsprüfung der Konfiguration implementieren                 | T-PERSIST-003 |
| R-010 | Signierung von Updates und Versionsfixierung hinzufügen             | T-PERSIST-002 |

---

## 7. Anhänge

### 7.1 Zuordnung der ATLAS-Techniken

| ATLAS-ID      | Technikname                     | OpenClaw-Bedrohungen                                              |
| ------------- | ------------------------------- | ----------------------------------------------------------------- |
| AML.T0006     | Aktives Scanning                | T-RECON-001, T-RECON-002                                         |
| AML.T0009     | Sammlung                        | T-EXFIL-001, T-EXFIL-002, T-EXFIL-003                            |
| AML.T0010.001 | Lieferkette: KI-Software        | T-PERSIST-001, T-PERSIST-002                                     |
| AML.T0010.002 | Lieferkette: Daten              | T-PERSIST-003                                                    |
| AML.T0031     | Integrität des KI-Modells beeinträchtigen | T-IMPACT-001, T-IMPACT-002, T-IMPACT-003                  |
| AML.T0040     | Zugriff auf die Inferenz-API des KI-Modells | T-ACCESS-001, T-ACCESS-002, T-ACCESS-003, T-DISC-001, T-DISC-002 |
| AML.T0043     | Adversariale Daten erstellen    | T-EXEC-004, T-EVADE-001, T-EVADE-002                             |
| AML.T0051.000 | LLM Prompt Injection: direkt    | T-EXEC-001, T-EXEC-003                                           |
| AML.T0051.001 | LLM Prompt Injection: indirekt  | T-EXEC-002                                                       |

### 7.2 Wichtige Sicherheitsdateien

| Pfad                                | Zweck                                | Risikostufe  |
| ----------------------------------- | ------------------------------------ | ------------ |
| `src/infra/exec-approvals.ts`                  | Logik für Befehlsfreigaben           | **Kritisch** |
| `src/gateway/auth.ts`                  | Gateway-Authentifizierung            | **Kritisch** |
| `src/infra/net/ssrf.ts`                  | SSRF-Schutz                          | **Kritisch** |
| `src/security/external-content.ts`                  | Eindämmung von Prompt Injection      | **Kritisch** |
| `src/agents/sandbox/tool-policy.ts`                  | Zulassungs-/Sperrrichtlinie für Sandbox-Tools | **Kritisch** |
| `src/routing/resolve-route.ts`                  | Sitzungsisolierung/-Routing          | **Mittel**   |

### 7.3 Glossar

| Begriff              | Definition                                                        |
| -------------------- | ----------------------------------------------------------------- |
| **ATLAS**            | MITREs Adversarial Threat Landscape for AI Systems                |
| **ClawHub**          | Skill-Marktplatz von OpenClaw                                     |
| **Gateway**          | Nachrichtenrouting- und Authentifizierungsschicht von OpenClaw    |
| **MCP**              | Model Context Protocol – Schnittstelle für Tool-Provider          |
| **Prompt Injection** | Angriff, bei dem schädliche Anweisungen in Eingaben eingebettet werden |
| **Skill**            | Herunterladbare Erweiterung für OpenClaw-Agenten                   |
| **SSRF**             | Server-Side Request Forgery                                       |

---

_Dieses Bedrohungsmodell ist ein fortlaufend aktualisiertes Dokument. Melden Sie Sicherheitsprobleme an `security@openclaw.ai` oder besuchen Sie die [Vertrauensseite](https://trust.openclaw.ai)._

## Verwandte Themen

- [Zum Bedrohungsmodell beitragen](/de/security/CONTRIBUTING-THREAT-MODEL)
- [Reaktion auf Sicherheitsvorfälle](/de/security/incident-response)
- [Netzwerkproxy](/de/security/network-proxy)
- [Formale Verifikation](/de/security/formal-verification)
