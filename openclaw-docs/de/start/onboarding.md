---
read_when:
    - Entwurf des macOS-Einrichtungsassistenten
    - Implementierung der Authentifizierungs- oder Identitätseinrichtung
sidebarTitle: 'Onboarding: macOS App'
summary: Ersteinrichtungsablauf für OpenClaw (macOS-App)
title: Onboarding (macOS-App)
x-i18n:
    generated_at: "2026-07-26T18:47:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

Der Ablauf beim ersten Start der macOS-App: Wählen Sie aus, wo der Gateway ausgeführt wird, verbinden Sie ein
verifiziertes KI-Backend, erteilen Sie Berechtigungen und übergeben Sie an das
eigene Bootstrap-Ritual des Agenten.
Informationen zum CLI-Onboarding und einen Vergleich beider Wege finden Sie in der [Onboarding-Übersicht](/de/start/onboarding-overview).

<Steps>
<Step title="macOS-Warnung bestätigen">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="Suche nach lokalen Netzwerken erlauben">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="Willkommen und Sicherheitshinweis">
<Frame caption="Lesen Sie den angezeigten Sicherheitshinweis und entscheiden Sie entsprechend">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

Sicherheits- und Vertrauensmodell:

- Standardmäßig ist OpenClaw ein persönlicher Agent: eine Vertrauensgrenze für einen vertrauenswürdigen Betreiber.
- Gemeinsam genutzte Setups und Mehrbenutzer-Setups müssen abgesichert werden: Trennen Sie Vertrauensgrenzen, beschränken Sie den Werkzeugzugriff auf ein Minimum und befolgen Sie die Hinweise unter [Sicherheit](/de/gateway/security).
- Beim lokalen Onboarding wird für neue Konfigurationen standardmäßig `tools.profile: "coding"` verwendet, damit neue Setups Dateisystem- und Laufzeitwerkzeuge behalten, ohne das uneingeschränkte Profil `full` zu verwenden.
- Wenn Hooks/Webhooks oder andere Quellen nicht vertrauenswürdiger Inhalte aktiviert sind, verwenden Sie eine leistungsstarke moderne Modellklasse und behalten Sie strikte Werkzeugrichtlinien sowie Sandboxing bei.

</Step>
<Step title="Lokal oder remote">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

Wo wird der **Gateway** ausgeführt?

- **Dieser Mac (nur lokal):** Das Onboarding konfiguriert die Authentifizierung und speichert die Anmeldedaten lokal.
- **Remote (über SSH/Tailnet):** Das Onboarding konfiguriert **keine** lokale Authentifizierung;
  die Anmeldedaten müssen bereits auf dem Gateway-Host vorhanden sein. Im Feld für das Remote-Gateway-Token
  wird das Token gespeichert, mit dem die macOS-App eine Verbindung zu diesem Gateway herstellt;
  bestehende `gateway.remote.token`-SecretRef-Werte bleiben erhalten, bis Sie
  sie ersetzen.
- **Später konfigurieren:** Überspringen Sie die Einrichtung und lassen Sie die App unkonfiguriert.

<Tip>
**Tipp zur Gateway-Authentifizierung:**

- Der Gateway-Authentifizierungsmodus verwendet selbst bei Loopback-Bindungen standardmäßig `token`, sodass sich lokale WS-Clients authentifizieren müssen.
- Durch das Festlegen von `gateway.auth.mode: "none"` kann jeder lokale Prozess eine Verbindung herstellen; verwenden Sie dies nur auf vollständig vertrauenswürdigen Rechnern.
- Verwenden Sie ein Token für den Zugriff von mehreren Rechnern oder für Nicht-Loopback-Bindungen.

</Tip>
</Step>
<Step title="CLI">
  Bei der lokalen Einrichtung wird die globale `openclaw`-CLI über npm, pnpm oder bun installiert,
  wobei npm bevorzugt wird. Node bleibt die empfohlene Laufzeit für den Gateway
  selbst. Vorhandene kompatible Installationen werden wiederverwendet.
</Step>
<Step title="KI verbinden">
  Bei einem verbundenen Gateway, für den bereits ein Agentenmodell konfiguriert ist, wird diese
  Seite vollständig übersprungen und die normale Agenten-Benutzeroberfläche geöffnet. Die Einrichtung von OpenClaw und des Providers
  wird nur bei einem neuen oder unvollständig konfigurierten Gateway ausgeführt.

Sobald der Gateway bereit ist, sucht das Onboarding nach bereits vorhandenem KI-Zugriff:
einer Anmeldung bei Claude Code oder Codex, `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` oder einem
werkzeugfähigen Modell mit mindestens 16K gemessenem effektivem Kontext, das bereits
auf einem erreichbaren Ollama- oder LM-Studio-Server installiert ist. Die Erkennung erfolgt auf dem
Gateway-Host, auch wenn die macOS-App eine Verbindung zu einem Linux-Gateway herstellt. Die beste
Option wird mit einer echten Vervollständigung getestet und erst gespeichert,
nachdem sie geantwortet hat. Wenn ein Test fehlschlägt, versucht die App automatisch die nächste Option
und zeigt an, warum die vorherige fehlgeschlagen ist. Wenn mehrere Optionen gefunden werden, können Sie
vor dem Fortfahren zwischen ihnen wechseln. Bei der automatischen lokalen Erkennung wird niemals ein Modell
abgerufen oder heruntergeladen.

Um ein Claude-Abonnement zu verwenden, wenn auf dem Gateway-Host keine Claude-CLI-Anmeldung vorhanden ist, führen Sie
`claude setup-token` auf einem beliebigen Rechner mit installiertem Claude Code aus und fügen Sie anschließend das
ausgegebene Token unter **Connect with an API key or
token** als **Anthropic setup-token** ein.

Installierte CLIs von Gemini CLI, Antigravity, Pi und OpenCode werden zur Einordnung angezeigt,
wenn sie nicht als wiederverwendbare Inferenzroute für die geführte Einrichtung ausgewählt werden können.
Gemini und Antigravity können die werkzeugfreie Inferenzprüfung nicht erzwingen. Pi und
OpenCode sind vollständige Agenten-Harnesses und keine Inferenzrouten für die Einrichtung; ihre
Sitzungsintegrationen erfordern eine separate Laufzeit- und Plugin-Einrichtung.

Sie können sich auch über den eigenen OAuth- oder Gerätekopplungsablauf des Providers anmelden.
Zu den integrierten Optionen gehören OpenAI/ChatGPT, OpenRouter, GitHub Copilot, Google
Gemini CLI, xAI, MiniMax Global und CN sowie Chutes. Die Liste stammt aus den
aktiven Textinferenz-Provider-Plugins des Gateways und nicht aus einer festen App-Liste,
sodass ein anderer Provider sich beteiligen kann, ohne providerspezifischen macOS-Code hinzuzufügen.

Die manuelle Schlüssel-/Token-Auswahl verwendet dieselbe Provider-Registry. Bei jedem Weg
stellt der Provider sein Startmodell und seine Konfiguration bereit; OpenClaw überprüft
die Anmeldedaten mit demselben Live-Test, bevor das zugehörige Authentifizierungsprofil gespeichert wird. Weiter
bleibt gesperrt, bis ein Backend den Test bestanden hat, sodass der erste Agenten-Chat nicht
ohne funktionierende Inferenz gestartet werden kann. Nachdem diese Live-Prüfung bestanden wurde, steht OpenClaw
zur Verfügung, um bei der Konfiguration des verbleibenden Arbeitsbereichs, des Gateways, der Kanäle und
anderer optionaler Funktionen zu helfen. Wenn OpenClaw eine kurze Auswahlliste anbietet, zeigt die App
native Optionskarten an; durch die Auswahl einer Option wird diese übermittelt, und **Skip for
now** lässt die Auswahl immer optional. OpenClaw ist später auch unter
Settings → OpenClaw verfügbar.
</Step>
<Step title="Erinnerungen importieren (wird bei Erkennung angezeigt)">
Bei einem lokalen Gateway prüft das Onboarding den Mac auf Erinnerungen aus unterstützten KI-
Werkzeugen: automatische Erinnerungen von Claude Code, konsolidierte Erinnerungen von Codex und Hermes-Erinnerungsdateien.
Wenn welche gefunden werden, listet diese Seite jede Quelle mit der Anzahl ihrer Erinnerungen auf
und ermöglicht den Import der ausgewählten Quellen in den Arbeitsbereich des Agenten unter
`memory/imports/` für die indizierte Wiederauffindung. Bereits importierte Dateien werden übersprungen, und
die Seite wird niemals angezeigt, wenn nichts importiert werden kann. Das Überspringen ist unbedenklich; die
Seite für den Erinnerungsimport im Dashboard bietet später denselben Import mit Kontrolle
auf Dateiebene an.
</Step>
<Step title="Berechtigungen">

<Frame caption="Wählen Sie aus, welche Berechtigungen Sie OpenClaw erteilen möchten">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

Das Onboarding fordert TCC-Berechtigungen für Folgendes an: Automatisierung (AppleScript), Mitteilungen, Bedienungshilfen, Bildschirmaufnahme, Mikrofon, Spracherkennung, Kamera und Standort.

</Step>
<Step title="Abschluss">
  Nachdem die Inferenzprüfung bestanden wurde, übernimmt OpenClaw die verbleibende optionale Einrichtung und kann
  Sie an den normalen Agenten-Chat übergeben. Nach Abschluss der Berechtigungsabfolge
  wird derselbe Chat geöffnet; die App erstellt vor OpenClaw keinen Arbeitsbereich und startet keine separate
  Unterhaltung zur Agenteneinrichtung. Unter
  [Bootstrapping](/de/start/bootstrapping) erfahren Sie, was auf dem Gateway-Host
  während des ersten tatsächlichen Durchlaufs des Agenten geschieht.
</Step>
</Steps>

## Verwandte Themen

- [Onboarding-Übersicht](/de/start/onboarding-overview)
- [Erste Schritte](/de/start/getting-started)
