---
read_when:
    - Reaktion auf eine Sicherheitsmeldung oder einen vermuteten Sicherheitsvorfall
    - Vorbereitung einer koordinierten Offenlegung oder einer Sicherheitsversion mit Fehlerbehebung
    - Überprüfung der Erwartungen an die Nachbereitung nach einem Vorfall
summary: Wie OpenClaw Sicherheitsvorfälle bewertet, darauf reagiert und nachverfolgt
title: Reaktion auf Sicherheitsvorfälle
x-i18n:
    generated_at: "2026-07-26T18:11:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30f2d754408e95133ee86254ce193c0d8aab293040df55e0c1cec0c4d7644c56
    source_path: security/incident-response.md
    workflow: 16
---

## 1. Erkennung und Triage

Sicherheitssignale stammen aus:

- GitHub Security Advisories (GHSA) und privaten Schwachstellenmeldungen.
- Öffentlichen GitHub-Issues/-Diskussionen, wenn Meldungen nicht vertraulich sind.
- Automatisierten Signalen: Dependabot, CodeQL, npm-Advisories und Secret Scanning.

Ersttriage:

1. Betroffene Komponente, Version und Auswirkungen auf die Vertrauensgrenze bestätigen.
2. Anhand der Regeln von `SECURITY.md` zum Geltungsbereich und zu ausgeschlossenen Bereichen als Sicherheitsproblem oder als Härtungsmaßnahme/kein Handlungsbedarf klassifizieren.
3. Eine für den Vorfall verantwortliche Person reagiert entsprechend.

## 2. Schweregrad

| Schweregrad | Definition                                                                                                                                                                                                 |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kritisch     | Kompromittierung eines Pakets, Releases oder Repositorys, aktive Ausnutzung oder nicht authentifizierte Umgehung einer Vertrauensgrenze mit weitreichender Kontrolle oder Offenlegung von Daten.             |
| Hoch         | Bestätigte Umgehung einer Vertrauensgrenze mit begrenzten Voraussetzungen (beispielsweise eine authentifizierte, aber nicht autorisierte Aktion mit weitreichenden Auswirkungen) oder Offenlegung vertraulicher, von OpenClaw verwalteter Anmeldedaten. |
| Mittel       | Erhebliche Sicherheitsschwäche mit praktischen Auswirkungen, aber eingeschränkter Ausnutzbarkeit oder beträchtlichen Voraussetzungen.                                                                        |
| Niedrig      | Maßnahmen zur mehrschichtigen Absicherung, eng begrenzte Denial-of-Service-Schwachstellen oder Lücken bei Härtung bzw. Funktionsgleichheit ohne nachgewiesene Umgehung einer Vertrauensgrenze.                 |

## 3. Reaktion

1. Eingang gegenüber der meldenden Person bestätigen (bei vertraulichen Inhalten auf privatem Weg).
2. Auf unterstützten Releases und dem neuesten `main` reproduzieren, anschließend einen Patch mit Abdeckung durch Regressionstests implementieren und validieren.
3. Kritisch/hoch: korrigierte Releases so schnell wie praktisch möglich vorbereiten.
4. Mittel/niedrig: Patch im regulären Release-Ablauf veröffentlichen und Hinweise zu Gegenmaßnahmen dokumentieren.

## 4. Kommunikation und Offenlegung

Die Kommunikation erfolgt über GitHub Security Advisories im betroffenen Repository, Release Notes/Changelog-Einträge für korrigierte Versionen sowie direkte Status- und Abschlussmeldungen an die meldende Person.

Bei kritischen und hoch eingestuften Vorfällen erfolgt eine koordinierte Offenlegung und gegebenenfalls die Vergabe einer CVE. Härtungsbefunde mit geringem Risiko können abhängig von den Auswirkungen und der Gefährdung der Benutzer ohne CVE in Release Notes oder Advisories dokumentiert werden.

## 5. Wiederherstellung und Nachbereitung

Nach der Veröffentlichung der Korrektur:

1. Behebungsmaßnahmen in CI und Release-Artefakten verifizieren.
2. Eine kurze Nachbesprechung des Vorfalls durchführen: zeitlicher Ablauf, Grundursache, Erkennungslücke, Präventionsplan.
3. Folgeaufgaben für Härtung, Tests und Dokumentation hinzufügen und bis zum Abschluss nachverfolgen.

## Verwandte Themen

- [Sicherheitsrichtlinie](https://github.com/openclaw/openclaw/blob/main/SECURITY.md) — Meldeumfang und Vertrauensmodell.
- [Bedrohungsmodell](/de/security/THREAT-MODEL-ATLAS)
