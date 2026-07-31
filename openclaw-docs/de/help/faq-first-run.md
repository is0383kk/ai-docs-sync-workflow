---
read_when:
    - Neuinstallation, festgefahrenes Onboarding oder Fehler beim ersten Start
    - Authentifizierung und Provider-Abonnements auswählen
    - Kein Zugriff auf docs.openclaw.ai, Dashboard lässt sich nicht öffnen, Installation hängt fest
sidebarTitle: First-run FAQ
summary: 'FAQ: Schnellstart und Ersteinrichtung — Installation, Onboarding, Authentifizierung, Abonnements, anfängliche Fehler'
title: 'FAQ: Ersteinrichtung'
x-i18n:
    generated_at: "2026-07-26T18:29:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

Schnellstart und Fragen und Antworten zur ersten Ausführung. Informationen zum täglichen Betrieb, zu Modellen, zur Authentifizierung, zu Sitzungen
und zur Fehlerbehebung finden Sie in den wichtigsten [häufig gestellten Fragen](/de/help/faq).

## Schnellstart und Einrichtung bei der ersten Ausführung

<AccordionGroup>
  <Accordion title="Ich komme nicht weiter – wie finde ich am schnellsten eine Lösung?">
    Verwenden Sie einen lokalen KI-Agenten, der **Ihren Rechner sehen kann**. Bei den meisten Fällen von „Ich komme nicht weiter“
    handelt es sich um **lokale Konfigurations- oder Umgebungsprobleme**, die eine Remote-Hilfsperson nicht untersuchen kann. Dies ist daher
    effektiver, als in Discord zu fragen.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Stellen Sie dem Agenten über die anpassbare Installation (git) den vollständigen Quellcode-Checkout bereit, damit er
    Code und Dokumentation lesen und die genaue von Ihnen ausgeführte Version berücksichtigen kann:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Bitten Sie den Agenten, die Korrektur schrittweise zu planen und zu überwachen und anschließend nur die
    erforderlichen Befehle auszuführen – kleinere Diffs lassen sich leichter prüfen.

    Teilen Sie diese Ausgaben, wenn Sie um Hilfe bitten (in Discord oder einem GitHub-Issue):

    | Befehl | Zeigt |
    | --- | --- |
    | `openclaw status` | Zustand von Gateway/Agent und grundlegende Konfigurationsübersicht |
    | `openclaw status --all` | Vollständige schreibgeschützte Diagnose zum Einfügen |
    | `openclaw models status` | Provider-Authentifizierung und Modellverfügbarkeit |
    | `openclaw doctor` | Validiert und behebt häufige Konfigurations-/Zustandsprobleme |
    | `openclaw logs --follow` | Live-Protokollausgabe |
    | `openclaw gateway status --deep` | Umfassende Zustandsprüfung von Gateway, Konfiguration und Plugins |
    | `openclaw health --verbose` | Detaillierter Zustandsbericht |

    Haben Sie einen echten Fehler oder eine Lösung gefunden? Erstellen Sie ein Issue oder senden Sie einen PR:
    [Issues](https://github.com/openclaw/openclaw/issues) /
    [Pull Requests](https://github.com/openclaw/openclaw/pulls).

    Schnelle Debugging-Schleife: [Die ersten 60 Sekunden, wenn etwas nicht funktioniert](/de/help/faq#first-60-seconds-if-something-is-broken).
    Installationsdokumentation: [Installation](/de/install), [Installer-Flags](/de/install/installer), [Aktualisierung](/de/install/updating).

  </Accordion>

  <Accordion title="Heartbeat wird ständig übersprungen. Was bedeuten die Gründe dafür?">
    | Grund für das Überspringen | Bedeutung |
    | --- | --- |
    | `quiet-hours` | Außerhalb des konfigurierten Zeitfensters für aktive Stunden |
    | `empty-heartbeat-file` | Der Heartbeat-Monitor-Entwurf ist vorhanden, enthält aber nur leere Elemente, Kommentare, Überschriften, Codeblöcke oder eine leere Checklistenstruktur |
    | `alerts-disabled` | Die gesamte Heartbeat-Sichtbarkeit ist deaktiviert (`showOk`, `showAlerts` und `useIndicator` sind alle deaktiviert) |

    Ältere Heartbeat-Blöcke vom Typ `tasks:` werden mit `openclaw doctor --fix` zu unabhängig geplanten Cron-Aufträgen migriert.

    Dokumentation: [Heartbeat](/de/gateway/heartbeat), [Automatisierung](/de/automation).

  </Accordion>

  <Accordion title="Empfohlene Vorgehensweise zur Installation und Einrichtung von OpenClaw">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    Aus dem Quellcode (Mitwirkende/Entwicklung):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    Noch keine globale Installation? Führen Sie stattdessen `pnpm openclaw onboard` aus. Wenn die Assets der Control UI
    fehlen, versucht das Onboarding, sie selbst zu erstellen, und greift andernfalls auf `pnpm ui:build` zurück.

  </Accordion>

  <Accordion title="Wie öffne ich nach dem Onboarding das Dashboard?">
    Das Onboarding öffnet unmittelbar nach der Einrichtung Ihren Browser mit einer sauberen Dashboard-URL (ohne Token)
    und gibt den Link in der Zusammenfassung aus. Lassen Sie diesen Tab geöffnet. Falls er nicht geöffnet wurde,
    kopieren Sie die ausgegebene URL und fügen Sie sie auf demselben Rechner ein.
  </Accordion>

  <Accordion title="Wie authentifiziere ich das Dashboard auf localhost beziehungsweise aus der Ferne?">
    **Localhost (derselbe Rechner):**

    - Öffnen Sie `http://127.0.0.1:18789/`.
    - Wenn die Authentifizierung über ein gemeinsames Geheimnis angefordert wird, fügen Sie das konfigurierte Token oder Passwort in die Einstellungen der Control UI ein.
    - Token-Quelle: `gateway.auth.token` (oder `OPENCLAW_GATEWAY_TOKEN`).
    - Passwortquelle: `gateway.auth.password` (oder `OPENCLAW_GATEWAY_PASSWORD`).
    - Noch kein gemeinsames Geheimnis konfiguriert? Führen Sie `openclaw doctor --generate-gateway-token` (oder `openclaw doctor --fix --generate-gateway-token`) aus.

    **Nicht auf localhost:**

    - **Tailscale Serve** (empfohlen): Belassen Sie die Bindung auf Loopback, führen Sie `openclaw gateway --tailscale serve` aus und öffnen Sie `https://<magicdns>/`. Mit `gateway.auth.allowTailscale: true` erfüllen Identitätsheader die Authentifizierungsanforderungen der Control UI/WebSocket-Verbindung (kein Einfügen eines gemeinsamen Geheimnisses; ein vertrauenswürdiger Gateway-Host wird vorausgesetzt). HTTP-APIs benötigen weiterhin die Authentifizierung über ein gemeinsames Geheimnis, sofern Sie nicht bewusst Private-Ingress über `none` oder die HTTP-Authentifizierung über einen vertrauenswürdigen Proxy verwenden.
      Gleichzeitige Serve-Versuche mit fehlerhafter Authentifizierung vom selben Client werden serialisiert, bevor die Begrenzung für fehlgeschlagene Authentifizierungen sie erfasst. Daher kann bereits beim zweiten fehlerhaften Versuch `retry later` angezeigt werden.
    - **Tailnet-Bindung**: Führen Sie `openclaw gateway --bind tailnet --token "<token>"` aus (oder konfigurieren Sie die Passwortauthentifizierung), öffnen Sie `http://<tailscale-ip>:18789/` und fügen Sie das passende gemeinsame Geheimnis in die Dashboard-Einstellungen ein.
    - **Identitätsbewusster Reverse-Proxy**: Belassen Sie das Gateway hinter einem vertrauenswürdigen Proxy, legen Sie `gateway.auth.mode: "trusted-proxy"` fest und öffnen Sie die Proxy-URL. Loopback-Proxys auf demselben Host benötigen ausdrücklich `gateway.auth.trustedProxy.allowLoopback: true`.
    - **SSH-Tunnel**: `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`, öffnen Sie anschließend `http://127.0.0.1:18789/`. Die Authentifizierung über ein gemeinsames Geheimnis gilt auch über den Tunnel. Fügen Sie bei Aufforderung das konfigurierte Token oder Passwort ein.

    Informationen zu Bindungsmodi und Authentifizierungsdetails finden Sie unter [Dashboard](/de/web/dashboard) und [Web-Oberflächen](/de/web).

  </Accordion>

  <Accordion title="Warum gibt es zwei Konfigurationen für die Bestätigung von exec-Ausführungen im Chat?">
    Sie steuern unterschiedliche Ebenen:

    - `approvals.exec` – leitet Bestätigungsaufforderungen an Chat-Ziele weiter.
    - `channels.<channel>.execApprovals` – macht diesen Kanal zu einem nativen Bestätigungsclient für exec-Ausführungen.

    Die exec-Richtlinie des Hosts bleibt die eigentliche Bestätigungsschranke. Die Chat-Konfiguration steuert lediglich, wo
    Aufforderungen angezeigt werden und wie Personen darauf antworten.

    Sie benötigen nur selten beides:

    - Wenn der Chat bereits Befehle und Antworten unterstützt, funktioniert `/approve` im selben Chat über den gemeinsamen Pfad.
    - Wenn ein unterstützter nativer Kanal die bestätigenden Personen sicher ermitteln kann, aktiviert OpenClaw automatisch native Bestätigungen mit vorrangiger Direktnachricht, wenn `channels.<channel>.execApprovals.enabled` nicht festgelegt ist oder `"auto"` entspricht.
    - Wenn native Bestätigungskarten/-schaltflächen verfügbar sind, ist diese Oberfläche maßgeblich. Erwähnen Sie einen manuellen Befehl `/approve` nur, wenn das Tool-Ergebnis angibt, dass Chat-Bestätigungen nicht verfügbar sind.
    - Verwenden Sie `approvals.exec` nur, wenn Aufforderungen zusätzlich andere Chats oder ausdrücklich angegebene Betriebsräume erreichen müssen.
    - Verwenden Sie `channels.<channel>.execApprovals.target: "channel"` oder `"both"` nur, wenn Bestätigungsaufforderungen erneut im ursprünglichen Raum/Thema veröffentlicht werden sollen.
    - Plugin-Bestätigungen sind separat: standardmäßig `/approve` im selben Chat, optional eine Weiterleitung über `approvals.plugin`, und nur einige native Kanäle behalten auch dafür die native Verarbeitung bei.

    Kurz gesagt: Die Weiterleitung dient dem Routing, die native Clientkonfiguration einer umfangreicheren kanalspezifischen Benutzererfahrung.
    Siehe [Bestätigungen für exec-Ausführungen](/de/tools/exec-approvals).

  </Accordion>

  <Accordion title="Welche Laufzeitumgebung benötige ich?">
    Node **22.22.3+**, **24.15+** oder **25.9+** ist erforderlich (Node 24 empfohlen). `pnpm` ist der Paketmanager des Repositorys.
    Bun kann Abhängigkeiten installieren und Paketskripte ausführen, aber nicht die OpenClaw-CLI oder das Gateway, da `node:sqlite` fehlt.
  </Accordion>

  <Accordion title="Läuft es auf Raspberry Pi?">
    Ja, prüfen Sie aber zuerst den RAM: Pi 5 und Pi 4 (2 GB+) sind ideal; Pi 3B+ (1 GB) funktioniert, ist aber langsam; Pi Zero 2 W (512 MB) wird nicht empfohlen.

    | Modell | RAM | Eignung |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | Am besten |
    | Pi 4 | 4 GB | Gut |
    | Pi 4 | 2 GB | In Ordnung, Auslagerungsspeicher hinzufügen |
    | Pi 4 | 1 GB | Knapp |
    | Pi 3B+ | 1 GB | Langsam |
    | Pi Zero 2 W | 512 MB | Nicht empfohlen |

    Absolutes Minimum: 1 GB RAM, 1 Kern, 500 MB freier Speicherplatz, 64-Bit-Betriebssystem. Da der Pi nur
    das Gateway ausführt (Modelle greifen auf Cloud-APIs zu), bewältigt selbst ein einfacher Pi die Last.

    Ein kleiner Pi/VPS kann auch nur das Gateway hosten, während Sie **Nodes** auf Ihrem
    Laptop/Telefon koppeln, um lokal Bildschirm, Kamera oder Canvas zu verwenden oder Befehle auszuführen. Siehe [Nodes](/de/nodes).

    Vollständige Einrichtungsanleitung: [Raspberry Pi](/de/install/raspberry-pi).

  </Accordion>

  <Accordion title="Gibt es Tipps für Installationen auf Raspberry Pi?">
    - Verwenden Sie ein **64-Bit**-Betriebssystem; verwenden Sie kein 32-Bit-Raspberry-Pi-OS.
    - Fügen Sie auf Boards mit 2 GB oder weniger Auslagerungsspeicher hinzu.
    - Bevorzugen Sie hinsichtlich Leistung und Lebensdauer eine **USB-SSD** gegenüber einer SD-Karte.
    - Bevorzugen Sie die anpassbare Installation (git), damit Sie Protokolle einsehen und schnelle Aktualisierungen durchführen können.
    - Beginnen Sie ohne Kanäle/Skills und fügen Sie sie einzeln hinzu.
    - Ungewöhnliche Binärdateifehler („exec format error“) sind normalerweise auf einen fehlenden ARM64-Build für ein optionales Skill-Tool zurückzuführen.

    Vollständiger Leitfaden: [Raspberry Pi](/de/install/raspberry-pi). Siehe auch [Linux](/de/platforms/linux).

  </Accordion>

  <Accordion title="„Wake up my friend“ hängt fest/das Onboarding startet nicht. Was nun?">
    Dieser Bildschirm setzt voraus, dass das Gateway erreichbar und authentifiziert ist. Die TUI sendet beim ersten Start außerdem automatisch
    „Wake up, my friend!“, wenn ein Modell-Provider konfiguriert ist. Wenn Sie die Einrichtung von Modell/Authentifizierung
    übersprungen haben, zeigt das Onboarding den Hinweis „Model auth missing“ an und öffnet die
    TUI, ohne etwas zu senden – fügen Sie mit `openclaw configure --section model` einen Provider hinzu.
    Wenn Sie die Aktivierungszeile **ohne Antwort** sehen und die Token-Anzahl bei 0 bleibt, wurde der Agent nie ausgeführt.

    1. Starten Sie das Gateway neu:

    ```bash
    openclaw gateway restart
    ```

    2. Prüfen Sie Status und Authentifizierung:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Hängt es weiterhin? Führen Sie Folgendes aus:

    ```bash
    openclaw doctor
    ```

    Wenn sich das Gateway auf einem anderen Rechner befindet, vergewissern Sie sich, dass die Tunnel-/Tailscale-Verbindung besteht und die UI
    auf das richtige Gateway verweist. Siehe [Remote-Zugriff](/de/gateway/remote).

  </Accordion>

  <Accordion title="Kann ich meine Einrichtung auf einen neuen Rechner migrieren, ohne das Onboarding zu wiederholen?">
    Ja. Kopieren Sie das **Zustandsverzeichnis** und den **Arbeitsbereich** und führen Sie anschließend einmal Doctor aus:

    1. Installieren Sie OpenClaw auf dem neuen Rechner.
    2. Kopieren Sie `$OPENCLAW_STATE_DIR` (Standard: `~/.openclaw`) vom alten Rechner.
    3. Kopieren Sie Ihren Arbeitsbereich (Standard: `~/.openclaw/workspace`).
    4. Führen Sie `openclaw doctor` aus und starten Sie den Gateway-Dienst neu.

    Dadurch bleiben Konfiguration, Authentifizierungsprofile, WhatsApp-Anmeldedaten, Sitzungen und Speicher erhalten – Ihr Bot bleibt
    exakt gleich, solange Sie **beide** Speicherorte kopieren. Im Remote-Modus verwaltet der
    Gateway-Host den Sitzungsspeicher und den Arbeitsbereich.

    **Wichtig:** Wenn Sie nur Ihren Arbeitsbereich auf GitHub committen/pushen, sichern Sie
    **Speicher und Bootstrap-Dateien**, aber weder den Sitzungsverlauf noch die Authentifizierung. Diese befinden sich unter
    `~/.openclaw/` (beispielsweise `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`).

    Verwandte Themen: [Migration](/de/install/migrating), [Speicherorte auf dem Datenträger](/de/help/faq#where-things-live-on-disk),
    [Agent-Arbeitsbereich](/de/concepts/agent-workspace), [Doctor](/de/gateway/doctor),
    [Remote-Modus](/de/gateway/remote).

  </Accordion>

  <Accordion title="Wo kann ich sehen, was in der neuesten Version neu ist?">
    Sehen Sie im GitHub-Änderungsprotokoll nach:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Die neuesten Einträge stehen oben. Wenn der oberste Abschnitt **Unveröffentlicht** ist, ist der nächste datierte
    Abschnitt die neueste veröffentlichte Version. Die Einträge sind unter **Highlights**, **Änderungen**
    und **Fehlerbehebungen** gruppiert (sowie bei Bedarf in Abschnitten für Dokumentation und Sonstiges).

  </Accordion>

  <Accordion title="Kein Zugriff auf docs.openclaw.ai (SSL-Fehler)">
    Einige Comcast-/Xfinity-Verbindungen blockieren `docs.openclaw.ai` fälschlicherweise über Xfinity
    Advanced Security. Deaktivieren Sie diese Funktion oder setzen Sie `docs.openclaw.ai` auf die Zulassungsliste und versuchen Sie es erneut. Helfen Sie uns,
    die Sperrung aufzuheben: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Immer noch blockiert? Die Dokumentation wird auf GitHub gespiegelt:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Unterschied zwischen Stable und Beta">
    **Stable** und **Beta** sind **npm-dist-tags**, keine separaten Codezweige:

    - `latest` = Stable
    - `beta` = früher Build zum Testen (fällt auf `latest` zurück, wenn Beta fehlt oder älter als die aktuelle Stable-Version ist)

    Ein Stable-Release landet normalerweise zuerst unter **Beta**; anschließend verschiebt ein expliziter
    Hochstufungsschritt dieselbe Version nach `latest`, ohne die Versionsnummer zu ändern. Maintainer
    können auch direkt unter `latest` veröffentlichen. Deshalb können Beta und Stable nach der
    Hochstufung auf **dieselbe Version** verweisen.

    Änderungen anzeigen: [CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md).

    Einzeilige Installationsbefehle und den Unterschied zwischen Beta und Dev finden Sie im nächsten Akkordeon.

  </Accordion>

  <Accordion title="Wie installiere ich die Beta-Version und worin besteht der Unterschied zwischen Beta und Dev?">
    **Beta** ist das npm-dist-tag `beta` (kann nach der Hochstufung `latest` entsprechen).
    **Dev** ist der sich fortlaufend ändernde Stand von `main` (Git); bei einer Veröffentlichung auf npm verwendet er das dist-tag `dev`.

    Einzeilige Befehle (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows-Installationsprogramm (PowerShell): `iwr -useb https://openclaw.ai/install.ps1 | iex`

    Weitere Details: [Entwicklungskanäle](/de/install/development-channels) und [Installationsparameter](/de/install/installer).

  </Accordion>

  <Accordion title="Wie teste ich die neuesten Änderungen?">
    Zwei Möglichkeiten:

    1. **Dev-Kanal (vorhandene Installation):**

    ```bash
    openclaw update --channel dev
    ```

    Dies wechselt zu einem Git-Checkout von `main`, führt einen Rebase auf den Upstream-Stand durch, erstellt den Build und installiert
    die CLI aus diesem Checkout.

    2. **Anpassbare Git-Installation (neues System):**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Bevorzugen Sie einen manuellen Klonvorgang:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Dokumentation: [Aktualisieren](/de/cli/update), [Entwicklungskanäle](/de/install/development-channels), [Installation](/de/install).

  </Accordion>

  <Accordion title="Wie lange dauern Installation und Onboarding normalerweise?">
    Grobe Orientierung:

    - **Installation:** 2-5 Minuten.
    - **QuickStart-Onboarding:** einige Minuten (Loopback-Gateway, automatisches Token, Standard-Workspace).
    - **Erweitertes/vollständiges Onboarding:** länger, wenn die Provider-Anmeldung, Kanalkopplung, Daemon-Installation, Netzwerk-Downloads oder Skills zusätzliche Einrichtung erfordern.

    Der Assistent zeigt diesen Zeitrahmen vorab an. Überspringen Sie optionale Schritte und kehren Sie später mit
    `openclaw configure` zurück.

    Hängt der Vorgang? Siehe oben [Ich komme nicht weiter](#quick-start-and-first-run-setup).

  </Accordion>

  <Accordion title="Installationsprogramm hängt? Wie erhalte ich mehr Rückmeldungen?">
    Führen Sie es erneut mit `--verbose` aus:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` verfügt über keinen eigenen Schalter für ausführliche Ausgaben; führen Sie es stattdessen über `Set-PSDebug -Trace 1` /
    `-Trace 0` aus. Vollständige Parameterreferenz: [Installationsparameter](/de/install/installer).

  </Accordion>

  <Accordion title="Die Windows-Installation meldet, dass Git nicht gefunden oder OpenClaw nicht erkannt wurde">
    Zwei häufige Windows-Probleme:

    **1) npm-Fehler „spawn git“ / Git nicht gefunden**

    - Installieren Sie **Git for Windows** und stellen Sie sicher, dass `git` im PATH enthalten ist.
    - Schließen Sie PowerShell, öffnen Sie es erneut und führen Sie das Installationsprogramm noch einmal aus.

    **2) OpenClaw wird nach der Installation nicht erkannt**

    - Der globale bin-Ordner von npm ist nicht im PATH enthalten.
    - Überprüfen Sie ihn: `npm config get prefix`.
    - Fügen Sie dieses Verzeichnis Ihrem Benutzer-PATH hinzu (kein `\bin`-Suffix erforderlich; auf den meisten Systemen lautet es `%AppData%\npm`).
    - Schließen Sie PowerShell und öffnen Sie es erneut.

    Bevorzugen Sie eine Desktop-App? Verwenden Sie **Windows Hub**. Für eine reine Terminal-Einrichtung werden sowohl das PowerShell-
    Installationsprogramm als auch WSL2-Gateway-Pfade unterstützt. Dokumentation: [Windows](/de/platforms/windows).

  </Accordion>

  <Accordion title="Die Windows-Ausgabe von exec zeigt verstümmelten chinesischen Text – was soll ich tun?">
    In der Regel liegt in nativen Windows-Shells eine nicht übereinstimmende Konsolen-Codepage vor.

    Symptome: Die Ausgabe von `system.run`/`exec` stellt Chinesisch als unleserlichen Zeichensalat dar; derselbe Befehl
    wird in einem anderen Terminalprofil korrekt angezeigt.

    Behelfslösung in PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Starten Sie anschließend das Gateway neu und versuchen Sie es erneut:

    ```powershell
    openclaw gateway restart
    ```

    Tritt dies mit der neuesten OpenClaw-Version weiterhin auf? Verfolgen/melden Sie es hier: [Issue #30640](https://github.com/openclaw/openclaw/issues/30640).

  </Accordion>

  <Accordion title="Die Dokumentation hat meine Frage nicht beantwortet – wie erhalte ich eine bessere Antwort?">
    Verwenden Sie die anpassbare Git-Installation, damit der vollständige Quellcode und die Dokumentation lokal verfügbar sind. Fragen Sie anschließend
    Ihren Bot (oder Claude/Codex) **aus diesem Ordner heraus**, damit er das Repository lesen und präzise antworten kann.

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Weitere Details: [Installation](/de/install) und [Installationsparameter](/de/install/installer).

  </Accordion>

  <Accordion title="Wie installiere ich OpenClaw unter Linux?">
    - Schnelleinrichtung unter Linux und Dienstinstallation: [Linux](/de/platforms/linux).
    - Vollständige Anleitung: [Erste Schritte](/de/start/getting-started).
    - Installationsprogramm und Aktualisierungen: [Installation und Aktualisierungen](/de/install/updating).

  </Accordion>

  <Accordion title="Wie installiere ich OpenClaw auf einem VPS?">
    Jeder Linux-VPS ist geeignet. Installieren Sie OpenClaw auf dem Server und greifen Sie anschließend über SSH/Tailscale auf das Gateway zu.

    Anleitungen: [exe.dev](/de/install/exe-dev), [Hetzner](/de/install/hetzner), [Fly.io](/de/install/fly).
    Remotezugriff: [Gateway-Remotezugriff](/de/gateway/remote).

  </Accordion>

  <Accordion title="Wo finde ich die Installationsanleitungen für Cloud/VPS?">
    Hosting-Übersicht für gängige Provider:

    - [VPS-Hosting](/de/vps) (alle Provider an einem Ort)
    - [Fly.io](/de/install/fly)
    - [Hetzner](/de/install/hetzner)
    - [exe.dev](/de/install/exe-dev)

    In der Cloud **wird das Gateway auf dem Server ausgeführt**, und Sie greifen von Ihrem Laptop/Telefon
    über die Control UI (oder Tailscale/SSH) darauf zu. Ihr Zustand und Workspace befinden sich auf dem Server;
    behandeln Sie den Host daher als maßgebliche Datenquelle und sichern Sie ihn.

    Koppeln Sie **Nodes** (Mac/iOS/Android/headless) mit diesem Cloud-Gateway, um lokale
    Bildschirm-/Kamera-/Canvas-Funktionen oder die Befehlsausführung auf Ihrem Laptop zu nutzen, während das Gateway in
    der Cloud verbleibt.

    Übersicht: [Plattformen](/de/platforms). Remotezugriff: [Gateway-Remotezugriff](/de/gateway/remote).
    Nodes: [Nodes](/de/nodes), [Nodes-CLI](/de/cli/nodes).

  </Accordion>

  <Accordion title="Kann ich OpenClaw anweisen, sich selbst zu aktualisieren?">
    Möglich, aber nicht empfohlen. Der Aktualisierungsvorgang kann das Gateway neu starten (wodurch die
    aktive Sitzung getrennt wird), einen sauberen Git-Checkout erfordern und eine Bestätigung abfragen.
    Sicherer ist es, Aktualisierungen als Betreiber über eine Shell auszuführen.

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Automatisierung durch einen Agenten:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Dokumentation: [Aktualisieren](/de/cli/update), [Aktualisierung](/de/install/updating).

  </Accordion>

  <Accordion title="Was geschieht beim Onboarding tatsächlich?">
    `openclaw onboard` ist der empfohlene Einrichtungsweg. Im **lokalen Modus** führt es durch folgende Schritte:

    1. **Modell/Authentifizierung** – Provider-OAuth, API-Schlüssel oder manuelle Authentifizierung (einschließlich lokaler Optionen wie LM Studio); Auswahl eines Standardmodells.
    2. **Workspace** – Speicherort und Bootstrap-Dateien.
    3. **Gateway** – Port, Bind-Adresse, Authentifizierungsmodus, Tailscale-Freigabe.
    4. **Kanäle** – integrierte und offizielle Plugin-Chatkanäle: iMessage, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp und weitere.
    5. **Daemon** – LaunchAgent (macOS), systemd-Benutzereinheit (Linux/WSL2) oder native geplante Windows-Aufgabe.
    6. **Integritätsprüfung** – startet das Gateway und überprüft, ob es ausgeführt wird.
    7. **Skills** – installiert empfohlene Skills und optionale Abhängigkeiten.

    Zu Beginn werden Angaben zur erwarteten Dauer angezeigt, und es erfolgt eine Warnung, wenn Ihr konfiguriertes Modell unbekannt ist
    oder die Authentifizierung fehlt. Vollständige Aufschlüsselung: [Onboarding (CLI)](/de/start/wizard).

  </Accordion>

  <Accordion title="Benötige ich ein Claude- oder OpenAI-Abonnement, um dies auszuführen?">
    Nein. Führen Sie OpenClaw mit **API-Schlüsseln** (Anthropic/OpenAI/andere) oder **ausschließlich lokalen Modellen** aus,
    sodass Ihre Daten auf Ihrem Gerät verbleiben. Abonnements (Claude Pro/Max, ChatGPT/Codex) sind
    optionale Möglichkeiten zur Authentifizierung bei diesen Providern.

    Bei Anthropic ermöglicht ein **API-Schlüssel** die standardmäßige nutzungsabhängige Abrechnung; die **Claude CLI**
    verwendet eine vorhandene Claude-Code-Anmeldung auf demselben Host erneut. Anthropic behandelt derzeit
    den nicht interaktiven `claude -p`-Pfad der Claude CLI als Agent-SDK-/programmatische Nutzung, die
    weiterhin auf die Limits Ihres Abonnementtarifs angerechnet wird. Prüfen Sie die aktuelle Abrechnungsdokumentation
    von Anthropic, bevor Sie sich auf das Abonnementverhalten verlassen. Für dauerhaft betriebene Gateway-Hosts und gemeinsam genutzte
    Automatisierungen ist ein Anthropic-API-Schlüssel die besser vorhersehbare Wahl.

    OpenAI-Codex-OAuth (ChatGPT-/Codex-Abonnement) wird für Agentenmodelle vollständig unterstützt.
    OpenClaw unterstützt außerdem gehostete abonnementbasierte Optionen, darunter den **Qwen Cloud
    Coding Plan**, **MiniMax Coding Plan** und **Z.AI / GLM Coding Plan**.

    Dokumentation: [Anthropic](/de/providers/anthropic), [OpenAI](/de/providers/openai),
    [Qwen Cloud](/de/providers/qwen), [MiniMax](/de/providers/minimax), [Z.AI (GLM)](/de/providers/zai),
    [Lokale Modelle](/de/gateway/local-models), [Modelle](/de/concepts/models).

  </Accordion>

  <Accordion title="Kann ich ein Claude-Max-Abonnement ohne API-Schlüssel verwenden?">
    Ja. OpenClaw unterstützt die Wiederverwendung der Claude CLI für Pro-/Max-/Team-/Enterprise-Tarife. Anthropic
    behandelt den von OpenClaw verwendeten `claude -p`-Pfad derzeit als Nutzung des Abonnementtarifs, die
    den Limits Ihres Tarifs unterliegt, und nicht als separates kostenloses Kontingent. Unter
    [Anthropic](/de/providers/anthropic) finden Sie aktuelle Abrechnungsdetails und Links zu den
    Supportartikeln von Anthropic. Verwenden Sie stattdessen einen
    Anthropic-API-Schlüssel, um eine möglichst vorhersehbare serverseitige Einrichtung zu erhalten.
  </Accordion>

  <Accordion title="Unterstützen Sie die Authentifizierung per Claude-Abonnement (Claude Pro oder Max)?">
    Ja, über die Wiederverwendung der Claude CLI. Die Abrechnungsweise von Anthropic für die Nutzung von `claude -p`/Agent SDK
    hat sich im Laufe der Zeit geändert. Unter [Anthropic](/de/providers/anthropic) finden Sie den aktuellen Stand und
    datierte Links zu den Supportartikeln von Anthropic, bevor Sie sich auf ein bestimmtes Abrechnungsverhalten
    verlassen.

    Die Anthropic-Authentifizierung per Setup-Token wird ebenfalls weiterhin als Token-Pfad unterstützt, OpenClaw bevorzugt jedoch
    die Wiederverwendung der Claude CLI und `claude -p`, sofern verfügbar. Für Produktions- oder Mehrbenutzer-
    Workloads bleibt ein Anthropic-API-Schlüssel die sicherere und besser vorhersehbare Wahl. Weitere
    gehostete Optionen mit Abonnementmodell: [OpenAI](/de/providers/openai), [Qwen Cloud](/de/providers/qwen),
    [MiniMax](/de/providers/minimax), [Z.AI (GLM)](/de/providers/zai).

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="Warum wird bei Anthropic der HTTP-Fehler 429 rate_limit_error angezeigt?">
    Ihr **Anthropic-Kontingent bzw. -Ratenlimit** ist für das aktuelle Zeitfenster ausgeschöpft. Warten Sie bei der **Claude
    CLI**, bis das Zeitfenster zurückgesetzt wird, oder führen Sie ein Upgrade Ihres Tarifs durch. Prüfen Sie bei einem **Anthropic-API-Schlüssel**
    die Nutzung und Abrechnung in der Anthropic Console und erhöhen Sie die Limits nach Bedarf.

    Wenn die Meldung ausdrücklich `Extra usage is required for long context requests` lautet,
    versucht die Anfrage, das 1M-Kontextfenster von Anthropic zu verwenden (ein allgemein verfügbares 1M-Modell der Claude-4.x-
    Reihe oder die veraltete Konfiguration `params.context1m: true`), und Ihre aktuellen Anmeldedaten sind nicht
    für die Abrechnung langer Kontexte berechtigt.

    Legen Sie ein **Fallback-Modell** fest, damit OpenClaw weiterhin antwortet, während das Ratenlimit eines Providers erreicht ist.
    Siehe [Modelle](/de/cli/models), [OAuth](/de/concepts/oauth) und
    [Anthropic 429: Zusätzliche Nutzung für langen Kontext erforderlich](/de/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="Wird AWS Bedrock unterstützt?">
    Ja. OpenClaw verfügt über einen gebündelten Provider für **Amazon Bedrock (Converse)**. Wenn AWS-Umgebungs-
    marker vorhanden sind (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`, `AWS_BEARER_TOKEN_BEDROCK`),
    aktiviert OpenClaw den impliziten Bedrock-Provider für die Modellerkennung automatisch; legen Sie andernfalls
    `plugins.entries.amazon-bedrock.config.discovery.enabled: true` fest oder fügen Sie einen manuellen
    Provider-Eintrag hinzu. Siehe [Amazon Bedrock](/de/providers/bedrock) und [Modell-Provider](/de/providers/models).
    Ein OpenAI-kompatibler Proxy vor Bedrock ist weiterhin eine gültige Option, wenn Sie einen verwalteten Schlüsselfluss bevorzugen.
  </Accordion>

  <Accordion title="Wie funktioniert die Codex-Authentifizierung?">
    OpenClaw unterstützt **OpenAI Codex** über OAuth (ChatGPT-Anmeldung). Eine neue
    Einrichtung ohne primäres Modell verwendet exakt `openai/gpt-5.6-sol` für die
    ChatGPT-/Codex-Abonnementauthentifizierung sowie die native Ausführung des Codex-App-Servers.
    Bei einer erneuten Authentifizierung bleibt ein vorhandenes explizites Modell erhalten, einschließlich
    `openai/gpt-5.5`. Wenn der Codex-Arbeitsbereich GPT-5.6 nicht bereitstellt, wählen Sie
    ausdrücklich `openai/gpt-5.5`; OpenClaw führt kein stillschweigendes Downgrade durch. Veraltete
    Modellreferenzen mit Codex-Präfix sind veraltete Konfigurationen, die durch `openclaw doctor
    --fix` repariert werden. Der direkte Zugriff per OpenAI-API-Schlüssel bleibt für OpenAI-
    API-Oberflächen außerhalb von Agenten und über ein geordnetes API-Schlüsselprofil `openai` auch für Agenten-
    modelle verfügbar. Siehe [Modell-Provider](/de/concepts/model-providers) und
    [Onboarding (CLI)](/de/start/wizard).
  </Accordion>

  <Accordion title="Warum erwähnt OpenClaw weiterhin das veraltete OpenAI-Codex-Präfix?">
    `openai` ist die aktuelle ID des Providers und Authentifizierungsprofils sowohl für OpenAI-API-Schlüssel als auch für
    ChatGPT-/Codex-OAuth – OpenAI Codex ist darin aufgegangen. In älteren Konfigurationen und Migrationswarnungen
    wird möglicherweise weiterhin das veraltete Präfix `openai-codex` angezeigt:

    - `openai/gpt-5.6-sol` = neue Einrichtung eines ChatGPT-/Codex-Abonnements mit der nativen Codex-Laufzeit für Agentendurchläufe.
    - `openai/gpt-5.5` = explizite unterstützte Auswahl für vorhandene Konfigurationen oder Konten ohne Zugriff auf GPT-5.6.
    - Veraltete Modellreferenzen `openai-codex/*` = veraltete Route, die durch `openclaw doctor --fix` repariert wird.
    - `openai/gpt-5.5` plus ein geordnetes API-Schlüsselprofil `openai` = Authentifizierung per API-Schlüssel für ein OpenAI-Agentenmodell.
    - Veraltete Authentifizierungsprofil-IDs `openai-codex` = veraltete IDs, die durch `openclaw doctor --fix` migriert werden.

    Möchten Sie direkt über die OpenAI Platform abrechnen? Legen Sie `OPENAI_API_KEY` fest. Möchten Sie die ChatGPT-/Codex-
    Abonnementauthentifizierung verwenden? Führen Sie `openclaw models auth login --provider openai` aus. Belassen Sie
    Modellreferenzen unter dem kanonischen Provider `openai/*`. Eine neue Abonnement-
    einrichtung verwendet exakt `openai/gpt-5.6-sol`; Doctor repariert veraltete Referenzen
    mit Codex-Präfix, ohne eine explizite Auswahl von `openai/gpt-5.5` zu aktualisieren.

  </Accordion>

  <Accordion title="Warum können sich die Codex-OAuth-Limits von denen im ChatGPT-Web unterscheiden?">
    Codex OAuth verwendet von OpenAI verwaltete, tarifabhängige Kontingentzeitfenster, die sich selbst beim
    selben Konto von der Nutzung auf der ChatGPT-Website bzw. in der App unterscheiden können.

    `openclaw models status` zeigt die derzeit sichtbaren Nutzungs- und Kontingentzeitfenster des Providers an,
    erfindet oder überführt ChatGPT-Web-Berechtigungen jedoch nicht in direkten API-Zugriff. Verwenden Sie für den
    direkten Abrechnungs- und Limitpfad der OpenAI Platform `openai/*` mit einem API-Schlüssel.

  </Accordion>

  <Accordion title="Wird die OpenAI-Abonnementauthentifizierung (Codex OAuth) unterstützt?">
    Ja, vollständig. OpenAI erlaubt ausdrücklich die Nutzung von Abonnement-OAuth in externen
    Tools und Workflows wie OpenClaw. Das Onboarding kann den OAuth-Ablauf für Sie ausführen.

    Siehe [OAuth](/de/concepts/oauth), [Modell-Provider](/de/concepts/model-providers) und [Onboarding (CLI)](/de/start/wizard).

  </Accordion>

  <Accordion title="Wie richte ich Gemini-CLI-OAuth ein?">
    Die Gemini CLI verwendet einen **Plugin-Authentifizierungsablauf**, keine Client-ID und kein Geheimnis in `openclaw.json`.

    1. Installieren Sie die Gemini CLI lokal, sodass `gemini` in `PATH` verfügbar ist:
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Aktivieren Sie das Plugin: `openclaw plugins enable google`
    3. Anmelden: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Standardmodell nach der Anmeldung: `google/gemini-3.1-pro-preview` (Laufzeit `google-gemini-cli`)
    5. Schlagen Anfragen nach der Anmeldung fehl? Legen Sie `GOOGLE_CLOUD_PROJECT` oder `GOOGLE_CLOUD_PROJECT_ID` auf dem Gateway-Host fest und versuchen Sie es erneut.

    OAuth-Token werden in Authentifizierungsprofilen auf dem Gateway-Host gespeichert. Details: [Google](/de/providers/google), [Modell-Provider](/de/concepts/model-providers).

  </Accordion>

  <Accordion title="Ist ein lokales Modell für ungezwungene Chats geeignet?">
    Normalerweise nicht. OpenClaw benötigt einen großen Kontext und starke Sicherheitsmechanismen; kleine Grafikkarten kürzen den Kontext
    und umgehen providerseitige Sicherheitsfilter. Falls unvermeidbar, führen Sie lokal den **größten**
    möglichen Modell-Build aus (LM Studio) – siehe [Lokale Modelle](/de/gateway/local-models). Kleinere bzw. quantisierte
    Modelle erhöhen das Risiko von Prompt-Injection – siehe [Sicherheit](/de/gateway/security).
  </Accordion>

  <Accordion title="Wie halte ich den Datenverkehr gehosteter Modelle in einer bestimmten Region?">
    Wählen Sie an eine Region gebundene Endpunkte. OpenRouter bietet in den USA gehostete Optionen für MiniMax, Kimi
    und GLM; wählen Sie die in den USA gehostete Variante, damit die Daten in der Region verbleiben. Sie können Anthropic/OpenAI
    weiterhin gemeinsam mit diesen unter `models.mode: "merge"` aufführen, damit Fallbacks verfügbar
    bleiben und gleichzeitig der ausgewählte regionale Provider berücksichtigt wird.
  </Accordion>

  <Accordion title="Muss ich für die Installation einen Mac Mini kaufen?">
    Nein. OpenClaw läuft unter macOS oder Linux (Windows über WSL2). Ein Mac mini ist eine beliebte Wahl
    als ständig verfügbarer Host, aber auch ein kleiner VPS, Heimserver oder ein System der Raspberry-Pi-Klasse funktioniert.

    Sie benötigen einen Mac nur **für Tools, die ausschließlich unter macOS verfügbar sind**. Verwenden Sie für iMessage [iMessage](/de/channels/imessage)
    mit `imsg` auf einem beliebigen Mac, der bei Messages angemeldet ist. Wenn der Gateway unter Linux oder anderswo ausgeführt wird,
    legen Sie `channels.imessage.cliPath` auf einen SSH-Wrapper fest, der `imsg` auf diesem Mac ausführt. Führen Sie für andere
    ausschließlich unter macOS verfügbare Tools den Gateway auf einem Mac aus oder koppeln Sie eine macOS-Node.

    Dokumentation: [iMessage](/de/channels/imessage), [Nodes](/de/nodes), [Mac-Remote-Modus](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Benötige ich einen Mac mini für die Unterstützung von iMessage?">
    Sie benötigen **ein beliebiges macOS-Gerät**, das bei Messages angemeldet ist – nicht zwingend einen Mac mini,
    jeder Mac funktioniert. Verwenden Sie [iMessage](/de/channels/imessage) mit `imsg`; der Gateway kann auf diesem
    Mac oder mit einem SSH-Wrapper `cliPath` an anderer Stelle ausgeführt werden.

    Übliche Konfigurationen:

    - Gateway unter Linux/auf einem VPS, wobei `channels.imessage.cliPath` auf einen SSH-Wrapper festgelegt ist, der `imsg` auf einem bei Messages angemeldeten Mac ausführt.
    - Alles auf einem Mac für die einfachste Konfiguration mit nur einem Computer.

    Dokumentation: [iMessage](/de/channels/imessage), [Nodes](/de/nodes), [Mac-Remote-Modus](/de/platforms/mac/remote).

  </Accordion>

  <Accordion title="Kann ich einen Mac mini für OpenClaw kaufen und ihn mit meinem MacBook Pro verbinden?">
    Ja. Auf dem **Mac mini kann der Gateway ausgeführt werden**, und Ihr MacBook Pro verbindet sich als **Node**
    (Begleitgerät). Nodes führen den Gateway nicht aus – sie ergänzen Funktionen wie
    Bildschirm, Kamera, Canvas und `system.run` auf diesem Gerät.

    Typisches Muster: Der Gateway läuft auf dem ständig verfügbaren Mac mini; auf dem MacBook Pro wird die macOS-App oder ein
    Node-Host ausgeführt und mit dem Gateway gekoppelt. Prüfen Sie dies mit `openclaw nodes status` / `openclaw nodes list`.

    Dokumentation: [Nodes](/de/nodes), [Nodes-CLI](/de/cli/nodes).

  </Accordion>

  <Accordion title="Kann ich Bun verwenden?">
    Sie können Bun verwenden, um Abhängigkeiten zu installieren oder Paketskripte auszuführen. Die OpenClaw-CLI und der
    Gateway benötigen **Node**, da der kanonische Zustandsspeicher `node:sqlite` verwendet; Bun stellt
    diese API nicht bereit.
  </Accordion>

  <Accordion title="Telegram: Was gehört in allowFrom?">
    `channels.telegram.allowFrom` ist die **Telegram-Benutzer-ID des menschlichen Absenders** (numerisch),
    nicht der Benutzername des Bots. Die Einrichtung fragt ausschließlich nach numerischen Benutzer-IDs; `openclaw doctor --fix`
    kann versuchen, veraltete Einträge vom Typ `@username` aufzulösen.

    Sicherer (kein Drittanbieter-Bot): Senden Sie Ihrem Bot eine Direktnachricht, führen Sie `openclaw logs --follow` aus und lesen Sie `from.id`.

    Offizielle Bot API: Senden Sie Ihrem Bot eine Direktnachricht, rufen Sie `https://api.telegram.org/bot<bot_token>/getUpdates` auf und lesen Sie `message.from.id`.

    Drittanbieter (weniger privat): Senden Sie `@userinfobot` oder `@getidsbot` eine Direktnachricht.

    Siehe [Telegram-Zugriffskontrolle](/de/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Können mehrere Personen eine WhatsApp-Nummer mit verschiedenen OpenClaw-Instanzen verwenden?">
    Ja, über **Multi-Agent-Routing**. Binden Sie die WhatsApp-Direktnachrichten jedes Absenders (`peer: { kind: "direct", id: "+15551234567" }`) an eine andere `agentId`, sodass jede Person einen eigenen Arbeitsbereich und Sitzungsspeicher erhält. Antworten stammen weiterhin vom **selben WhatsApp-Konto**; die Zugriffskontrolle für Direktnachrichten (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) gilt global pro Konto. Siehe [Multi-Agent-Routing](/de/concepts/multi-agent) und [WhatsApp](/de/channels/whatsapp).
  </Accordion>

  <Accordion title='Kann ich einen Agenten für „schnelle Chats“ und einen Agenten mit „Opus zum Programmieren“ ausführen?'>
    Ja. Verwenden Sie Multi-Agent-Routing: Weisen Sie jedem Agenten ein eigenes Standardmodell zu und binden Sie anschließend eingehende
    Routen (Provider-Konto oder bestimmte Gegenstellen) an den jeweiligen Agenten. Beispielkonfiguration:
    [Multi-Agent-Routing](/de/concepts/multi-agent). Siehe auch [Modelle](/de/concepts/models) und
    [Konfiguration](/de/gateway/configuration).
  </Accordion>

  <Accordion title="Funktioniert Homebrew unter Linux?">
    Ja, über Linuxbrew:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Wenn OpenClaw über systemd ausgeführt wird, stellen Sie sicher, dass der PATH des Dienstes
    `/home/linuxbrew/.linuxbrew/bin` (oder Ihr brew-Präfix) enthält, damit mit `brew` installierte Tools
    in Nicht-Anmelde-Shells aufgelöst werden. Neuere Builds stellen Linux-
    systemd-Diensten außerdem gängige benutzerspezifische bin-Verzeichnisse voran (zum Beispiel `~/.local/bin`, `~/.npm-global/bin`,
    `~/.local/share/pnpm`, `~/.bun/bin`) und berücksichtigen `PNPM_HOME`, `NPM_CONFIG_PREFIX`,
    `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` und `FNM_DIR`, sofern festgelegt.

  </Accordion>

  <Accordion title="Unterschied zwischen der anpassbaren Git-Installation und der npm-Installation">
    - **Anpassbare Git-Installation:** vollständiger, bearbeitbarer Quellcode-Checkout; ideal für Mitwirkende. Sie erstellen den Build lokal und können Code und Dokumentation ändern.
    - **npm-Installation:** globale CLI-Installation ohne Repository; ideal, wenn Sie es „einfach nur ausführen“ möchten. Updates werden über npm-dist-tags bereitgestellt.

    Dokumentation: [Erste Schritte](/de/start/getting-started), [Aktualisierung](/de/install/updating).

  </Accordion>

  <Accordion title="Kann ich später zwischen npm- und Git-Installationen wechseln?">
    Ja, mit `openclaw update --channel ...` bei einer bestehenden Installation. Dabei werden **Ihre Daten
    nicht gelöscht** – nur die Installation des OpenClaw-Codes ändert sich. Status (`~/.openclaw`) und
    Arbeitsbereich (`~/.openclaw/workspace`) bleiben unverändert.

    Von npm zu Git:

    ```bash
    openclaw update --channel dev
    ```

    Von Git zu npm:

    ```bash
    openclaw update --channel stable
    ```

    Fügen Sie `--dry-run` hinzu, um zunächst eine Vorschau des geplanten Moduswechsels anzuzeigen. Der Updater führt nachfolgende Doctor-Schritte
    aus, aktualisiert die Plugin-Quellen für den Zielkanal und startet das Gateway neu,
    sofern Sie nicht `--no-restart` übergeben.

    Das Installationsprogramm kann ebenfalls einen der beiden Modi erzwingen:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    Tipps zur Sicherung: [Wo sich die Daten auf dem Datenträger befinden](/de/help/faq#where-things-live-on-disk).

  </Accordion>

  <Accordion title="Sollte ich das Gateway auf meinem Laptop oder einem VPS ausführen?">
    Möchten Sie 24/7 Zuverlässigkeit? Verwenden Sie einen **VPS**. Möchten Sie möglichst wenig Aufwand und sind mit
    Ruhezustand und Neustarts einverstanden? Führen Sie es lokal aus.

    **Laptop (lokales Gateway)**

    - **Vorteile:** keine Serverkosten, direkter Zugriff auf lokale Dateien, sichtbares Browserfenster.
    - **Nachteile:** Ruhezustand und Netzwerkunterbrechungen trennen die Verbindung, Betriebssystemupdates und Neustarts unterbrechen den Betrieb, das Gerät muss aktiv bleiben.

    **VPS / Cloud**

    - **Vorteile:** ständig verfügbar, stabiles Netzwerk, keine durch den Laptop-Ruhezustand verursachten Probleme, leichter dauerhaft zu betreiben.
    - **Nachteile:** häufig ohne grafische Oberfläche (verwenden Sie Screenshots), nur Remotezugriff auf Dateien, SSH für Updates erforderlich.

    WhatsApp/Telegram/Slack/Mattermost/Discord funktionieren alle problemlos auf einem VPS – der eigentliche
    Kompromiss besteht zwischen einem Browser ohne grafische Oberfläche und einem sichtbaren Fenster. Siehe [Browser](/de/tools/browser).

    Standardempfehlung: Verwenden Sie einen VPS, wenn es bereits zu Gateway-Verbindungsabbrüchen gekommen ist. Die lokale Ausführung eignet sich hervorragend,
    wenn Sie den Mac aktiv verwenden und lokalen Dateizugriff oder UI-Automatisierung
    mit einem sichtbaren Browser wünschen.

  </Accordion>

  <Accordion title="Wie wichtig ist es, OpenClaw auf einem dedizierten Rechner auszuführen?">
    Es ist nicht erforderlich, wird aber für Zuverlässigkeit und Isolation empfohlen.

    - **Dedizierter Host (VPS/Mac mini/Raspberry Pi):** ständig verfügbar, weniger Unterbrechungen durch Ruhezustand oder Neustarts, übersichtlichere Berechtigungen, leichter dauerhaft zu betreiben.
    - **Gemeinsam genutzter Laptop/Desktop-PC:** für Tests und die aktive Nutzung geeignet, aber rechnen Sie mit Unterbrechungen, wenn das Gerät in den Ruhezustand wechselt oder Updates installiert.

    Die beste Kombination aus beiden Ansätzen: Betreiben Sie das Gateway auf einem dedizierten Host und koppeln Sie Ihren Laptop als
    **Node** für lokale Bildschirm-, Kamera- und Ausführungswerkzeuge. Siehe [Nodes](/de/nodes) und [Sicherheit](/de/gateway/security).

  </Accordion>

  <Accordion title="Welche Mindestanforderungen gelten für einen VPS und welches Betriebssystem wird empfohlen?">
    - **Absolutes Minimum:** 1 vCPU, 1 GB RAM, ~500 MB Speicherplatz.
    - **Empfohlen:** 1–2 vCPU, mindestens 2 GB RAM als Reserve (Protokolle, Medien, mehrere Kanäle). Node-Werkzeuge und Browserautomatisierung können viele Ressourcen beanspruchen.

    Betriebssystem: **Ubuntu LTS** (oder jede moderne Debian-/Ubuntu-Version) – der am besten getestete Installationsweg unter Linux.

    Dokumentation: [Linux](/de/platforms/linux), [VPS-Hosting](/de/vps).

  </Accordion>

  <Accordion title="Kann ich OpenClaw in einer VM ausführen und welche Anforderungen gelten?">
    Ja. Behandeln Sie eine VM wie einen VPS: Sie muss ständig eingeschaltet und erreichbar sein sowie über genügend RAM
    für das Gateway und alle von Ihnen aktivierten Kanäle verfügen.

    - **Absolutes Minimum:** 1 vCPU, 1 GB RAM.
    - **Empfohlen:** mindestens 2 GB RAM für mehrere Kanäle, Browserautomatisierung oder Medienwerkzeuge.
    - **Betriebssystem:** Ubuntu LTS oder eine andere moderne Debian-/Ubuntu-Version.

    Verwenden Sie unter Windows den **Windows Hub** für die Desktop-Einrichtung oder WSL2 für eine Linux-ähnliche Gateway-VM
    mit umfassender Werkzeugkompatibilität. Siehe [Windows](/de/platforms/windows), [VPS-Hosting](/de/vps).
    macOS in einer VM ausführen: siehe [macOS-VM](/de/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [FAQ](/de/help/faq) – die wichtigsten häufig gestellten Fragen (Modelle, Sitzungen, Gateway, Sicherheit und mehr)
- [Installationsübersicht](/de/install)
- [Erste Schritte](/de/start/getting-started)
- [Fehlerbehebung](/de/help/troubleshooting)
