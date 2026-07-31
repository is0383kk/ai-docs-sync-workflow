---
read_when:
    - Sitzungs-Dashboards in der Control UI verwenden oder erläutern
    - Festlegen, was Agenten auf einem Board tun dürfen und wofür eine Betreiberfreigabe erforderlich ist
summary: 'Sitzungs-Dashboards: von Agenten erstellte Widgets, Boards, Tabs und der angedockte Chat'
title: Sitzungs-Dashboards
x-i18n:
    generated_at: "2026-07-26T18:11:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3babbc859e261aa959740ea778b44fdc1a07bce8ce7628cbabcfbc5fa207a0ce
    source_path: web/dashboards.md
    workflow: 16
---

Jeder Thread in der Control UI hat zwei Ansichten: die vertraute Unterhaltung und ein
**Dashboard** – ein Raster aus Live-Widgets, das Ihr Agent für Sie erstellt. Ein Thread
ohne Widgets ist lediglich ein Chat. Sobald ein Widget angeheftet wird, erscheint im
Header ein Umschalter **Chat | Dashboard**, und das Dashboard wird zur Hauptansicht,
während Ihr Chat daneben angedockt ist.

Es muss nichts eingerichtet und keine separate App konfiguriert werden: Dashboards sind
eine Kernfunktion, gehören zum Thread, werden beim Agenten gespeichert und bleiben auch
nach `/new` und `/reset` erhalten (der Unterhaltungskontext wird gelöscht, das Board bleibt bestehen).

## Ein Dashboard per Anfrage erstellen

Bitten Sie Ihren Agenten um das, was Sie sehen möchten:

> Erstelle ein Widget namens revenue-graph: ein interaktives Balkendiagramm der
> monatlichen Umsätze. Füge die Schaltflächen „Balken“ und „Trend“ hinzu, mit denen
> zwischen den Ansichten gewechselt werden kann. Hefte es an mein Dashboard an.

Der Agent rendert das Widget zunächst inline im Chat, damit Sie es ansehen können,
bevor es irgendwohin verschoben wird. Anschließend gilt:

- **Sie heften es an**: Bewegen Sie den Mauszeiger über ein Inline-Widget und wählen Sie **An Dashboard anheften**.
- **Oder der Agent heftet es direkt an**, wenn Sie darum bitten, und aktualisiert es später
  anhand seines Namens – Widgets haben stabile Namen, sodass „Aktualisiere
  revenue-graph mit den Zahlen für Juni“ den Inhalt an Ort und Stelle ersetzt,
  während das Board unverändert bleibt.

Widgets sind eigenständige kleine Apps (HTML/JS/SVG in einer strikt isolierten Sandbox).
Schaltflächen und Ansichtsumschalter innerhalb eines Widgets funktionieren sofort – zum
Wechseln einer Diagrammansicht wird der Agent nie benötigt.

## Das Board

- **Flexibles Raster.** Ziehen Sie Widgets an ihrem Griff; alles wird automatisch
  neu angeordnet und kompakt ausgerichtet. Ändern Sie die Größe am Griff oder wählen
  Sie im Widget-Menü eine Größenvoreinstellung (klein, mittel, groß, extra groß).
  Niemand positioniert Pixel – weder Sie noch der Agent.
- **Tabs.** Ein Board kann mehrere Seiten haben – beispielsweise einen Übersichtstab
  und einen fokussierten Tab mit einem großen Widget. Jeder Tab merkt sich seine
  eigene Position des Chat-Docks.
- **Angedockter Chat.** In der Dashboard-Ansicht wird Ihre Unterhaltung links, rechts
  oder unten angedockt, lässt sich wie die Seitenleiste in der Größe ändern und kann
  vollständig ausgeblendet werden – der Agent hört Sie weiterhin, sobald Sie den
  Chat wieder einblenden.
- **Gleichwertige Agentensteuerung.** Alles, was Sie tun können, kann auch der Agent
  mit seinem Werkzeug `dashboard` tun: Widgets hinzufügen, aktualisieren, verschieben,
  in der Größe ändern und entfernen, Tabs verwalten, den sichtbaren Tab wechseln
  sowie das Chat-Dock verschieben oder ausblenden. Bitten Sie ihn: „Platziere den
  Chat links und zeige den Finanz-Tab an“, und beobachten Sie, wie es geschieht.

## Was Widgets tun dürfen

Ein Widget, das ausschließlich Inhalte rendert, benötigt keine Genehmigung – es erscheint
sofort, genau wie Inline-Chat-Widgets, und sein Netzwerkzugriff ist vollständig deaktiviert.

Widgets, die **Zugriff** benötigen, müssen diesen deklarieren, und Sie gewähren ihn einmal
pro Widget mit einem Tippen:

- **Netzwerk** (`net`): deklarierte HTTPS-Ursprünge direkt aus der Sandbox
  abrufen – beispielsweise für eine Wetterkarte, die sich selbst über eine API
  aktualisiert.
- **Gateway-Daten** (`data`): schreibgeschützte Feeds wie Sitzungen, Nutzung
  oder Cron-Status, die vom Gateway aufgelöst werden – das Widget enthält niemals
  Ihr Token.
- **Automatisierung** (`actions`): einen bestimmten Cron-Job auslösen, sodass
  eine Schaltfläche eine echte Aufgabe ausführen kann (die möglicherweise ein
  kleineres Modell verwendet), ohne Ihre Hauptunterhaltung zu aktivieren.
- **Prompt** (`prompt`): Nachrichten an Ihren Thread senden, ohne dass bei jedem
  Klick die Bestätigung erforderlich ist, die nicht genehmigte Widgets benötigen.

Aktivierte Plugins können diesen Funktionslisten eigene benannte schreibgeschützte Feeds und Aktionen hinzufügen; durch das Deaktivieren des Plugins werden diese Integrationen entfernt.

Genehmigungen sind an die exakten Widget-Bytes und die von Ihnen geprüfte Revision gebunden.
Wenn der Agent das Widget ändert und _mehr_ anfordert, als Sie genehmigt haben, erhält es
wieder den Status „ausstehend“; beim Aktualisieren von Inhalten innerhalb derselben
Berechtigungen bleibt die Genehmigung bestehen. Widget-Interaktionen, über die der Agent
informiert sein sollte (von Ihnen ausgewählte Filter oder gewechselte Ansichten), erreichen
ihn unaufdringlich als Sitzungshinweise – so bleibt er informiert, ohne unterbrochen zu
werden.

## MCP-Apps auf dem Board

Wenn auf Ihrem Gateway MCP-Server konfiguriert sind, können interaktive MCP-Apps, die im
Chat erscheinen, wie jedes andere Widget angeheftet werden. Angeheftete Apps werden auf
dem Board mit neuen Sitzungen wieder aktiv; standardmäßig dienen sie nur zur Anzeige.
Wenn dem Widget seine deklarierten Serverwerkzeuge gewährt werden, wird es vollständig
interaktiv – mit derselben Genehmigung per einmaligem Tippen, die wie bei allen anderen
Widgets an die Revision gebunden ist.

## Gut zu wissen

- Beim Zurücksetzen eines Threads mit einem Board wird eine Bestätigung angefordert,
  und das Board bleibt erhalten.
- Beim Löschen eines Threads wird auch sein Board gelöscht.
- Boards befinden sich auf Ihrem Gateway (in der Datenbank des zuständigen Agenten)
  und erscheinen auf jedem Gerät, über das Sie eine Verbindung herstellen.
- Das Sicherheitsmodell, Einzelheiten zur Speicherung und die Begründung des Designs
  finden Sie unter [Dashboard-Architektur](/de/web/dashboard-architecture), einschließlich
  der dokumentierten Kompromisse der Sandbox.
