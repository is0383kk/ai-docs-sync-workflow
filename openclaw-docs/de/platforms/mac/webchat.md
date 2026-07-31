---
read_when:
    - Debugging der Mac-WebChat-Ansicht oder des Loopback-Ports
summary: Wie die Mac-App den Gateway-WebChat einbettet und wie Sie ihn debuggen können
title: WebChat (macOS)
x-i18n:
    generated_at: "2026-07-26T18:34:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5e5983954e12d8546a01d089eda54e7eb0c60b4c92eff670f91797cd022c9fd
    source_path: platforms/mac/webchat.md
    workflow: 16
---

Die macOS-Menüleisten-App bettet die WebChat-Benutzeroberfläche als native SwiftUI-Ansicht ein. Sie stellt eine Verbindung zum Gateway her und verwendet standardmäßig die primäre Sitzung für den ausgewählten Agenten (`main` oder `global`, wenn `session.scope` den Wert `global` hat).

Das vollständige Chatfenster ist eine native geteilte Ansicht:

- **Sitzungsseitenleiste**: durchsuchbare Sitzungsliste mit Bereichen für angeheftete Sitzungen, Gateway-gestützte Gruppen und kürzlich verwendete Sitzungen. Erzeugte untergeordnete Sitzungen werden innerhalb jedes Bereichs unter ihrer übergeordneten Sitzung verschachtelt; eingeklappte übergeordnete Sitzungen fassen laufende, fehlgeschlagene und ungelesene untergeordnete Sitzungen zusammen. Kontextmenüs unterstützen Sitzungsinformationen, Umbenennen, Anheften, Forken, Gelesen/Ungelesen, Archivieren/Wiederherstellen, das Kopieren des Sitzungsschlüssels und Löschen. Die primäre Aktion für eine neue Sitzung (oder Shift-Cmd-N) erstellt diese sofort über `sessions.create`; im danebenliegenden Optionen-Popover können ein Agent ausgewählt und ein verwalteter Worktree mit einer optionalen Basisreferenz angefordert werden.
- **Fenster-Symbolleiste**: Ring für die Kontextnutzung (Tokens und Sitzungskosten, mit einer kompakten Aktion), Modellsteuerungen und ein Menü für Sitzungsaktionen. Modelle werden nach Provider gruppiert, wobei der Standard-Provider zuerst erscheint; angeheftete und kürzlich verwendete Modelle bleiben ganz oben. Die Steuerungen können die Denkstufe des Modells übernehmen oder überschreiben, die Ausführlichkeit von Tool-Aufrufen festlegen und schnelle Antworten ein- oder ausschalten. Über das Menü kann die aktuelle Sitzung umbenannt oder geforkt sowie ihr Status für Anheften, Gelesen und Archivieren aktualisiert werden. **Sitzungen…** (Shift-Cmd-S) öffnet den Manager für aktive/archivierte Sitzungen zur Gateway-Suche, Gruppenverwaltung, Sitzungsprüfung sowie zum Umbenennen, Anheften, Archivieren und Wiederherstellen. Im Auswahlmodus können mehrere aktive Sitzungen angeheftet, gelöst, archiviert oder gelöscht werden, wobei einzelne Fehler sichtbar bleiben. Separate Häkchen im Menü blenden die Gedankengänge des Assistenten und die Tool-Aktivität ein oder aus; beide sind standardmäßig aktiviert und werden über App-Starts hinweg gespeichert.
- **Transkript und Eingabebereich**: Assistentennachrichten werden als Klartext mit einem Avatar dargestellt, Benutzernachrichten als farblich hervorgehobene Sprechblasen. Ausstehende Fragen des Agenten werden als native Karten mit Optionen zur Einzel- oder Mehrfachauswahl, Freitextantworten unter **Sonstiges**, Ablauf-Countdowns und gemeinsamem Endstatus dargestellt. Leere Chats bieten Einstiegsaufforderungen für den Desktop. Die Eingabe von `/` öffnet die durch `commands.list` bereitgestellte Autovervollständigung für Slash-Befehle mit Tastaturnavigation über Pfeiltasten/Tab/Return/Escape. Klicken Sie mit der rechten Maustaste auf eine Nachricht, um ihr sichtbares Markdown ohne ausgeblendete Gedankengänge zu kopieren. Abgeschnittene Assistentennachrichten bieten außerdem **Vollständige Nachricht öffnen**, wodurch eine auswählbare Markdown-Leseansicht geladen wird. Verwenden Sie **Anhören** für Gateway-TTS mit lokaler Sprachausgabe als Fallback.
- **Sprachsteuerungen**: Über den Eingabebereich kann der bestehende macOS-Sprechmodus gestartet oder beendet werden, ohne dessen Menüleisten-Overlay zu ersetzen. Während der Sprechmodus aktiv ist, zeigt der Eingabebereich seinen Status „Zuhören“, „Denken“ oder „Sprechen“, die aktuelle Audioaktivität und ein erweiterbares fortlaufendes Transkript an. Klicken Sie mit der rechten Maustaste auf die Sprechtaste, um **Systemstandard** oder ein verbundenes Mikrofon auszuwählen; dieselbe Mikrofonauswahl wird für die Sprachaktivierung und Push-to-Talk verwendet. Wenn ein ausgewähltes Mikrofon getrennt wird, greift die aktive Sprechsitzung auf den Systemstandard zurück und versucht beim nächsten Start des Sprechmodus erneut, die Auswahl zu verwenden. Eine separate Mikrofonaktion zeichnet eine Sprachnachricht auf, wenn der Sprechmodus die Audioaufnahme nicht belegt.

Das verankerte kompakte Chatpanel der Menüleiste behält das kompakte einspaltige Layout mit denselben direkt eingebetteten Steuerungen für Modell, Denkstufe, Ausführlichkeit und schnelle Antworten sowie Einstiegsaufforderungen, Sprechmodus, Sprachnachrichten und Anhören bei. Gedankengänge des Assistenten und Tool-Aktivität bleiben in dieser kompakten Oberfläche ausgeblendet.

## Mehrere Gateway-Fenster

Öffnen Sie **Einstellungen → Gateways**, um wiederverwendbare Gateway-Profile hinzuzufügen oder zu entfernen. Jedes
Profil enthält einen privaten Netzwerkendpunkt vom Typ `ws://` oder einen sicheren Endpunkt vom Typ `wss://` sowie dessen
optionales Token oder Passwort; Anmeldedaten werden im macOS-Schlüsselbund gespeichert.
Sichere Profile verwalten jeweils eine eigene, durch das Systemvertrauen abgesicherte Zertifikatsanheftung bei der ersten Verwendung
und übernehmen `gateway.remote.tlsFingerprint` nicht vom primären Gateway.
Beim Entfernen eines Profils werden außerdem dessen geöffnete Fenster geschlossen und seine sekundäre
Verbindung beendet.

Wählen Sie **Ablage → Neues Gateway-Fenster…** oder drücken Sie Cmd-N und wählen Sie anschließend eines dieser
gespeicherten Profile aus. Die Auswahl merkt sich das zuletzt verwendete Profil. Jede
Auswahl erstellt ein neues unabhängiges Fenster, sodass dasselbe Gateway in
mehreren Fenstern mit unterschiedlichen aktiven Sitzungen und Navigationszuständen angezeigt werden kann.

Jedes gespeicherte Profil besitzt eine gemeinsam genutzte Gateway-Verbindung, einen Geräteauthentifizierungsbereich,
einen Transkript-Cache, einen Offline-Postausgang und Routen-Leases. Fenster für dieses Profil
verwenden diese Ressourcen gemeinsam, bleiben jedoch unabhängig voneinander navigierbar. Fenster für
unterschiedliche Profile bleiben verbunden und führen Chats gleichzeitig aus.

Das konfigurierte Gateway der Menüleisten-App bleibt Eigentümer der Fähigkeiten des Mac-Node
und des Sprechmodus. Zusätzliche Gateway-Fenster dienen ausschließlich der Bedienung, sodass ein
zweites Gateway die globalen Mikrofon- oder Gerätesteuerungen nicht unbemerkt auf ein anderes Ziel umstellen kann.
Anhören/TTS und normale Chataktionen verwenden die eigene Gateway-Verbindung des Fensters.

## Quick-Chat-Leiste

Drücken Sie Option-Leertaste (⌥Leertaste) oder wählen Sie **Quick Chat** im Menü der Menüleiste, um einen schwebenden Eingabebereich für die Hauptsitzung zu öffnen. Ändern Sie das globale Tastenkürzel mit dem Rekorder unter **Einstellungen → Allgemein → Quick-Chat-Kürzel**.

Quick Chat zeigt den Zielagenten an (Avatar oder Emoji, wobei der Name des Agenten als Platzhalter dient) und sendet an die Hauptsitzung dieses Agenten. Nachdem Return das Senden bestätigt hat, bleibt die Leiste geöffnet und wird nach unten um die gestreamte Markdown-Antwort und das aktuelle Transkript erweitert. Das Eingabefeld der Leiste bleibt der Eingabebereich. Drücken Sie Befehlstaste-Return, um zu senden und dasselbe Ziel im vollständigen Chatfenster zu öffnen, Shift-Return für einen Zeilenumbruch oder Escape, um die gesamte Leiste einschließlich Antwortbereich zu schließen. Ein Klick außerhalb schließt sie ebenfalls. Wenn relevante macOS-Berechtigungen fehlen, bietet eine angefügte Leiste die Aktionen **Gewähren** und **Nicht jetzt** an.

Verwenden Sie die Mikrofontaste, um in den Eingabebereich zu diktieren. Vorläufige Spracherkennungsergebnisse ersetzen den diktierten Abschnitt fortlaufend, während bereits im Eingabebereich vorhandener Text erhalten bleibt. Drücken Sie zum Beenden erneut die Taste, Return oder Escape; auch beim Senden, Ausblenden oder Verlust des Fokus von Quick Chat wird das Mikrofon freigegeben. Bei der ersten Verwendung wird um Zugriff auf das macOS-Mikrofon und die Spracherkennung gebeten. Quick Chat verwendet Apple Speech und kann dessen Netzwerkdienste nutzen; nur die passive Sprachaktivierung erfordert eine Erkennung auf dem Gerät.

Die kompakte Modellsteuerung zeigt das aktuelle Modell und die Denkstufe der Zielsitzung an. Eine Modellauswahl aktualisiert diese Sitzung und bleibt daher dort bestehen, während eine Auswahl der Denkstufe nur für jede Nachricht gilt, die aus der aktuellen Quick-Chat-Darstellung gesendet wird. Lokale Auswahlen werden zurückgesetzt, wenn die Leiste ausgeblendet wird. Beim Wechseln des Agenten oder Auswählen einer kürzlich verwendeten Sitzung bleiben explizite Auswahlen erhalten, der zugrunde liegende Modellzustand der neu ausgewählten Sitzung wird jedoch neu geladen.

Klicken Sie auf die Verlaufstaste, um eine der fünf zuletzt aktualisierten Sitzungen auszuwählen oder zu **Neue Nachricht an &lt;agent&gt;** zurückzukehren. Bei Auswahl einer kürzlich verwendeten Sitzung wird an genau diese Sitzung gesendet und der Platzhalter in **Antwort in &lt;session&gt;** geändert. Beim Ausblenden von Quick Chat wird dieses temporäre Ziel auf die Hauptsitzung des ausgewählten Agenten zurückgesetzt; ein Wechsel des Agenten über das Avatar-Menü löscht es ebenfalls.

Befehlstaste-Return öffnet die Unterhaltung des Agenten, der die Sendung empfangen hat, auch wenn der Sitzungsbereich global ist.

Die Kamerataste öffnet ein Menü für **Fenster aufnehmen…** oder **Bereich aufnehmen…**. Bei der Fensteraufnahme wird jedes sichtbare Fenster beschriftet; bei der Bereichsaufnahme wird jedes Display abgedunkelt, während Sie einen Bereich aufziehen, und dessen aktuelle Größe wird angezeigt. Der ausgewählte Screenshot wird zusammen mit gegebenenfalls eingegebenem Text als Beschriftung an den ausgewählten Agenten gesendet. Bei der ersten Verwendung wird um Zugriff auf die macOS-Bildschirmaufnahme gebeten. Escape, ein Klick auf eine leere Stelle oder ein Klick ohne ausreichend großen aufgezogenen Bereich bricht den Vorgang ab.

Verwenden Sie die Dokumenttext-Taste, um Text aus dem fokussierten Fenster der fokussierten App anzuhängen. Quick Chat zeigt das Ergebnis als entfernbaren Kontext-Chip an, statt den erfassten Text in den Eingabebereich einzufügen; beim Senden wird der Text des Chips an die ausgehende Nachricht angehängt und anschließend gelöscht. Dies erfordert die macOS-Bedienungshilfen-Berechtigung. Angehängter Text wird außerdem bei jedem Schließen von Quick Chat gelöscht, sodass Kontext aus einer Darstellung nicht unbeabsichtigt in einen späteren Sendevorgang gelangen kann.

Wählen Sie nach Abschluss einer Antwort **In &lt;app&gt; einsetzen**, um den sichtbaren Assistententext ohne ausgeblendete Gedankengänge in die allgemeine Zwischenablage zu kopieren und in die zuvor im Vordergrund befindliche App einzufügen. Dies erfordert die macOS-Bedienungshilfen-Berechtigung. Die Aktion ersetzt den aktuellen Inhalt der Zwischenablage und blendet Quick Chat anschließend aus.

Deaktivieren Sie die Funktion vollständig unter **Einstellungen → Allgemein → Quick Chat**; derselbe Abschnitt enthält den Rekorder für das Tastenkürzel.

- **Lokaler Modus**: stellt eine direkte Verbindung zum lokalen Gateway-WebSocket her.
- **Remote-Modus**: verwendet die konfigurierte direkte Route `ws://`/`wss://` oder den von der App verwalteten SSH-Tunnel als Datenebene.

## Start und Fehlerbehebung

- Manuell: Lobster-Menü -> "Chat öffnen".
- Automatisches Öffnen für Tests:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --chat
  ```

  (`--webchat` wird als Legacy-Alias akzeptiert.)

- Protokolle: `./scripts/clawlog.sh` (Subsystem `ai.openclaw`, Kategorie `WebChatSwiftUI`).

## Technische Anbindung

- Datenebene: Gateway-WS-Methoden `chat.history`, `chat.message.get`, `chat.send`, `chat.abort`, `chat.inject` sowie `question.list` und `question.resolve` und die Ereignisse `chat`, `agent`, `presence`, `tick`, `health`; Fragekarten folgen den Ereignissen `question.requested` und `question.resolved` und werden nach erneuten Verbindungen über `question.list` aktualisiert.
- `chat.history` gibt ein für die Anzeige normalisiertes Transkript zurück: Eingebettete Direktiven-Tags werden aus dem sichtbaren Text entfernt, XML-Nutzdaten von Tool-Aufrufen im Klartext (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`, einschließlich abgeschnittener Blöcke) und unbeabsichtigt ausgegebene Modellsteuerungs-Tokens werden entfernt, reine Assistentenzeilen mit stillen Tokens wie exakt `NO_REPLY`/`no_reply` werden ausgelassen und übergroße Zeilen können durch einen Platzhalter für abgeschnittene Inhalte ersetzt werden.
- Sitzung: verwendet standardmäßig die oben beschriebene primäre Sitzung; in der Benutzeroberfläche kann zwischen Sitzungen gewechselt werden.
- Sitzungsgruppen: `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` und `sessions.groups.delete` verwalten den Gruppenkatalog. Die Zugehörigkeit ist das über `sessions.patch` aktualisierte Sitzungsfeld `category`.
- Ungelesen-Status: Nachdem eine Sitzung aktiviert und ihr aktueller Verlauf erfolgreich geladen wurde, löscht die App die Ungelesen-Markierung dieser Sitzung. Fehlgeschlagene Ladevorgänge des Verlaufs löschen sie nicht; ein vorübergehender Patch-Fehler wird bei der nächsten Aktivierung erneut versucht.
- Das Onboarding verwendet eine eigene Sitzung, um die Ersteinrichtung getrennt zu halten.
- Offline-Cache: Die App verwaltet pro Gateway einen kleinen schreibgeschützten Cache der letzten Chatsitzungen und Transkripte (`~/Library/Application Support/OpenClaw/chat-cache.sqlite`): Bei einem Kaltstart wird das zuletzt bekannte Transkript sofort angezeigt und aktualisiert, sobald das Gateway antwortet; kürzlich verwendete Chats bleiben auch ohne Verbindung durchsuchbar (das Senden bleibt deaktiviert, bis die Verbindung wiederhergestellt ist).

## Sicherheitsrelevante Oberfläche

- Der Remote-Modus leitet über SSH ausschließlich den WebSocket-Steuerport des Gateway weiter.

## Bekannte Einschränkungen

- Die Benutzeroberfläche ist für Chatsitzungen optimiert, nicht als vollständige Browser-Sandbox.

## Verwandte Themen

- [WebChat](/de/web/webchat)
- [macOS-App](/de/platforms/macos)
