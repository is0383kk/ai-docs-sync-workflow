---
read_when:
    - Arbeiten an Sprachaktivierungs- oder PTT-Pfaden
summary: Sprachaktivierungs- und Push-to-Talk-Modi sowie Routingdetails in der Mac-App
title: Sprachaktivierung (macOS)
x-i18n:
    generated_at: "2026-07-26T18:27:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d3b2a01ee997b4158bf88b9ef54b1e523503722620f943d594323516619e7502
    source_path: platforms/mac/voicewake.md
    workflow: 16
---

# Sprachaktivierung und Push-to-Talk

## Anforderungen

Sprachaktivierung und Push-to-Talk erfordern macOS 26 oder neuer. Unter älteren macOS-Versionen sind die Steuerelemente auf der Seite für Spracheinstellungen ausgeblendet; stattdessen wird dort auf die Anforderung von macOS 26 hingewiesen.

Die Sprachaktivierung setzt voraus, dass Apple Speech für die ausgewählte Sprache die Erkennung auf dem Gerät unterstützt. Die App startet die passive Erkennung des Aktivierungsworts nicht, wenn diese ausschließlich lokale Verarbeitung nicht verfügbar ist; sie greift niemals auf die Netzwerkerkennung zurück. Push-to-Talk, Sprechmodus und die Diktierfunktion von Quick Chat sind explizite Benutzeraktionen und können für eine breitere Sprachabdeckung die Netzwerkdienste von Apple Speech nutzen.

## Modi

- **Aktivierungswortmodus** (Standard): Eine ständig aktive Speech-Erkennung auf dem Gerät wartet auf Aktivierungstoken (`swabbleTriggerWords`). Bei einer Übereinstimmung beginnt die Aufnahme, das Overlay mit dem vorläufigen Text wird angezeigt und die Nachricht wird nach einer Sprechpause automatisch gesendet.
- **Push-to-Talk (rechte Wahltaste gedrückt halten)**: Halten Sie die rechte Wahltaste gedrückt, um die Aufnahme sofort zu starten; eine Aktivierung ist nicht erforderlich. Das Overlay wird angezeigt, solange die Taste gedrückt ist. Beim Loslassen wird die Aufnahme abgeschlossen und nach einer kurzen Verzögerung weitergeleitet, damit Sie den Text bearbeiten können.

## Laufzeitverhalten (Aktivierungswort)

- Die Erkennung befindet sich in `VoiceWakeRuntime`.
- Die Aktivierung erfolgt nur, wenn zwischen dem Aktivierungswort und dem nächsten Wort eine deutliche Pause liegt (`triggerPauseWindow` = 0.55s). Overlay und Signalton können bereits während der Pause starten, noch bevor der Befehl beginnt.
- Zeitfenster für Stille: 2.0s (`silenceWindow`) bei fortlaufender Sprache, 5.0s (`triggerOnlySilenceWindow`), wenn nur das Aktivierungswort erkannt wurde.
- Harte Begrenzung: 120s (`captureHardStop`), um unkontrolliert fortlaufende Sitzungen zu verhindern.
- Entprellzeit zwischen Sitzungen: 350ms (`debounceAfterSend`) nach dem Senden.
- Das Overlay wird über `VoiceWakeOverlayController` gesteuert und stellt bestätigten und vorläufigen Text farblich unterschiedlich dar.
- Nach dem Senden wird die Erkennung sauber neu gestartet, um auf die nächste Aktivierung zu warten.

## Lebenszyklusinvarianten

- Wenn die Sprachaktivierung aktiviert ist und die Berechtigungen erteilt wurden, bleibt die Aktivierungsworterkennung aktiv, außer während einer laufenden Push-to-Talk-Aufnahme.
- Beim Schließen des Overlays wird die Erkennung immer fortgesetzt, auch beim manuellen Schließen über die Schaltfläche X: `VoiceSessionCoordinator.overlayDidDismiss` ruft bei jedem Schließpfad `VoiceWakeRuntime.refresh(state:)` auf. Informationen zum Sitzungs-/Tokenmodell finden Sie unter [Sprach-Overlay](/de/platforms/mac/voice-overlay).

## Besonderheiten von Push-to-Talk

- Die Tastenkombinationserkennung verwendet einen globalen `.flagsChanged`-Monitor für die rechte Wahltaste (`keyCode 61` + `.option`). Er beobachtet Ereignisse lediglich und unterdrückt sie niemals.
- Die Aufnahme befindet sich in `VoicePushToTalk`: Sie startet Speech sofort, überträgt vorläufige Ergebnisse fortlaufend an das Overlay und ruft beim Loslassen `VoiceWakeForwarder` auf.
- Beim Start von Push-to-Talk wird die Aktivierungswort-Laufzeit angehalten, um konkurrierende Audioabgriffe zu vermeiden; nach dem Loslassen wird sie automatisch neu gestartet.
- Berechtigungen: Mikrofon und Spracherkennung sind erforderlich; für den Empfang von Tastenereignissen ist die Genehmigung für Bedienungshilfen/Eingabeüberwachung erforderlich.
- Externe Tastaturen: Einige stellen die rechte Wahltaste nicht wie erwartet bereit. Bieten Sie eine alternative Tastenkombination an, wenn Benutzer von nicht erkannten Betätigungen berichten.

## Benutzereinstellungen

- Umschalter **Sprachaktivierung**: Aktiviert die Aktivierungswort-Laufzeit.
- **Rechte Wahltaste zum Sprechen gedrückt halten**: Aktiviert die Push-to-Talk-Überwachung.
- Wenn die ausgewählte Sprache auf diesem Mac keine Erkennung auf dem Gerät unterstützt, bleibt die Sprachaktivierung deaktiviert, während Push-to-Talk und der Sprechmodus weiterhin verfügbar sind.
- Auswahlfelder für Sprache und Mikrofon, eine Live-Pegelanzeige, eine Tabelle mit Aktivierungswörtern und eine Testfunktion (nur lokal, leitet niemals etwas weiter).
- Die Mikrofonauswahl behält die letzte Auswahl bei, wenn die Verbindung zu einem Gerät getrennt wird, zeigt einen Hinweis zur getrennten Verbindung an und greift vorübergehend auf den Systemstandard zurück, bis das Gerät wieder verfügbar ist.
- **Töne**: Signaltöne bei Erkennung der Aktivierung und beim Senden; standardmäßig wird der macOS-Systemton „Glass“ verwendet. Wählen Sie für jedes Ereignis eine beliebige von `NSSound` ladbare Datei (z. B. MP3/WAV/AIFF) oder wählen Sie **Kein Ton**.

## Weiterleitungsverhalten

- Bei der Weiterleitung wählt `VoiceWakeForwarder.selectedSessionOptions` den aktiven WebChat-Sitzungsschlüssel aus, sofern einer festgelegt ist, andernfalls den Hauptsitzungsschlüssel des Gateways.
- Die Sitzung wird über `sessions.list` ermittelt. Über den Zustellungskontext der Sitzung werden der Zustellungskanal und das Ziel abgeleitet; als Rückfall dienen zunächst der letzte Kanal bzw. das letzte Ziel und anschließend ein analysierter Sitzungsschlüssel. Wenn keine Zuordnung möglich ist, wird standardmäßig WebChat verwendet.
- Wenn die Zustellung fehlschlägt, wird der Fehler protokolliert (Kategorie `voicewake.forward`) und der Lauf bleibt weiterhin über WebChat bzw. die Sitzungsprotokolle sichtbar.

## Weiterleitungsnutzlast

- `VoiceWakeForwarder.prefixedTranscript(_:)` stellt dem Transkript eine Zeile mit einem Computerhinweis voran (ermittelter Hostname, ersatzweise „dieser Mac“), die sowohl für den Aktivierungswort- als auch den Push-to-Talk-Pfad verwendet wird.

## Schnelle Überprüfung

- Aktivieren Sie Push-to-Talk, halten Sie die rechte Wahltaste gedrückt, sprechen Sie und lassen Sie die Taste los: Das Overlay sollte zunächst vorläufige Ergebnisse anzeigen und sie anschließend senden.
- Während die Taste gedrückt wird, sollten die Ohren in der Menüleiste vergrößert bleiben (`triggerVoiceEars(ttl: nil)`); nach dem Loslassen werden sie wieder kleiner.

## Verwandte Themen

- [Sprachaktivierung](/de/nodes/voicewake)
- [Sprach-Overlay](/de/platforms/mac/voice-overlay)
- [macOS-App](/de/platforms/macos)
