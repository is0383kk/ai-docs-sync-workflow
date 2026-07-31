---
read_when:
    - Nachfassen nach Feedback von Barnacle oder ClawSweeper
    - ClawSweeper um eine Überprüfung bitten
    - Fehlerbehebung bei Barnacle, ClawSweeper, veralteten Labels oder automatischen Schließungen
sidebarTitle: PR review flow
summary: Wie Feedback von Barnacle und ClawSweeper dazu beiträgt, OpenClaw-Pull-Requests durch den Review zu bringen.
title: Pull-Request-Review-Ablauf
x-i18n:
    generated_at: "2026-07-26T18:05:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9bec4578d55d2279450e991480467946db7da5ca956f85c35b4221190b2babe
    source_path: reference/pull-request-review-flow.md
    workflow: 16
---

Diese Seite erläutert den Review-Ablauf, nachdem Sie einen OpenClaw-Pull-Request
geöffnet oder aktualisiert haben: was Barnacle und ClawSweeper tun, wie Sie den PR anhand ihres
Feedbacks verbessern und was Sie prüfen sollten, wenn die Automatisierung keine Rückmeldung gibt.

Barnacle und ClawSweeper helfen den Maintainern, die Review-Warteschlange nutzbar zu halten. Sie
ersetzen nicht das Urteil der Maintainer.

## Barnacle

Barnacle führt deterministisches GitHub-Triage durch. Es sucht nach bekannten Fällen der
Warteschlangenverwaltung und reagiert mit Labels, Kommentaren oder Schließungen.

Barnacle kann aktiv werden, wenn:

- ein PR-Text größtenteils leer ist oder der Problemkontext fehlt;
- ein PR keine brauchbaren Nachweise enthält;
- bei einer reinen Dokumentations-, Test-, Refactoring-, CI- oder Infrastrukturänderung
  ein verknüpfter Maintainer-Kontext fehlt;
- eine Änderung so aussieht, als gehöre sie statt in den Core zu ClawHub oder einem Plugin;
- ein Branch nicht zugehörige Arbeiten enthält;
- ein Autor mehr als 20 offene PRs hat.

Barnacle wird mit vertrauenswürdigem Workflow-Code des Repositorys ausgeführt. Es checkt
Beitragenden-Code weder aus noch führt es ihn aus.

Die meisten Routing-Labels sind Signale für Maintainer oder die Automatisierung, daher müssen Beitragende
selbst keine Labels hinzufügen.

## ClawSweeper

ClawSweeper ist der KI-gestützte Review- und Wartungs-Bot für OpenClaw-
Repositorys. Er kann PRs prüfen, Nachweise bewerten, dauerhafte Review-Kommentare hinterlassen
und Maintainer bei abgesicherten Reparatur- oder Automerge-Abläufen unterstützen.

Ein positives ClawSweeper-Ergebnis ist ein unterstützender Nachweis, keine Genehmigung durch Maintainer.
Die Maintainer entscheiden weiterhin, ob und wann ein PR zum Mergen bereit ist.

ClawSweeper arbeitet warteschlangenbasiert. Erwarten Sie keine sofortige Antwort, nachdem Sie einen
PR geöffnet, einen Commit gepusht oder eine Review-Anfrage hinzugefügt haben. Auch Label-Aktualisierungen nach einem
ClawSweeper-Durchlauf können einige Zeit dauern.

Neue PRs gelangen in die ClawSweeper-Review-Warteschlange. Maintainer können außerdem Review-,
Reparatur- oder Automerge-Abläufe mit Labels oder Befehlen in die Warteschlange einreihen. Fordern Sie bei gewöhnlichen Aktualisierungen durch Beitragende
erst dann ein weiteres Review von ClawSweeper an, nachdem Sie den
Branch, die PR-Beschreibung, die Nachweise oder den Code aktualisiert haben. Fordern Sie anschließend mit einem neuen
PR-Kommentar ein frisches Review an:

```text
@clawsweeper re-review
```

PR-Autoren können auch `@clawsweeper re-run` verwenden; Benutzer mit Schreibzugriff auf das Repository
können für jedes offene Element beide Befehle verwenden. Der einfache
Befehl `@clawsweeper review` ist ausschließlich Maintainern vorbehalten. Haben Sie Geduld: Eine erneute Anfrage,
bevor die angeforderten Änderungen vorliegen, erzeugt lediglich zusätzliche Störungen in der Warteschlange.

Wenn ClawSweeper Review-Konversationen hinterlässt, behandeln Sie diese wie normales Review-
Feedback und verwenden Sie die nachfolgende Checkliste.

Wenn ein menschlicher Beitragender oder Maintainer den PR übernommen hat und aktiv
daran arbeitet, rufen Sie nicht gleichzeitig ClawSweeper auf und arbeiten Sie auch nicht anderweitig
am PR. Lassen Sie das menschliche Review oder die Reparatur zuerst abschließen. Wenn die Aktivität endet, prüfen Sie,
ob der Autor aufgefordert wurde, Nachweise bereitzustellen oder andere Aktualisierungen vorzunehmen.

## Einen PR während des Reviews verbessern

Sobald Barnacle, ClawSweeper oder ein Maintainer antwortet, verwenden Sie dieses Feedback als
Checkliste für die nächsten Schritte des PRs.

1. Lesen Sie ClawSweepers `Rank-up moves:` und `Proof guidance:` als Maßnahmenliste
   für diesen PR. Bewertungen und Labels sind Review-Signale, keine festen Merge-Ziele.
2. Pushen Sie die angeforderte Code- oder Dokumentationsänderung und aktualisieren Sie die PR-Beschreibung, wenn
   sich das Problem, die Lösung, die Auswirkungen auf Benutzer oder die Nachweise geändert haben.
3. Fügen Sie den angeforderten Nachweis hinzu und verwenden Sie dafür Nachweise, die zur Änderung passen.
4. Lösen Sie bearbeitete Review-Konversationen selbst auf. Antworten Sie und lassen Sie eine
   Konversation nur dann offen, wenn Sie eine Entscheidung durch Maintainer oder Reviewer benötigen.
5. Fordern Sie erst dann ein erneutes Review an, wenn der Branch, die PR-Beschreibung, die Nachweise und
   die relevanten CI-Ergebnisse aktuell sind. Mehrere Aktualisierungs- und Review-Zyklen zwischen
   Autor, Maintainer und ClawSweeper sind normal.
6. Führen Sie die Diskussion nach Möglichkeit im PR. Wechseln Sie nur dann zu `#clawtributors` auf Discord,
   wenn der PR eine Koordination mit Maintainern erfordert, die Automatisierung blockiert zu sein scheint
   oder sich die nächste Entscheidung nur schwer in GitHub-Kommentaren klären lässt. Geben Sie den PR-
   Link, den aktuellen Status und die konkrete Frage oder den noch fehlenden Nachweis an.

Halten Sie den PR-Text aktuell. Kommentare helfen bei der Diskussion, aber die PR-
Beschreibung ist die dauerhafte Zusammenfassung, auf die Maintainer und Automatisierung erneut zurückgreifen.

`status: ⏳ waiting on author` bedeutet, dass die nächste Aktion beim PR-Autor liegt:
Aktualisieren Sie den Branch, die PR-Beschreibung oder die Nachweise beziehungsweise antworten Sie mit dem fehlenden Kontext,
bevor Sie ein weiteres Review anfordern.

Nützliche Nachweise umfassen die Ausgabe fokussierter Tests, CI-Ergebnisse, Screenshots,
Aufzeichnungen, Terminalausgaben, Live-Beobachtungen, bereinigte Protokolle oder Links zu
Artefakten. Fügen Sie bei visuellen Änderungen nach Möglichkeit Vorher- und Nachher-Screenshots hinzu.
Verlinken Sie für Nachweisdateien vorzugsweise CI-Artefakte, auf GitHub hochgeladene Screenshots oder
Aufzeichnungen oder einen kurzen bereinigten Protokollauszug. Committen Sie keine generierten Nachweisdateien,
sofern diese nicht Teil der eigentlichen Dokumentations-, Test- oder Produktänderung sind.

Die Bereinigung sensibler Daten liegt in der Verantwortung des Beitragenden. Entfernen Sie Secrets,
Tokens, private URLs, Benutzerdaten und nicht zugehörige Protokolle, bevor Sie Nachweise veröffentlichen.

OpenClaw verwendet außerdem eine separate Automatisierung für inaktive Elemente. Nicht zugewiesene Issues und PRs können
nach 14 Tagen ohne Aktivität als inaktiv markiert und nach weiteren 7 inaktiven Tagen geschlossen werden.
Zugewiesene PRs werden 27 Tage nach dem Öffnen als inaktiv markiert, unabhängig von späteren
Aktualisierungen, und anschließend nach 7 inaktiven Tagen ohne Aktivität geschlossen. Wenn ein zugewiesener PR
noch aktiv ist, stimmen Sie sich mit dem daran arbeitenden Maintainer ab.

## Wenn die Automatisierung keine Rückmeldung gibt

Die Automatisierung kann ohne Rückmeldung bleiben, wenn ein Maintainer das Element bereits bearbeitet, eine
Review- oder Reparaturanfrage noch in der Warteschlange steht, das Ereignis routinemäßig ist oder die
ClawSweeper-Ausführungsspur nicht für die angeforderte Aktion konfiguriert ist.

Sie kann auch auf eine Aktion verzichten, wenn ein vertrauenswürdiger Workflow nicht vertrauenswürdigen
Beitragenden-Code ausführen müsste. In diesem Fall verwenden Maintainer stattdessen ein normales Review oder einen sichereren
Workflow.

## Fehlerbehebung

Wenn ClawSweeper nicht sofort antwortet, warten Sie, bevor Sie es erneut versuchen. Der Dienst arbeitet
warteschlangenbasiert, und wiederholte Kommentare oder Label-Änderungen können die Prüfung des Threads erschweren,
ohne die Warteschlange zu beschleunigen.

Prüfen Sie Folgendes, bevor Sie um Hilfe bitten:

- Die PR-Beschreibung ist aktuell;
- der neueste Commit enthält die angeforderte Änderung;
- die CI ist abgeschlossen oder der PR-Text erläutert, warum ein verbleibender Fehler
  nicht mit dem PR zusammenhängt;
- die neueste Review-Anfrage wurde als PR-Kommentar gestellt:
  `@clawsweeper re-review`;
- kein Maintainer oder Beitragender arbeitet bereits aktiv am PR;
- die neueste Anfrage liegt nicht noch innerhalb der normalen Wartezeit der ClawSweeper-Warteschlange.

Wenn mehrere Stunden, nachdem der PR auf dem aktuellen Stand ist, weiterhin keine ClawSweeper-Antwort vorliegt
oder der PR durch die Automatisierung blockiert zu sein scheint, fragen Sie in `#clawtributors` auf Discord nach.
Geben Sie den PR-Link an, was Sie erwartet haben, wann Sie die Anfrage gestellt haben und was sich seit
dem letzten Bot-Kommentar geändert hat.

## Die Automatisierung forken

Projekte, die eine ähnliche Review-Automatisierung wünschen, können ClawSweeper untersuchen oder forken:

- [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)
- [ClawSweeper-Dokumentation](https://clawsweeper.bot/)

## Verwandte Themen

- [Mitwirken](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [CI-Pipeline](/de/ci)
