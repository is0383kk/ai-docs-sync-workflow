---
read_when:
    - ClawHub-Sicherheitsauditergebnisse verstehen
    - Entscheidung, ob ein Skill oder Plugin installiert werden soll
    - Erläuterung des ClawHub-Prüfstatus, der Risikostufe oder der Feststellungen
sidebarTitle: Security Audits
summary: So verstehen Sie die Ergebnisse der ClawHub-Sicherheitsprüfung, bevor Sie Skills oder Plugins installieren.
title: Sicherheitsaudits
x-i18n:
    generated_at: "2026-07-26T17:41:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4178a568c9b8e202da666ed95d2200ad73f931a22c7e473aeaba84545e8bb25
    source_path: clawhub/security-audits.md
    workflow: 16
---

# Sicherheitsprüfungen

Die Sicherheitsprüfungen von ClawHub helfen Ihnen bei der Entscheidung, ob ein Skill oder Plugin sicher genug für die Installation ist. Sie zeigen, was ein Release bewirkt, welche Berechtigungen es anfordert und ob etwas besondere Aufmerksamkeit erfordert, bevor es auf Dateien, Konten, Anmeldedaten, Code oder externe Dienste zugreifen kann.

Prüfungen sind aussagekräftige Sicherheitssignale, aber keine Garantie dafür, dass ein Release risikofrei ist. Wägen Sie stets sorgfältig ab, bevor Sie vertraulichen Zugriff gewähren.

Siehe auch [Sicherheit](/de/clawhub/security), [Zulässige Nutzung](/de/clawhub/acceptable-usage) und [Moderation und Kontosicherheit](/de/clawhub/moderation).

## Was vor der Installation zu prüfen ist

Prüfen Sie vor der Installation:

- den Gesamtstatus der Prüfung
- die Risikostufe
- alle aufgeführten Befunde
- erforderliche Anmeldedaten, Berechtigungen oder Umgebungsvariablen
- Eigentümer, Quelle, Version, Änderungsprotokoll, Downloads, Sterne und andere Vertrauenssignale

Installieren Sie nur Inhalte, die Sie verstehen und denen Sie vertrauen.

## Prüfstatus

Der Prüfstatus gibt an, wie Sie auf das Prüfergebnis reagieren sollten:

| Status      | Bedeutung                                                                   |
| ----------- | ------------------------------------------------------------------------- |
| `Pass`      | Es wurde kein sichtbares Problem oberhalb der niedrigen Risikostufe gefunden.                                |
| `Review`    | Lesen Sie vor der Installation die Befunde. Das Release kann dennoch legitim sein. |
| `Warn`      | Seien Sie besonders vorsichtig. ClawHub hat ein Problem mit weitreichenden Auswirkungen oder ein Warnsignal gefunden. |
| `Malicious` | Nicht installieren.                                                           |
| `Pending`   | Die Prüfungen sind noch nicht abgeschlossen.                                             |
| `Error`     | Die Prüfung konnte nicht abgeschlossen werden.                                         |

Ein `Pass` ist beruhigend, ersetzt aber nicht Ihre eigene Einschätzung. Dies ist besonders wichtig bei Tools, die Inhalte veröffentlichen, Daten bearbeiten, Befehle ausführen, Dateien lesen oder auf Produktionssysteme zugreifen können.

## Risikostufe

Die Risikostufe beschreibt den potenziellen Wirkungsbereich: wie viel Macht das Release offenbar hat, wenn Sie es wie vorgesehen verwenden.

| Risikostufe | Bedeutung                                                                       |
| ---------- | ----------------------------------------------------------------------------- |
| `Low`      | Es wurden nur geringe vertrauliche Berechtigungen oder Auswirkungen auf Benutzer festgestellt.                          |
| `Medium`   | Das Release verfügt über bedeutende Berechtigungen, etwa Kontozugriff oder die Möglichkeit, Daten zu ändern. |
| `High`     | Das Release verfügt über Berechtigungen mit weitreichenden Auswirkungen, schwerwiegende Befunde oder Anzeichen für Schadsoftware. |

Risikostufe und Prüfstatus beantworten unterschiedliche Fragen:

- Die Risikostufe fragt: „Wie viel Macht steckt hierin?“
- Der Prüfstatus fragt: „Wie soll ich mit diesem Ergebnis umgehen?“

Beispielsweise kann ein Skill zum Veröffentlichen `Review` mit dem Risiko `Medium` anzeigen. Das bedeutet nicht, dass er bösartig ist. Es bedeutet, dass der Skill offenbar seinem Zweck entspricht, aber mit bedeutenden Kontoberechtigungen handeln kann.

## Befunde

Befunde erläutern, warum ein bestimmtes Prüfergebnis angezeigt wurde. Jeder Befund enthält in der Regel:

- was er bedeutet
- warum er markiert wurde
- die relevanten Inhalte des Skills oder Plugins
- eine Empfehlung

Befunde können als `Info`, `Low`, `Medium`, `High` oder `Critical` gekennzeichnet sein. Befunde mit höherem Schweregrad wirken sich stärker auf die Risikostufe und den Prüfstatus aus.

Befunde mit geringer Zuverlässigkeit werden in der öffentlichen Prüfungsübersicht ausgeblendet, damit die Seite auf aussagekräftige Nachweise ausgerichtet bleibt.

## Was ClawHub prüft

ClawHub prüft eingereichte Release-Artefakte, darunter:

- Skill-Anweisungen oder Plugin-Metadaten
- deklarierte Umgebungsvariablen und Berechtigungen
- Installationsanweisungen und Paketmetadaten
- enthaltene Dateien und Dateimanifeste
- Kompatibilitäts- und Funktionsmetadaten

Die zentrale Frage lautet, ob alles stimmig ist: Entsprechen Name, Zusammenfassung, Metadaten, angeforderte Berechtigungen und tatsächliche Inhalte dem, was Benutzer vernünftigerweise erwarten würden?

Leistungsfähiges Verhalten ist nicht automatisch schlecht. Viele nützliche Tools benötigen Anmeldedaten, lokale Befehle, Provider-APIs oder Paketinstallationen. Die Prüfung ermittelt, ob diese Befugnisse erwartet werden können, offengelegt sind und in einem angemessenen Verhältnis stehen.

Artefaktseiten verweisen unter folgender Adresse auf die vollständige Prüfung:

```text
/<owner>/skills/<slug>/security-audit
```

Die Prüfseite kombiniert:

1. SkillSpector
2. VirusTotal
3. Risikoanalyse

## VirusTotal

ClawHub verwendet VirusTotal als Malware-Telemetrie im Prüfungsverbund. VirusTotal ist ein vertrauenswürdiger Branchenstandard für die Bewertung des Rufs von Dateien und die Suche nach Malware. Durch unsere Partnerschaft kann ClawHub die Prüfung von Skills und Plugins um umfassendere Sicherheitsinformationen ergänzen.

VirusTotal ist besonders nützlich für bekannte schädliche Artefakte, Treffer von Scan-Engines und Reputationssignale, welche die agentenbezogene Prüfung von ClawHub ergänzen. Wenn die Anzahl der Bewertungen durch Anbieter-Engines verfügbar ist, fasst die Prüfung sie in allgemein verständlicher Form zusammen, zum Beispiel:

```text
62/62 Anbieter haben diesen Skill als unbedenklich eingestuft.
```

oder:

```text
2/64 Anbieter haben diesen Skill als schädlich eingestuft, 1/64 als verdächtig und 61/64 als unbedenklich.
```

Wenn ClawHub keine Telemetrie zur Anzahl der Anbieterbewertungen zusammenfassen kann, lautet die Meldung der Prüfung:

```text
Keine VirusTotal-Befunde
```

VirusTotal bleibt eine Telemetriequelle. Es ersetzt nicht die eigene artefaktbezogene Risikoanalyse von ClawHub.

## Risikoanalyse

Die Risikoanalyse wird intern von ClawScan unterstützt, dem eigenen Sicherheitsprüfungssystem von ClawHub. Es prüft jedes Release als Artefakt für Agenten: Anweisungen, Metadaten, deklarierte Berechtigungen, Dateien, Funktionssignale, Signale statischer Scans, SkillSpector-Befunde, VirusTotal-Telemetrie und vom Herausgeber bereitgestellten Kontext. Signale statischer Scans dienen als interner Kontext für diese Prüfung; sie sind weder ein eigenständiger öffentlicher Prüfungsabschnitt noch ein Urteil, das die Installation blockiert.

Die Risikoanalyse verwendet die
[OWASP Agentic Skills Top 10](https://owasp.org/www-project-agentic-skills-top-10/)
als Orientierungsrahmen für Risiken wie Prompt-Injection, Missbrauch von Tools, Offenlegung von Anmeldedaten, unsichere Ausführung, Vergiftung des Speichers oder Kontexts und übermäßige Handlungsautonomie.

ClawScan stuft eine bedrohlich wirkende Funktion nicht automatisch als bösartig ein. Es prüft, ob die Funktion offengelegt ist, dem Zweck entspricht und durch den angegebenen Anwendungsfall des Releases gerechtfertigt wird.
