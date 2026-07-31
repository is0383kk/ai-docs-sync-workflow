---
read_when:
    - Listings, Versionen, Installationen, Veröffentlichung und Moderation verstehen
summary: So funktionieren ClawHub-Einträge, Versionen, Installationen, Veröffentlichungen, Scans und Aktualisierungen.
x-i18n:
    generated_at: "2026-07-26T18:50:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 747079343899e42d00f84b00c553447abe0b83f2c4f1c9cdbf54725e34779eaf
    source_path: clawhub/how-it-works.md
    workflow: 16
---

# Funktionsweise von ClawHub

ClawHub ist die Registry-Ebene für OpenClaw-Skills und -Plugins. Sie bietet Benutzern einen
Ort, um Pakete zu entdecken, Herausgebern einen Ort, um Versionen zu veröffentlichen, und
OpenClaw genügend Metadaten, um diese Pakete sicher zu installieren und zu aktualisieren.

## Registry-Einträge

Jeder öffentliche Eintrag ist ein Registry-Eintrag mit:

- einem Eigentümer und Slug oder Paketnamen
- einer oder mehreren veröffentlichten Versionen
- Metadaten, Zusammenfassung, Dateien und Quellenangabe
- Änderungsprotokoll und Tag-Informationen wie `latest`
- Download-, Installations- und Sternsignalen
- Status von Sicherheitsscan und Moderation

Die Eintragsseite ist die maßgebliche Stelle, an der Benutzer vor der Installation prüfen
können, welche Funktionalität ein Skill oder Plugin laut eigener Beschreibung bietet.

## Skills

Ein Skill ist ein versioniertes Textpaket, dessen Kern `SKILL.md` bildet. Es kann
unterstützende Dateien, Beispiele, Vorlagen und Skripte enthalten.

ClawHub liest das Frontmatter von `SKILL.md`, um den Namen, die
Beschreibung, die Anforderungen, die Umgebungsvariablen und die Metadaten des Skills zu
ermitteln. Präzise Metadaten sind wichtig, da sie Benutzern bei der Entscheidung helfen, ob
sie den Skill installieren sollen, und automatisierten Scans ermöglichen, Abweichungen
zwischen deklariertem und beobachtetem Verhalten zu erkennen.

Siehe [Skill-Format](/de/clawhub/skill-format).

## Plugins

Plugins sind paketierte OpenClaw-Erweiterungen. ClawHub speichert Paketmetadaten,
Kompatibilitätsinformationen, Quelllinks, Artefakte und Versionseinträge.

Wenn OpenClaw ein Plugin von ClawHub installiert, prüft es vor der Installation die
angegebenen Kompatibilitätsmetadaten. Paketeinträge können API-Kompatibilität,
Gateway-Mindestversion, Zielhosts, Umgebungsanforderungen und Artefakt-Digests
enthalten.

Verwenden Sie eine explizite ClawHub-Installationsquelle, wenn die Registry als maßgebliche
Quelle dienen soll:

```bash
openclaw plugins install clawhub:<package>
```

## Veröffentlichung

Durch die Veröffentlichung wird ein neuer unveränderlicher Versionseintrag erstellt. Herausgeber
verwenden die CLI `clawhub` für authentifizierte Registry-Workflows:

```bash
clawhub skill publish ./my-skill
clawhub package publish <source> --family code-plugin --dry-run
clawhub package publish <source> --family code-plugin
```

Verwenden Sie Testläufe, um die aufgelöste Nutzlast vor dem Hochladen in der Vorschau
anzuzeigen. Die öffentlichen Seiten zeigen anschließend die veröffentlichten Metadaten,
Dateien, Quellenangaben und den Scanstatus an.

## Installationen und Aktualisierungen

OpenClaw-Installationsbefehle verwenden ClawHub als Paketquelle:

```bash
openclaw skills install @openclaw/demo
openclaw plugins install clawhub:<package>
```

OpenClaw zeichnet die Metadaten der Installationsquelle auf, damit bei späteren
Aktualisierungen dasselbe Registry-Paket aufgelöst werden kann. Die ClawHub-CLI unterstützt
außerdem direkte Workflows zur Installation und Aktualisierung von Skills für Benutzer, die
von der Registry verwaltete Skill-Ordner außerhalb eines vollständigen OpenClaw-Arbeitsbereichs
verwenden möchten.

## Sicherheitsstatus

ClawHub steht für Veröffentlichungen offen, doch Releases unterliegen weiterhin
Upload-Prüfungen, automatisierten Kontrollen, Benutzerberichten und Maßnahmen der
Moderatoren.

Öffentliche Seiten zeigen Scan-Zusammenfassungen an, sofern verfügbar. Zurückgehaltene,
ausgeblendete oder blockierte Inhalte können aus der öffentlichen Suche und den
Installationsabläufen verschwinden, während sie für den Eigentümer zu Diagnosezwecken
sichtbar bleiben.

Siehe [Sicherheit](/de/clawhub/security), [Sicherheitsaudits](/de/clawhub/security-audits),
[Moderation und Kontosicherheit](/de/clawhub/moderation) und
[Zulässige Nutzung](/de/clawhub/acceptable-usage).

## API-Zugriff

ClawHub stellt öffentliche Lese-APIs für Ermittlung, Suche, Paketdetails und
Downloads bereit. Drittanbieterkataloge dürfen diese APIs verwenden, wenn sie auf den
maßgeblichen ClawHub-Eintrag zurückverlinken, Ratenbegrenzungen einhalten und nicht den
Eindruck einer Befürwortung erwecken.

Siehe [Öffentliche API](/de/clawhub/api) und [HTTP-API](/de/clawhub/http-api).
