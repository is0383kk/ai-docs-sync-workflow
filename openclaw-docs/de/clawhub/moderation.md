---
read_when:
    - Einen Skill, ein Plugin oder ein Paket melden
    - Wiederherstellung eines zurückgehaltenen, ausgeblendeten oder blockierten Eintrags
    - Informationen zu ClawHub-Moderation, Sperren oder Kontostatus
sidebarTitle: Moderation and Account Safety
summary: Funktionsweise von Meldungen, Moderationssperren, ausgeblendeten Einträgen, Ausschlüssen und dem Kontostatus in ClawHub.
title: Moderation und Kontosicherheit
x-i18n:
    generated_at: "2026-07-26T17:41:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54c1e0860411e6599923ef4d7db65d5cd5406ec63bf67c52968b4f99d893ffef
    source_path: clawhub/moderation.md
    workflow: 16
---

# Moderation und Kontosicherheit

ClawHub ermöglicht das freie Veröffentlichen, doch die öffentliche Auffindbarkeit und die Installationsoberflächen benötigen weiterhin Schutzmechanismen. Meldungen, Moderationssperren, ausgeblendete Einträge und Kontomaßnahmen tragen zum Schutz der Benutzer bei, wenn eine Veröffentlichung oder ein Konto unsicher, irreführend oder richtlinienwidrig erscheint.

Diese Seite behandelt Moderation und Kontostatus. Informationen zu Audit-Bezeichnungen wie `Pass`, `Review`, `Warn`, `Malicious` und zur Risikostufe finden Sie unter
[Sicherheitsaudits](/de/clawhub/security-audits).

Siehe auch [Sicherheit](/de/clawhub/security) und
[Zulässige Nutzung](/de/clawhub/acceptable-usage). Bei Bedenken hinsichtlich Urheberrechten oder anderen Inhaltsrechten verwenden Sie [Anfragen zu Inhaltsrechten](/de/clawhub/content-rights).

## Meldungen

Angemeldete Benutzer können Skills, Plugins und Pakete melden.

Verwenden Sie ClawHub-Meldungen ausschließlich für unsichere Marketplace-Inhalte, beispielsweise:

- schädliche Einträge
- irreführende Metadaten
- nicht offengelegte Anmeldedaten oder Berechtigungsanforderungen
- verdächtige Installationsanweisungen
- Identitätsvortäuschung
- böswillige Registrierungen oder Markenmissbrauch
- Inhalte, die gegen die [Zulässige Nutzung](/de/clawhub/acceptable-usage) verstoßen

Verwenden Sie die Schaltfläche **Report skill** auf einer Skill-Seite oder den Befehl beziehungsweise die API zum Melden von Paketen.

Verwenden Sie ClawHub-Meldungen nicht für Schwachstellen im eigenen Quellcode eines Drittanbieter-Skills oder -Plugins. Melden Sie diese direkt dem Herausgeber oder dem im Eintrag verlinkten Quell-Repository. ClawHub wartet oder korrigiert keinen Skill- oder Plugin-Code von Drittanbietern.

GitHub Security Advisories für `openclaw/clawhub` sind für Schwachstellen in ClawHub selbst vorgesehen. Beispiele sind Fehler in der Website, API, CLI, Registry, Authentifizierung, Überprüfung, Moderation oder in den Vertrauensgrenzen für Download und Installation. Verwenden Sie ClawHub-Advisories nicht für Schwachstellen in Skills oder Plugins von Drittanbietern.

Gute Meldungen sind konkret und umsetzbar. Der Missbrauch der Meldefunktion kann selbst zu Kontomaßnahmen führen.

## Ansprüche auf Organisationen und Namespaces

Streitigkeiten über die Inhaberschaft von Organisationen, Marken, Paketbereichen, Inhaber-Handles oder Namespaces sollten über das Verfahren für [Ansprüche auf Organisationen und Namespaces](/de/clawhub/namespace-claims) geklärt werden, nicht über den produktinternen Meldeablauf oder das Einspruchsformular für Konten.

Verwenden Sie dieses Verfahren, wenn ClawHub-Mitarbeiter nicht vertrauliche Nachweise prüfen sollen, dass ein Namespace reserviert, übertragen, umbenannt, ausgeblendet, unter Quarantäne gestellt, mit einem Alias versehen oder anderweitig überprüft werden sollte. Fügen Sie einem öffentlichen Issue keine Geheimnisse, privaten Dokumente, vertraulichen Rechtsunterlagen, persönlichen Identitätsdokumente, API-Tokens oder DNS-Challenge-Tokens bei.

## Moderationssperren

Einige schwerwiegende Feststellungen oder Richtlinienverstöße können dazu führen, dass ein Herausgeber oder Eintrag einer Moderationssperre unterliegt. In diesem Fall können betroffene Inhalte aus der öffentlichen Auffindbarkeit ausgeblendet werden oder zukünftige Veröffentlichungen zunächst ausgeblendet erscheinen, bis das Problem überprüft wurde.

Moderationssperren sollen Benutzer schützen, während ClawHub Fälle mit hohem Risiko klärt. Sie können auch aufgehoben werden, wenn ein falsch positives Ergebnis bestätigt wird.

## Ausgeblendete oder gesperrte Einträge

Ein Eintrag kann zurückgehalten, ausgeblendet, unter Quarantäne gestellt, widerrufen oder anderweitig auf öffentlichen Installationsoberflächen nicht verfügbar sein.

Wenn einer dieser Zustände angezeigt wird, installieren Sie die Veröffentlichung nicht, solange der Inhaber das Problem nicht behoben oder die Moderation sie nicht wiederhergestellt hat.

Inhaber können weiterhin Diagnosedaten zu ihren eigenen zurückgehaltenen oder ausgeblendeten Einträgen einsehen. Diese Diagnosedaten erläutern, was geschehen ist und was geändert werden muss, bevor der Eintrag wieder auf öffentlichen Oberflächen erscheinen kann.

## Sperren und Kontostatus

Konten, die gegen die ClawHub-Richtlinien verstoßen, können den Veröffentlichungszugriff verlieren. Schwerwiegender Missbrauch kann zu Kontosperren, dem Widerruf von Tokens, ausgeblendeten Inhalten oder entfernten Einträgen führen. Belastungssignale für Missbrauch durch Herausgeber werden täglich überprüft. Signale, die den ClawHub-Schwellenwert für eine mögliche Sperre erreichen, können eine automatische Warnung auslösen. Wenn der nächste zulässige Scan nach Ablauf der Warnfrist den Herausgeber weiterhin dem Schwellenwert für eine mögliche Sperre zuordnet, kann ClawHub die Kontomaßnahme automatisch anwenden. Prüfsignale mit geringerer Konfidenz und zeitlich begrenztem Umfang werden nicht automatisch durchgesetzt.

Gelöschte, gesperrte oder deaktivierte Konten können keine ClawHub-API-Tokens verwenden. Wenn die CLI-Authentifizierung nach einer Kontomaßnahme fehlschlägt, melden Sie sich an der Weboberfläche an, um den Kontostatus zu überprüfen. Wenn die Anmeldung oder der normale CLI-Zugriff aufgrund einer Sperre oder eines deaktivierten Kontos blockiert ist, verwenden Sie für eine Wiederherstellungsprüfung das [ClawHub-Einspruchsformular](https://appeals.openclaw.ai/).

Wenn eine durch einen Scanner ausgelöste E-Mail eine Skill- oder Plugin-Version als schädlich bezeichnet, laden Sie die gespeicherten Scanergebnisse für die gesperrte eingereichte Version herunter:
`clawhub scan download <slug> --version <version>`. Fügen Sie bei Plugins
`--kind plugin` hinzu. Prüfen Sie die Scanausgabe, korrigieren Sie den Eintrag, erhöhen Sie die Versionsnummer und laden Sie die korrigierte Version hoch.

## Hinweise für Herausgeber

So reduzieren Sie falsch positive Ergebnisse und stärken das Vertrauen der Benutzer:

- halten Sie Namen, Zusammenfassungen, Tags und Änderungsprotokolle korrekt
- geben Sie erforderliche Umgebungsvariablen und Berechtigungen an
- vermeiden Sie verschleierte Installationsbefehle
- verlinken Sie nach Möglichkeit den Quellcode
- verwenden Sie vor der Veröffentlichung von Plugins Testläufe
- antworten Sie klar, wenn Benutzer oder Moderatoren nach dem Verhalten einer Veröffentlichung fragen
