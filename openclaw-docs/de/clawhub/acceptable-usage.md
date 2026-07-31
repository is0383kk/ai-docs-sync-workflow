---
read_when:
    - Uploads auf Missbrauch oder Richtlinienverstöße prüfen
    - Moderationsdokumentation oder Prüfer-Runbooks verfassen
    - Entscheiden, ob ein Skill ausgeblendet oder ein Benutzer gesperrt werden sollte
sidebarTitle: Acceptable Usage
summary: 'Marketplace-Richtlinie: Was ClawHub erlaubt und was dort nicht gehostet wird.'
title: Akzeptable Nutzung
x-i18n:
    generated_at: "2026-07-26T17:40:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ace357e7a3e9f4d242f113ad791b254e94ae8a841dd9a864a77c5bac15713132
    source_path: clawhub/acceptable-usage.md
    workflow: 16
---

# Akzeptable Nutzung

ClawHub hostet Skills, Plugins, Pakete und Marketplace-Metadaten für OpenClaw.
Diese Seite hilft bei der Entscheidung, ob Inhalte oder Veröffentlichungsverhalten auf
ClawHub gehören.

Diese Regeln gelten dafür, was ein Eintrag tut, welche Ausführungen er von Benutzern
verlangt, wie er sich darstellt und wie Herausgeber die Entdeckungs-, Installations- und
Vertrauensfunktionen von ClawHub nutzen. Informationen zu Moderationsstatus und Kontostatus finden Sie unter
[Moderation und Kontosicherheit](/de/clawhub/moderation). Informationen zu Urheberrechts- oder anderen Rechtsansprüchen
finden Sie unter [Anfragen zu Inhaltsrechten](/de/clawhub/content-rights).

## Zulässige Inhalte

ClawHub begrüßt Inhalte, die nützlich und verständlich sind und nach Treu und Glauben
veröffentlicht werden.

| Kategorie                                        | Zulässig, wenn                                                                                                                            |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Entwicklerproduktivität                          | Der Eintrag Benutzern beim Entwickeln, Testen, Migrieren, Debuggen, Dokumentieren oder Betreiben von Software hilft.                      |
| UI-, Daten- und Automatisierungsabläufe           | Der Umfang klar ist, erforderliche Zugangsdaten ausdrücklich angegeben sind und riskante Aktionen Prüf-, Testlauf-, Vorschau- oder Bestätigungspfade enthalten. |
| Defensive Sicherheit, Moderation und Missbrauchsprüfung | Das Tool für autorisierte Prüfungen vorgesehen ist, Beweismittel bewahrt und die Grenzen menschlicher Genehmigung klar einhält.      |
| Persönliche oder Team-Abläufe                     | Der Ablauf auf Einwilligung basierende Konten, eine transparente Einrichtung und ausdrückliche Berechtigungen verwendet.                 |
| Gepflegte Kataloge                                | Jeder Eintrag eigenständig und nützlich ist, korrekt beschrieben und angemessen gepflegt wird.                                           |

Der Kontext ist entscheidend. Dasselbe Thema kann in einem eng begrenzten defensiven oder
auf Einwilligung basierenden Umfeld akzeptabel und als Missbrauchsablauf verpackt inakzeptabel sein.

## Unzulässige Inhalte

ClawHub hostet keine Inhalte, deren Hauptzweck Missbrauch, Täuschung, unsichere
Ausführung oder Rechtsverletzungen sind.

| Kategorie                                                    | Nicht zulässig                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Unbefugter Zugriff oder Umgehung von Sicherheitsmaßnahmen    | Umgehung der Authentifizierung, Kontoübernahme, Missbrauch von Ratenbegrenzungen, Übernahme laufender Anrufe oder Agenten, wiederverwendbarer Diebstahl von Sitzungen oder automatische Genehmigung von Kopplungsabläufen für nicht genehmigte Benutzer.                                                             |
| Plattformmissbrauch und Umgehung von Sperren                 | Verdeckte Konten nach Sperren, Aufwärmen oder Farmen von Konten, vorgetäuschte Interaktionen, Automatisierung mehrerer Konten, massenhafte Veröffentlichungen, Spam-Bots oder Automatisierung zur Vermeidung der Erkennung.                                                                                            |
| Betrug, Scams und irreführende Finanzabläufe                 | Gefälschte Zertifikate oder Rechnungen, irreführende Zahlungsabläufe, betrügerische Kontaktaufnahme, vorgetäuschte soziale Bestätigung, Abläufe mit synthetischen Identitäten für Betrug oder Tools zum Ausgeben bzw. Abbuchen von Geld ohne klare menschliche Genehmigung.                                            |
| In die Privatsphäre eingreifende Datenanreicherung oder Überwachung | Auslesen von Kontaktdaten für Spam, Doxxing, Stalking, Lead-Extraktion in Verbindung mit unaufgeforderter Kontaktaufnahme, verdeckte Überwachung, biometrischer Abgleich ohne Einwilligung oder Verwendung geleakter Daten bzw. Daten aus Sicherheitsverletzungen.                                                |
| Nachahmung oder Identitätsmanipulation ohne Einwilligung     | Gesichtsaustausch, digitale Zwillinge, geklonte Influencer, gefälschte Persönlichkeiten oder andere Tools, die zum Nachahmen oder Irreführen verwendet werden.                                                                                                                                                      |
| Explizite sexuelle Inhalte oder nicht sicherheitsbeschränkte Generierung von Inhalten für Erwachsene | Generierung von NSFW-Bildern, -Videos oder -Inhalten; Wrapper für Inhalte für Erwachsene um APIs von Drittanbietern; oder Einträge, deren Hauptzweck explizite sexuelle Inhalte sind.                                                                                                           |
| Verborgene, unsichere oder irreführende Ausführungsanforderungen | Verschleierte Installationsbefehle, Pipe-to-Shell-Installationsprogramme, etwa heruntergeladene Inhalte, die ohne klare Prüfbarkeit mit `sh` oder `bash` ausgeführt werden, nicht deklarierte Anforderungen an Secrets oder private Schlüssel, entfernte Ausführung von `npx @latest` ohne klare Prüfbarkeit oder Metadaten, die verschleiern, was der Eintrag tatsächlich zur Ausführung benötigt. |
| Urheberrechtsverletzendes oder anderweitig rechtsverletzendes Material | Erneute Veröffentlichung von Skills, Plugins, Dokumentation, Markenressourcen oder proprietärem Code anderer Personen ohne Erlaubnis; Verletzung von Lizenzbedingungen; oder Nachahmung des ursprünglichen Autors oder Herausgebers.                                                                            |

## Unzulässiges Marketplace-Verhalten

ClawHub prüft auch, wie Herausgeber den Marketplace nutzen. Verwenden Sie ClawHub nicht,
um Auffindbarkeit, Kennzahlen, Vertrauenssignale, Moderationssysteme oder die
Aufmerksamkeit von Benutzern zu manipulieren.

Zu unzulässigem Marketplace-Verhalten gehören:

- massenhaftes Veröffentlichen einer großen Zahl von mit geringem Aufwand erstellten, duplizierten, als Platzhalter dienenden oder
  maschinell generierten Einträgen, die keinen echten Nutzen für Benutzer zu haben scheinen
- Überfluten von Such- oder Kategorieansichten mit nahezu identischen Skills oder Plugins
- Veröffentlichen Hunderter Einträge mit geringer oder keiner Nutzung, Pflege, Quelltransparenz
  oder sinnvollen Unterscheidung
- künstliches Aufblähen von Installationen, Downloads, Sternen oder anderen Interaktionskennzahlen
  durch Automatisierung, Selbstinstallationsschleifen, gefälschte Konten, koordinierte
  Aktivitäten, bezahlte Interaktionen oder anderes nicht organisches Verhalten
- Erstellen oder Wechseln von Konten zur Umgehung von Moderation, Sperren, Herausgeberbeschränkungen oder
  Marketplace-Prüfungen
- Irreführen von Benutzern hinsichtlich Eigentum, Quelle, Fähigkeiten, Sicherheitsstatus,
  Installationsanforderungen oder Zugehörigkeit zu einem anderen Projekt oder Herausgeber
- wiederholtes Hochladen von Inhalten, die bereits verborgen, entfernt oder blockiert wurden,
  ohne das zugrunde liegende Problem zu beheben

Das Veröffentlichen großer Mengen stellt nicht automatisch Missbrauch dar. Große Kataloge sind akzeptabel,
wenn sich die Einträge wesentlich unterscheiden, korrekt beschrieben und gepflegt werden
und von echten Benutzern verwendet werden. Große Kataloge werden zu einem Vertrauens- und Sicherheitsproblem, wenn
hohe Mengen mit oberflächlichen, duplizierten, irreführenden, ungepflegten oder
künstlich beworbenen Einträgen einhergehen.

## Inhaltsrechte

Wenn Sie der Ansicht sind, dass Inhalte auf ClawHub Ihr Urheberrecht oder andere Rechte verletzen, verwenden Sie
[Anfragen zu Inhaltsrechten](/de/clawhub/content-rights). Verwenden Sie normale Marketplace-
Meldungen nicht für Urheberrechts- oder andere Rechtsansprüche, sofern der Eintrag nicht zugleich unsicher,
bösartig oder irreführend ist.

## Prüfung und Durchsetzung

ClawHub kann automatisierte Prüfungen, statistische Missbrauchssignale, Benutzermeldungen und
Prüfungen durch Mitarbeiter einsetzen, um unsichere Inhalte oder missbräuchliches Veröffentlichungsverhalten zu erkennen. Ein Signal
beweist für sich allein keinen Missbrauch; es hilft ClawHub bei der Entscheidung, was geprüft werden muss.

Wir können:

- rechtsverletzende Einträge verbergen, zurückhalten, entfernen, vorläufig löschen oder, sofern für den Ressourcentyp unterstützt,
  endgültig löschen
- Downloads oder Installationen unsicherer Releases blockieren
- API-Tokens widerrufen
- zugehörige Inhalte vorläufig löschen
- den Veröffentlichungszugriff einschränken
- wiederholt oder schwerwiegend gegen die Regeln verstoßende Personen sperren

Bei offensichtlichem Missbrauch garantieren wir keine vorherige Warnung. Informationen zu Meldungen, Moderationssperren,
verborgenen Einträgen, Sperren und Kontostatus finden Sie unter
[Moderation und Kontosicherheit](/de/clawhub/moderation).
