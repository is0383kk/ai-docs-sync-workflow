---
read_when:
    - Sie möchten die Standardmodelle ändern oder den Authentifizierungsstatus des Providers anzeigen.
    - Sie möchten verfügbare Modelle/Provider durchsuchen und Authentifizierungsprofile debuggen
summary: CLI-Referenz für `openclaw models` (Status/auflisten/festlegen/scannen, Aliasse, Fallbacks, Authentifizierung)
title: Modelle
x-i18n:
    generated_at: "2026-07-26T18:52:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f7405c25694f04afe9c3029a8af64ae3ae7e1bdcf4c4ac31b8b84ff512d6a90e
    source_path: cli/models.md
    workflow: 16
---

# `openclaw models`

Modellerkennung, Scans und Konfiguration (Standardmodell, Fallbacks, Authentifizierungsprofile).

Verwandte Themen:

- Provider und Modelle: [Modelle](/de/providers/models)
- Konzepte zur Modellauswahl und `/models`-Slash-Befehl: [Modellkonzept](/de/concepts/models)
- Einrichtung der Provider-Authentifizierung: [Erste Schritte](/de/start/getting-started)

## Häufig verwendete Befehle

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
openclaw models scan
```

Die Unterbefehle `status` und `auth` akzeptieren `--agent <id>`, um einen konfigurierten Agenten anzugeben; `list`, `scan`, `aliases` und `fallbacks`/`image-fallbacks` verwenden immer den konfigurierten Standardagenten, und `set`/`set-image` lehnen `--agent` grundsätzlich ab. Wenn die Angabe fehlt, verwenden `--agent`-fähige Befehle `OPENCLAW_AGENT_DIR`, sofern festgelegt, andernfalls den konfigurierten Standardagenten.

### Status

`openclaw models status` zeigt das aufgelöste Standardmodell und die Fallbacks sowie eine Authentifizierungsübersicht an. Bei Plugin-eigenen Agenten-Runtimes wie Codex wird außerdem geprüft, ob das zuständige Plugin aktiviert ist und die Überprüfung der Startnutzlast bestanden hat. Eine Route mit gültigen Anmeldedaten, aber nicht verfügbarer Runtime meldet `status: unavailable` statt `usable`; die JSON-Ausgabe enthält separate Angaben für `authStatus` und `runtimeStatus` sowie begrenzte Runtime-Diagnosen. Wenn Momentaufnahmen zur Provider-Nutzung verfügbar sind, enthält der OAuth-/API-Schlüssel-Statusabschnitt Nutzungszeiträume und Kontingentmomentaufnahmen des Providers. Derzeitige Provider für Nutzungszeiträume: Anthropic, GitHub Copilot, Gemini CLI, OpenAI, MiniMax, Xiaomi und z.ai. Die Authentifizierung für Nutzungsdaten erfolgt, sofern verfügbar, über Provider-spezifische Hooks; andernfalls greift OpenClaw auf passende OAuth-/API-Schlüssel-Anmeldedaten aus Authentifizierungsprofilen, Umgebungsvariablen oder der Konfiguration zurück.

In der Ausgabe von `--json` ist `auth.providers` die Umgebungsvariablen, Konfiguration und Speicher berücksichtigende Provider-Übersicht, während `auth.oauth` ausschließlich den Zustand der Profile im Authentifizierungsspeicher darstellt.

Optionen:

| Flag                      | Wirkung                                                                                                                                   |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`                  | JSON-Ausgabe; Diagnoseinformationen zu Authentifizierungsprofilen, Providern und Startvorgängen werden an stderr ausgegeben, damit stdout per Pipe an `jq` weitergeleitet werden kann.                            |
| `--plain`                 | Nur-Text-Ausgabe.                                                                                                                       |
| `--check`                 | Beendet den Prozess mit einem von null verschiedenen Status, wenn die Authentifizierung bald abläuft oder abgelaufen ist oder eine ausgewählte Agenten-Runtime nicht verfügbar ist: `1` = nicht verfügbar/abgelaufen/fehlend, `2` = läuft bald ab. |
| `--probe`                 | Live-Prüfung konfigurierter Authentifizierungsprofile. Sendet echte Anfragen; kann Token verbrauchen und Ratenbegrenzungen auslösen.                                       |
| `--probe-provider <name>` | Prüft nur einen Provider.                                                                                                                 |
| `--probe-profile <id>`    | Prüft bestimmte IDs von Authentifizierungsprofilen (wiederholbar oder durch Kommas getrennt).                                                                             |
| `--probe-timeout <ms>`    | Zeitlimit je Prüfung.                                                                                                                       |
| `--probe-concurrency <n>` | Gleichzeitige Prüfungen.                                                                                                                       |
| `--probe-max-tokens <n>`  | Maximale Anzahl von Token für die Prüfung (Best Effort).                                                                                                          |
| `--agent <id>`            | ID des konfigurierten Agenten; überschreibt `OPENCLAW_AGENT_DIR`.                                                                                     |

Prüfzeilen können aus Authentifizierungsprofilen, Anmeldedaten in Umgebungsvariablen oder `models.json` stammen. Statuskategorien der Prüfung: `ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`.

Zu erwartende Detail-/Ursachencodes, wenn eine Prüfung keinen Modellaufruf erreicht:

- `excluded_by_auth_order`: Es ist ein gespeichertes Profil vorhanden, wurde jedoch durch die explizite Angabe `auth.order.<provider>` ausgelassen. Daher meldet die Prüfung den Ausschluss, statt das Profil zu testen.
- `missing_credential`, `invalid_expires`, `expired`, `unresolved_ref`: Das Profil ist vorhanden, aber nicht geeignet oder nicht auflösbar.
- `ineligible_profile`: Das Profil ist aus einem anderen Grund nicht mit der Provider-Konfiguration kompatibel.
- `no_model`: Es ist eine Provider-Authentifizierung vorhanden, OpenClaw konnte jedoch keinen prüfbaren Modellkandidaten für diesen Provider auflösen.

Für die Fehlerbehebung bei OpenAI-ChatGPT-/Codex-OAuth lässt sich mit `openclaw models status`, `openclaw models auth list --provider openai` und `openclaw config get agents.defaults.model --json` am schnellsten feststellen, ob ein Agent über ein verwendbares `openai`-OAuth-Profil für `openai/*` über die native Codex-Runtime verfügt. Siehe [Einrichtung des OpenAI-Providers](/de/providers/openai#check-and-recover-codex-oauth-routing).

### Auflisten

`openclaw models list` ist schreibgeschützt: Der Befehl liest die Konfiguration, Authentifizierungsprofile, den vorhandenen Katalogstatus und Provider-eigene Katalogzeilen, schreibt `models.json` jedoch niemals neu.

Optionen: `--all` (vollständiger Katalog), `--local` (auf lokale Modelle beschränken), `--provider <id>`, `--json`, `--plain`.

Hinweise:

- Die Spalte `Auth` ist schreibgeschützt. Bei Provider-eigenen Modellrouten wie OpenAI gleicht sie die API-/Basis-URL-Route jeder Zeile mit geeigneten Profilen in der effektiven Konfiguration `auth.order`, Anmeldedaten aus Umgebungsvariablen oder der Konfiguration sowie aufgelösten, befehlsbezogenen SecretRefs ab. Eine konkrete OpenAI-Zeile behält einen unbekannten Status, wenn ihre Routenrichtlinie nicht verfügbar ist, statt die Authentifizierung auf Provider-Ebene zu übernehmen; alte, ausschließlich Provider-bezogene Prüfungen und andere Provider behalten das Verhalten auf Provider-Ebene bei. Metadaten zur synthetischen Authentifizierung eines Plugins sind lediglich ein Hinweis auf eine Runtime-Fähigkeit und kein Nachweis einer nativen Kontoauthentifizierung. Daher behalten kontoabhängige Routen ohne positive Registry-Nachweise einen unbekannten Status. Der Befehl lädt weder die Provider-Runtime noch liest er Schlüsselbundgeheimnisse, ruft Provider-APIs auf oder weist die genaue Ausführungsbereitschaft nach.
- `models list --all --provider <id>` kann Provider-eigene statische Katalogzeilen aus Plugin-Manifesten oder gebündelten Provider-Katalogmetadaten enthalten, auch wenn Sie sich noch nicht bei diesem Provider authentifiziert haben. Diese Zeilen werden weiterhin als nicht verfügbar angezeigt, bis eine passende Authentifizierung konfiguriert ist.
- `models list` hält die Steuerungsebene reaktionsfähig, wenn die Provider-Katalogerkennung langsam ist. Die Standardansicht und die konfigurierte Ansicht greifen nach einer kurzen Wartezeit auf konfigurierte oder synthetische Modellzeilen zurück und lassen die Erkennung im Hintergrund abschließen. Verwenden Sie `--all`, wenn Sie den vollständigen, exakt erkannten Katalog benötigen und bereit sind, auf die Provider-Erkennung zu warten.
- Das allgemeine `models list --all` führt Manifest-Katalogzeilen mit Registry-Zeilen zusammen, ohne ergänzende Hooks der Provider-Runtime zu laden. Nach Provider gefilterte Manifest-Schnellpfade verwenden nur Provider mit der Kennzeichnung `static`; Provider mit der Kennzeichnung `refreshable` bleiben Registry-/Cache-basiert und hängen Manifestzeilen als Ergänzungen an, während Provider mit der Kennzeichnung `runtime` weiterhin die Registry-/Runtime-Erkennung verwenden.
- `models list` behandelt native Modellmetadaten und Runtime-Begrenzungen getrennt. In der Tabellenausgabe zeigt `Ctx` den Wert `contextTokens/contextWindow`, wenn sich eine effektive Runtime-Begrenzung vom nativen Kontextfenster unterscheidet; JSON-Zeilen enthalten `contextTokens`, wenn ein Provider diese Begrenzung bereitstellt.
- Bei Provider-eigenen Routen projiziert `models list` eine logische Provider-/Modellzeile auf die ausgewählte Route. `Input` und `Ctx` stammen ausschließlich aus einer Katalogzeile für die exakte physische Route, wobei explizit konfigurierte logische Überschreibungen zuletzt angewendet werden; bei einer nicht aufgelösten Routenauswahl werden unbekannte Fähigkeitsfelder angezeigt, statt Metadaten einer benachbarten Route zu übernehmen.
- `models list --provider <id>` filtert nach der Provider-ID, beispielsweise `moonshot` oder `openai`. Anzeigenamen aus interaktiven Provider-Auswahlmenüs wie `Moonshot AI` werden nicht akzeptiert.
- Modellreferenzen werden am **ersten** `/` getrennt. Wenn die Modell-ID `/` enthält (OpenRouter-Stil), geben Sie das Provider-Präfix an (Beispiel: `openrouter/moonshotai/kimi-k2`).
- Wenn Sie den Provider auslassen, löst OpenClaw die Eingabe zunächst als Alias auf, danach als eindeutige Übereinstimmung mit der exakten Modell-ID bei einem konfigurierten Provider und greift erst anschließend mit einer Veraltungswarnung auf den konfigurierten Standard-Provider zurück. Wenn dieser Provider das konfigurierte Standardmodell nicht mehr bereitstellt, greift OpenClaw auf die erste konfigurierte Provider-/Modellkombination zurück, statt einen veralteten Standardwert eines entfernten Providers anzuzeigen.
- `models status` kann bei nicht geheimen Platzhaltern in der Authentifizierungsausgabe `marker(<value>)` anzeigen (beispielsweise `OPENAI_API_KEY`, `secretref-managed`, `minimax-oauth`, `oauth:chutes`, `ollama-local`), statt sie wie Geheimnisse zu maskieren.

### Standard-/Bildmodell festlegen

```bash
openclaw models set <model-or-alias>
openclaw models set-image <model-or-alias>
```

`set` schreibt `agents.defaults.model.primary`; `set-image` schreibt `agents.defaults.imageModel.primary`. Beide akzeptieren `provider/model` oder einen konfigurierten Alias. `set` repariert außerdem Installationen von Codex-/Copilot-Runtime-Plugins, wenn das neu ausgewählte Modell ein solches benötigt; `set-image` tut dies nicht. Keiner der beiden Befehle akzeptiert `--agent`; sie schreiben immer die Standardwerte des Agenten.

### Scannen

`models scan` liest den öffentlichen `:free`-Katalog von OpenRouter und ordnet Kandidaten für die Verwendung als Fallback. Der Katalog selbst ist öffentlich, sodass reine Metadatenscans keinen OpenRouter-Schlüssel benötigen.

Standardmäßig versucht OpenClaw, die Unterstützung für Tools und Bilder mit Live-Modellaufrufen zu prüfen. Wenn kein OpenRouter-Schlüssel konfiguriert ist, greift der Befehl auf eine reine Metadatenausgabe zurück und erläutert, dass `:free`-Modelle weiterhin `OPENROUTER_API_KEY` für Prüfungen und Inferenz benötigen.

Optionen:

- `--no-probe` (nur Metadaten; kein Zugriff auf Konfiguration/Geheimnisse)
- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>` (Zeitlimit für Kataloganfrage und einzelne Prüfungen)
- `--concurrency <n>`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

`--set-default` und `--set-image` erfordern Live-Prüfungen; Ergebnisse reiner Metadatenscans dienen nur zur Information und werden nicht auf die Konfiguration angewendet.

## Aliasse

```bash
openclaw models aliases list [--json] [--plain]
openclaw models aliases add <alias> <model-or-alias>
openclaw models aliases remove <alias>
```

Aliasse werden pro Modelleintrag als `agents.defaults.models.<key>.alias` gespeichert. `add` löst `<model-or-alias>` zunächst in einen kanonischen Provider-/Modellschlüssel auf. Wird einem Alias daher ein weiterer Alias zugewiesen, wird er neu zugeordnet, statt eine Kette zu bilden.
Das Hinzufügen eines Alias ändert `agents.defaults.modelPolicy.allow` nicht und schränkt Modellüberschreibungen nicht ein.

## Fallbacks

```bash
openclaw models fallbacks list [--json] [--plain]
openclaw models fallbacks add <model-or-alias>
openclaw models fallbacks remove <model-or-alias>
openclaw models fallbacks clear
```

Verwaltet `agents.defaults.model.fallbacks`. `openclaw models image-fallbacks list|add|remove|clear` verwaltet die parallele Liste `agents.defaults.imageModel.fallbacks` mit derselben Unterbefehlsstruktur.

## Authentifizierungsprofile

```bash
openclaw models auth add
openclaw models auth list [--provider <id>] [--json]
openclaw models auth login --provider <id>
openclaw models auth login --provider openai --profile-id openai:work
openclaw models auth login-github-copilot
openclaw models auth paste-api-key --provider <id>
openclaw models auth setup-token --provider <id>
openclaw models auth paste-token --provider <id>
openclaw models auth order get --provider <id>
openclaw models auth order set --provider <id> <profileIds...>
openclaw models auth order clear --provider <id>
```

`models auth add` ist die interaktive Authentifizierungshilfe. Abhängig vom ausgewählten Provider kann sie einen Authentifizierungsablauf des Providers (OAuth/API-Schlüssel) starten oder Sie beim manuellen Einfügen eines Tokens anleiten.

`models auth list` listet die gespeicherten Authentifizierungsprofile für den ausgewählten Agenten auf, ohne Token, API-Schlüssel oder geheime OAuth-Daten auszugeben. Verwenden Sie `--provider <id>`, um nach einem einzelnen Provider wie `openai` zu filtern, und `--json` für Skripte.

`models auth login` führt den Authentifizierungsablauf eines Provider-Plugins (OAuth/API-Schlüssel) aus. Mit `openclaw plugins list` können Sie anzeigen, welche Provider installiert sind. `login` akzeptiert `--profile-id <id>` für Provider, die bei der Anmeldung benannte Profile unterstützen (verwenden Sie dies, um mehrere Anmeldungen beim selben Provider voneinander zu trennen), `--method <id>` zur Auswahl einer bestimmten Authentifizierungsmethode, `--device-code` als Kurzform für `--method device-code`, `--set-default` zum Anwenden des vom Provider empfohlenen Standardmodells und `--force`, um zuerst vorhandene Profile dieses Providers zu entfernen (verwenden Sie dies, wenn ein zwischengespeichertes OAuth-Profil nicht mehr reagiert oder Sie das Konto wechseln möchten).

`models auth login-github-copilot` ist eine Kurzform für `models auth login --provider github-copilot --method device` (GitHub-Geräteablauf); der Befehl akzeptiert `--yes`, um ein vorhandenes Profil ohne Rückfrage zu überschreiben.

Verwenden Sie `openclaw models auth --agent <id> <subcommand>`, um Authentifizierungsergebnisse in den Speicher eines bestimmten konfigurierten Agenten zu schreiben. Das übergeordnete Flag `--agent` wird von `add`, `list`, `login`, `paste-api-key`, `setup-token`, `paste-token`, `login-github-copilot` und `order get`/`set`/`clear` berücksichtigt.

Für OpenAI-Modelle verwendet `--provider openai` standardmäßig die Anmeldung mit einem ChatGPT-/Codex-Konto. Verwenden Sie `--method api-key` nur, wenn Sie ein OpenAI-API-Schlüsselprofil hinzufügen möchten, üblicherweise als Absicherung für die Limits eines Codex-Abonnements. Führen Sie `openclaw doctor --fix` aus, um ältere Authentifizierungs-/Profilzustände mit dem veralteten OpenAI-Codex-Präfix zu `openai` zu migrieren.

Beispiele:

```bash
openclaw models auth login --provider openai --set-default
openclaw models auth login --provider openai --method api-key
openclaw models auth paste-api-key --provider openai
openclaw models auth list --provider openai
```

Hinweise:

- `paste-api-key` akzeptiert an anderer Stelle generierte API-Schlüssel, fragt den Schlüsselwert ab und schreibt ihn unter der standardmäßigen Profil-ID `<provider>:manual`, sofern Sie nicht `--profile-id` übergeben. Leiten Sie bei der Automatisierung den Schlüssel über die Standardeingabe weiter, beispielsweise mit `printf "%s\n" "$OPENAI_API_KEY" | openclaw models auth paste-api-key --provider openai`.
- `setup-token` und `paste-token` bleiben generische Token-Befehle für Provider, die Token-Authentifizierungsmethoden bereitstellen.
- `setup-token` erfordert ein interaktives TTY und führt die Token-Authentifizierungsmethode des Providers aus (standardmäßig dessen Methode `setup-token`, wenn der Provider eine solche bereitstellt).
- `paste-token` erfordert `--provider`, fragt standardmäßig den Token-Wert ab und schreibt ihn unter der standardmäßigen Profil-ID `<provider>:manual`, sofern Sie nicht `--profile-id` übergeben. Leiten Sie bei der Automatisierung das Token über die Standardeingabe weiter, statt es als Argument zu übergeben, damit die Zugangsdaten des Providers nicht im Shell-Verlauf oder in Prozesslisten erscheinen.
- `paste-token --expires-in <duration>` speichert anhand einer relativen Dauer wie `365d` oder `12h` einen absoluten Ablaufzeitpunkt für das Token.
- Bei `openai` haben OpenAI-API-Schlüssel und ChatGPT-/OAuth-Token-Daten unterschiedliche Authentifizierungsformen. Verwenden Sie `paste-api-key` für OpenAI-API-Schlüssel vom Typ `sk-...` und `paste-token` ausschließlich für Token-Authentifizierungsdaten.
- Anthropic: `setup-token`/`paste-token` sind unterstützte OpenClaw-Authentifizierungswege für `anthropic`; OpenClaw bevorzugt jedoch die Wiederverwendung der Claude CLI (`claude -p`) auf dem Host, sofern sie verfügbar ist.
- `auth order get/set/clear` verwaltet für einen Provider eine agentenspezifische Überschreibung der Reihenfolge von Authentifizierungsprofilen, die in `auth-state.json` gespeichert wird (getrennt vom Konfigurationsschlüssel `auth.order.<provider>`). `set` akzeptiert eine oder mehrere Profil-IDs in Prioritätsreihenfolge; `clear` greift wieder auf die Konfigurations-/Round-Robin-Reihenfolge zurück.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Modellauswahl](/de/concepts/model-providers)
- [Modell-Failover](/de/concepts/model-failover)
