---
read_when:
    - Modelle auswählen oder wechseln, Aliasse konfigurieren
    - Debugging des Modell-Failovers / „Alle Modelle sind fehlgeschlagen“
    - Authentifizierungsprofile verstehen und verwalten
sidebarTitle: Models FAQ
summary: 'FAQ: Modellstandards, Auswahl, Aliase, Wechsel, Failover und Authentifizierungsprofile'
title: 'FAQ: Modelle und Authentifizierung'
x-i18n:
    generated_at: "2026-07-26T18:31:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

Fragen und Antworten zu Modellen und Auth-Profilen. Informationen zu Einrichtung, Sitzungen, Gateway, Kanälen und
Fehlerbehebung finden Sie in den zentralen [FAQ](/de/help/faq).

## Modelle: Standardwerte, Auswahl, Aliasse, Wechsel

<AccordionGroup>
  <Accordion title='Was ist das „Standardmodell“?'>
    Festgelegt wird es mit:

    ```text
    agents.defaults.model.primary
    ```

    Modelle sind `provider/model`-Referenzen (Beispiel: `openai/gpt-5.5`,
    `anthropic/claude-sonnet-4-6`). Legen Sie `provider/model` immer ausdrücklich fest. Wenn
    Sie den Provider weglassen, versucht OpenClaw zunächst, einen Alias zuzuordnen, dann für diese
    Modell-ID eine eindeutige Übereinstimmung unter den konfigurierten Providern zu finden, und greift anschließend auf den
    konfigurierten Standard-Provider zurück (veralteter Kompatibilitätspfad). Wenn dieser
    Provider das konfigurierte Standardmodell nicht mehr führt, greift OpenClaw
    statt eines veralteten Standardwerts auf den ersten konfigurierten Provider samt Modell zurück.

  </Accordion>

  <Accordion title="Welches Modell empfehlen Sie?">
    Verwenden Sie das leistungsfähigste Modell der neuesten Generation, das Ihr Provider-Stack anbietet,
    insbesondere für Agents mit Werkzeugzugriff oder nicht vertrauenswürdigen Eingaben – schwächere oder
    übermäßig quantisierte Modelle sind anfälliger für Prompt-Injection und unsicheres
    Verhalten (siehe [Sicherheit](/de/gateway/security)). Weisen Sie günstigere Modelle anhand der Agent-Rolle
    routinemäßigen Chats mit geringem Risiko zu.

    Weisen Sie Modelle pro Agent zu und verwenden Sie Sub-Agents, um langwierige Aufgaben zu parallelisieren (jeder
    Sub-Agent verbraucht eigene Tokens). Siehe [Modelle](/de/concepts/models),
    [Sub-Agents](/de/tools/subagents), [MiniMax](/de/providers/minimax) und
    [Lokale Modelle](/de/gateway/local-models).

  </Accordion>

  <Accordion title="Wie wechsle ich Modelle, ohne meine Konfiguration zu löschen?">
    Ändern Sie nur die Modellfelder – vermeiden Sie das vollständige Ersetzen der Konfiguration.

    - `/model` im Chat (pro Sitzung, siehe [Slash-Befehle](/de/tools/slash-commands))
    - `openclaw models set ...` (aktualisiert nur die Modellkonfiguration)
    - `openclaw configure --section model` (interaktiv)
    - `agents.defaults.model` direkt in `~/.openclaw/openclaw.json` bearbeiten

    Prüfen Sie bei RPC-Änderungen zunächst mit `config.schema.lookup` (normalisierter
    Pfad, kompakte Schemadokumentation, Zusammenfassungen untergeordneter Elemente) und bevorzugen Sie dann `config.patch`
    gegenüber `config.apply` mit einem partiellen Objekt. Falls Sie die Konfiguration überschrieben haben,
    stellen Sie sie aus einer Sicherung wieder her oder führen Sie zur Reparatur `openclaw doctor` aus.

    Dokumentation: [Modelle](/de/concepts/models), [Konfigurieren](/de/cli/configure),
    [Konfiguration](/de/cli/config), [Doctor](/de/gateway/doctor).

  </Accordion>

  <Accordion title="Kann ich selbst gehostete Modelle verwenden (llama.cpp, vLLM, Ollama)?">
    Ja – Ollama ist der einfachste Weg. Schnelle Einrichtung:

    1. Ollama von `https://ollama.com/download` installieren
    2. Ein lokales Modell abrufen, z. B. `ollama pull gemma4`
    3. Auch für Cloud-Modelle `ollama signin` ausführen
    4. `openclaw onboard` ausführen, `Ollama` auswählen und anschließend `Local` oder `Cloud + Local`

    `Cloud + Local` stellt Ihnen Cloud-Modelle zusammen mit Ihren lokalen Ollama-Modellen bereit;
    Cloud-Modelle wie `kimi-k2.5:cloud` müssen nicht lokal abgerufen werden. So wechseln Sie
    manuell: `openclaw models list`, dann `openclaw models set ollama/<model>`.

    Kleinere oder stark quantisierte Modelle sind anfälliger für Prompt-Injection.
    Verwenden Sie große Modelle für jeden Bot mit Werkzeugzugriff. Falls Sie dennoch kleine Modelle
    verwenden, aktivieren Sie Sandboxing und strenge Zulassungslisten für Werkzeuge.

    Dokumentation: [Ollama](/de/providers/ollama), [Lokale Modelle](/de/gateway/local-models),
    [Modell-Provider](/de/concepts/model-providers), [Sicherheit](/de/gateway/security),
    [Sandboxing](/de/gateway/sandboxing).

  </Accordion>

  <Accordion title="Wie wechsle ich Modelle im laufenden Betrieb (ohne Neustart)?">
    Senden Sie `/model <name>` als eigenständige Nachricht. Unter
    [Slash-Befehle](/de/tools/slash-commands) finden Sie die
    vollständige Befehlsliste, einschließlich der nummerierten Auswahl (`/model`, `/model
    list`, `/model 3`), `/model default` zum Löschen einer sitzungsspezifischen Überschreibung und
    `/model status` für Details zum Endpunkt/API-Modus.

    Erzwingen Sie mit `@profile` ein bestimmtes Auth-Profil pro Sitzung:

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Um die Bindung eines mit `@profile` festgelegten Profils aufzuheben, führen Sie `/model` erneut ohne
    Suffix aus (z. B. `/model anthropic/claude-opus-4-6`) oder wählen Sie den Standardwert unter
    `/model` aus. Verwenden Sie `/model status`, um das aktive Auth-Profil zu bestätigen.

  </Accordion>

  <Accordion title="Wenn zwei Provider dieselbe Modell-ID anbieten, welchen verwendet /model?">
    `/model provider/model` wählt exakt diese Provider-Route aus. Beispielsweise sind
    `qianfan/deepseek-v4-flash` und `deepseek/deepseek-v4-flash` unterschiedliche
    Referenzen, obwohl die Modell-ID übereinstimmt – OpenClaw wechselt bei einer Übereinstimmung allein anhand der ID nicht stillschweigend
    den Provider.

    Eine vom Benutzer ausgewählte `/model`-Referenz ist beim Fallback strikt: Wenn dieses
    Provider-/Modell-Paar nicht mehr verfügbar ist, schlägt die Antwort sichtbar fehl, statt
    auf `agents.defaults.model.fallbacks` zurückzugreifen. Konfigurierte Fallback-
    Ketten gelten weiterhin für konfigurierte Standardwerte, primäre Modelle von Cron-Aufträgen und
    automatisch ausgewählte Fallback-Zustände. Wenn ein Lauf ohne Sitzungsüberschreibung
    einen Fallback verwenden darf, versucht OpenClaw zuerst das angeforderte Provider-/Modell-Paar, dann
    die konfigurierten Fallbacks und anschließend das konfigurierte primäre Modell – identische reine
    Modell-IDs springen daher nie direkt zum Standard-Provider zurück.

    Siehe [Modelle](/de/concepts/models) und [Modell-Failover](/de/concepts/model-failover).

  </Accordion>

  <Accordion title="Kann ich GPT 5.5 für tägliche Aufgaben und Codex 5.5 zum Programmieren verwenden?">
    Ja – Modellauswahl und Runtime-Auswahl sind voneinander unabhängig:

    - **Nativer Codex-Programmier-Agent:** Legen Sie `agents.defaults.model.primary` auf
      `openai/gpt-5.5` fest. Melden Sie sich mit `openclaw models auth login --provider
      openai` für die Authentifizierung über ein ChatGPT-/Codex-Abonnement an.
    - **Direkte OpenAI-API-Aufgaben außerhalb der Agent-Schleife:** Konfigurieren Sie
      `OPENAI_API_KEY` für Bilder, Embeddings, Sprache, Echtzeit und andere
      OpenAI-API-Oberflächen außerhalb von Agents.
    - **OpenAI-Agent-Authentifizierung per API-Schlüssel:** `/model openai/gpt-5.5` mit einem geordneten
      `openai`-API-Schlüsselprofil.
    - **Sub-Agents:** Weisen Sie Programmieraufgaben einem auf Codex ausgerichteten Agent mit einem
      eigenen `openai/gpt-5.5`-Modell zu.

    Siehe [Modelle](/de/concepts/models) und [Slash-Befehle](/de/tools/slash-commands).

  </Accordion>

  <Accordion title="Wie konfiguriere ich den schnellen Modus für GPT 5.5?">
    - **Pro Sitzung:** Senden Sie `/fast on`, während Sie `openai/gpt-5.5` verwenden.
    - **Standardwert pro Modell:** Legen Sie
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` auf `true` fest.
    - **Automatischer Grenzwert:** `/fast auto` oder `params.fastMode: "auto"` führt neue
      Modellaufrufe bis zum Grenzwert im schnellen Modus aus; spätere Wiederholungs-, Fallback-,
      Werkzeugergebnis- oder Fortsetzungsaufrufe erfolgen danach ohne schnellen Modus. Der Grenzwert beträgt standardmäßig
      60 Sekunden; überschreiben Sie ihn mit `params.fastAutoOnSeconds` für das Modell.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    Der schnelle Modus wird bei nativen OpenAI-Responses-
    Anfragen auf `service_tier = "priority"` abgebildet; vorhandene `service_tier`-Werte bleiben erhalten, und der schnelle Modus
    ändert weder `reasoning` noch `text.verbosity`. Sitzungsbezogene `/fast`-Überschreibungen haben
    Vorrang vor den Konfigurationsstandardwerten.

    Siehe [Denk- und schneller Modus](/de/tools/thinking) sowie den Abschnitt zum schnellen Modus
    unter „Erweiterte Konfiguration“ auf der Provider-Seite
    [OpenAI](/de/providers/openai).

  </Accordion>

  <Accordion title='Warum wird „Model ... is not allowed“ angezeigt und anschließend keine Antwort ausgegeben?'>
    Wenn `agents.defaults.modelPolicy.allow` nicht leer ist, wird es zur
    **Zulassungsliste** für `/model`, Sitzungsüberschreibungen und `--model`. Bei Auswahl eines Modells außerhalb dieser Liste wird
    statt einer normalen Antwort Folgendes zurückgegeben:

    ```text
    Die Modellüberschreibung "provider/model" ist laut agents.defaults.modelPolicy.allow nicht zulässig.
    ```

    Lösung: Fügen Sie das genaue Modell oder einen Provider-Platzhalter wie `"provider/*"` zur
    genannten `modelPolicy.allow`-Liste hinzu, entfernen oder leeren Sie diese Liste oder wählen Sie ein Modell
    aus `/model list`. Wenn der Befehl außerdem
    `--runtime codex` enthielt, aktualisieren Sie zuerst die Zulassungsliste und versuchen Sie dann denselben
    `/model provider/model --runtime codex`-Befehl erneut.

  </Accordion>

  <Accordion title='Warum wird „Unknown model: minimax/MiniMax-M3“ angezeigt?'>
    Wenn Sie eine ältere OpenClaw-Version verwenden, führen Sie zuerst ein Upgrade durch (oder starten Sie mit
    `main` aus dem Quellcode) und starten Sie das Gateway neu – `MiniMax-M3` ist möglicherweise noch nicht im Katalog
    Ihrer installierten Version enthalten. Andernfalls ist der MiniMax-Provider nicht
    konfiguriert (es wurde kein Provider-Eintrag oder Auth-Profil gefunden), sodass das Modell nicht
    aufgelöst werden kann. Die vollständige Checkliste zur Behebung,
    die Tabelle der Provider-/Modell-IDs und ein Beispiel für einen Konfigurationsblock finden Sie im Abschnitt zur Fehlerbehebung auf der
    Provider-Seite [MiniMax](/de/providers/minimax).

  </Accordion>

  <Accordion title="Kann ich MiniMax als Standard und OpenAI für komplexe Aufgaben verwenden?">
    Ja. Verwenden Sie MiniMax als Standard und wechseln Sie das Modell pro Sitzung – Fallbacks
    sind für Fehler vorgesehen, nicht für „schwierige Aufgaben“. Verwenden Sie daher `/model` oder einen separaten Agent.

    **Option A: pro Sitzung wechseln**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Anschließend `/model gpt`.

    **Option B: separate Agents** – Agent A verwendet standardmäßig MiniMax, Agent B
    standardmäßig OpenAI; weisen Sie anhand des Agents zu oder verwenden Sie zum Wechseln `/agent`.

    Dokumentation: [Modelle](/de/concepts/models), [Multi-Agent-Routing](/de/concepts/multi-agent),
    [MiniMax](/de/providers/minimax), [OpenAI](/de/providers/openai).

  </Accordion>

  <Accordion title="Sind opus / sonnet / gpt integrierte Kurzformen?">
    Ja – integrierte Kurzformen, die nur angewendet werden, wenn das Zielmodell in
    `agents.defaults.models` vorhanden ist:

    | Alias | Wird aufgelöst zu |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    Ein eigener Alias mit demselben Namen überschreibt den integrierten Alias.

  </Accordion>

  <Accordion title="Wie definiere oder überschreibe ich Modellkurzformen (Aliasse)?">
    Aliasse befinden sich unter `agents.defaults.models.<modelId>.alias`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    Anschließend wird `/model sonnet` (oder, falls unterstützt, `/<alias>`) zu dieser
    Modell-ID aufgelöst.

  </Accordion>

  <Accordion title="Wie füge ich Modelle anderer Provider wie OpenRouter oder Z.AI hinzu?">
    OpenRouter (Abrechnung pro Token; viele Modelle):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (GLM-Modelle):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Ein fehlender Provider-Schlüssel für ein referenziertes Provider-/Modell-Paar verursacht zur Laufzeit
    einen Authentifizierungsfehler (z. B. `No API key found for provider "zai"`).

    **Nach dem Hinzufügen eines neuen Agents wurde kein API-Schlüssel für den Provider gefunden**

    Ein neuer Agent verfügt über einen leeren Authentifizierungsspeicher – die Authentifizierung erfolgt pro Agent und wird hier gespeichert:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Fehlerbehebung: Führen Sie `openclaw agents add <id>` aus und konfigurieren Sie die Authentifizierung im Assistenten oder
    kopieren Sie nur portable statische `api_key`/`token`-Profile aus dem Speicher des
    Hauptagenten. Melden Sie sich für OAuth über den neuen Agenten an, wenn dieser
    ein eigenes Konto benötigt. Unter [Multi-Agent-Routing](/de/concepts/multi-agent) finden Sie die
    vollständigen Regeln zur Wiederverwendung von `agentDir` und zur gemeinsamen Nutzung von Anmeldedaten — verwenden Sie
    `agentDir` niemals agentenübergreifend wieder.

  </Accordion>
</AccordionGroup>

## Modell-Failover und „Alle Modelle fehlgeschlagen“

<AccordionGroup>
  <Accordion title="Wie funktioniert Failover?">
    Zwei Phasen:

    1. **Rotation der Authentifizierungsprofile** innerhalb desselben Providers.
    2. **Modell-Fallback** auf das nächste Modell in `agents.defaults.model.fallbacks`.

    Für fehlgeschlagene Profile gelten Abklingzeiten (exponentielles Backoff), sodass OpenClaw
    weiterhin antwortet, wenn für einen Provider eine Ratenbegrenzung gilt oder dieser vorübergehend ausfällt.

    Der Ratenbegrenzungsbereich umfasst mehr als nur `429`: `Too many concurrent
    requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai
    ... quota limit exceeded`, `resource exhausted` und periodische
    Nutzungslimits (`weekly/monthly limit reached`) gelten sämtlich als
    Ratenbegrenzungen, die ein Failover rechtfertigen.

    Abrechnungsantworten sind nicht immer `402`, und manche `402`s verbleiben im
    Bereich für vorübergehende Fehler/Ratenbegrenzungen, statt dem Abrechnungsbereich zugeordnet zu werden. Explizite
    Abrechnungstexte bei `401`/`403` können weiterhin der Abrechnung zugeordnet werden; providerspezifische
    Textabgleiche (z. B. OpenRouter `Key limit exceeded`) bleiben auf den jeweiligen
    Provider beschränkt. Ein `402`, das wie ein wiederholbares Nutzungslimit oder
    Ausgabenlimit einer Organisation/eines Arbeitsbereichs formuliert ist (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`), wird als `rate_limit` behandelt, nicht als
    langfristige Deaktivierung aufgrund der Abrechnung.

    Kontextüberlauffehler werden vollständig vom Fallback-Pfad ferngehalten — Signaturen
    wie `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`, `input is
    too long for the model` oder `ollama error: context length exceeded` führen zu
    Compaction/Wiederholung, statt den Modell-Fallback voranzutreiben.

    Generischer Serverfehlertext ist enger gefasst als „alles, worin unknown/error
    vorkommt“. Providergebundene vorübergehende Fehlerformen, die als Failover-
    Signale gelten: Anthropic mit bloßem `An unknown error occurred`, OpenRouter mit bloßem
    `Provider returned error`, Stoppgrundfehler wie `Unhandled stop reason:
    error`, JSON-`api_error`-Nutzdaten mit Text zu vorübergehenden Serverfehlern (`internal
    server error`, `unknown error, 520`, `upstream error`, `backend error`)
    sowie Provider-ausgelastet-Fehler wie `ModelNotReadyException`, wenn der Provider-
    Kontext übereinstimmt. Generischer interner Fallback-Text wie `LLM request failed
    with an unknown error.` wird konservativ behandelt und löst allein
    keinen Fallback aus.

  </Accordion>

  <Accordion title='Was bedeutet „No credentials found for profile anthropic:default“?'>
    Für die Authentifizierungsprofil-ID `anthropic:default` sind im
    erwarteten Authentifizierungsspeicher keine Anmeldedaten vorhanden.

    **Checkliste zur Fehlerbehebung:**

    - Prüfen Sie, wo die Profile gespeichert sind — aktuell:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`; veraltet:
      `~/.openclaw/agent/*` (migriert durch `openclaw doctor`).
    - Prüfen Sie, ob das Gateway Ihre Umgebungsvariable lädt. Ein nur in
      Ihrer Shell gesetztes `ANTHROPIC_API_KEY` erreicht kein über systemd/launchd ausgeführtes Gateway — tragen Sie es in
      `~/.openclaw/.env` ein oder aktivieren Sie `env.shellEnv`.
    - Prüfen Sie, ob Sie den richtigen Agenten bearbeiten — Multi-Agent-Konfigurationen verfügen über
      mehrere `auth-profiles.json`-Dateien.
    - Führen Sie `openclaw models status` aus, um konfigurierte Modelle und den
      Authentifizierungsstatus des Providers anzuzeigen.

    **Für „No credentials found for profile anthropic“ (ohne E-Mail-Suffix):**

    Die Ausführung ist an ein Anthropic-Profil gebunden, das das Gateway nicht finden kann.

    - Verwenden Sie die Claude CLI: Führen Sie `openclaw models auth login --provider anthropic
      --method cli --set-default` auf dem Gateway-Host aus.
    - Wenn Sie stattdessen einen API-Schlüssel bevorzugen: Tragen Sie `ANTHROPIC_API_KEY` in
      `~/.openclaw/.env` auf dem Gateway-Host ein und löschen Sie anschließend jede festgelegte Reihenfolge,
      die das fehlende Profil erzwingt:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - Remote-Modus: Authentifizierungsprofile befinden sich auf dem Gateway-Rechner, nicht auf Ihrem
      Laptop — stellen Sie sicher, dass Sie die Befehle dort ausführen.

  </Accordion>

  <Accordion title="Warum wurde auch Google Gemini ausprobiert und ist fehlgeschlagen?">
    Wenn Ihre Modellkonfiguration Google Gemini als Fallback enthält (oder Sie
    zu einer Gemini-Kurzform gewechselt haben), versucht OpenClaw es während des Fallbacks. Sind keine
    Google-Anmeldedaten konfiguriert, ergibt sich `No API key found for provider
    "google"`. Fehlerbehebung: Fügen Sie die Google-Authentifizierung hinzu oder entfernen Sie Google-Modelle aus
    `agents.defaults.model.fallbacks`/Aliasen.

    **LLM-Anfrage abgelehnt: Denksignatur erforderlich (Google Antigravity)**

    Ursache: Der Sitzungsverlauf enthält Denkblöcke ohne Signaturen (häufig
    aufgrund eines abgebrochenen/unvollständigen Streams); Google Antigravity erfordert Signaturen
    für Denkblöcke. OpenClaw entfernt nicht signierte Denkblöcke für Google
    Antigravity Claude; wenn das Problem weiterhin auftritt, starten Sie eine neue Sitzung oder setzen Sie
    `/thinking off` für diesen Agenten.

  </Accordion>
</AccordionGroup>

## Authentifizierungsprofile: Was sie sind und wie sie verwaltet werden

Verwandt: [/concepts/oauth](/de/concepts/oauth) (OAuth-Abläufe, Token-Speicherung, Muster für mehrere Konten)

<AccordionGroup>
  <Accordion title="Was ist ein Authentifizierungsprofil?">
    Ein benannter Datensatz mit Anmeldedaten (OAuth oder API-Schlüssel), der einem Provider zugeordnet und
    hier gespeichert ist:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Prüfen Sie gespeicherte Profile, ohne Geheimnisse auszugeben: `openclaw models auth
    list` (optional `--provider <id>` oder `--json`). Siehe
    [Modelle-CLI](/de/cli/models#auth-profiles).

  </Accordion>

  <Accordion title="Was sind typische Profil-IDs?">
    Mit Provider-Präfix: `anthropic:default` (üblich, wenn keine E-Mail-Identität
    vorhanden ist), `anthropic:<email>` für OAuth-Identitäten oder eine benutzerdefinierte ID Ihrer
    Wahl (z. B. `anthropic:work`).

  </Accordion>

  <Accordion title="Kann ich steuern, welches Authentifizierungsprofil zuerst ausprobiert wird?">
    Ja. Die Konfiguration `auth.order.<provider>` legt die Rotationsreihenfolge pro Provider fest
    (nur Metadaten — es werden keine Geheimnisse gespeichert).

    OpenClaw überspringt möglicherweise ein Profil während einer kurzen **Abklingzeit** (Ratenbegrenzungen,
    Zeitüberschreitungen, Authentifizierungsfehler) oder eines längeren **deaktivierten** Zustands
    (Abrechnung/unzureichendes Guthaben). Prüfen Sie dies mit `openclaw models status
    --json` und kontrollieren Sie `auth.unusableProfiles`. Abklingzeiten aufgrund von Ratenbegrenzungen können
    modellspezifisch sein — ein Profil, das für ein Modell abkühlt, kann weiterhin ein
    verwandtes Modell beim selben Provider bedienen; Abrechnungs-/Deaktivierungszeiträume sperren das
    gesamte Profil.

    Legen Sie eine agentenspezifische Reihenfolgeüberschreibung fest (gespeichert in `auth-state.json` dieses Agenten):

    ```bash
    # Standardmäßig wird der konfigurierte Standardagent verwendet (--agent weglassen)
    openclaw models auth order get --provider anthropic

    # Rotation auf ein einzelnes Profil beschränken
    openclaw models auth order set --provider anthropic anthropic:default

    # Oder eine explizite Reihenfolge festlegen (Fallback innerhalb des Providers)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # Überschreibung löschen (auf die Konfiguration auth.order / Round-Robin zurückfallen)
    openclaw models auth order clear --provider anthropic

    # Einen bestimmten Agenten auswählen
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    Prüfen Sie, was tatsächlich ausprobiert wird: `openclaw models status --probe`. Ein
    gespeichertes Profil, das in einer expliziten Reihenfolge fehlt, meldet
    `excluded_by_auth_order`, statt stillschweigend ausprobiert zu werden.

  </Accordion>

  <Accordion title="OAuth oder API-Schlüssel – worin besteht der Unterschied?">
    - **OAuth-/CLI-Anmeldung** nutzt häufig den Abonnementzugriff, sofern der
      Provider ihn unterstützt. Für Anthropic verwendet das Claude-CLI-Backend von OpenClaw
      Claude Code `claude -p`, das Anthropic derzeit als
      Agent-SDK-/programmatische Nutzung behandelt, die auf die Nutzungslimits des Abonnements angerechnet wird —
      unter [Anthropic](/de/providers/anthropic) finden Sie den aktuellen Status der Abrechnungspause
      und Quellenlinks.
    - **API-Schlüssel** verwenden eine tokenbasierte Abrechnung.

    Der Assistent unterstützt Anthropic Claude CLI, OpenAI Codex OAuth und API-
    Schlüssel.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [FAQ](/de/help/faq) — die wichtigsten häufig gestellten Fragen
- [FAQ — Schnellstart und Einrichtung beim ersten Start](/de/help/faq-first-run)
- [Modellauswahl](/de/concepts/model-providers)
- [Modell-Failover](/de/concepts/model-failover)
