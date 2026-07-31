---
read_when:
    - Sie teilen einen OpenClaw-Agenten mit anderen Betreibern
    - Sie müssen die Anzeigen für Sitzungseigentümer und Anwesenheit verstehen
    - Sie entscheiden, ob ein gemeinsam genutzter Agent ausreichende Isolation bietet
summary: Funktionsweise von Sitzungseigentümerschaft und Präsenz, wenn mehrere Personen einen Agenten bedienen
title: Mehrbenutzermodus
x-i18n:
    generated_at: "2026-07-26T18:25:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c6a5a0e37b8dbeb2ebb7f32c3518acc6f3995dbfc09102f4d58c85e9cd62dfc2
    source_path: concepts/multi-user.md
    workflow: 16
---

Im Mehrbenutzermodus können mehrere vertrauenswürdige Personen denselben OpenClaw-Agenten bedienen. Er ergänzt Sitzungsverantwortlichkeit, Live-Präsenz und Erstellerfilter, sodass ein Team erkennen kann, wer eine Arbeit begonnen hat und wer sie gerade beobachtet.

## Vertrauensgrenze

Jede Person, die einen Agenten bedienen kann, kann ihn alles tun lassen, wozu dieser Agent in der Lage ist. Sitzungsverantwortlichkeit, Sichtbarkeit in der Seitenleiste und Präsenzanzeigen sind Bedienungshilfen und keine Sicherheitsgrenzen.

Wenn Personen nicht auf die Sitzungen, Tools, Anmeldedaten oder Dateien anderer Personen zugreifen dürfen, weisen Sie ihnen separate Agenten oder separate Vertrauensgrenzen für Gateway beziehungsweise Host zu. Verlassen Sie sich zur Isolation nicht auf Avatare der Verantwortlichen oder Filter.

## Verantwortlichkeit und Präsenz

Neue Sitzungen zeichnen einen unveränderlichen `createdActor` auf, wenn der Erstellungspfad nachweisen kann, wer sie ausgelöst hat. Bei authentifizierten Personen wird die dauerhafte ID ihres Gateway-Profils verwendet; anfragende Agenten und Systempfade verwenden dasselbe Akteursfeld. Sitzungen, die ohne nachgewiesenen Akteur erstellt werden, bleiben ohne Zuordnung.

Anzeigenamen von Personen werden anhand des aktuellen Gateway-Profils aufgelöst, wenn Sitzungszeilen zurückgegeben werden. OpenClaw speichert keine Bezeichnungen in Sitzungseinträgen. Daher aktualisiert eine Änderung des Profilnamens die Benutzeroberfläche für die Verantwortlichkeit, ohne den Sitzungsverlauf neu zu schreiben.

Die Web-App stellt Verantwortlichkeit und Präsenz visuell getrennt dar:

- Ein ausgefüllter Avatar des Verantwortlichen bleibt für die gesamte Lebensdauer dieser Sitzung bestehen.
- Umrandete oder durchscheinende Präsenzavatare zeigen Personen, die derzeit verbunden sind oder zuschauen.
- Der Personenfilter der Seitenleiste zeigt Sitzungen an, die von einer bestimmten Identität erstellt wurden, und behält dabei die vorhandenen benutzerdefinierten Gruppen bei.

Wenn in der geladenen Sitzungsliste weniger als zwei unterschiedliche Ersteller vorkommen, blendet OpenClaw alle Elemente für Verantwortlichkeit und Personenfilter aus. Ein Gateway für eine einzelne Person sieht daher unverändert aus.

## Entwürfe

Starten Sie eine Sitzung als Entwurf, damit laufende Arbeiten nicht in den Seitenleisten Ihrer Teammitglieder erscheinen, bis Sie sie veröffentlichen. Entwürfe werden Administratoren nie vorenthalten; sie sehen die Entwürfe anderer Personen mit einer verblassten Geistmarkierung. Dies ist eine Koordinationsfunktion und keine Sicherheitsgrenze.

## Zuordnung von Beiträgen

Die Absenderzuordnung für Beiträge erfolgt nach bestem Bemühen. Durch Steuerung können Eingaben in einen aktiven Beitrag eingefügt werden, sodass das Transkript den Beitrag jeder Person nicht immer als separaten Beitrag darstellen kann.

## Verwandte Themen

- [Die Hauptsitzung](/de/concepts/main-session)
- [Sitzungsverwaltung](/de/concepts/session)
- [Präsenz](/de/concepts/presence)
- [Gateway-Sicherheit](/de/gateway/security)
