---
read_when:
    - WebChat-Zugriff debuggen oder konfigurieren
summary: Statischer Loopback-WebChat-Host und Gateway-WS-Nutzung für die Chat-Benutzeroberfläche
title: WebChat
x-i18n:
    generated_at: "2026-07-26T18:54:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19c301af1eb1b28650849cdd90924805dd0f5189516693505d9b75f62197007f
    source_path: web/webchat.md
    workflow: 16
---

Status: Die macOS/iOS-SwiftUI-Chat-Benutzeroberfläche kommuniziert direkt mit dem Gateway-WebSocket. Kein eingebetteter Browser, kein lokaler statischer Server.

## Was es ist

- Eine native Chat-Benutzeroberfläche für das Gateway.
- Verwendet dieselben Sitzungen und Routingregeln wie andere Kanäle.
- Deterministisches Routing: Antworten werden immer an WebChat zurückgesendet.
- Der Verlauf wird stets vom Gateway abgerufen (keine lokale Dateiüberwachung). Wenn das Gateway nicht erreichbar ist, ist WebChat schreibgeschützt.

## Schnellstart

1. Starten Sie das Gateway.
2. Öffnen Sie die WebChat-Benutzeroberfläche (macOS/iOS-App) oder den Chat-Tab der Control UI.
3. Stellen Sie sicher, dass ein gültiger Gateway-Authentifizierungspfad konfiguriert ist (standardmäßig ein gemeinsames Geheimnis, auch bei Loopback).

## Funktionsweise

- Die Benutzeroberfläche stellt eine Verbindung zum Gateway-WebSocket her und verwendet die RPC-Methoden `chat.history`, `chat.send`, `chat.inject` und `chat.message.get`.
- `chat.history` ist aus Stabilitätsgründen begrenzt: Das Gateway kann lange Textfelder kürzen, umfangreiche Metadaten auslassen und übergroße Einträge durch `[chat.history omitted: message too large]` ersetzen. API-Clients können pro Anfrage `maxChars` senden, um das Standardlimit für einen Aufruf zu überschreiben.
- Wenn eine sichtbare Assistentennachricht in `chat.history` gekürzt wurde, kann die Control UI einen seitlichen Lesebereich öffnen und den vollständigen, für die Anzeige normalisierten Eintrag bei Bedarf über `chat.message.get` abrufen, ohne die standardmäßige Verlaufsnutzlast zu erhöhen. `chat.message.get` verwendet denselben Transkriptzweig und dieselben Anzeigeregeln wie `chat.history`, zielt jedoch anhand von `messageId` auf einen einzelnen Eintrag und gibt einen zutreffenden Grund für die Nichtverfügbarkeit zurück, wenn der vollständige Inhalt nicht mehr zurückgegeben werden kann.
- `chat.history` folgt bei nur erweiterbaren Sitzungsdateien dem aktiven Transkriptzweig, sodass verworfene Neuschreibzweige und ersetzte Prompt-Kopien nicht in WebChat dargestellt werden.
- Compaction-Einträge werden als Trennlinie „Komprimierter Verlauf“ dargestellt. Sie erklärt, dass das komprimierte Transkript als Prüfpunkt erhalten bleibt, und bietet eine Aktion zum Öffnen der Sitzungsprüfpunkte (Verzweigen oder Wiederherstellen, sofern die Berechtigungen dies zulassen).
- Die Control UI merkt sich das zugrunde liegende Gateway-`sessionId`, das von `chat.history` zurückgegeben wird, und schließt es in nachfolgende `chat.send`-Aufrufe ein. Dadurch setzen erneute Verbindungen und Seitenaktualisierungen dieselbe gespeicherte Unterhaltung fort, sofern der Benutzer keine Sitzung startet oder zurücksetzt.
- Beim Senden im Vordergrund wird außerdem das Blatt des angezeigten Zweigs aus dem dargestellten Verlauf als `expectedLeafEntryId` einbezogen. Falls ein anderer Client zuvor den Zweig gewechselt hat, stellt die Control UI die Nachricht zur Prüfung zurück und aktualisiert das Transkript, anstatt sie im neuen Zweig zu veröffentlichen. Wiederholungen nach einer erneuten Verbindung oder aus dem wiederhergestellten Postausgang lassen diese Vorbedingung nach dem Abgleich des aktuellen Verlaufs absichtlich aus.
- `chat.send` akzeptiert einen Idempotenzschlüssel (die Control UI verwendet die Ausführungs-ID). Das Gateway dedupliziert wiederholte Anfragen, die denselben Schlüssel erneut verwenden, sodass wiederholte oder doppelte laufende Übermittlungen für dieselbe Sitzung, Nachricht und dieselben Anhänge keine zweite Ausführung erzeugen.
- Beim Antworten auf eine bestimmte Nachricht (Rechtsklick → Reply) wird die Transkript-ID des Ziels als `replyToId` mit `chat.send` gesendet. Das Gateway löst diese Nachricht aus dem Sitzungsverlauf auf und ergänzt dieselben kanalunabhängigen Antwortkontext-Metadaten, die Discord-Antworten verwenden: Agenten sehen `has_reply_context` sowie den nicht vertrauenswürdigen Block „Antwortziel der aktuellen Benutzernachricht“ mit Absenderbezeichnung und Inhalt. (Bei WebChat-Prompts bleiben flüchtige Unterhaltungs-IDs wie `reply_to_id` gemäß der bestehenden byte-stabilen Prompt-Richtlinie für direkte WebChat-Sitzungen unterdrückt.) Antwortziele ohne persistierte Transkript-ID (beispielsweise ausstehende Sendevorgänge) verwenden ersatzweise ein Inline-Zitat im Nachrichtentext.
- Arbeitsbereich-Startdateien und ausstehende `BOOTSTRAP.md`-Anweisungen werden über den Abschnitt `# Project Context` des Agentensystem-Prompts bereitgestellt und nicht in die WebChat-Benutzernachricht kopiert. Wenn Bootstrap-Inhalte gekürzt werden, erhält der System-Prompt stattdessen einen kurzen „Hinweis zum Bootstrap-Kontext“; detaillierte Zählwerte und Konfigurationsoptionen verbleiben auf den Diagnoseoberflächen.
- Die Anzeigenormalisierung in `chat.history` entfernt Folgendes: ausschließlich zur Laufzeit verwendeten OpenClaw-Kontext, Wrapper eingehender Umschläge, Inline-Tags für Zustellungsanweisungen wie `[[reply_to_current]]`, `[[reply_to:<id>]]` und `[[audio_as_voice]]`, Nur-Text-XML-Nutzlasten von Werkzeugaufrufen (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`, einschließlich gekürzter Blöcke) sowie unbeabsichtigt ausgegebene Modellsteuerungstoken in ASCII- oder vollbreiter Schreibweise. Assistenteneinträge, deren gesamter sichtbarer Text ausschließlich aus dem Stille-Token `NO_REPLY` besteht (ohne Beachtung der Groß-/Kleinschreibung), werden ausgelassen.
- Als Reasoning gekennzeichnete Antwortnutzlasten (`isReasoning: true`) werden aus den WebChat-Assistenteninhalten, dem Text bei der Transkriptwiedergabe und den Audioinhaltsblöcken ausgeschlossen, sodass ausschließlich aus Gedankengängen bestehende Nutzlasten weder als sichtbare Assistentennachrichten noch als abspielbares Audio erscheinen.
- `chat.inject` hängt eine Assistentennotiz direkt an das Transkript an und überträgt sie an die Benutzeroberfläche (keine Agentenausführung).
- Bei abgebrochenen Ausführungen können teilweise Assistentenausgaben in der Benutzeroberfläche sichtbar bleiben. Das Gateway persistiert diesen Teiltext im Transkriptverlauf, wenn eine gepufferte Ausgabe vorhanden ist, und kennzeichnet den Eintrag mit Abbruchmetadaten.

### Transkript- und Zustellungsmodell

WebChat verfügt über zwei separate Datenpfade:

- Die SQLite-Transkriptzeilen bilden das dauerhafte Modell-/Laufzeittranskript. Bei normalen Agentenausführungen persistiert die eingebettete OpenClaw-Laufzeit die für das Modell sichtbaren Nachrichten `user`, `assistant` und `toolResult` über den Sitzungszugriff. WebChat schreibt keine beliebigen Zustellungs-, Status- oder Hilfstexte in dieses Transkript.
- Gateway-`ReplyPayload`-Ereignisse bilden die Live-Zustellungsprojektion: normalisiert für die WebChat-/Kanalanzeige, Block-Streaming, Anweisungstags, Medieneinbettung, TTS-/Audio-Kennzeichnungen und das Ausweichverhalten der Benutzeroberfläche. Sie sind nicht selbst das kanonische Sitzungsprotokoll.
- Testsysteme, die sichtbare Antworten über `tools.message` benötigen, verwenden WebChat weiterhin als interne Senke für Quellantworten der aktuellen Ausführung. Ein zielloses `message.send` aus dieser aktiven WebChat-Ausführung wird in denselben Chat projiziert und in das Sitzungstranskript gespiegelt. WebChat wird dadurch nicht zu einem wiederverwendbaren ausgehenden Kanal und übernimmt niemals `lastChannel`.
- WebChat fügt Assistenteneinträge nur dann in das Transkript ein, wenn das Gateway außerhalb eines normalen eingebetteten Agentendurchlaufs für eine angezeigte Nachricht verantwortlich ist: `chat.inject`, Antworten auf Befehle ohne Agent, abgebrochene Teilausgaben und von WebChat verwaltete Medientranskript-Ergänzungen.
- Wenn während einer Ausführung Live-Assistententext erscheint, aber nach dem Neuladen des Verlaufs verschwindet, prüfen Sie in dieser Reihenfolge: ob das SQLite-Transkript den Assistententext enthält, ob die `chat.history`-Anzeigeprojektion ihn entfernt hat und anschließend, ob die optimistische Tail-Zusammenführung der Control UI den lokalen Zustellungsstatus durch den persistierten Snapshot ersetzt hat.

Die endgültigen Antworten normaler Agentenausführungen sollten dauerhaft sein, da die eingebettete Laufzeit die Assistenten-`message_end` schreibt. Jeder Ausweichmechanismus, der eine zugestellte endgültige Nutzlast in das Transkript spiegelt, muss zunächst verhindern, dass ein bereits von der eingebetteten Laufzeit geschriebener Assistentendurchlauf dupliziert wird.

## Werkzeugbereich für Agenten in der Control UI

- Der Werkzeugbereich `/agents` der Control UI verfügt über eine Ansicht „Jetzt verfügbar“, die auf `tools.effective(sessionKey=...)` basiert: eine vom Server abgeleitete, schreibgeschützte Projektion des Werkzeuginventars der aktuellen Sitzung, einschließlich Kern-, Plugin- und kanaleigener Werkzeuge sowie bereits erkannter MCP-Serverwerkzeuge.
- Eine separate Ansicht zur Konfigurationsbearbeitung (basierend auf `tools.catalog`) umfasst Profile, agentenspezifische Überschreibungen und Katalogsemantik.
- Die Laufzeitverfügbarkeit ist sitzungsbezogen. Beim Wechsel zwischen Sitzungen desselben Agenten kann sich die Liste „Jetzt verfügbar“ ändern. Wenn konfigurierte MCP-Server seit der letzten Erkennung nicht verbunden oder geändert wurden, zeigt der Bereich einen Hinweis an, anstatt stillschweigend MCP-Transporte über den Lesepfad zu starten.
- Der Konfigurationseditor bedeutet nicht, dass zur Laufzeit Zugriff besteht; der effektive Zugriff folgt weiterhin der Richtlinienrangfolge (`allow`/`deny`, agenten- sowie provider-/kanalspezifische Überschreibungen).

## Remote-Nutzung

- Der Remote-Modus tunnelt den Gateway-WebSocket über SSH/Tailscale.
- Sie müssen keinen separaten WebChat-Server ausführen.

## Konfigurationsreferenz (WebChat)

Vollständige Konfiguration: [Konfiguration](/de/gateway/configuration)

WebChat verfügt über keinen persistierten Konfigurationsabschnitt. Das Gateway verwendet das integrierte Anzeigelimit `chat.history`; API-Clients können pro Anfrage `maxChars` senden, um es für einen einzelnen Aufruf zu überschreiben. Die Legacy-Konfigurationen `channels.webchat` und `gateway.webchat` werden nicht mehr unterstützt; führen Sie `openclaw doctor --fix` aus, um sie zu entfernen.

Zugehörige globale Optionen:

- `gateway.port`, `gateway.bind`: WebSocket-Host/-Port.
- `gateway.auth.mode`, `gateway.auth.token`, `gateway.auth.password`:
  WebSocket-Authentifizierung mit gemeinsamem Geheimnis.
- `gateway.auth.allowTailscale`: Der Chat-Tab der browserbasierten Control UI kann bei Aktivierung
  Tailscale-Serve-Identitätsheader verwenden.
- `gateway.auth.mode: "trusted-proxy"`: Reverse-Proxy-Authentifizierung für Browser-Clients hinter einer identitätsbewussten **Nicht-Loopback**-Proxyquelle (siehe [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth)).
- `gateway.remote.url`, `gateway.remote.token`, `gateway.remote.password`: entferntes Gateway-Ziel.
- `session.*`: Sitzungsspeicher und Standardwerte für den Hauptschlüssel.

## Verwandte Themen

- [Control UI](/de/web/control-ui)
- [Dashboard](/de/web/dashboard)
