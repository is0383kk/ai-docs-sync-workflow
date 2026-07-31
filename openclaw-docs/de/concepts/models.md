---
read_when:
    - Ändern des Modell-Fallback-Verhaltens oder der Auswahl-UX
    - Fehlerbehebung bei „Modell ist nicht zulässig“ oder einem veralteten Fallback auf den Standard-Provider
    - Arbeiten am Zusammenführungs-/Secret-Verhalten von models.json
sidebarTitle: Models CLI
summary: Wie OpenClaw Provider-/Modellreferenzen, Konfigurationsschlüssel und den Chatbefehl `/model` auflöst
title: Modell-CLI
x-i18n:
    generated_at: "2026-07-26T18:20:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cd13a2aae6575bdfeefb477b7fe8be740b77c66cb76454b07d82481f6612152
    source_path: concepts/models.md
    workflow: 16
---

<CardGroup cols={2}>
  <Card title="Modell-Failover" href="/de/concepts/model-failover">
    Rotation von Auth-Profilen, Abklingzeiten und deren Zusammenspiel mit Fallbacks.
  </Card>
  <Card title="Modell-Provider" href="/de/concepts/model-providers">
    Kurzübersicht über Provider und Beispiele.
  </Card>
  <Card title="CLI-Referenz für Modelle" href="/de/cli/models">
    Vollständige Referenz zum Befehl `openclaw models` und seinen Flags.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/config-agents#agent-defaults">
    Modellkonfigurationsschlüssel, Standardwerte und Beispiele.
  </Card>
</CardGroup>

Eine Modellreferenz (`provider/model`) wählt einen Provider und ein Modell aus, nicht die
Low-Level-Agent-Runtime. Wenn keine Runtime-Richtlinie festgelegt ist oder `auto`
verwendet wird, kann die Provider-eigene Routing-Richtlinie von OpenAI Codex nur für eine
exakte offizielle HTTPS-Platform-Responses- oder ChatGPT-Responses-Route auswählen, bei der
die Anfrage keine explizite Überschreibung enthält; das Präfix `openai/*` allein
wählt niemals Codex aus. Completions-Adapter, benutzerdefinierte Endpunkte und explizit
festgelegtes Anfrageverhalten verbleiben bei OpenClaw. Offizielle Klartext-HTTP-Endpunkte
werden abgelehnt. Siehe [Implizite Agent-Runtime von OpenAI](/de/providers/openai#implicit-agent-runtime).

Copilot-Abonnementreferenzen (`github-copilot/*`) können für das externe
GitHub-Copilot-Agent-Runtime-Plugin aktiviert werden, dieser Pfad ist jedoch immer explizit
(und wird niemals durch `auto` ausgewählt). Runtime-Überschreibungen gehören in
die Provider-/Modellrichtlinie, nicht auf den gesamten Agenten oder die gesamte Sitzung.
Die Runtime-Auswahl bestimmt nicht die Abrechnung: Anmeldedaten für OpenAI-API-Schlüssel
und ChatGPT-/Codex-Abonnements bleiben getrennt. Siehe
[Agent-Runtimes](/de/concepts/agent-runtimes) und
[GitHub-Copilot-Agent-Runtime](/de/plugins/copilot).

## Auswahlreihenfolge

<Steps>
  <Step title="Primäres Modell">
    `agents.defaults.model.primary` (oder `agents.defaults.model` als einfache Zeichenfolge).
  </Step>
  <Step title="Fallbacks">
    `agents.defaults.model.fallbacks`, die der Reihe nach versucht werden.
  </Step>
  <Step title="Auth-Failover">
    Die Rotation von Auth-Profilen erfolgt innerhalb eines Providers, bevor OpenClaw zum nächsten Fallback-Modell wechselt.
  </Step>
</Steps>

Zugehörige Oberflächen zur Modellkonfiguration:

- `agents.defaults.models` speichert Aliasse und modellspezifische Einstellungen. Das Hinzufügen eines Eintrags schränkt Modellüberschreibungen nicht ein.
- `agents.defaults.modelPolicy.allow` ist die optionale Positivliste für Überschreibungen. Verwenden Sie exakte Referenzen oder abschließende Präfix-Platzhalter wie `provider/*` und `provider/namespace/*`; lassen Sie sie weg oder setzen Sie `[]`, um jedes Modell zuzulassen. Das agentenspezifische `agents.entries.*.modelPolicy.allow` ersetzt die Standardrichtlinie für diesen Agenten.
- `agents.defaults.utilityModel` ist ein optionales kostengünstigeres Modell für kurze interne Aufgaben wie generierte Sitzungstitel im Dashboard, unterstützte Kanal-Thread-/Thementitel und Fortschrittsbeschreibungen. Das agentenspezifische `agents.entries.*.utilityModel` überschreibt es. Wenn es nicht festgelegt ist, verwendet OpenClaw den deklarierten Standard des primären Providers für kleine Modelle, sofern vorhanden (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`), andernfalls das primäre Modell des Agenten; setzen Sie es auf eine leere Zeichenfolge, um das Utility-Routing zu deaktivieren. Generierte Titel werden einmal mit dem primären Modell erneut versucht, wenn ein separates Utility-Modell fehlschlägt. Bei Dashboard-Titeln folgen die automatische Utility-Ableitung und der reguläre Fallback dem effektiven Sitzungs-Provider und Auth-Profil; ein explizites Utility-Modell behält seinen konfigurierten Provider und seine konfigurierte Authentifizierung bei. Ein leeres Utility-Modell überspringt nur die alternative Route über das kleine Modell, nicht die Generierung von Dashboard-Titeln. Utility-Aufgaben sind separate Modellaufrufe und können begrenzte Aufgabeninhalte an den ausgewählten Modell-Provider senden.
- `agents.defaults.imageModel` wird nur verwendet, wenn das primäre Modell keine Bilder akzeptieren kann.
- `agents.defaults.pdfModel` wird vom Werkzeug `pdf` verwendet. Wenn es nicht festgelegt ist, greift das Werkzeug auf `imageModel` und anschließend auf das aufgelöste Sitzungs-/Standardmodell zurück.
- `agents.defaults.mediaModels.{image,music,video}` dient als Grundlage für die gemeinsam genutzten Werkzeuge zur Mediengenerierung. Wenn es nicht festgelegt ist, leitet jedes Werkzeug einen durch Authentifizierung gestützten Provider-Standard ab: zuerst den aktuellen Standard-Provider, dann die übrigen registrierten Provider für diese Fähigkeit in der Reihenfolge ihrer Provider-IDs. Provider-übergreifender Fallback ist das fest vorgegebene Standardverhalten.
- Das agentenspezifische `agents.entries.*.model` (zusammen mit Bindungen) überschreibt `agents.defaults.model` – siehe [Multi-Agent-Routing](/de/concepts/multi-agent).

Vollständige Schlüsselreferenz, Standardwerte und JSON5-Beispiele: [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults).

## Auswahlquelle und Fallback-Strenge

Dasselbe `provider/model` verhält sich je nach Herkunft unterschiedlich:

| Quelle                                                                  | Verhalten                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Konfigurierter Standard (`agents.defaults.model.primary`, agentenspezifisches primäres Modell) | Normaler Ausgangspunkt; verwendet `agents.defaults.model.fallbacks`.                                                                                                                                                                                                 |
| Automatischer Fallback                                                  | Temporärer Wiederherstellungszustand, gespeichert als `modelOverrideSource: "auto"`. OpenClaw prüft das ursprüngliche primäre Modell regelmäßig erneut, löscht die automatische Auswahl bei der Wiederherstellung und kündigt Fallback-/Wiederherstellungsübergänge einmal pro Zustandsänderung an.                              |
| Benutzerauswahl für die Sitzung                                         | Exakt und strikt. `/model`, die Modellauswahl, `session_status(model=...)` und `sessions.patch` speichern `modelOverrideSource: "user"`. Wenn dieser Provider bzw. dieses Modell nicht mehr erreichbar ist, schlägt der Lauf sichtbar fehl, statt auf ein anderes konfiguriertes Modell zurückzugreifen. |
| Cron `--model` / Nutzlast `model`                  | Primäres Modell pro Auftrag. Verwendet weiterhin konfigurierte Fallbacks, sofern der Auftrag nicht sein eigenes Nutzlast-`fallbacks` bereitstellt (`fallbacks: []` erzwingt einen strikten Lauf).                                                                                                                    |

Weitere Auswahlregeln:

- Eine Änderung von `agents.defaults.model.primary` schreibt vorhandene Sitzungsfixierungen nicht um. Wenn der Status `This session is pinned to X; config primary Y will apply to new/unpinned sessions.` meldet, führen Sie `/model default` aus, um die Fixierung zu löschen.
- CLI-Auswahlmenüs für das Standardmodell und die Positivliste berücksichtigen `models.mode: "replace"`, indem sie nur `models.providers.*.models` statt des vollständigen integrierten Katalogs auflisten.
- Die Modellauswahl der Control UI fragt beim Gateway dessen konfigurierte Modellansicht ab. Ein explizites `modelPolicy.allow` filtert sie einschließlich Einträgen mit abschließendem Präfix-Platzhalter; andernfalls zeigt sie konfigurierte Modelle sowie Provider mit verwendbarer Authentifizierung an. Der vollständige integrierte Katalog ist expliziten Suchansichten vorbehalten (`models.list` mit `view: "all"` oder `openclaw models list --all`).
- Benutzeroberflächen für das Provider-Inventar verwenden `models.list` mit `view: "provider-config"`, um von der Quelle definierte `models.providers.*.models`-Zeilen anzuzeigen, ohne Positivlisten der Auswahl anzuwenden.

Vollständige Funktionsweise: [Modell-Failover](/de/concepts/model-failover).

## Kurze Modellrichtlinie

- Legen Sie als primäres Modell das leistungsfähigste verfügbare Modell der neuesten Generation fest.
- Verwenden Sie Fallbacks für kosten-/latenzsensitive Aufgaben und Chats mit geringeren Anforderungen.
- Vermeiden Sie für Agenten mit aktivierten Werkzeugen oder bei nicht vertrauenswürdigen Eingaben ältere bzw. schwächere Modellstufen.

## Onboarding

```bash
openclaw onboard
```

Richtet Modell und Authentifizierung für gängige Provider ein, ohne die Konfiguration manuell zu bearbeiten, einschließlich OAuth für OpenAI-Codex-Abonnements und Anthropic (API-Schlüssel oder Wiederverwendung der Claude CLI).

Wenn kein primäres Modell konfiguriert ist, wählt eine neue Einrichtung mit OpenAI-API-Schlüssel
`openai/gpt-5.6`; die reine Direct-API-ID wird in die Sol-Stufe aufgelöst. Eine neue
Einrichtung mit ChatGPT-/Codex-OAuth wählt die exakte Katalogreferenz `openai/gpt-5.6-sol`.
Eine erneute Authentifizierung behält ein vorhandenes explizites primäres Modell bei,
einschließlich `openai/gpt-5.5`. Wenn GPT-5.6 für das Konto nicht verfügbar ist, wählen
Sie `openai/gpt-5.5` explizit aus; OpenClaw stuft es nicht stillschweigend herab.

## „Modell ist nicht zulässig“ (und warum Antworten ausbleiben)

Wenn `agents.defaults.modelPolicy.allow` nicht leer ist, wird es zur Positivliste für `/model`, Sitzungsüberschreibungen und `--model`. Die Auswahl eines Modells außerhalb dieser Positivliste führt zur Rückkehr, bevor eine normale Antwort generiert wird. Ein agentenspezifisches `agents.entries.*.modelPolicy.allow` ersetzt die Standardrichtlinie für diesen Agenten.

```text
Die Modellüberschreibung "provider/model" ist durch agents.defaults.modelPolicy.allow nicht zulässig.
Fügen Sie "provider/model", "provider/*" oder ein enger gefasstes Präfix "provider/namespace/*" zu agents.defaults.modelPolicy.allow hinzu oder entfernen/leeren Sie die Liste, um jedes Modell zuzulassen.
```

Beheben Sie dies, indem Sie das Modell oder einen Provider-Platzhalter zum genannten Schlüssel `modelPolicy.allow` hinzufügen, diese Liste entfernen/leeren oder ein Modell aus `/model list` auswählen. Wenn der abgelehnte Befehl eine Runtime-Überschreibung wie `/model openai/gpt-5.5 --runtime codex` enthielt, korrigieren Sie zuerst die Positivliste und versuchen Sie anschließend denselben Befehl erneut.

Für lokale/GGUF-Modelle muss die Positivliste die vollständige Referenz mit Provider-Präfix enthalten, beispielsweise `ollama/gemma4:26b` oder `lmstudio/Gemma4-26b-a4-it-gguf` – prüfen Sie `openclaw models list --provider <provider>` auf die exakte Zeichenfolge. Reine Dateinamen oder Anzeigenamen reichen nicht aus, sobald die Positivliste aktiv ist.

Um Provider einzuschränken, ohne jedes Modell aufzulisten, verwenden Sie Einträge mit abschließendem Präfix-Platzhalter. Ein Provider-weiter Eintrag `provider/*` entspricht jedem Modell dieses Providers; ein enger gefasstes Präfix wie `clawrouter/anthropic/*` entspricht nur diesem Namespace:

```json5
{
  agents: {
    defaults: {
      modelPolicy: {
        allow: ["openai/*", "vllm/*"],
      },
    },
  },
}
```

`/model`, `/models` und Modellauswahlmenüs zeigen dann nur den ermittelten Katalog dieser Provider an, und neue Modelle können erscheinen, ohne die Positivliste zu bearbeiten. Kombinieren Sie exakte `provider/model`-Einträge mit `provider/*`-Einträgen, um ein bestimmtes Modell eines anderen Providers einzubeziehen.

Beispiel einer Positivliste mit Aliassen und modellspezifischen Einstellungen:

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      modelPolicy: {
        allow: ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
}
```

<Accordion title="Positivliste explizit bearbeiten">
Legen Sie die vollständige Liste direkt fest:

```bash
openclaw config set agents.defaults.modelPolicy.allow '["openai/gpt-5.4","anthropic/*"]' --strict-json
```

`openclaw models set`, die Provider-Einrichtung und `openclaw models aliases add` können Einträge unter `agents.defaults.models` hinzufügen, ändern jedoch niemals `modelPolicy.allow`. Dadurch bleiben Modellmetadaten und Aliasse von der Überschreibungsrichtlinie unabhängig.
</Accordion>

## `/model` im Chat

```text
/model
/model list
/model 3
/model openai/gpt-5.4
/model default
/model status
```

- `/model` und `/model list` zeigen eine kompakte nummerierte Auswahl (Modellfamilie + verfügbare Provider); `/model <#>` wählt daraus aus. Auf Discord öffnet dies Dropdown-Menüs für Provider und Modell mit einem Submit-Schritt; auf Telegram gelten Auswahlen in der Auswahl nur für die Sitzung und überschreiben niemals den dauerhaften Standard des Agenten in `openclaw.json`. `/models add` ist veraltet und gibt eine Nachricht zurück, statt Modelle über den Chat zu registrieren.
- `/model` speichert die neue Sitzungsauswahl sofort. Wenn der Agent inaktiv ist, verwendet der nächste Lauf sie unmittelbar; wenn bereits ein Lauf aktiv ist, wird der Wechsel für den nächsten einwandfreien Wiederholungszeitpunkt vorgemerkt (oder einen späteren, falls bereits Tool-Aktivität oder Antwortausgabe begonnen hat).
- `/model default` löscht die Sitzungsauswahl, sodass sie wieder den konfigurierten Primärwert übernimmt.
- Eine vom Benutzer ausgewählte `/model`-Referenz gilt strikt für diese Sitzung: Wenn sie nicht mehr erreichbar ist, schlägt die Antwort sichtbar fehl, statt stillschweigend über `agents.defaults.model.fallbacks` auszuweichen. Konfigurierte Standardwerte und Primärmodelle von Cron-Aufträgen verwenden weiterhin Fallback-Ketten.
- `/model status` ist die Detailansicht: Authentifizierungskandidaten pro Provider sowie (falls konfiguriert) der Provider-Endpunkt `baseUrl` und der Modus `api`.
- Modellreferenzen werden durch Aufteilen am ersten `/` geparst; geben Sie `provider/model` ein. Wenn die Modell-ID selbst `/` enthält (im OpenRouter-Stil), fügen Sie das Provider-Präfix hinzu, z. B. `/model openrouter/moonshotai/kimi-k2`. Wenn Sie den Provider weglassen, versucht OpenClaw Folgendes: (1) Übereinstimmung mit einem Alias, (2) eindeutige Übereinstimmung mit einem konfigurierten Provider für genau diese Modell-ID ohne Präfix, (3) den konfigurierten Standard-Provider (veralteter Fallback) — und wenn dieser Provider das konfigurierte Standardmodell nicht mehr bereitstellt, stattdessen das erste konfigurierte Provider-/Modellpaar, damit kein veralteter Standardwert eines entfernten Providers angezeigt wird.
- Modellreferenzen werden in Kleinbuchstaben normalisiert; Provider-IDs müssen ansonsten exakt übereinstimmen. Verwenden Sie daher die vom Plugin angegebene ID.

Vollständiges Befehlsverhalten und Konfiguration: [Slash-Befehle](/de/tools/slash-commands).

## CLI

```bash
openclaw models status
openclaw models list
openclaw models set <provider/model>
openclaw models set-image <provider/model>
openclaw models scan
openclaw models aliases list|add|remove
openclaw models fallbacks list|add|remove|clear
openclaw models image-fallbacks list|add|remove|clear
openclaw models auth list|add|login|paste-api-key|paste-token|setup-token|order
```

`openclaw models` ohne Unterbefehl ist eine Kurzform für `models status`, das auch den Ablaufzeitpunkt von OAuth für Profile im Authentifizierungsspeicher anzeigt (standardmäßig erfolgt innerhalb von 24h eine Warnung). Vollständige Flags, JSON-Strukturen und Unterbefehle für Authentifizierungsprofile: [CLI-Referenz für Modelle](/de/cli/models).

<AccordionGroup>
  <Accordion title="Suche (kostenlose OpenRouter-Modelle)">
    `openclaw models scan` untersucht den öffentlichen Katalog kostenloser Modelle von OpenRouter und kann Kandidaten live auf Tool- und Bildunterstützung prüfen. Der Katalog selbst ist öffentlich, daher benötigen reine Metadatensuchen (`--no-probe`) keinen Schlüssel; Live-Prüfungen sowie `--set-default`/`--set-image` erfordern einen OpenRouter-API-Schlüssel (Authentifizierungsprofil oder `OPENROUTER_API_KEY`) und beschränken die Ausgabe ohne einen solchen sicher auf Metadaten.

    Die Ergebnisse werden nach folgenden Kriterien gewichtet: Bildunterstützung, dann Tool-Latenz, dann Kontextgröße und schließlich Parameteranzahl. In einem TTY fordern geprüfte Ergebnisse zur interaktiven Auswahl eines Fallbacks auf; im nicht interaktiven Modus ist `--yes` erforderlich, um die Standardwerte zu übernehmen.

  </Accordion>
</AccordionGroup>

## Modellregistrierung (`models.json`)

Unter `models.providers` konfigurierte benutzerdefinierte Provider werden unter dem Agentenverzeichnis (standardmäßig `~/.openclaw/agents/<agentId>/agent/models.json`) in `models.json` geschrieben. Kataloge von Provider-Plugins werden separat als generierte, Plugin-eigene Katalogfragmente gespeichert und automatisch geladen. Diese Datei wird standardmäßig mit der Konfiguration zusammengeführt; setzen Sie `models.mode: "replace"`, um ausschließlich Ihre konfigurierten Provider zu verwenden.

<AccordionGroup>
  <Accordion title="Priorität im Zusammenführungsmodus">
    Bei übereinstimmenden Provider-IDs:

    - Ein nicht leeres `baseUrl`, das bereits im `models.json` des Agenten vorhanden ist, hat Vorrang.
    - Ein nicht leeres `apiKey` in `models.json` hat nur dann Vorrang, wenn dieser Provider im aktuellen Kontext der Konfiguration bzw. des Authentifizierungsprofils nicht durch SecretRef verwaltet wird.
    - Durch SecretRef verwaltete `apiKey`-Werte werden anhand von Quellmarkierungen aktualisiert, statt aufgelöste Secrets dauerhaft zu speichern: der Name der Umgebungsvariablen bei Umgebungsreferenzen, `secretref-managed` bei Datei-/Ausführungsreferenzen.
    - Durch SecretRef verwaltete Header-Werte werden auf dieselbe Weise aktualisiert, wobei für Umgebungsreferenzen `secretref-env:ENV_VAR_NAME` verwendet wird.
    - Leere oder fehlende `apiKey`/`baseUrl` in `models.json` greifen auf `models.providers` aus der Konfiguration zurück.
    - Andere Provider-Felder werden anhand der Konfiguration und der normalisierten Katalogdaten aktualisiert.

  </Accordion>
</AccordionGroup>

Bei der dauerhaften Speicherung von Markierungen ist die Quelle maßgeblich: OpenClaw schreibt bei jeder Neugenerierung von `models.json` Markierungen aus dem aktiven Snapshot der Quellkonfiguration (vor der Auflösung), nicht aus aufgelösten Laufzeitwerten von Secrets — einschließlich befehlsgesteuerter Pfade wie `openclaw agent`.

## Verwandte Themen

- [Agentenlaufzeiten](/de/concepts/agent-runtimes) — OpenClaw, Codex und andere Laufzeiten für Agentenschleifen
- [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults) — Konfigurationsschlüssel für Modelle
- [Bilderzeugung](/de/tools/image-generation) — Konfiguration von Bildmodellen
- [Modell-Failover](/de/concepts/model-failover) — Fallback-Ketten
- [Modell-Provider](/de/concepts/model-providers) — Provider-Routing und Authentifizierung
- [CLI-Referenz für Modelle](/de/cli/models) — vollständige Referenz für Befehle und Flags
- [Musikerzeugung](/de/tools/music-generation) — Konfiguration von Musikmodellen
- [Videoerzeugung](/de/tools/video-generation) — Konfiguration von Videomodellen
