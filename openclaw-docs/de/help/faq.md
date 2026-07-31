---
read_when:
    - Beantwortung häufiger Supportfragen zu Einrichtung, Installation, Onboarding oder Laufzeit
    - Triage von gemeldeten Benutzerproblemen vor einer eingehenderen Fehlerbehebung
summary: Häufig gestellte Fragen zur Einrichtung, Konfiguration und Verwendung von OpenClaw
title: Häufig gestellte Fragen
x-i18n:
    generated_at: "2026-07-26T19:01:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

Schnelle Antworten und ausführlichere Fehlerbehebung für praxisnahe Setups (lokale Entwicklung, VPS, mehrere Agenten, OAuth/API-Schlüssel, Modell-Failover). Informationen zur Laufzeitdiagnose finden Sie unter [Fehlerbehebung](/de/gateway/troubleshooting). Die vollständige Konfigurationsreferenz finden Sie unter [Konfiguration](/de/gateway/configuration).

## Die ersten 60 Sekunden, wenn etwas nicht funktioniert

<Steps>
  <Step title="Schnellstatus">
    ```bash
    openclaw status
    ```
    Schnelle lokale Zusammenfassung: Betriebssystem und Update, Erreichbarkeit von Gateway/Dienst, Agenten/Sitzungen, Provider-Konfiguration und Laufzeitprobleme (wenn das Gateway erreichbar ist).
  </Step>
  <Step title="Einfügbarer Bericht (kann sicher geteilt werden)">
    ```bash
    openclaw status --all
    ```
    Schreibgeschützte Diagnose mit einem Auszug der letzten Protokolleinträge (Token unkenntlich gemacht).
  </Step>
  <Step title="Daemon- und Portstatus">
    ```bash
    openclaw gateway status
    ```
    Zeigt die Supervisor-Laufzeit im Vergleich zur RPC-Erreichbarkeit, die Ziel-URL der Prüfung und welche Konfiguration der Dienst wahrscheinlich verwendet hat.
  </Step>
  <Step title="Ausführliche Prüfungen">
    ```bash
    openclaw status --deep
    ```
    Live-Zustandsprüfung des Gateways, einschließlich Kanalprüfungen, sofern unterstützt (erfordert ein erreichbares Gateway). Siehe [Zustand](/de/gateway/health).
  </Step>
  <Step title="Aktuelles Protokoll fortlaufend anzeigen">
    ```bash
    openclaw logs --follow
    ```
    Wenn RPC nicht verfügbar ist, verwenden Sie ersatzweise:
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # Beispiel für ein benanntes Profil:
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    Dateiprotokolle sind von Dienstprotokollen getrennt; siehe [Protokollierung](/de/logging) und [Fehlerbehebung](/de/gateway/troubleshooting).
  </Step>
  <Step title="Doctor ausführen (Reparaturen)">
    ```bash
    openclaw doctor
    ```
    Repariert/migriert Konfiguration und Zustand und führt anschließend Zustandsprüfungen aus. Siehe [Doctor](/de/gateway/doctor).
  </Step>
  <Step title="Gateway-Momentaufnahme (nur WS)">
    ```bash
    openclaw health --json
    openclaw health --verbose   # zeigt bei Fehlern die Ziel-URL und den Konfigurationspfad
    ```
    Fordert vom laufenden Gateway eine vollständige Momentaufnahme an. Siehe [Zustand](/de/gateway/health).
  </Step>
</Steps>

## Schnellstart und Ersteinrichtung

Fragen und Antworten zum ersten Start – Installation, Onboarding, Authentifizierungsrouten, Abonnements, anfängliche Fehler – finden Sie in den [FAQ zum ersten Start](/de/help/faq-first-run).

## Was ist OpenClaw?

<AccordionGroup>
  <Accordion title="Was ist OpenClaw, kurz zusammengefasst?">
    OpenClaw ist ein persönlicher KI-Assistent, den Sie auf Ihren eigenen Geräten ausführen. Er antwortet über die bereits von Ihnen verwendeten Messaging-Plattformen (Discord, Google Chat, iMessage, Mattermost, Signal, Slack, Telegram, WebChat, WhatsApp und gebündelte Kanal-Plugins wie QQ Bot) und unterstützt auf kompatiblen Plattformen außerdem Sprache sowie ein Live-Canvas. Das **Gateway** ist die ständig aktive Steuerungsebene; der Assistent ist das Produkt.
  </Accordion>

  <Accordion title="Nutzenversprechen">
    OpenClaw ist nicht „nur ein Claude-Wrapper“. Es ist eine **Local-First-Steuerungsebene**, die einen leistungsfähigen Assistenten auf **Ihrer eigenen Hardware** ausführt, über die bereits von Ihnen verwendeten Chat-Apps erreichbar ist und zustandsbehaftete Sitzungen, Speicher und Tools bietet – ohne Ihre Workflows einem gehosteten SaaS zu überlassen.

    - **Ihre Geräte, Ihre Daten**: Führen Sie das Gateway an einem beliebigen Ort aus (Mac, Linux, VPS) und speichern Sie Arbeitsbereich und Sitzungsverlauf lokal.
    - **Echte Kanäle statt einer Web-Sandbox**: Discord/iMessage/Signal/Slack/Telegram/WhatsApp usw. sowie mobile Sprachfunktionen und Canvas auf unterstützten Plattformen.
    - **Modellunabhängig**: Verwenden Sie Anthropic, MiniMax, OpenAI, OpenRouter usw. mit agentenspezifischem Routing und Failover.
    - **Option für rein lokalen Betrieb**: Führen Sie lokale Modelle aus, damit alle Daten auf Ihrem Gerät verbleiben können.
    - **Routing mit mehreren Agenten**: Verwenden Sie separate Agenten pro Kanal, Konto oder Aufgabe, jeweils mit eigenem Arbeitsbereich und eigenen Standardeinstellungen.
    - **Open Source und anpassbar**: Prüfen, erweitern und selbst hosten – ohne Anbieterbindung.

    Dokumentation: [Gateway](/de/gateway), [Kanäle](/de/channels), [Mehrere Agenten](/de/concepts/multi-agent), [Speicher](/de/concepts/memory).

  </Accordion>

  <Accordion title="Ich habe es gerade eingerichtet – was sollte ich zuerst tun?">
    Gute erste Projekte: eine Website erstellen (WordPress, Shopify oder eine statische Website); den Prototyp einer mobilen App entwickeln (Konzept, Ansichten, API-Plan); Dateien und Ordner organisieren; Gmail verbinden und Zusammenfassungen oder Nachfassaktionen automatisieren.

    OpenClaw kann umfangreiche Aufgaben bewältigen, funktioniert jedoch am besten, wenn diese in Phasen aufgeteilt und Sub-Agenten für parallele Arbeiten eingesetzt werden.

  </Accordion>

  <Accordion title="Was sind die fünf wichtigsten alltäglichen Anwendungsfälle für OpenClaw?">
    - **Persönliche Briefings**: Zusammenfassungen Ihres Posteingangs, Kalenders und der für Sie relevanten Nachrichten.
    - **Recherche und Entwürfe**: schnelle Recherchen, Zusammenfassungen und erste Entwürfe für E-Mails oder Dokumente.
    - **Erinnerungen und Nachfassaktionen**: durch Cron oder Heartbeat ausgelöste Hinweise und Checklisten.
    - **Browserautomatisierung**: Formulare ausfüllen, Daten sammeln und wiederkehrende Webaufgaben erledigen.
    - **Geräteübergreifende Koordination**: Senden Sie eine Aufgabe von Ihrem Telefon, lassen Sie das Gateway sie auf einem Server ausführen und erhalten Sie das Ergebnis im Chat zurück.

  </Accordion>

  <Accordion title="Kann OpenClaw bei Leadgenerierung, Kontaktaufnahme, Anzeigen und Blogs für ein SaaS helfen?">
    Ja, bei **Recherche, Qualifizierung und Entwürfen**: Websites durchsuchen, Auswahllisten erstellen, potenzielle Kunden zusammenfassen und Entwürfe für Kontaktaufnahmen oder Anzeigentexte verfassen.

    Bei **Kontaktaufnahme oder Anzeigenkampagnen** sollte stets ein Mensch eingebunden sein. Vermeiden Sie Spam, halten Sie lokale Gesetze und Plattformrichtlinien ein und prüfen Sie alles vor dem Versand. Lassen Sie OpenClaw den Entwurf erstellen; Sie genehmigen ihn.

    Dokumentation: [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Welche Vorteile bietet OpenClaw gegenüber Claude Code bei der Webentwicklung?">
    OpenClaw ist ein **persönlicher Assistent** und eine Koordinationsschicht, kein Ersatz für eine IDE. Verwenden Sie Claude Code oder Codex für den schnellsten direkten Programmierzyklus innerhalb eines Repositorys. Verwenden Sie OpenClaw für dauerhaften Speicher, geräteübergreifenden Zugriff und Tool-Orchestrierung.

    - Dauerhafter Speicher und Arbeitsbereich über Sitzungen hinweg.
    - Plattformübergreifender Zugriff (Telegram, WhatsApp, TUI, WebChat).
    - Tool-Orchestrierung (Browser, Dateien, Zeitplanung, Hooks).
    - Ständig aktives Gateway (auf einem VPS ausführen und von überall interagieren).
    - Nodes für lokale Browser-, Bildschirm-, Kamera- und Ausführungsfunktionen.

    Beispiele: [https://openclaw.ai/showcase](https://openclaw.ai/showcase).

  </Accordion>
</AccordionGroup>

## Skills und Automatisierung

<AccordionGroup>
  <Accordion title="Wie passe ich Skills an, ohne Änderungen im Repository zu hinterlassen?">
    Verwenden Sie verwaltete Überschreibungen, anstatt die Repository-Kopie zu bearbeiten. Legen Sie Änderungen in `~/.openclaw/skills/<name>/SKILL.md` ab (oder fügen Sie über `skills.load.extraDirs` in `~/.openclaw/openclaw.json` einen Ordner hinzu). Rangfolge: `<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> gebündelt -> `skills.load.extraDirs`. Dadurch haben verwaltete Überschreibungen Vorrang vor gebündelten Skills, ohne Git zu verändern. Um Skills global zu installieren, ihre Sichtbarkeit jedoch auf bestimmte Agenten zu beschränken, behalten Sie die gemeinsam genutzte Kopie in `~/.openclaw/skills` und steuern Sie die Sichtbarkeit mit `agents.defaults.skills` / `agents.entries.*.skills`. Nur Änderungen, die sich für die Übernahme in das Upstream-Projekt eignen, sollten als PRs für die Repository-Kopie eingereicht werden.
  </Accordion>

  <Accordion title="Kann ich Skills aus einem benutzerdefinierten Ordner laden?">
    Ja: Fügen Sie Verzeichnisse über `skills.load.extraDirs` in `~/.openclaw/openclaw.json` hinzu (niedrigste Priorität in der oben genannten Reihenfolge). `clawhub` installiert standardmäßig in `./skills`, das OpenClaw in der nächsten Sitzung als `<workspace>/skills` behandelt. Um die Sichtbarkeit auf bestimmte Agenten zu beschränken, kombinieren Sie dies mit `agents.defaults.skills` oder `agents.entries.*.skills`.
  </Accordion>

  <Accordion title="Wie kann ich für verschiedene Aufgaben unterschiedliche Modelle oder Einstellungen verwenden?">
    Unterstützte Muster:

    - **Cron-Aufträge**: Isolierte Aufträge können pro Auftrag eine `model`-Überschreibung festlegen.
    - **Agenten**: Leiten Sie Aufgaben an separate Agenten mit unterschiedlichen Standardmodellen, Denkstufen und Streaming-Parametern weiter.
    - **Wechsel bei Bedarf**: `/model` wechselt jederzeit das Modell der aktuellen Sitzung.

    Beispiel – dasselbe Modell mit unterschiedlichen agentenspezifischen Einstellungen:

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    Legen Sie gemeinsame modellspezifische Standardeinstellungen in `agents.defaults.models["provider/model"].params` und anschließend agentenspezifische Überschreibungen im flachen `agents.entries.*.params` ab. Duplizieren Sie dasselbe Modell nicht unter dem verschachtelten `agents.entries.*.models["provider/model"].params`; dieser Pfad ist für agentenspezifische Modellkatalog- und Laufzeitüberschreibungen vorgesehen.

    Siehe [Cron-Aufträge](/de/automation/cron-jobs), [Routing mit mehreren Agenten](/de/concepts/multi-agent), [Konfiguration](/de/gateway/config-agents), [Slash-Befehle](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Der Bot reagiert bei aufwendigen Arbeiten nicht mehr. Wie kann ich diese auslagern?">
    Verwenden Sie **Sub-Agenten** für lang laufende oder parallele Aufgaben: Sie werden in einer eigenen Sitzung ausgeführt, geben eine Zusammenfassung zurück und sorgen dafür, dass Ihr Hauptchat reaktionsfähig bleibt. Bitten Sie den Bot, „für diese Aufgabe einen Sub-Agenten zu starten“, oder verwenden Sie `/subagents`. Mit `/status` können Sie prüfen, ob das Gateway derzeit ausgelastet ist.

    Sowohl lange Aufgaben als auch Sub-Agenten verbrauchen Token; legen Sie über `agents.defaults.subagents.model` ein günstigeres Modell für Sub-Agenten fest, wenn die Kosten relevant sind.

    Dokumentation: [Sub-Agenten](/de/tools/subagents), [Hintergrundaufgaben](/de/automation/tasks).

  </Accordion>

  <Accordion title="Wie funktionieren an Threads gebundene Sub-Agenten-Sitzungen auf Discord?">
    Binden Sie einen Discord-Thread an einen Sub-Agenten oder ein Sitzungsziel, damit nachfolgende Nachrichten dort in dieser gebundenen Sitzung verbleiben.

    - Starten Sie mit `sessions_spawn` und verwenden Sie dabei `thread: true` (optional `mode: "session"` für dauerhafte Nachfassaktionen).
    - Alternativ können Sie die Bindung manuell mit `/focus <target>` vornehmen.
    - `/agents` zeigt den Bindungsstatus an.
    - `/session idle <duration|off>` und `/session max-age <duration|off>` steuern die automatische Aufhebung des Fokus.
    - `/unfocus` löst den Thread.

    Konfiguration: `session.threadBindings.enabled` (globaler Schalter), `session.threadBindings.idleHours` (Standard `24`, `0` deaktiviert), `session.threadBindings.maxAgeHours` (Standard `0` = keine feste Obergrenze) und `session.threadBindings.spawnSessions` für die automatische Bindung beim Starten (Standard `true`).

    Dokumentation: [Sub-Agenten](/de/tools/subagents), [Discord](/de/channels/discord), [Konfigurationsreferenz](/de/gateway/configuration-reference), [Slash-Befehle](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Ein Sub-Agent wurde abgeschlossen, aber die Abschlussmeldung wurde am falschen Ort oder gar nicht veröffentlicht. Was sollte ich prüfen?">
    Prüfen Sie die aufgelöste Route des Anforderers:

    - Bei der Zustellung im Abschlussmodus bevorzugen Sub-Agenten einen gebundenen Thread oder eine gebundene Konversationsroute, sofern vorhanden.
    - Wenn der Abschlussursprung nur einen Kanal enthält, greift OpenClaw auf die gespeicherte Route der Anforderersitzung zurück (`lastChannel` / `lastTo` / `lastAccountId`), sodass die direkte Zustellung weiterhin erfolgreich sein kann.
    - Wenn weder eine gebundene Route noch eine nutzbare gespeicherte Route vorhanden ist, kann die direkte Zustellung fehlschlagen; das Ergebnis wird dann in die Warteschlange für die Sitzungszustellung gestellt, statt sofort veröffentlicht zu werden.
    - Ungültige oder veraltete Ziele können ebenfalls einen Rückfall auf die Warteschlange oder ein endgültiges Fehlschlagen der Zustellung erzwingen.
    - Wenn die letzte sichtbare Assistentenantwort des untergeordneten Agenten exakt `NO_REPLY` / `no_reply` oder `ANNOUNCE_SKIP` lautet, unterdrückt OpenClaw die Ankündigung absichtlich, statt veraltete frühere Fortschrittsmeldungen zu veröffentlichen.

    Fehlerbehebung: `openclaw tasks show <lookup>`, wobei `<lookup>` eine Aufgaben-ID, Ausführungs-ID oder ein Sitzungsschlüssel ist.

    Dokumentation: [Sub-Agenten](/de/tools/subagents), [Hintergrundaufgaben](/de/automation/tasks), [Sitzungstools](/de/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron oder Erinnerungen werden nicht ausgelöst. Was sollte ich prüfen?">
    Cron wird innerhalb des Gateway-Prozesses ausgeführt und löst nichts aus, wenn das Gateway nicht kontinuierlich läuft.

    - Bestätigen Sie, dass Cron aktiviert ist (`cron.enabled`) und `OPENCLAW_SKIP_CRON` nicht gesetzt ist.
    - Bestätigen Sie, dass der Gateway 24/7 ausgeführt wird (kein Ruhezustand/keine Neustarts).
    - Überprüfen Sie die Zeitzone des Jobs (`--tz` gegenüber der Zeitzone des Hosts).

    Debugging:
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Dokumentation: [Cron-Jobs](/de/automation/cron-jobs), [Automatisierung](/de/automation).

  </Accordion>

  <Accordion title="Cron wurde ausgelöst, aber es wurde nichts an den Kanal gesendet. Warum?">
    Überprüfen Sie den Zustellungsmodus:

    - `--no-deliver` / `delivery.mode: "none"`: Es wird kein ersatzweises Senden durch den Runner erwartet.
    - Fehlendes oder ungültiges Ankündigungsziel (`channel` / `to`): Der Runner hat die ausgehende Zustellung übersprungen.
    - Fehler bei der Kanalauthentifizierung (`unauthorized`, `Forbidden`): Der Runner hat die Zustellung versucht, sie wurde jedoch durch die Anmeldedaten verhindert.
    - Ein stilles isoliertes Ergebnis (nur `NO_REPLY` / `no_reply`) gilt als absichtlich nicht zustellbar, daher wird auch die in die Warteschlange eingereihte Ersatzzustellung unterdrückt.

    Bei isolierten Cron-Jobs kann der Agent weiterhin direkt mit dem Tool `message` senden, wenn eine Chatroute verfügbar ist. `--announce` steuert nur die Ersatzzustellung durch den Runner für abschließenden Text, den der Agent nicht bereits selbst gesendet hat.

    Debugging:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    Dokumentation: [Cron-Jobs](/de/automation/cron-jobs), [Hintergrundaufgaben](/de/automation/tasks).

  </Accordion>

  <Accordion title="Warum hat ein isolierter Cron-Lauf das Modell gewechselt oder es einmal erneut versucht?">
    Dies ist der aktive Modellwechselpfad und keine doppelte Planung. Ein isolierter Cron-Lauf speichert eine Übergabe des Laufzeitmodells dauerhaft und versucht es erneut, wenn der aktive Lauf `LiveSessionModelSwitchError` auslöst. Dabei werden der gewechselte Provider/das gewechselte Modell (und eine etwaige gewechselte Überschreibung des Authentifizierungsprofils) vor dem erneuten Versuch beibehalten.

    Rangfolge der Modellauswahl: zuerst die Modellüberschreibung des Gmail-Hooks (`hooks.gmail.model`), dann `model` pro Job, anschließend eine gespeicherte Modellüberschreibung der Cron-Sitzung und danach die normale Agenten-/Standardmodellauswahl.

    Die Wiederholungsschleife ist auf den ersten Versuch plus 2 Wechselwiederholungen begrenzt; anschließend bricht Cron ab, statt endlos weiterzulaufen.

    Debugging:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    Dokumentation: [Cron-Jobs](/de/automation/cron-jobs), [Cron-CLI](/de/cli/cron).

  </Accordion>

  <Accordion title="Wie installiere ich Skills unter Linux?">
    Verwenden Sie native `openclaw skills`-Befehle oder legen Sie Skills in Ihrem Arbeitsbereich ab; die macOS-Benutzeroberfläche für Skills ist unter Linux nicht verfügbar. Durchsuchen Sie Skills unter [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    Native `openclaw skills install` schreibt standardmäßig in das Verzeichnis `skills/` des aktiven Arbeitsbereichs. Fügen Sie `--global` hinzu, um im gemeinsamen verwalteten Skills-Verzeichnis für alle lokalen Agenten zu installieren. Installieren Sie die separate `clawhub`-CLI nur, um eigene Skills zu veröffentlichen oder zu synchronisieren. Verwenden Sie `agents.defaults.skills` oder `agents.entries.*.skills`, um einzuschränken, welche Agenten gemeinsame Skills sehen.

  </Accordion>

  <Accordion title="Kann OpenClaw Aufgaben nach einem Zeitplan oder kontinuierlich im Hintergrund ausführen?">
    Ja, über den Gateway-Scheduler:

    - **Cron-Jobs** für geplante oder wiederkehrende Aufgaben (bleiben über Neustarts hinweg bestehen).
    - **Heartbeat** für regelmäßige Prüfungen der Hauptsitzung.
    - **Isolierte Jobs** für autonome Agenten, die Zusammenfassungen veröffentlichen oder an Chats zustellen.

    Dokumentation: [Cron-Jobs](/de/automation/cron-jobs), [Automatisierung](/de/automation), [Heartbeat](/de/gateway/heartbeat).

  </Accordion>

  <Accordion title="Kann ich ausschließlich für Apple macOS vorgesehene Skills unter Linux ausführen?">
    Nicht direkt. macOS-Skills werden durch `metadata.openclaw.os` und erforderliche Binärdateien eingeschränkt und nur geladen, wenn sie auf dem **Gateway-Host** zulässig sind. Unter Linux werden ausschließlich für `darwin` vorgesehene Skills (`apple-notes`, `apple-reminders`, `things-mac`) nur geladen, wenn Sie diese Einschränkung überschreiben.

    Drei unterstützte Vorgehensweisen:

    **Option A – Gateway auf einem Mac ausführen (am einfachsten)**. Führen Sie den Gateway dort aus, wo die macOS-Binärdateien vorhanden sind, und stellen Sie anschließend unter Linux im [Remote-Modus](#gateway-ports-already-running-and-remote-mode) oder über Tailscale eine Verbindung her. Skills werden normal geladen, da der Gateway-Host macOS verwendet.

    **Option B – eine macOS-Node verwenden (ohne SSH)**. Führen Sie den Gateway unter Linux aus, koppeln Sie eine macOS-Node (Menüleisten-App) und setzen Sie **Node Run Commands** auf dem Mac auf "Always Ask" oder "Always Allow". OpenClaw betrachtet ausschließlich für macOS vorgesehene Skills als zulässig, wenn die erforderlichen Binärdateien auf der Node vorhanden sind; der Agent führt sie über das Tool `nodes` aus. Wenn "Always Ask" ausgewählt ist, wird der Befehl durch die Bestätigung von "Always Allow" in der Eingabeaufforderung zur Zulassungsliste hinzugefügt.

    **Option C – macOS-Binärdateien über SSH weiterleiten (fortgeschritten)**. Belassen Sie den Gateway unter Linux, sorgen Sie jedoch dafür, dass die erforderlichen CLI-Binärdateien in SSH-Wrapper aufgelöst werden, die auf einem Mac ausgeführt werden. Überschreiben Sie anschließend den Skill, um Linux zuzulassen, sodass er weiterhin verwendet werden kann.

    1. Erstellen Sie einen SSH-Wrapper für die Binärdatei (Beispiel: `memo` für Apple Notes):
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. Legen Sie den Wrapper auf dem Linux-Host in `PATH` ab (zum Beispiel `~/bin/memo`).
    3. Überschreiben Sie die Skill-Metadaten (im Arbeitsbereich oder in `~/.openclaw/skills`), um Linux zuzulassen:
       ```markdown
       ---
       name: apple-notes
       description: Apple Notes über die memo-CLI unter macOS verwalten.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. Starten Sie eine neue Sitzung, damit der Skills-Snapshot aktualisiert wird.

  </Accordion>

  <Accordion title="Gibt es eine Integration für Notion oder HeyGen?">
    Derzeit nicht integriert. Optionen:

    - **Benutzerdefinierter Skill / benutzerdefiniertes Plugin**: am besten für zuverlässigen API-Zugriff (beide verfügen über APIs).
    - **Browserautomatisierung**: funktioniert ohne Code, ist jedoch langsamer und fehleranfälliger.

    Für einen agenturähnlichen Kontext pro Kunde: Verwenden Sie eine Notion-Seite pro Kunde (Kontext + Präferenzen + aktive Arbeiten) und weisen Sie den Agenten an, diese Seite zu Beginn einer Sitzung abzurufen.

    Für eine native Integration können Sie eine Funktionsanfrage stellen oder einen Skill für diese APIs erstellen.

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    Native Installationen werden im Verzeichnis `skills/` des aktiven Arbeitsbereichs abgelegt; verwenden Sie `--global` für alle lokalen Agenten oder konfigurieren Sie `agents.defaults.skills` / `agents.entries.*.skills`, um die Sichtbarkeit einzuschränken. Einige Skills erwarten über Homebrew installierte Binärdateien; unter Linux bedeutet dies Linuxbrew.

    Siehe [Skills](/de/tools/skills), [Skills-Konfiguration](/de/tools/skills-config), [ClawHub](/de/clawhub).

  </Accordion>

  <Accordion title="Wie verwende ich mein bereits angemeldetes Chrome mit OpenClaw?">
    Verwenden Sie das integrierte Browserprofil `user`, das über Chrome DevTools MCP eine Verbindung herstellt:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Erstellen Sie für einen benutzerdefinierten Namen ein ausdrückliches MCP-Profil:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Hierbei kann der lokale Browser des Hosts oder eine verbundene Browser-Node verwendet werden. Wenn der Gateway an einem anderen Ort ausgeführt wird, führen Sie einen Node-Host auf dem Browsercomputer aus oder verwenden Sie stattdessen Remote-CDP.

    Aktuelle Einschränkungen der Profile `existing-session` / `user` gegenüber dem verwalteten Profil `openclaw`:

    - `click`, `type`, `hover`, `scrollIntoView`, `drag` und `select` erfordern Snapshot-Referenzen und keine CSS-Selektoren.
    - Upload-Hooks erfordern `ref` oder `inputRef`, jeweils eine Datei; CSS-`element` wird nicht unterstützt.
    - `responsebody`, PDF-Export, Abfangen von Downloads und Stapelaktionen erfordern weiterhin den verwalteten Browserpfad.

    Den vollständigen Vergleich finden Sie unter [Browser](/de/tools/browser#existing-session-via-chrome-devtools-mcp).

  </Accordion>
</AccordionGroup>

## Sandboxing und Speicher

<AccordionGroup>
  <Accordion title="Gibt es eine eigene Dokumentation zum Sandboxing?">
    Ja: [Sandboxing](/de/gateway/sandboxing). Informationen zur Docker-spezifischen Einrichtung (vollständiger Gateway in Docker oder Sandbox-Images) finden Sie unter [Docker](/de/install/docker).
  </Accordion>

  <Accordion title="Docker wirkt eingeschränkt – wie aktiviere ich den vollständigen Funktionsumfang?">
    Das Standard-Image stellt Sicherheit an erste Stelle und wird als Benutzer `node` ausgeführt. Daher enthält es keine Systempakete, kein Homebrew und keine mitgelieferten Browser. Für eine umfassendere Einrichtung:

    - Speichern Sie `/home/node` mit `OPENCLAW_HOME_VOLUME` dauerhaft, damit Caches erhalten bleiben.
    - Integrieren Sie Systemabhängigkeiten mit `OPENCLAW_IMAGE_APT_PACKAGES` in das Image.
    - Installieren Sie Playwright-Browser über die mitgelieferte CLI: `node /app/node_modules/playwright-core/cli.js install chromium`.
    - Legen Sie `PLAYWRIGHT_BROWSERS_PATH` fest und speichern Sie diesen Pfad dauerhaft.

    Dokumentation: [Docker](/de/install/docker), [Browser](/de/tools/browser).

  </Accordion>

  <Accordion title="Kann ich Direktnachrichten persönlich halten, Gruppen jedoch mit einem Agenten öffentlich und in einer Sandbox ausführen?">
    Ja, wenn der private Datenverkehr aus **Direktnachrichten** und der öffentliche Datenverkehr aus **Gruppen** besteht. Legen Sie `agents.defaults.sandbox.mode: "non-main"` fest, damit Gruppen-/Kanalsitzungen (Nicht-Hauptschlüssel) im konfigurierten Sandbox-Backend ausgeführt werden, während die Hauptsitzung für Direktnachrichten auf dem Host verbleibt. Docker ist das Standard-Backend, sobald Sandboxing aktiviert ist. Schränken Sie die in Sandbox-Sitzungen verfügbaren Tools über `tools.sandbox.tools` ein.

    Einrichtungsanleitung: [Gruppen: persönliche Direktnachrichten + öffentliche Gruppen](/de/channels/groups#pattern-personal-dms-public-groups-single-agent). Wichtige Referenz: [Gateway-Konfiguration](/de/gateway/config-agents#agentsdefaultssandbox).

  </Accordion>

  <Accordion title="Wie binde ich einen Hostordner in die Sandbox ein?">
    Setzen Sie `agents.defaults.sandbox.docker.binds` auf `["host:container:mode"]` (zum Beispiel `"/home/user/src:/src:ro"`). Globale und agentenspezifische Einbindungen werden zusammengeführt; agentenspezifische Einbindungen werden ignoriert, wenn `scope: "shared"`. Verwenden Sie `:ro` für vertrauliche Inhalte; Einbindungen umgehen die Dateisystembegrenzungen der Sandbox.

    OpenClaw validiert Einbindungsquellen sowohl anhand des normalisierten Pfads als auch anhand des kanonischen Pfads, der über den tiefsten vorhandenen Vorgänger aufgelöst wurde. Dadurch schlagen Ausbruchsversuche über übergeordnete Symlinks sicher fehl, selbst wenn das letzte Pfadsegment noch nicht vorhanden ist.

    Siehe [Sandboxing](/de/gateway/sandboxing#custom-bind-mounts) und [Sandbox im Vergleich zu Tool-Richtlinie und Elevated](/de/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check).

  </Accordion>

  <Accordion title="Wie funktioniert der Speicher?">
    Der OpenClaw-Speicher besteht aus Markdown-Dateien im Arbeitsbereich des Agenten: tägliche Notizen in `memory/YYYY-MM-DD.md`, kuratierte Langzeitnotizen in `MEMORY.md` (nur Haupt-/private Sitzungen).

    OpenClaw führt außerdem vor der Compaction, die die Unterhaltung zusammenfasst, eine stille **Speicherleerung vor der Compaction** aus und erinnert das Modell daran, zunächst dauerhafte Notizen zu schreiben. Sie wird nur ausgeführt, wenn der Arbeitsbereich beschreibbar ist (schreibgeschützte Sandboxes überspringen sie); mit `agents.defaults.compaction.memoryFlush.enabled: false` können Sie sie deaktivieren. Siehe [Speicher](/de/concepts/memory).

  </Accordion>

  <Accordion title="Der Speicher vergisst ständig Dinge. Wie sorge ich dafür, dass sie erhalten bleiben?">
    Weisen Sie den Bot an, **die Tatsache in den Speicher zu schreiben**: Langzeitnotizen werden in `MEMORY.md`, kurzfristiger Kontext in `memory/YYYY-MM-DD.md` gespeichert. Das Modell daran zu erinnern, Erinnerungen zu speichern, behebt das Problem normalerweise. Falls es weiterhin Dinge vergisst, überprüfen Sie, ob der Gateway bei jedem Lauf denselben Arbeitsbereich verwendet.

    Dokumentation: [Speicher](/de/concepts/memory), [Agentenarbeitsbereich](/de/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Bleibt der Speicher für immer bestehen? Welche Grenzen gibt es?">
    Speicherdateien liegen auf der Festplatte und bleiben erhalten, bis sie gelöscht werden; die Grenze bildet Ihr Speicherplatz, nicht das Modell. Der **Sitzungskontext** ist weiterhin durch das Kontextfenster des Modells begrenzt, sodass lange Unterhaltungen komprimiert oder gekürzt werden können – deshalb gibt es die Speichersuche, die nur die relevanten Teile wieder in den Kontext lädt.

    Dokumentation: [Speicher](/de/concepts/memory), [Kontext](/de/concepts/context).

  </Accordion>

  <Accordion title="Erfordert die semantische Speichersuche einen OpenAI-API-Schlüssel?">
    Nur wenn Sie **OpenAI-Embeddings** verwenden, den standardmäßigen Provider. Codex OAuth deckt Chat/Vervollständigungen ab und gewährt **keinen** Zugriff auf Embeddings. Daher aktiviert die Anmeldung mit Codex (OAuth oder die Anmeldung über die Codex CLI) nicht die semantische Speichersuche. OpenAI-Embeddings benötigen weiterhin einen echten API-Schlüssel (`OPENAI_API_KEY` oder `models.providers.openai.apiKey`).

    Für einen rein lokalen Betrieb setzen Sie `memory.search.provider: "local"` (GGUF/llama.cpp). Weitere unterstützte Provider: Bedrock, DeepInfra, Gemini (`GEMINI_API_KEY` oder `memory.search.remote.apiKey`), GitHub Copilot, LM Studio, Mistral, Ollama, OpenAI-kompatible Provider und Voyage. Einzelheiten zur Einrichtung finden Sie unter [Speicher](/de/concepts/memory) und [Speichersuche](/de/concepts/memory-search).

  </Accordion>
</AccordionGroup>

## Speicherorte auf der Festplatte

<AccordionGroup>
  <Accordion title="Werden alle mit OpenClaw verwendeten Daten lokal gespeichert?">
    Nein: **Der eigene Zustand von OpenClaw ist lokal**, aber **externe Dienste sehen weiterhin, was Sie ihnen senden**.

    - **Standardmäßig lokal**: Sitzungen, Speicherdateien, Konfiguration und Arbeitsbereich befinden sich auf dem Gateway-Host (`~/.openclaw` sowie Ihr Arbeitsbereichsverzeichnis).
    - **Notwendigerweise extern**: An Modell-Provider (Anthropic/OpenAI/usw.) gesendete Nachrichten werden an deren APIs übermittelt, und Chatplattformen (Slack/Telegram/WhatsApp/usw.) speichern Nachrichtendaten auf ihren Servern.
    - **Sie bestimmen den Umfang**: Lokale Modelle behalten Prompts auf Ihrem Rechner, der Kanalverkehr läuft jedoch weiterhin über die Server des jeweiligen Kanals.

    Siehe auch: [Agent-Arbeitsbereich](/de/concepts/agent-workspace), [Speicher](/de/concepts/memory).

  </Accordion>

  <Accordion title="Wo speichert OpenClaw seine Daten?">
    Alles befindet sich unter `$OPENCLAW_STATE_DIR` (Standard: `~/.openclaw`):

    | Pfad                                                               | Zweck                                                            |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | Hauptkonfiguration (JSON5)                                                 |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | Veralteter OAuth-Import (wird bei der ersten Verwendung in Authentifizierungsprofile kopiert)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | Authentifizierungsprofile (OAuth, API-Schlüssel, optional `keyRef`/`tokenRef`)        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | Optionale dateibasierte geheime Nutzlast für `file`-SecretRef-Provider   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | Veraltete Kompatibilitätsdatei (statische `api_key`-Einträge entfernt)        |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | Provider-Zustand (zum Beispiel `whatsapp/<accountId>/creds.json`)      |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | Agent-spezifischer Zustand (agentDir sowie veraltete/archivierte Sitzungsartefakte)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | Agent-spezifischer SQLite-Zustand einschließlich Sitzungszeilen und Transkripten      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | Quellen für die Migration veralteter Sitzungen sowie Archiv-/Supportartefakte      |

    Der veraltete Einzel-Agent-Pfad `~/.openclaw/agent/*` wird durch `openclaw doctor` migriert.

    Ihr **Arbeitsbereich** (AGENTS.md, Speicherdateien, Skills usw.) ist davon getrennt und wird über `agents.defaults.workspace` konfiguriert (Standard: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Wo sollten sich AGENTS.md / SOUL.md / USER.md / MEMORY.md befinden?">
    Diese Dateien befinden sich im **Agent-Arbeitsbereich**, nicht unter `~/.openclaw`.

    - **Arbeitsbereich (pro Agent)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `memory/YYYY-MM-DD.md`, optional `HEARTBEAT.md`. Die kleingeschriebene Stammdatei `memory.md` dient nur als Eingabe für die Reparatur veralteter Daten; `openclaw doctor --fix` kann sie mit `MEMORY.md` zusammenführen, wenn beide vorhanden sind.
    - **Zustandsverzeichnis (`~/.openclaw`)**: Konfiguration, Kanal-/Provider-Zustand, Authentifizierungsprofile, Sitzungen, Protokolle und gemeinsam genutzte Skills (`~/.openclaw/skills`).

    Der standardmäßige Arbeitsbereich ist `~/.openclaw/workspace` und kann konfiguriert werden:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Wenn der Bot nach einem Neustart etwas „vergisst“, stellen Sie sicher, dass das Gateway bei jedem Start denselben Arbeitsbereich verwendet (im Remote-Modus wird der Arbeitsbereich des **Gateway-Hosts** verwendet, nicht der Ihres lokalen Laptops).

    Tipp: Bitten Sie den Bot bei dauerhaftem Verhalten oder dauerhaften Präferenzen, diese **in AGENTS.md oder MEMORY.md zu schreiben**, statt sich auf den Chatverlauf zu verlassen.

    Siehe [Agent-Arbeitsbereich](/de/concepts/agent-workspace) und [Speicher](/de/concepts/memory).

  </Accordion>

  <Accordion title="Kann ich SOUL.md vergrößern?">
    Ja. `SOUL.md` ist eine der Bootstrap-Dateien des Arbeitsbereichs, die in den Agent-Kontext eingefügt werden. Die standardmäßige Einfügungsgrenze pro Datei beträgt `20000` Zeichen; das gesamte Bootstrap-Budget über alle Dateien hinweg beträgt `60000` Zeichen.

    Ändern Sie die gemeinsam genutzten Standardwerte:

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    Alternativ können Sie einen einzelnen Agent unter `agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars` abweichend konfigurieren.

    Verwenden Sie `/context`, um die Rohgrößen und eingefügten Größen zu prüfen und festzustellen, ob eine Kürzung erfolgt ist. Beschränken Sie `SOUL.md` auf Stimme, Haltung und Persönlichkeit; legen Sie Betriebsregeln in `AGENTS.md` und dauerhafte Fakten im Speicher ab.

    Siehe [Kontext](/de/concepts/context) und [Agent-Konfiguration](/de/gateway/config-agents).

  </Accordion>

  <Accordion title="Empfohlene Sicherungsstrategie">
    Legen Sie Ihren **Agent-Arbeitsbereich** in einem **privaten** Git-Repository ab und sichern Sie ihn an einem privaten Ort (zum Beispiel in einem privaten GitHub-Repository). Dadurch werden der Speicher sowie die AGENTS-/SOUL-/USER-Dateien erfasst, und Sie können den „Geist“ des Assistenten später wiederherstellen.

    Committen Sie **nichts** unter `~/.openclaw` (Anmeldedaten, Sitzungen, Tokens, verschlüsselte geheime Nutzlasten). Sichern Sie für eine vollständige Wiederherstellung den Arbeitsbereich und das Zustandsverzeichnis separat.

    Dokumentation: [Agent-Arbeitsbereich](/de/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Wie deinstalliere ich OpenClaw vollständig?">
    Siehe [Deinstallation](/de/install/uninstall).
  </Accordion>

  <Accordion title="Können Agents außerhalb des Arbeitsbereichs arbeiten?">
    Ja. Der Arbeitsbereich ist das **standardmäßige aktuelle Arbeitsverzeichnis** und der Speicheranker, keine feste Sandbox. Relative Pfade werden innerhalb des Arbeitsbereichs aufgelöst; absolute Pfade können auf andere Orte des Hosts zugreifen, sofern Sandboxing nicht aktiviert ist. Verwenden Sie zur Isolation [`agents.defaults.sandbox`](/de/gateway/sandboxing) oder Agent-spezifische Sandbox-Einstellungen. Um ein Repository zum standardmäßigen Arbeitsverzeichnis zu machen, richten Sie `workspace` dieses Agents auf das Stammverzeichnis des Repositorys – das OpenClaw-Repository selbst ist lediglich Quellcode. Halten Sie den Arbeitsbereich daher getrennt, sofern der Agent nicht absichtlich darin arbeiten soll.

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Remote-Modus: Wo befindet sich der Sitzungsspeicher?">
    Der Sitzungszustand gehört dem **Gateway-Host**. Im Remote-Modus befindet sich der für Sie relevante Sitzungsspeicher auf dem entfernten Rechner, nicht auf Ihrem lokalen Laptop. Siehe [Sitzungsverwaltung](/de/concepts/session).
  </Accordion>
</AccordionGroup>

## Grundlagen der Konfiguration

<AccordionGroup>
  <Accordion title="Welches Format hat die Konfiguration? Wo befindet sie sich?">
    OpenClaw liest eine optionale **JSON5**-Konfiguration aus `$OPENCLAW_CONFIG_PATH` (Standard: `~/.openclaw/openclaw.json`). Fehlt die Datei, werden einigermaßen sichere Standardwerte verwendet, darunter der Standardarbeitsbereich `~/.openclaw/workspace`.
  </Accordion>

  <Accordion title='Ich habe gateway.bind auf "lan" (oder "tailnet") gesetzt, und jetzt lauscht nichts mehr / die Benutzeroberfläche meldet eine fehlende Autorisierung'>
    Bindungen außerhalb der Loopback-Schnittstelle **erfordern einen gültigen Gateway-Authentifizierungspfad**: Authentifizierung mit einem gemeinsamen Geheimnis (Token oder Passwort) oder `gateway.auth.mode: "trusted-proxy"` hinter einem korrekt konfigurierten identitätsbewussten Reverse-Proxy.

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password` aktivieren die lokale Gateway-Authentifizierung **nicht** von selbst; lokale Aufrufpfade können `gateway.remote.*` nur dann als Fallback verwenden, wenn `gateway.auth.*` nicht gesetzt ist.
    - Legen Sie für die Passwortauthentifizierung `gateway.auth.mode: "password"` sowie `gateway.auth.password` (oder `OPENCLAW_GATEWAY_PASSWORD`) fest.
    - Wenn `gateway.auth.token` / `.password` explizit über SecretRef konfiguriert ist und nicht aufgelöst werden kann, schlägt die Auflösung sicher geschlossen fehl (ohne verschleiernden Remote-Fallback).
    - Control-UI-Konfigurationen mit gemeinsamem Geheimnis authentifizieren sich über `connect.params.auth.token` oder `connect.params.auth.password` (in den App-/UI-Einstellungen gespeichert). Identitätstragende Modi wie Tailscale Serve oder `trusted-proxy` verwenden stattdessen Anfrage-Header – vermeiden Sie es, gemeinsame Geheimnisse in URLs einzufügen.
    - Bei `gateway.auth.mode: "trusted-proxy"` erfordern Loopback-Reverse-Proxys auf demselben Host eine explizite Angabe von `gateway.auth.trustedProxy.allowLoopback = true` sowie einen Loopback-Eintrag in `gateway.trustedProxies`.

  </Accordion>

  <Accordion title="Warum benötige ich jetzt auf localhost ein Token?">
    OpenClaw erzwingt standardmäßig die Gateway-Authentifizierung, auch für Loopback. Wenn kein expliziter Authentifizierungspfad konfiguriert ist, wird beim Start der Token-Modus ausgewählt und ein nur für diesen Start gültiges Laufzeit-Token erzeugt. Lokale WS-Clients müssen sich daher authentifizieren. Dadurch wird verhindert, dass andere lokale Prozesse das Gateway aufrufen.

    Konfigurieren Sie `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN` oder `OPENCLAW_GATEWAY_PASSWORD` explizit, wenn Clients über Neustarts hinweg ein stabiles Geheimnis benötigen. Sie können auch den Passwortmodus oder `trusted-proxy` für identitätsbewusste Reverse-Proxys auswählen. Setzen Sie für offenen Loopback `gateway.auth.mode: "none"` explizit. `openclaw doctor --generate-gateway-token` erzeugt jederzeit ein Token.

  </Accordion>

  <Accordion title="Muss ich nach einer Konfigurationsänderung neu starten?">
    Das Gateway überwacht die Konfiguration und unterstützt Hot-Reload: `gateway.reload.mode: "hybrid"` (Standard) übernimmt sichere Änderungen im laufenden Betrieb und startet bei kritischen Änderungen neu. `hot`, `restart` und `off` werden ebenfalls unterstützt. Die meisten Änderungen an `tools.*`, der `agents.*`-Richtlinie, `session.*` und `messages.*` werden sofort und ganz ohne Reload-Aktion wirksam; Änderungen an Bindung/Port von `gateway.*` erfordern einen Neustart.
  </Accordion>

  <Accordion title="Wie aktiviere ich die Websuche (und den Webabruf)?">
    `web_fetch` funktioniert ohne API-Schlüssel. `web_search` hängt vom ausgewählten Provider ab:

    | Provider | Ohne Schlüssel | Umgebungsvariable(n) |
    | --- | --- | --- |
    | Brave | Nein | `BRAVE_API_KEY` |
    | DuckDuckGo | Ja (inoffiziell, HTML-basiert) | - |
    | Exa | Nein | `EXA_API_KEY` |
    | Firecrawl | Nein | `FIRECRAWL_API_KEY` |
    | Gemini | Nein | `GEMINI_API_KEY` |
    | Grok | Nein (xAI OAuth oder Schlüssel) | `XAI_API_KEY` |
    | Kimi | Nein | `KIMI_API_KEY` oder `MOONSHOT_API_KEY` |
    | MiniMax Search | Nein | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` oder `MINIMAX_API_KEY` |
    | Ollama Web Search | Ja (benötigt `ollama signin`) | - |
    | Perplexity | Nein | `PERPLEXITY_API_KEY` oder `OPENROUTER_API_KEY` |
    | SearXNG | Ja (selbst gehostet) | `SEARXNG_BASE_URL` |
    | Tavily | Nein | `TAVILY_API_KEY` |

    Grok kann außerdem xAI OAuth aus der Modellauthentifizierung (`openclaw onboard --auth-choice xai-oauth`) wiederverwenden.

    **Empfohlen**: `openclaw configure --section web` und wählen Sie einen Provider aus.

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // optional; für automatische Erkennung weglassen
          },
        },
      },
    }
    ```

    Die providerspezifische Websuchkonfiguration befindet sich unter `plugins.entries.<plugin>.config.webSearch.*`. Ältere `tools.web.search.*`-Providerpfade werden aus Kompatibilitätsgründen weiterhin geladen, sollten jedoch in neuen Konfigurationen nicht verwendet werden. Die Fallback-Konfiguration für Firecrawl-Webabrufe befindet sich unter `plugins.entries.firecrawl.config.webFetch.*`.

    - Zulassungslisten: Fügen Sie `web_search`/`web_fetch`/`x_search` oder `group:web` für alle drei hinzu.
    - `web_fetch` ist standardmäßig aktiviert.
    - Wenn `tools.web.fetch.provider` weggelassen wird, erkennt OpenClaw anhand der verfügbaren Anmeldedaten automatisch den ersten einsatzbereiten Fallback-Provider für Abrufe; das offizielle Firecrawl-Plugin stellt diesen Fallback bereit.
    - Daemons lesen Umgebungsvariablen aus `~/.openclaw/.env` (oder aus der Dienstumgebung).

    Dokumentation: [Webtools](/de/tools/web).

  </Accordion>

  <Accordion title="config.apply hat meine Konfiguration gelöscht. Wie kann ich sie wiederherstellen und dies vermeiden?">
    `config.apply` ersetzt die **gesamte Konfiguration**; bei einem Teilobjekt wird alles andere entfernt.

    Die aktuelle OpenClaw-Version schützt vor den meisten versehentlichen Überschreibungen:

    - OpenClaw-eigene Konfigurationsschreibvorgänge validieren vor dem Schreiben die vollständige Konfiguration nach der Änderung.
    - Ungültige oder destruktive OpenClaw-eigene Schreibvorgänge werden abgelehnt und als `openclaw.json.rejected.*` gespeichert.
    - Eine direkte Bearbeitung, die den Start oder das Hot-Reload beeinträchtigt, führt dazu, dass der Gateway geschlossen fehlschlägt oder das Neuladen überspringt; `openclaw.json` wird dabei nicht neu geschrieben.
    - `openclaw doctor --fix` ist für die Reparatur zuständig, kann den letzten als funktionsfähig bekannten Zustand wiederherstellen und speichert die abgelehnte Datei als `openclaw.json.clobbered.*`.

    Wiederherstellung:

    - Prüfen Sie `openclaw logs --follow` auf `Invalid config at`, `Config write rejected:` oder `config reload skipped (invalid config)`.
    - Prüfen Sie die neueste `openclaw.json.clobbered.*` oder `openclaw.json.rejected.*` neben der aktiven Konfiguration.
    - Führen Sie `openclaw config validate` und `openclaw doctor --fix` aus.
    - Kopieren Sie mit `openclaw config set` oder `config.patch` nur die beabsichtigten Schlüssel zurück.
    - Wenn weder ein letzter als funktionsfähig bekannter Zustand noch abgelehnte Daten vorliegen: Stellen Sie eine Sicherung wieder her oder führen Sie `openclaw doctor` erneut aus und konfigurieren Sie Kanäle/Modelle neu.
    - Bei unerwartetem Verlust: Melden Sie einen Fehler und fügen Sie Ihre letzte bekannte Konfiguration oder eine Sicherung bei. Ein lokaler Coding-Agent kann häufig anhand von Protokollen oder dem Verlauf eine funktionsfähige Konfiguration rekonstruieren.

    So vermeiden Sie dies: Verwenden Sie `openclaw config set` für kleine Änderungen, `openclaw configure` für interaktive Bearbeitungen, `config.schema.lookup` zum Prüfen eines unbekannten Pfads (gibt einen flachen Schemaknoten sowie Zusammenfassungen der unmittelbaren untergeordneten Elemente zurück) und `config.patch` für partielle RPC-Bearbeitungen – reservieren Sie `config.apply` für das Ersetzen der vollständigen Konfiguration. Das agentenseitige Laufzeittool `gateway` verweigert selbst über ältere `tools.bash.*`-Aliasse das Neuschreiben von `tools.exec.ask` / `tools.exec.security`.

    Dokumentation: [Konfiguration](/de/cli/config), [Konfigurieren](/de/cli/configure), [Gateway-Fehlerbehebung](/de/gateway/troubleshooting#gateway-rejected-invalid-config), [Doctor](/de/gateway/doctor).

  </Accordion>

  <Accordion title="Wie betreibe ich einen zentralen Gateway mit spezialisierten Workern auf mehreren Geräten?">
    Gängiges Muster: **ein Gateway** (zum Beispiel ein Raspberry Pi) plus **Nodes** und **Agenten**.

    - **Gateway (zentral)**: Verwaltet Kanäle (Signal/WhatsApp), Routing und Sitzungen.
    - **Nodes (Geräte)**: Macs/iOS-/Android-Geräte stellen als Peripheriegeräte eine Verbindung her und stellen lokale Tools bereit (`system.run`, `canvas`, `camera`).
    - **Agenten (Worker)**: Separate Steuerinstanzen/Arbeitsbereiche für besondere Rollen (zum Beispiel Betrieb im Gegensatz zu persönlichen Daten).
    - **Unteragenten**: Starten Hintergrundarbeit von einem Hauptagenten aus, um Aufgaben parallel auszuführen.
    - **TUI**: Stellt eine Verbindung zum Gateway her und wechselt zwischen Agenten/Sitzungen.

    Dokumentation: [Nodes](/de/nodes), [Remotezugriff](/de/gateway/remote), [Multi-Agent-Routing](/de/concepts/multi-agent), [Unteragenten](/de/tools/subagents), [TUI](/de/web/tui).

  </Accordion>

  <Accordion title="Kann der OpenClaw-Browser headless ausgeführt werden?">
    Ja:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    Standardmäßig ist `false` (mit sichtbarer Oberfläche) eingestellt. Der Headless-Betrieb löst bei manchen Websites eher Anti-Bot-Prüfungen aus (X/Twitter blockiert Headless-Sitzungen häufig). Er verwendet dieselbe Chromium-Engine und eignet sich für die meisten Automatisierungen; der Hauptunterschied besteht darin, dass kein Browserfenster sichtbar ist (verwenden Sie Screenshots für visuelle Inhalte). Siehe [Browser](/de/tools/browser).

  </Accordion>

  <Accordion title="Wie verwende ich Brave zur Browsersteuerung?">
    Setzen Sie `browser.executablePath` auf Ihre Brave-Binärdatei (oder einen beliebigen Chromium-basierten Browser) und starten Sie den Gateway neu. Siehe [Browser](/de/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Remote-Gateways und Nodes

<AccordionGroup>
  <Accordion title="Wie werden Befehle zwischen Telegram, dem Gateway und Nodes weitergegeben?">
    Telegram-Nachrichten werden vom **Gateway** verarbeitet, der den Agenten ausführt und erst danach über den **Gateway-WebSocket** Nodes aufruft, wenn ein Node-Tool benötigt wird:

    Telegram -> Gateway -> Agent -> `node.*` -> Node -> Gateway -> Telegram

    Nodes sehen keinen eingehenden Provider-Datenverkehr; sie empfangen ausschließlich Node-RPC-Aufrufe.

  </Accordion>

  <Accordion title="Wie kann mein Agent auf meinen Computer zugreifen, wenn der Gateway remote gehostet wird?">
    Koppeln Sie Ihren Computer als **Node**. Der Gateway wird an einem anderen Ort ausgeführt, kann jedoch über den Gateway-WebSocket `node.*`-Tools (Bildschirm, Kamera, System) auf Ihrem lokalen Computer aufrufen.

    1. Führen Sie den Gateway auf dem ständig aktiven Host aus (VPS/Heimserver).
    2. Fügen Sie den Gateway-Host und Ihren Computer demselben Tailnet hinzu.
    3. Stellen Sie sicher, dass der Gateway-WS erreichbar ist (Tailnet-Bindung oder SSH-Tunnel).
    4. Öffnen Sie die macOS-App lokal und stellen Sie im Modus **Remote over SSH** (oder direkt über das Tailnet) eine Verbindung her, damit sie als Node registriert wird.
    5. Genehmigen Sie den Node:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Es ist keine separate TCP-Bridge erforderlich; Nodes stellen die Verbindung über den Gateway-WebSocket her.

    Sicherheitshinweis: Durch das Koppeln eines macOS-Nodes wird `system.run` auf diesem Computer ermöglicht. Koppeln Sie nur vertrauenswürdige Geräte; lesen Sie [Sicherheit](/de/gateway/security).

    Dokumentation: [Nodes](/de/nodes), [Gateway-Protokoll](/de/gateway/protocol), [macOS-Remote-Modus](/de/platforms/mac/remote), [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Tailscale ist verbunden, aber ich erhalte keine Antworten. Was nun?">
    Prüfen Sie zunächst die Grundlagen:

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    Überprüfen Sie anschließend Authentifizierung und Routing: Wenn Sie Tailscale Serve verwenden, vergewissern Sie sich, dass `gateway.auth.allowTailscale` korrekt festgelegt ist. Wenn Sie die Verbindung über einen SSH-Tunnel herstellen, vergewissern Sie sich, dass der Tunnel aktiv ist und auf den richtigen Port verweist. Prüfen Sie außerdem, ob die Zulassungslisten für Direktnachrichten/Gruppen Ihr Konto enthalten.

    Dokumentation: [Tailscale](/de/gateway/tailscale), [Remotezugriff](/de/gateway/remote), [Kanäle](/de/channels).

  </Accordion>

  <Accordion title="Können zwei OpenClaw-Instanzen miteinander kommunizieren (lokal + VPS)?">
    Ja, allerdings gibt es keine integrierte Bot-zu-Bot-Bridge.

    **Am einfachsten**: Verwenden Sie einen normalen Chatkanal, auf den beide Bots zugreifen können (Slack/Telegram/WhatsApp). Lassen Sie Bot A eine Nachricht an Bot B senden und Bot B anschließend wie gewohnt antworten.

    **CLI-Bridge (generisch)**: Führen Sie ein Skript aus, das den anderen Gateway mit `openclaw agent --message ... --deliver` aufruft und dabei einen Chat als Ziel angibt, in dem der andere Bot Nachrichten empfängt. Wenn sich ein Bot auf einem entfernten VPS befindet, richten Sie Ihre CLI über SSH/Tailscale auf diesen Remote-Gateway aus (siehe [Remotezugriff](/de/gateway/remote)):

    ```bash
    openclaw agent --message "Hallo vom lokalen Bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Fügen Sie eine Schutzmaßnahme hinzu, damit die beiden Bots keine Endlosschleife erzeugen (nur bei Erwähnungen, Kanal-Zulassungslisten oder eine Regel „nicht auf Bot-Nachrichten antworten“).

    Dokumentation: [Remotezugriff](/de/gateway/remote), [Agenten-CLI](/de/cli/agent), [Senden durch Agenten](/de/tools/agent-send).

  </Accordion>

  <Accordion title="Benötige ich separate VPSes für mehrere Agenten?">
    Nein. Ein Gateway hostet mehrere Agenten, jeweils mit eigenem Arbeitsbereich, eigenen Modellstandards und eigenem Routing – dies ist die normale Einrichtung und wesentlich günstiger/einfacher als ein VPS pro Agent. Verwenden Sie separate VPSes nur für eine strikte Isolation (Sicherheitsgrenzen) oder für stark unterschiedliche Konfigurationen, die nicht gemeinsam genutzt werden sollen.
  </Accordion>

  <Accordion title="Bietet die Verwendung eines Nodes auf meinem persönlichen Laptop Vorteile gegenüber SSH von einem VPS aus?">
    Ja: Nodes sind die primäre Methode, um von einem Remote-Gateway aus auf Ihren Laptop zuzugreifen, und ermöglichen mehr als nur Shell-Zugriff. Der Gateway läuft unter macOS/Linux (Windows über WSL2) und benötigt nur wenige Ressourcen (ein kleiner VPS oder ein Gerät der Raspberry-Pi-Klasse reicht aus; 4 GB RAM sind ausreichend). Eine gängige Einrichtung besteht daher aus einem ständig aktiven Host und Ihrem Laptop als Node.

    - **Kein eingehendes SSH erforderlich** – Nodes stellen über die Gerätekopplung ausgehend eine Verbindung zum Gateway-WebSocket her.
    - **Sicherere Ausführungssteuerung** – `system.run` wird auf diesem Laptop durch Node-Zulassungslisten/Genehmigungen beschränkt.
    - **Mehr Gerätetools** – Nodes stellen zusätzlich zu `system.run` auch `canvas`, `camera` und `screen` bereit.
    - **Lokale Browserautomatisierung** – Lassen Sie den Gateway auf einem VPS laufen, führen Sie Chrome jedoch lokal über einen Node-Host aus, oder stellen Sie über Chrome MCP eine Verbindung zum lokalen Chrome her.

    SSH eignet sich für gelegentlichen Shell-Zugriff; für fortlaufende Agenten-Workflows und Geräteautomatisierung sind Nodes einfacher.

    Dokumentation: [Nodes](/de/nodes), [Nodes-CLI](/de/cli/nodes), [Browser](/de/tools/browser).

  </Accordion>

  <Accordion title="Führen Nodes einen Gateway-Dienst aus?">
    Nein. Pro Host sollte nur **ein Gateway** ausgeführt werden, sofern Sie nicht absichtlich isolierte Profile verwenden (siehe [Mehrere Gateways](/de/gateway/multiple-gateways)). Nodes sind Peripheriegeräte, die eine Verbindung zum Gateway herstellen (iOS-/Android-Nodes oder der macOS-„Node-Modus“ in der Menüleisten-App). Informationen zu Headless-Node-Hosts und zur CLI-Steuerung finden Sie unter [Node-Host-CLI](/de/cli/node).

    Für Änderungen an `gateway`, `discovery` und gehosteten Plugin-Oberflächen ist ein vollständiger Neustart erforderlich.

  </Accordion>

  <Accordion title="Gibt es eine API-/RPC-Möglichkeit, eine Konfiguration anzuwenden?">
    Ja:

    - `config.schema.lookup`: Prüft vor dem Schreiben einen Konfigurationsunterbaum mit seinem flachen Schemaknoten, dem passenden UI-Hinweis und Zusammenfassungen der unmittelbaren untergeordneten Elemente.
    - `config.get`: Ruft den aktuellen Snapshot einschließlich Hash ab.
    - `config.patch`: Sichere partielle Aktualisierung (für die meisten RPC-Bearbeitungen bevorzugt); führt nach Möglichkeit ein Hot-Reload durch und startet neu, wenn dies erforderlich ist.
    - `config.apply`: Validiert und ersetzt die vollständige Konfiguration; führt nach Möglichkeit ein Hot-Reload durch und startet neu, wenn dies erforderlich ist.
    - Das agentenseitige Laufzeittool `gateway` verweigert weiterhin das Neuschreiben von `tools.exec.ask` / `tools.exec.security`; ältere `tools.bash.*`-Aliasse werden auf dieselben geschützten Pfade normalisiert.

  </Accordion>

  <Accordion title="Minimale sinnvolle Konfiguration für eine Erstinstallation">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Legt Ihren Arbeitsbereich fest und beschränkt, wer den Bot auslösen kann.

  </Accordion>

  <Accordion title="Wie richte ich Tailscale auf einem VPS ein und stelle von meinem Mac aus eine Verbindung her?">
    1. **Auf dem VPS installieren und anmelden**:
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. **Auf Ihrem Mac installieren und anmelden** – verwenden Sie dazu die Tailscale-App und dasselbe Tailnet.
    3. **Aktivieren Sie MagicDNS** in der Tailscale-Administrationskonsole, damit der VPS einen stabilen Namen erhält.
    4. **Verwenden Sie den Tailnet-Hostnamen**: SSH `ssh user@your-vps.tailnet-xxxx.ts.net`; Gateway-WS `ws://your-vps.tailnet-xxxx.ts.net:18789`.

    Verwenden Sie für die Control UI ohne SSH Tailscale Serve auf dem VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Dadurch bleibt das Gateway an die Loopback-Schnittstelle gebunden und HTTPS wird über Tailscale bereitgestellt. Siehe [Tailscale](/de/gateway/tailscale).

  </Accordion>

  <Accordion title="Wie verbinde ich einen Mac-Node mit einem entfernten Gateway (Tailscale Serve)?">
    Serve stellt die **Gateway Control UI und WS** bereit; Nodes verbinden sich über denselben Gateway-WS-Endpunkt.

    1. Stellen Sie sicher, dass sich der VPS und der Mac im selben Tailnet befinden.
    2. Verwenden Sie die macOS-App im Remote-Modus (das SSH-Ziel kann der Tailnet-Hostname sein) – sie tunnelt den Gateway-Port und stellt die Verbindung als Node her.
    3. Genehmigen Sie den Node:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Dokumentation: [Gateway-Protokoll](/de/gateway/protocol), [Erkennung](/de/gateway/discovery), [macOS-Remote-Modus](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Soll ich OpenClaw auf einem zweiten Laptop installieren oder nur einen Node hinzufügen?">
    Fügen Sie den zweiten Laptop für **ausschließlich lokale Tools** (Bildschirm/Kamera/Ausführung) als **Node** hinzu – ein Gateway, keine doppelte Konfiguration. Lokale Node-Tools sind derzeit nur unter macOS verfügbar. Installieren Sie ein zweites Gateway nur für **strikte Isolation** oder zwei vollständig getrennte Bots.

    Dokumentation: [Nodes](/de/nodes), [Nodes-CLI](/de/cli/nodes), [Mehrere Gateways](/de/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Umgebungsvariablen und Laden von .env

<AccordionGroup>
  <Accordion title="Wie lädt OpenClaw Umgebungsvariablen?">
    OpenClaw liest Umgebungsvariablen aus dem übergeordneten Prozess (Shell, launchd/systemd, CI usw.) und lädt zusätzlich:

    - `.env` aus dem aktuellen Arbeitsverzeichnis.
    - einen globalen Fallback `.env` aus `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`).

    Keine der `.env`-Dateien überschreibt vorhandene Umgebungsvariablen. Eine Ausnahme bilden Provider-Zugangsdaten und Schlüssel für das Endpunkt-Routing in der Workspace-Datei `.env`: Schlüssel wie `GEMINI_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY` oder alle Schlüssel, die auf `_ENDPOINT` enden (sowie andere Umgebungsvariablen für Authentifizierung oder Endpunkte gebündelter Provider), werden aus der Workspace-Datei `.env` ignoriert und sollten in der Prozessumgebung, in `~/.openclaw/.env` oder in der Konfiguration `env` hinterlegt werden.

    Inline-Umgebungsvariablen in der Konfiguration gelten nur, wenn sie in der Prozessumgebung fehlen:

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Die vollständige Rangfolge und alle Quellen finden Sie unter [/environment](/de/help/environment).

  </Accordion>

  <Accordion title="Ich habe das Gateway über den Dienst gestartet und meine Umgebungsvariablen sind verschwunden. Was nun?">
    Zwei Lösungen:

    1. Hinterlegen Sie die fehlenden Schlüssel in `~/.openclaw/.env`, damit sie auch geladen werden, wenn der Dienst Ihre Shell-Umgebung nicht übernimmt.
    2. Aktivieren Sie den Shell-Import (optionale Komfortfunktion):
       ```json5
       {
         env: {
           shellEnv: {
             enabled: true,
             timeoutMs: 15000,
           },
         },
       }
       ```
       Dadurch wird Ihre Anmelde-Shell ausgeführt und es werden nur fehlende erwartete Schlüssel importiert (vorhandene werden niemals überschrieben). Entsprechende Umgebungsvariablen: `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Ich habe COPILOT_GITHUB_TOKEN gesetzt, aber der Modellstatus zeigt „Shell env: off.“ an. Warum?'>
    `openclaw models status` gibt an, ob der **Shell-Umgebungsimport** aktiviert ist. „Shell env: off“ bedeutet **nicht**, dass Ihre Umgebungsvariablen fehlen – es bedeutet lediglich, dass OpenClaw Ihre Anmelde-Shell nicht automatisch lädt.

    Wenn das Gateway als Dienst (launchd/systemd) ausgeführt wird, übernimmt es Ihre Shell-Umgebung nicht. Beheben Sie dies, indem Sie das Token in `~/.openclaw/.env` hinterlegen, `env.shellEnv.enabled: true` aktivieren oder es der Konfiguration `env` hinzufügen (gilt nur, wenn es fehlt). Starten Sie anschließend das Gateway neu und prüfen Sie den Status erneut:

    ```bash
    openclaw models status
    ```

    Copilot-Tokens werden in dieser Reihenfolge aufgelöst: `OPENCLAW_GITHUB_TOKEN`, dann `COPILOT_GITHUB_TOKEN`, dann `GH_TOKEN`, dann `GITHUB_TOKEN`.

    Siehe [/concepts/model-providers](/de/concepts/model-providers) und [/environment](/de/help/environment).

  </Accordion>
</AccordionGroup>

## Sitzungen und mehrere Chats

<AccordionGroup>
  <Accordion title="Wie beginne ich eine neue Unterhaltung?">
    Senden Sie `/new` oder `/reset` als eigenständige Nachricht. Siehe [Sitzungsverwaltung](/de/concepts/session).
  </Accordion>

  <Accordion title="Werden Sitzungen automatisch zurückgesetzt, wenn ich nie /new sende?">
    Nein, standardmäßig nicht. Sitzungen behalten dieselbe `sessionId`, und Compaction begrenzt den aktiven Modellkontext, wenn Unterhaltungen länger werden. `/new` und `/reset` bleiben verfügbar. Alternativ können Sie automatische Zurücksetzungen mit `mode: "daily"` oder `mode: "idle"` aktivieren. Der tägliche Modus wechselt um `session.reset.atHour` (Standard: `4`, 0-23) auf dem Gateway-Host; der Leerlaufmodus verwendet `session.reset.idleMinutes` seit der letzten tatsächlichen Interaktion, nicht seit Heartbeat-/Cron-/Ausführungs-Systemereignissen.

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType` unterstützt `direct`, `group` und `thread`. Doctor migriert veraltete `dm`-Einträge zu `direct`; das Schema lehnt `dm` ab. Das veraltete `session.idleMinutes` auf oberster Ebene funktioniert weiterhin als Kompatibilitätsalias für einen Standardwert im Leerlaufmodus, wenn kein `session.reset`- oder `resetByType`-Block festgelegt ist. Den vollständigen Lebenszyklus finden Sie unter [Sitzungsverwaltung](/de/concepts/session).

  </Accordion>

  <Accordion title="Kann ich ein Team aus OpenClaw-Instanzen erstellen (ein CEO und viele Agenten)?">
    Ja, über **Multi-Agent-Routing** und **Unteragenten**: ein koordinierender Agent sowie mehrere Arbeitsagenten mit eigenen Workspaces und Modellen.

    Dies sollte eher als interessantes Experiment betrachtet werden – es verbraucht viele Tokens und ist häufig weniger effizient als ein einzelner Bot mit getrennten Sitzungen. Das typische Modell besteht aus einem Bot, mit dem Sie kommunizieren, unterschiedlichen Sitzungen für parallele Arbeiten und bei Bedarf gestarteten Unteragenten.

    Dokumentation: [Multi-Agent-Routing](/de/concepts/multi-agent), [Unteragenten](/de/tools/subagents), [Agenten-CLI](/de/cli/agents).

  </Accordion>

  <Accordion title="Warum wurde der Kontext mitten in einer Aufgabe abgeschnitten? Wie kann ich das verhindern?">
    Der Sitzungskontext ist durch das Kontextfenster des Modells begrenzt. Lange Chats, umfangreiche Tool-Ausgaben oder viele Dateien können Compaction oder eine Kürzung auslösen.

    - Bitten Sie den Bot, den aktuellen Stand zusammenzufassen und in eine Datei zu schreiben.
    - Verwenden Sie `/compact` vor langen Aufgaben und `/new` beim Wechseln des Themas.
    - Bewahren Sie wichtigen Kontext im Workspace auf und bitten Sie den Bot, ihn erneut einzulesen.
    - Verwenden Sie Unteragenten für lange oder parallele Arbeiten, damit der Hauptchat kleiner bleibt.
    - Wählen Sie ein Modell mit einem größeren Kontextfenster, wenn dies häufig vorkommt.

  </Accordion>

  <Accordion title="Wie setze ich OpenClaw vollständig zurück, ohne es zu deinstallieren?">
    ```bash
    openclaw reset
    ```

    Nicht interaktives vollständiges Zurücksetzen:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Führen Sie anschließend die Einrichtung erneut aus:

    ```bash
    openclaw onboard --install-daemon
    ```

    Das Onboarding bietet außerdem **Zurücksetzen** an, wenn eine vorhandene Konfiguration erkannt wird; siehe [Onboarding (CLI)](/de/start/wizard). Wenn Sie Profile verwendet haben (`--profile` / `OPENCLAW_PROFILE`), setzen Sie jedes Zustandsverzeichnis zurück (Standard: `~/.openclaw-<profile>`). Nur für die Entwicklung vorgesehenes Zurücksetzen: `openclaw gateway --dev --reset` löscht Entwicklungskonfiguration, Zugangsdaten, Sitzungen und Workspace.

  </Accordion>

  <Accordion title='Ich erhalte Fehler vom Typ „context too large“ – wie kann ich zurücksetzen oder komprimieren?'>
    - **Komprimieren** (behält die Unterhaltung bei und fasst ältere Beiträge zusammen): `/compact` oder `/compact <instructions>`, um die Zusammenfassung zu steuern.
    - **Zurücksetzen** (neue Sitzungs-ID für denselben Chat-Schlüssel): `/new` oder `/reset`.

    Wenn das Problem weiterhin auftritt, passen Sie die **Sitzungsbereinigung** (`agents.defaults.contextPruning`) an, um alte Tool-Ausgaben zu kürzen, oder verwenden Sie ein Modell mit einem größeren Kontextfenster.

    Dokumentation: [Compaction](/de/concepts/compaction), [Sitzungsbereinigung](/de/concepts/session-pruning), [Sitzungsverwaltung](/de/concepts/session).

  </Accordion>

  <Accordion title='Warum wird „LLM request rejected: messages.content.tool_use.input field required“ angezeigt?'>
    Provider-Validierungsfehler: Das Modell hat einen `tool_use`-Block ohne das erforderliche `input` ausgegeben. Dies bedeutet in der Regel, dass der Sitzungsverlauf veraltet oder beschädigt ist (häufig nach langen Threads oder einer Tool-/Schemaänderung).

    Lösung: Starten Sie mit `/new` eine neue Sitzung (als eigenständige Nachricht).

  </Accordion>

  <Accordion title="Warum erhalte ich alle 30 Minuten Heartbeat-Nachrichten?">
    Heartbeats werden standardmäßig alle **30m** ausgeführt oder alle **1h**, wenn der aufgelöste Authentifizierungsmodus Anthropic-OAuth-/Token-Authentifizierung ist (einschließlich der Wiederverwendung der Claude-CLI) und `heartbeat.every` nicht festgelegt ist. Anpassen oder deaktivieren:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // or "0m" to disable
          },
        },
      },
    }
    ```

    Wenn `HEARTBEAT.md` vorhanden, aber faktisch leer ist (nur Leerzeilen, Markdown-/HTML-Kommentare, ATX-Überschriften, Fence-Markierungen oder leere Listeneintragsgerüste), überspringt OpenClaw die Heartbeat-Ausführung, um API-Aufrufe einzusparen. Wenn die Datei fehlt, wird der Heartbeat dennoch ausgeführt und das Modell entscheidet, was zu tun ist.

    Agentenspezifische Überschreibungen verwenden `agents.entries.*.heartbeat`. Dokumentation: [Heartbeat](/de/gateway/heartbeat).

  </Accordion>

  <Accordion title='Muss ich einer WhatsApp-Gruppe ein „Bot-Konto“ hinzufügen?'>
    Nein. OpenClaw wird über **Ihr eigenes Konto** ausgeführt – wenn Sie Mitglied der Gruppe sind, kann OpenClaw sie sehen. Standardmäßig werden Gruppenantworten blockiert, bis Sie Absender zulassen (`groupPolicy: "allowlist"`).

    So beschränken Sie Gruppenantworten ausschließlich auf sich selbst:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Wie erhalte ich die JID einer WhatsApp-Gruppe?">
    Am schnellsten geht es, indem Sie die Logs fortlaufend anzeigen und eine Testnachricht in der Gruppe senden.

    ```bash
    openclaw logs --follow --json
    ```

    Suchen Sie nach `chatId` (oder `from`), das auf `@g.us` endet, beispielsweise `1234567890-1234567890@g.us`.

    Wenn die Gruppen bereits konfiguriert oder in der Zulassungsliste enthalten sind, listen Sie sie aus der Konfiguration auf:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Dokumentation: [WhatsApp](/de/channels/whatsapp), [Verzeichnis](/de/cli/directory), [Logs](/de/cli/logs).

  </Accordion>

  <Accordion title="Warum antwortet OpenClaw nicht in einer Gruppe?">
    Zwei häufige Ursachen: Die Erwähnungsbeschränkung ist standardmäßig aktiviert (Sie müssen den Bot mit @ erwähnen oder `mentionPatterns` erfüllen), oder Sie haben `channels.whatsapp.groups` ohne `"*"` konfiguriert und die Gruppe befindet sich nicht in der Zulassungsliste.

    Siehe [Gruppen](/de/channels/groups) und [Gruppennachrichten](/de/channels/group-messages).

  </Accordion>

  <Accordion title="Teilen Gruppen/Threads ihren Kontext mit Direktnachrichten?">
    Direkte Chats werden standardmäßig in der Hauptsitzung zusammengeführt. Gruppen/Kanäle verfügen über eigene Sitzungsschlüssel, und Telegram-Themen sowie Discord-Threads sind separate Sitzungen. Siehe [Gruppen](/de/channels/groups) und [Gruppennachrichten](/de/channels/group-messages).
  </Accordion>

  <Accordion title="Wie viele Workspaces und Agenten kann ich erstellen?">
    Es gibt keine festen Grenzen – Dutzende oder sogar Hunderte sind möglich, achten Sie jedoch auf:

    - **Speicherplatzwachstum**: Aktive Sitzungen und Transkripte befinden sich in der agentenspezifischen SQLite-Datenbank; Legacy-/Archivartefakte können sich weiterhin unter `~/.openclaw/agents/<agentId>/sessions/` ansammeln.
    - **Token-Kosten**: Mehr Agenten bedeuten eine höhere gleichzeitige Modellnutzung.
    - **Betriebsaufwand**: agentenspezifische Auth-Profile, Arbeitsbereiche und Kanal-Routing.

    Behalten Sie pro Agent einen **aktiven** Arbeitsbereich (`agents.defaults.workspace`), bereinigen Sie alte Sitzungen mit `openclaw sessions cleanup`, wenn der Speicherplatzverbrauch wächst (bearbeiten Sie den aktiven SQLite-Zustand nicht manuell), und verwenden Sie `openclaw doctor`, um verwaiste Arbeitsbereiche und nicht übereinstimmende Profile zu erkennen.

  </Accordion>

  <Accordion title="Kann ich mehrere Bots oder Chats gleichzeitig ausführen (Slack), und wie sollte ich das einrichten?">
    Ja, über **Multi-Agent-Routing**: Führen Sie mehrere isolierte Agenten aus und leiten Sie eingehende Nachrichten anhand von Kanal/Konto/Peer weiter. Slack wird als Kanal unterstützt und kann bestimmten Agenten zugeordnet werden.

    Der Browserzugriff ist leistungsfähig, ermöglicht aber nicht, „alles zu tun, was ein Mensch kann“ – Anti-Bot-Maßnahmen, CAPTCHAs und MFA können die Automatisierung weiterhin blockieren. Für die zuverlässigste Steuerung verwenden Sie lokales Chrome MCP auf dem Host oder CDP auf dem Rechner, auf dem der Browser tatsächlich ausgeführt wird.

    Empfohlene Einrichtung: ein ständig verfügbarer Gateway-Host (VPS/Mac mini), ein Agent pro Rolle (Zuordnungen), diesen Agenten zugeordnete Slack-Kanäle und bei Bedarf ein lokaler Browser über Chrome MCP oder eine Node.

    Dokumentation: [Multi-Agent-Routing](/de/concepts/multi-agent), [Slack](/de/channels/slack), [Browser](/de/tools/browser), [Nodes](/de/nodes).

  </Accordion>
</AccordionGroup>

## Modelle, Failover und Auth-Profile

Fragen und Antworten zu Modellen – Standardwerte, Auswahl, Aliasse, Wechsel, Failover und Auth-Profile – finden Sie in den [häufig gestellten Fragen zu Modellen](/de/help/faq-models).

## Gateway: Ports, „bereits ausgeführt“ und Remote-Modus

<AccordionGroup>
  <Accordion title="Welchen Port verwendet das Gateway?">
    `gateway.port` steuert den einzelnen multiplexierten Port für WebSocket + HTTP (Control UI, Hooks usw.). Rangfolge:

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > Standardwert 18789
    ```

  </Accordion>

  <Accordion title='Warum meldet openclaw gateway status „Runtime: running“, aber „Connectivity probe: failed“?'>
    „Running“ ist die Ansicht des **Supervisors** (launchd/systemd/schtasks); bei der Konnektivitätsprüfung stellt die CLI tatsächlich eine Verbindung zum Gateway-WebSocket her. Verlassen Sie sich auf diese Zeilen aus `openclaw gateway status`: `Probe target:` (die von der Prüfung verwendete URL), `Listening:` (was tatsächlich an den Port gebunden ist), `Last gateway error:` (häufige Ursache, wenn der Prozess aktiv ist, der Port aber nicht lauscht).
  </Accordion>

  <Accordion title='Warum zeigt openclaw gateway status unterschiedliche Werte für „Config (cli)“ und „Config (service)“ an?'>
    Sie bearbeiten eine Konfigurationsdatei, während der Dienst eine andere verwendet (häufig aufgrund einer Nichtübereinstimmung von `--profile` / `OPENCLAW_STATE_DIR`).

    Führen Sie zur Behebung den folgenden Befehl aus derselben `--profile` / Umgebung aus, die der Dienst verwenden soll:

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='Was bedeutet „another gateway instance is already listening“?'>
    OpenClaw erzwingt eine Laufzeitsperre, indem der WebSocket-Listener unmittelbar beim Start gebunden wird (standardmäßig `ws://127.0.0.1:18789`). Wenn die Bindung mit `EADDRINUSE` fehlschlägt, wird `GatewayLockError` („another gateway instance is already listening“) ausgelöst.

    Behebung: Stoppen Sie die andere Instanz, geben Sie den Port frei oder führen Sie den Vorgang mit `openclaw gateway --port <port>` aus.

  </Accordion>

  <Accordion title="Wie führe ich OpenClaw im Remote-Modus aus (der Client verbindet sich mit einem Gateway an einem anderen Ort)?">
    Legen Sie `gateway.mode: "remote"` fest und geben Sie eine Remote-WebSocket-URL an, optional mit Remote-Anmeldedaten für ein gemeinsames Geheimnis:

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` startet nur, wenn `gateway.mode` den Wert `local` hat (oder Sie ein Überschreibungs-Flag übergeben).
    - Die macOS-App überwacht die Konfigurationsdatei und wechselt bei Änderungen dieser Werte unmittelbar zwischen den Modi.
    - `gateway.remote.token` / `.password` sind ausschließlich clientseitige Remote-Anmeldedaten; sie aktivieren nicht eigenständig die Authentifizierung des lokalen Gateways.

  </Accordion>

  <Accordion title='Die Control UI meldet „unauthorized“ (oder stellt ständig erneut eine Verbindung her). Was nun?'>
    Der Authentifizierungspfad Ihres Gateways und die Authentifizierungsmethode der UI stimmen nicht überein.

    Fakten (aus dem Code):

    - Die Control UI speichert das Token in `sessionStorage`, beschränkt auf den aktuellen Browser-Tab und die ausgewählte Gateway-URL. Dadurch funktionieren Aktualisierungen im selben Tab weiterhin, ohne dass das Token langfristig in localStorage gespeichert wird.
    - Bei `AUTH_TOKEN_MISMATCH` können vertrauenswürdige Clients einen einzigen begrenzten Wiederholungsversuch mit einem zwischengespeicherten Geräte-Token unternehmen, wenn das Gateway Hinweise für einen erneuten Versuch zurückgibt (`canRetryWithDeviceToken=true`, `recommendedNextStep=retry_with_device_token`).
    - Dieser Wiederholungsversuch mit dem zwischengespeicherten Token verwendet die zusammen mit dem Geräte-Token gespeicherten, genehmigten Geltungsbereiche erneut; Aufrufer mit explizitem `deviceToken` / explizitem `scopes` behalten ihre angeforderten Geltungsbereiche bei, statt zwischengespeicherte Geltungsbereiche zu übernehmen.
    - Außerhalb dieses Wiederholungspfads gilt für die Verbindungsauthentifizierung folgende Rangfolge: zuerst explizites gemeinsames Token/Passwort, dann explizites `deviceToken`, danach gespeichertes Geräte-Token und schließlich Bootstrap-Token.
    - Der integrierte Bootstrap über Einrichtungscode gibt ein Node-Geräte-Token mit `scopes: []` sowie ein zeitlich begrenztes Operator-Übergabe-Token für das vertrauenswürdige mobile Onboarding zurück. Die Operator-Übergabe kann die native Konfiguration während der Einrichtung lesen, gewährt jedoch weder Geltungsbereiche zur Änderung der Kopplung noch `operator.admin`.

    Behebung:

    - Am schnellsten: `openclaw dashboard` (gibt die Dashboard-URL aus, kopiert sie und versucht, sie zu öffnen; zeigt in einer Headless-Umgebung einen SSH-Hinweis an).
    - Noch kein Token: `openclaw doctor --generate-gateway-token`.
    - Remote: Erstellen Sie zuerst mit `ssh -N -L 18789:127.0.0.1:18789 user@host` einen Tunnel und öffnen Sie anschließend `http://127.0.0.1:18789/`.
    - Modus mit gemeinsamem Geheimnis: Legen Sie `gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` oder `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` fest und fügen Sie anschließend das entsprechende Geheimnis in den Einstellungen der Control UI ein.
    - Tailscale-Serve-Modus: Vergewissern Sie sich, dass `gateway.auth.allowTailscale` aktiviert ist und Sie die Serve-URL öffnen, nicht eine direkte Loopback-/Tailnet-URL, die Tailscale-Identitätsheader umgeht.
    - Trusted-Proxy-Modus: Vergewissern Sie sich, dass die Verbindung über den konfigurierten identitätsbewussten Proxy erfolgt. Loopback-Proxys auf demselben Host benötigen außerdem `gateway.auth.trustedProxy.allowLoopback = true`.
    - Wenn die Nichtübereinstimmung nach dem einen Wiederholungsversuch weiterhin besteht, rotieren/genehmigen Sie das Token des gekoppelten Geräts erneut:
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - Rotation abgelehnt: Sitzungen gekoppelter Geräte können nur ihr **eigenes** Gerät rotieren, sofern sie nicht zusätzlich über `operator.admin` verfügen. Explizite Werte für `--scope` dürfen die aktuellen Operator-Geltungsbereiche des Aufrufers nicht überschreiten.
    - Falls das Problem weiterhin besteht: `openclaw status --all` sowie [Fehlerbehebung](/de/gateway/troubleshooting). Einzelheiten zur Authentifizierung finden Sie unter [Dashboard](/de/web/dashboard).

  </Accordion>

  <Accordion title="Ich habe gateway.bind auf tailnet gesetzt, aber es lauscht nur auf Loopback">
    Die Bindung `tailnet` wählt eine Tailscale-IP aus Ihren Netzwerkschnittstellen (100.64.0.0/10). Wenn der Rechner nicht mit Tailscale verbunden ist (oder die Schnittstelle inaktiv ist), weicht das Gateway auf Loopback aus, statt eine andere Netzwerkschnittstelle freizugeben.

    Behebung: Starten Sie Tailscale auf diesem Host und starten Sie das Gateway neu, oder wechseln Sie ausdrücklich zu `gateway.bind: "loopback"` / `"lan"`.

    `tailnet` ist explizit; `auto` bevorzugt Loopback. Verwenden Sie `gateway.bind: "tailnet"`, um die Freigabe außerhalb von Loopback auf das Tailnet zu beschränken und gleichzeitig den erforderlichen `127.0.0.1`-Listener auf demselben Host beizubehalten.

  </Accordion>

  <Accordion title="Kann ich mehrere Gateways auf demselben Host ausführen?">
    Normalerweise nicht – ein Gateway kann mehrere Nachrichtenkanäle und Agenten ausführen. Verwenden Sie mehrere Gateways nur für Redundanz (beispielsweise einen Rettungs-Bot) oder strikte Isolation, und isolieren Sie jedes mit eigenen `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `agents.defaults.workspace` und einem eindeutigen `gateway.port`.

    Empfohlen: `openclaw --profile <name> ...` pro Instanz (erstellt automatisch `~/.openclaw-<name>`), ein eindeutiges `gateway.port` pro Profilkonfiguration (oder `--port` für manuelle Ausführungen) und ein profilspezifischer Dienst mit `openclaw --profile <name> gateway install`.

    Profile ergänzen außerdem Dienstnamen um ein Suffix: launchd `ai.openclaw.<profile>`, systemd `openclaw-gateway-<profile>.service`, Windows `OpenClaw Gateway (<profile>)`. Die nicht qualifizierte systemd-Unit `openclaw-gateway` ist nur für das Standardprofil vorhanden; der alte systemd-Unit-Name `clawdbot-gateway` aus der Zeit vor der Umbenennung wird automatisch migriert.

    Vollständige Anleitung: [Mehrere Gateways](/de/gateway/multiple-gateways).

  </Accordion>

  <Accordion title='Was bedeutet „invalid handshake“ / Code 1008?'>
    Das Gateway ist ein **WebSocket-Server** und erwartet als erste Nachricht einen `connect`-Frame. Alles andere schließt die Verbindung mit **Code 1008** (Richtlinienverstoß).

    Häufige Ursachen: Sie haben die **HTTP**-URL in einem Browser statt in einem WS-Client geöffnet, den falschen Port/Pfad verwendet oder ein Proxy/Tunnel hat Authentifizierungsheader entfernt oder eine Anfrage gesendet, die nicht für das Gateway bestimmt war.

    Behebung: Verwenden Sie die WS-URL (`ws://<host>:18789` oder `wss://...` über HTTPS), öffnen Sie den WS-Port nicht in einem normalen Browser-Tab und fügen Sie Token/Passwort in den `connect`-Frame ein, wenn die Authentifizierung aktiviert ist. CLI-/TUI-Beispiel:

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    Protokolldetails: [Gateway-Protokoll](/de/gateway/protocol).

  </Accordion>
</AccordionGroup>

## Protokollierung und Debugging

<AccordionGroup>
  <Accordion title="Wo befinden sich die Protokolle?">
    Dateiprotokolle (strukturiert): `/tmp/openclaw/openclaw-YYYY-MM-DD.log` für das Standardprofil oder `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` für ein benanntes Profil. Legen Sie über `logging.file` einen stabilen Pfad fest; die Protokollstufe für Dateien über `logging.level`; die Ausführlichkeit der Konsolenausgabe über `--verbose` und `logging.consoleLevel`.

    Schnellste Live-Anzeige:

    ```bash
    openclaw logs --follow
    ```

    Dienst-/Supervisor-Protokolle (wenn das Gateway über launchd/systemd ausgeführt wird):

    - macOS-launchd-Standardausgabe: `~/Library/Logs/openclaw/gateway.log` (Profile verwenden `gateway-<profile>.log`; die Standardfehlerausgabe wird unterdrückt).
    - Linux: `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`.
    - Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`.

    Weitere Informationen finden Sie unter [Fehlerbehebung](/de/gateway/troubleshooting).

  </Accordion>

  <Accordion title="Wie starte, stoppe oder starte ich den Gateway-Dienst neu?">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    Wenn Sie das Gateway manuell ausführen, kann `openclaw gateway --force` den Port zurückfordern. Siehe [Gateway](/de/gateway).

  </Accordion>

  <Accordion title="Ich habe mein Terminal unter Windows geschlossen – wie starte ich OpenClaw neu?">
    Drei Windows-Installationsmodi:

    **1) Lokale Einrichtung mit Windows Hub**: Die native App verwaltet ein lokales, App-eigenes WSL-Gateway. Öffnen Sie **OpenClaw Companion** über das Startmenü oder den Infobereich und verwenden Sie anschließend **Gateway Setup** oder die Registerkarte „Connections“.

    **2) Manuelles WSL2-Gateway**: Das Gateway wird unter Linux ausgeführt.
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    Wenn Sie den Dienst nie installiert haben, starten Sie ihn im Vordergrund: `openclaw gateway run`.

    **3) Native Windows-CLI/Gateway**: Wird direkt unter Windows ausgeführt.
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    Bei manueller Ausführung (ohne Dienst): `openclaw gateway run`.

    Dokumentation: [Windows](/de/platforms/windows), [Betriebshandbuch für den Gateway-Dienst](/de/gateway).

  </Accordion>

  <Accordion title="Das Gateway läuft, aber es kommen keine Antworten an. Was sollte ich überprüfen?">
    Schnelle Zustandsprüfung:

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    Häufige Ursachen: Die Modellauthentifizierung wurde auf dem **Gateway-Host** nicht geladen (prüfen Sie `models status`), die Kanalkopplung/Zulassungsliste blockiert Antworten (prüfen Sie die Kanalkonfiguration und Protokolle) oder WebChat/Dashboard wurde ohne das richtige Token geöffnet. Vergewissern Sie sich bei Remote-Verbindungen, dass die Tunnel-/Tailscale-Verbindung aktiv und der Gateway-WebSocket erreichbar ist.

    Docs: [Kanäle](/de/channels), [Fehlerbehebung](/de/gateway/troubleshooting), [Remotezugriff](/de/gateway/remote).

  </Accordion>

  <Accordion title='"Verbindung zum Gateway getrennt: kein Grund" – was nun?'>
    Dies bedeutet normalerweise, dass die UI die WebSocket-Verbindung verloren hat. Prüfen Sie: Läuft das Gateway (`openclaw gateway status`)? Ist es fehlerfrei (`openclaw status`)? Verwendet die UI das richtige Token (`openclaw dashboard`)? Bei Remotezugriff: Ist die Tunnel-/Tailscale-Verbindung aktiv?

    Verfolgen Sie anschließend die Logs:

    ```bash
    openclaw logs --follow
    ```

    Docs: [Dashboard](/de/web/dashboard), [Remotezugriff](/de/gateway/remote), [Fehlerbehebung](/de/gateway/troubleshooting).

  </Accordion>

  <Accordion title="Telegram setMyCommands schlägt fehl. Was sollte ich prüfen?">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    Ordnen Sie anschließend den Fehler zu:

    - `BOT_COMMANDS_TOO_MUCH`: Das Telegram-Menü enthält zu viele Einträge. OpenClaw kürzt es bereits auf das Telegram-Limit und versucht es mit weniger Befehlen erneut, dennoch können einige Menüeinträge entfallen. Reduzieren Sie Plugin-/Skill-/benutzerdefinierte Befehle oder deaktivieren Sie `channels.telegram.commands.native`, wenn Sie das Menü nicht benötigen.
    - `TypeError: fetch failed`, `Network request for 'setMyCommands' failed!` oder ähnliche Netzwerkfehler: Prüfen Sie auf einem VPS oder hinter einem Proxy, ob ausgehendes HTTPS zugelassen ist und DNS für `api.telegram.org` funktioniert.

    Wenn das Gateway remote ausgeführt wird, prüfen Sie die Logs auf dem Gateway-Host.

    Docs: [Telegram](/de/channels/telegram), [Fehlerbehebung für Kanäle](/de/channels/troubleshooting).

  </Accordion>

  <Accordion title="Die TUI zeigt keine Ausgabe. Was sollte ich prüfen?">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    Verwenden Sie in der TUI `/status`, um den aktuellen Status anzuzeigen. Wenn Sie Antworten in einem Chatkanal erwarten, prüfen Sie, ob die Zustellung aktiviert ist (`/deliver on`).

    Docs: [TUI](/de/web/tui), [Slash-Befehle](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Wie stoppe ich das Gateway vollständig und starte es anschließend neu?">
    Wenn Sie den Dienst installiert haben (launchd unter macOS, systemd unter Linux):

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    Beenden Sie die Ausführung im Vordergrund mit Ctrl-C und führen Sie anschließend `openclaw gateway run` aus.

    Docs: [Runbook für den Gateway-Dienst](/de/gateway).

  </Accordion>

  <Accordion title="Einfach erklärt: openclaw gateway restart im Vergleich zu openclaw gateway">
    `openclaw gateway restart` startet den **Hintergrunddienst** (launchd/systemd) neu. `openclaw gateway` führt das Gateway für diese Terminalsitzung **im Vordergrund** aus. Verwenden Sie die Gateway-Unterbefehle, wenn Sie den Dienst installiert haben; verwenden Sie die direkte Ausführung im Vordergrund für eine einmalige Ausführung.
  </Accordion>

  <Accordion title="Schnellster Weg zu weiteren Details bei einem Fehler">
    Starten Sie das Gateway mit `--verbose`, um ausführlichere Konsolendetails zu erhalten, und untersuchen Sie anschließend die Logdatei auf Fehler bei Kanalauthentifizierung, Modell-Routing und RPC.
  </Accordion>
</AccordionGroup>

## Medien und Anhänge

<AccordionGroup>
  <Accordion title="Mein Skill hat ein Bild/PDF erzeugt, aber nichts wurde gesendet">
    Ausgehende Anhänge des Agenten müssen strukturierte Medienfelder wie `media`, `mediaUrl`, `path` oder `filePath` verwenden. Siehe [OpenClaw-Assistent einrichten](/de/start/openclaw) und [Senden durch den Agenten](/de/tools/agent-send).

    ```bash
    openclaw message send --target +15555550123 --message "Hier ist es" --media /path/to/file.png
    ```

    Prüfen Sie außerdem: Der Zielkanal unterstützt ausgehende Medien und wird nicht durch Positivlisten blockiert; die Datei liegt innerhalb der Größenbeschränkungen des Providers (Bilder werden auf eine maximale Seitenlänge von 2048px skaliert); `tools.fs.workspaceOnly=true` beschränkt das Senden über lokale Pfade auf Dateien im Workspace, im temporären/Medien-Speicher und auf durch die Sandbox validierte Dateien; `tools.fs.workspaceOnly=false` (Standardwert) erlaubt bei strukturierten Sendungen lokaler Medien die Verwendung hostlokaler Dateien, die der Agent bereits lesen kann, für Medien sowie sichere Dokumenttypen (Bilder, Audio, Video, PDF, Office-Dokumente und validierte Textdokumente wie Markdown/MD, TXT, JSON, YAML/YML). Dies ist kein Geheimnis-Scanner – eine für den Agenten lesbare `secret.txt`- oder `config.json`-Datei kann angehängt werden, wenn Erweiterung und Inhaltsvalidierung übereinstimmen. Bewahren Sie vertrauliche Dateien außerhalb der für den Agenten lesbaren Pfade auf oder behalten Sie `tools.fs.workspaceOnly=true` bei, um das Senden über lokale Pfade strenger einzuschränken.

    Siehe [Bilder](/de/nodes/images).

  </Accordion>
</AccordionGroup>

## Sicherheit und Zugriffskontrolle

<AccordionGroup>
  <Accordion title="Ist es sicher, OpenClaw für eingehende Direktnachrichten zugänglich zu machen?">
    Behandeln Sie eingehende Direktnachrichten als nicht vertrauenswürdige Eingaben. Die Standardwerte reduzieren das Risiko:

    - Das Standardverhalten bei Kanälen mit Direktnachrichten-Unterstützung ist **Kopplung**: Unbekannte Absender erhalten einen Kopplungscode, und ihre Nachricht wird nicht verarbeitet. Genehmigen Sie sie mit `openclaw pairing approve --channel <channel> [--account <id>] <code>`. Ausstehende Anfragen sind auf **3 pro Kanal** begrenzt; prüfen Sie `openclaw pairing list --channel <channel> [--account <id>]`, wenn kein Code eingetroffen ist.
    - Das öffentliche Öffnen von Direktnachrichten erfordert eine ausdrückliche Aktivierung (`dmPolicy: "open"` und Positivliste `"*"`).

    Führen Sie `openclaw doctor` aus, um riskante Richtlinien für Direktnachrichten aufzuzeigen.

  </Accordion>

  <Accordion title="Ist Prompt-Injection nur bei öffentlichen Bots problematisch?">
    Nein. Bei Prompt-Injection geht es um **nicht vertrauenswürdige Inhalte**, nicht nur darum, wer dem Bot Direktnachrichten senden kann. Wenn Ihr Assistent externe Inhalte liest (Websuche/-abruf, Browserseiten, E-Mails, Dokumente, Anhänge, eingefügte Logs), können diese Inhalte Anweisungen enthalten, mit denen versucht wird, das Modell zu kapern – selbst wenn Sie der einzige Absender sind.

    Das größte Risiko besteht bei aktivierten Tools: Das Modell kann dazu verleitet werden, Kontext auszuschleusen oder in Ihrem Namen Tools aufzurufen. Reduzieren Sie den möglichen Schadensumfang:

    - Verwenden Sie einen schreibgeschützten oder Tool-deaktivierten „Leser“-Agenten, um nicht vertrauenswürdige Inhalte zusammenzufassen.
    - Lassen Sie `web_search` / `web_fetch` / `browser` für Agenten mit aktivierten Tools deaktiviert.
    - Behandeln Sie auch dekodierten Datei-/Dokumenttext als nicht vertrauenswürdig: Sowohl OpenResponses `input_file` als auch die Extraktion von Medienanhängen schließen extrahierten Text in explizite Begrenzungsmarkierungen für externe Inhalte ein, anstatt unverarbeiteten Dateitext weiterzugeben.
    - Verwenden Sie eine Sandbox und strenge Tool-Positivlisten.

    Details: [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Ist OpenClaw weniger sicher, weil es TypeScript/Node statt Rust/WASM verwendet?">
    Sprache und Laufzeit sind relevant, stellen jedoch nicht das Hauptrisiko für einen persönlichen Agenten dar. Die praktischen Risiken sind die Zugänglichkeit des Gateways, wer dem Bot Nachrichten senden kann, Prompt-Injection, der Tool-Umfang, die Handhabung von Zugangsdaten, Browserzugriff, Ausführungszugriff und das Vertrauen in Skills/Plugins von Drittanbietern.

    Rust und WASM können für einige Codeklassen eine stärkere Isolation bieten, lösen jedoch weder Prompt-Injection noch ungeeignete Positivlisten, öffentliche Gateway-Zugänglichkeit, zu weitreichende Tools oder ein Browserprofil, das bereits bei vertraulichen Konten angemeldet ist. Betrachten Sie Folgendes als die primären Schutzmaßnahmen: Halten Sie das Gateway privat oder authentifiziert, verwenden Sie Kopplung und Positivlisten für Direktnachrichten/Gruppen, verweigern Sie riskante Tools bei nicht vertrauenswürdigen Eingaben oder führen Sie sie in einer Sandbox aus, installieren Sie nur vertrauenswürdige Plugins und Skills und führen Sie nach Konfigurationsänderungen `openclaw security audit --deep` aus.

    Details: [Sicherheit](/de/gateway/security), [Sandboxing](/de/gateway/sandboxing).

  </Accordion>

  <Accordion title="Ich habe Berichte über öffentlich zugängliche OpenClaw-Instanzen gesehen. Was sollte ich prüfen?">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    Eine sicherere Ausgangskonfiguration: Das Gateway ist an `loopback` gebunden oder ausschließlich über authentifizierten privaten Zugriff erreichbar (Tailnet, SSH-Tunnel, Token-/Passwortauthentifizierung oder ein korrekt konfigurierter vertrauenswürdiger Proxy); Direktnachrichten befinden sich im Modus `pairing` oder `allowlist`; Gruppen stehen auf einer Positivliste und erfordern Erwähnungen, sofern nicht allen Mitgliedern vertraut wird; Tools mit hohem Risiko (`exec`, `browser`, `gateway`, `cron`) werden Agenten, die nicht vertrauenswürdige Inhalte lesen, verweigert oder für sie eng begrenzt; Sandboxing ist dort aktiviert, wo die Tool-Ausführung einen kleineren möglichen Schadensumfang benötigt.

    Öffentliche Bindungen ohne Authentifizierung, offene Direktnachrichten/Gruppen mit Tools und öffentlich zugängliche Browsersteuerung sind die Befunde, die zuerst behoben werden sollten. Details: [openclaw security audit](/de/gateway/security#openclaw-security-audit).

  </Accordion>

  <Accordion title="Können ClawHub-Skills und Plugins von Drittanbietern sicher installiert werden?">
    Behandeln Sie Skills und Plugins von Drittanbietern als Code, dem Sie bewusst vertrauen. ClawHub-Skillseiten zeigen vor der Installation den Scanstatus an, Scans bilden jedoch keine vollständige Sicherheitsgrenze. OpenClaw führt während der Installation oder Aktualisierung von Plugins/Skills keine integrierte lokale Blockierung gefährlichen Codes aus; verwenden Sie für lokale Zulassungs-/Blockierungsentscheidungen das betreiberverwaltete `security.installPolicy`.

    Sichereres Vorgehen: Bevorzugen Sie vertrauenswürdige Autoren und festgelegte Versionen, lesen Sie den Skill/das Plugin vor der Aktivierung, halten Sie Positivlisten für Plugins/Skills eng, führen Sie Arbeitsabläufe mit nicht vertrauenswürdigen Eingaben in einer Sandbox mit minimalen Tools aus und gewähren Sie Drittanbietercode keinen weitreichenden Dateisystem-, Ausführungs-, Browser- oder Geheimniszugriff.

    Details: [Skills](/de/tools/skills), [Plugins](/de/tools/plugin), [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Sollte mein Bot ein eigenes E-Mail-, GitHub-Konto oder eine eigene Telefonnummer haben?">
    Ja, für die meisten Konfigurationen. Die Isolierung des Bots durch separate Konten und Telefonnummern reduziert den möglichen Schadensumfang, wenn etwas schiefgeht, und erleichtert es, Zugangsdaten zu rotieren oder Zugriffe zu widerrufen, ohne Ihre persönlichen Konten zu beeinträchtigen.

    Beginnen Sie mit wenig: Gewähren Sie nur Zugriff auf die Tools und Konten, die Sie tatsächlich benötigen, und erweitern Sie ihn später bei Bedarf.

    Docs: [Sicherheit](/de/gateway/security), [Kopplung](/de/channels/pairing).

  </Accordion>

  <Accordion title="Kann ich ihm Autonomie über meine Textnachrichten geben, und ist das sicher?">
    Wir empfehlen **keine** vollständige Autonomie über Ihre persönlichen Nachrichten. Das sicherste Vorgehen: Belassen Sie Direktnachrichten im **Kopplungsmodus** oder verwenden Sie eine enge Positivliste, nutzen Sie eine **separate Nummer oder ein separates Konto**, wenn der Bot in Ihrem Namen Nachrichten senden soll, und lassen Sie ihn Entwürfe erstellen, die Sie **vor dem Senden genehmigen**.

    Experimentieren Sie ausschließlich mit einem dedizierten, isolierten Konto. Siehe [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Kann ich günstigere Modelle für Aufgaben eines persönlichen Assistenten verwenden?">
    Ja, **wenn** der Agent nur für Chats vorgesehen ist und die Eingabe vertrauenswürdig ist. Kleinere Modellklassen sind anfälliger für die Übernahme durch Anweisungen. Vermeiden Sie sie daher für Agenten mit aktivierten Tools oder beim Lesen nicht vertrauenswürdiger Inhalte. Wenn Sie ein kleineres Modell verwenden müssen, schränken Sie die Tools ein und führen Sie es innerhalb einer Sandbox aus. Siehe [Sicherheit](/de/gateway/security).
  </Accordion>

  <Accordion title="Ich habe /start in Telegram ausgeführt, aber keinen Kopplungscode erhalten">
    Kopplungscodes werden **nur** gesendet, wenn ein unbekannter Absender dem Bot eine Nachricht sendet und `dmPolicy: "pairing"` aktiviert ist; `/start` allein erzeugt keinen Code.

    Prüfen Sie ausstehende Anfragen:

    ```bash
    openclaw pairing list telegram
    ```

    Für sofortigen Zugriff nehmen Sie Ihre Absender-ID in die Positivliste auf oder legen Sie für dieses Konto `dmPolicy: "open"` fest.

  </Accordion>

  <Accordion title="WhatsApp: Sendet es meinen Kontakten Nachrichten? Wie funktioniert die Kopplung?">
    Nein. Die standardmäßige WhatsApp-Richtlinie für Direktnachrichten ist **Kopplung**. Unbekannte Absender erhalten lediglich einen Kopplungscode; ihre Nachricht wird **nicht verarbeitet**. OpenClaw antwortet nur auf eingehende Chats oder auf ausdrücklich von Ihnen ausgelöste Sendungen.

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    Die Abfrage der Telefonnummer im Assistenten legt Ihre **Positivliste/Ihren Eigentümer** fest, damit Ihre eigenen Direktnachrichten zugelassen werden – sie wird nicht für automatisches Senden verwendet. Verwenden Sie bei Ihrer persönlichen WhatsApp-Nummer diese Nummer und aktivieren Sie `channels.whatsapp.selfChatMode`.

  </Accordion>
</AccordionGroup>

## Chatbefehle, Abbrechen von Aufgaben und „es hört nicht auf“

<AccordionGroup>
  <Accordion title="Wie verhindere ich, dass interne Systemmeldungen im Chat angezeigt werden?">
    Die meisten internen/Tool-Meldungen erscheinen nur, wenn für diese Sitzung **ausführliche Ausgabe**, **Ablaufverfolgung** oder **Reasoning** aktiviert ist.

    Beheben Sie dies in dem Chat, in dem die Meldungen angezeigt werden:

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    Wenn es weiterhin zu viele Meldungen gibt: Prüfen Sie die Sitzungseinstellungen in der Control UI und setzen Sie die ausführliche Ausgabe auf **inherit**; stellen Sie sicher, dass Sie kein Bot-Profil verwenden, dessen Konfiguration `verboseDefault: "on"` enthält.

    Docs: [Denken und ausführliche Ausgabe](/de/tools/thinking), [Sicherheit](/de/gateway/security/index#reasoning-and-verbose-output-in-groups).

  </Accordion>

  <Accordion title="Wie stoppe/breche ich eine laufende Aufgabe ab?">
    Senden Sie eine der folgenden Angaben **als eigenständige Nachricht** (ohne Schrägstrich), um einen Abbruch auszulösen: `stop`, `stop action`, `stop current action`, `stop run`, `stop current run`, `stop agent`, `stop the agent`, `stop openclaw`, `openclaw stop`, `stop don't do anything`, `stop do not do anything`, `stop doing anything`, `do not do that`, `please stop`, `stop please`, `abort`, `esc`, `exit`, `interrupt`, `halt`. Gängige nicht englischsprachige Auslöser (Französisch, Deutsch, Spanisch, Chinesisch, Japanisch, Hindi, Arabisch, Russisch) funktionieren ebenfalls.

    Bitten Sie den Agenten bei Hintergrundprozessen, die vom exec-Tool gestartet wurden, Folgendes auszuführen:

    ```text
    process action:kill sessionId:XXX
    ```

    Die meisten Slash-Befehle müssen als **eigenständige** Nachricht gesendet werden, die mit `/` beginnt. Einige Kurzbefehle (wie `/status`) funktionieren für Absender auf der Zulassungsliste jedoch auch inline. Siehe [Slash-Befehle](/de/tools/slash-commands).

  </Accordion>

  <Accordion title='Wie sende ich eine Discord-Nachricht über Telegram? („Kontextübergreifendes Messaging verweigert“)'>
    OpenClaw blockiert standardmäßig das Messaging **zwischen Providern**. Wenn ein Tool-Aufruf an Telegram gebunden ist, sendet er keine Nachricht an Discord, sofern Sie dies nicht ausdrücklich zulassen. Die Änderung wird sofort wirksam; ein Neustart des Gateways ist nicht erforderlich:

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='Warum wirkt es so, als würde der Bot schnell aufeinanderfolgende Nachrichten „ignorieren“?'>
    Während eines laufenden Durchlaufs eingegebene Prompts werden standardmäßig in den aktiven Durchlauf eingesteuert. Verwenden Sie `/queue`, um das Verhalten für den aktiven Durchlauf auszuwählen:

    - `steer` (Standard) – den aktiven Durchlauf an der nächsten Modellgrenze steuern.
    - `followup` – Nachrichten in die Warteschlange stellen und nach dem Ende des aktuellen Durchlaufs einzeln ausführen.
    - `collect` – kompatible Nachrichten in die Warteschlange stellen und nach dem Ende des aktuellen Durchlaufs einmal antworten.
    - `interrupt` – den aktuellen Durchlauf abbrechen und neu beginnen.

    Fügen Sie Warteschlangenmodi Optionen wie `debounce:0.5s cap:25 drop:summarize` hinzu. Siehe [Befehlswarteschlange](/de/concepts/queue) und [Steuerungswarteschlange](/de/concepts/queue-steering).

  </Accordion>
</AccordionGroup>

## Sonstiges

<AccordionGroup>
  <Accordion title='Welches ist das Standardmodell für Anthropic mit einem API-Schlüssel?'>
    Anmeldedaten und Modellauswahl sind voneinander getrennt. Das Festlegen von `ANTHROPIC_API_KEY` (oder das Speichern eines Anthropic-API-Schlüssels in Authentifizierungsprofilen) ermöglicht die Authentifizierung. Das tatsächliche Standardmodell ist jedoch das Modell, das Sie in `agents.defaults.model.primary` konfigurieren (zum Beispiel `anthropic/claude-sonnet-4-6` oder `anthropic/claude-opus-4-6`). `No credentials found for profile "anthropic:default"` bedeutet, dass das Gateway im erwarteten `auth-profiles.json` keine Anthropic-Anmeldedaten für den ausgeführten Agenten finden konnte.
  </Accordion>
</AccordionGroup>

---

Kommen Sie weiterhin nicht weiter? Fragen Sie in [Discord](https://discord.com/invite/clawd) nach oder eröffnen Sie eine [GitHub-Diskussion](https://github.com/openclaw/openclaw/discussions).

## Verwandte Themen

- [FAQ zum ersten Start](/de/help/faq-first-run) – Installation, Onboarding, Authentifizierung, Abonnements, frühe Fehler
- [Modell-FAQ](/de/help/faq-models) – Modellauswahl, Failover, Authentifizierungsprofile
- [Fehlerbehebung](/de/help/troubleshooting) – symptomorientierte Triage
