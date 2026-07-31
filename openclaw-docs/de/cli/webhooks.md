---
read_when:
    - Sie möchten Gmail-Pub/Sub-Ereignisse in OpenClaw integrieren
    - Sie benötigen die vollständige Liste der Flags und die Standardwerte
summary: CLI-Referenz für `openclaw webhooks` (Einrichtung und Runner für Gmail Pub/Sub)
title: Webhooks
x-i18n:
    generated_at: "2026-07-26T18:23:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

Webhook-Hilfsfunktionen und -Integrationen. Derzeit ist diese Oberfläche auf Gmail-Pub/Sub-Abläufe beschränkt, die auf dem gebündelten `gog`-Watcher basieren.

## Unterbefehle

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| Unterbefehl    | Beschreibung                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | Einmaliger Assistent: Gmail-Watch, Pub/Sub-Thema/-Abonnement und Zustellung an den OpenClaw-Hook. |
| `gmail run`   | Führt `gog watch serve` zusammen mit der Schleife zur automatischen Verlängerung des Watch im Vordergrund aus.               |

<Note>
Das Gateway startet `gog gmail watch serve` beim Hochfahren ebenfalls automatisch, sobald `hooks.enabled=true` und `hooks.gmail.account` festgelegt sind (durch `gmail setup`). `gmail run` verwendet dieselbe Logik im Vordergrund und eignet sich zur Fehlerbehebung oder wenn der Gateway-Watcher deaktiviert ist. Einzelheiten zum automatischen Start und zur Deaktivierung über `OPENCLAW_SKIP_GMAIL_WATCHER` finden Sie unter [Gmail-Pub/Sub-Integration](/de/automation/cron-jobs#gmail-pubsub-integration).
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

Installiert `gcloud` und `gog`, falls sie fehlen, authentifiziert `gcloud`, erstellt das Pub/Sub-Thema und -Abonnement, startet den Gmail-Watch und schreibt die `hooks.gmail`-Konfiguration mit `hooks.enabled=true`. Gibt `Next: openclaw webhooks gmail run` aus.

### Erforderlich

| Flag                | Beschreibung             |
| ------------------- | ----------------------- |
| `--account <email>` | Zu überwachendes Gmail-Konto. |

### Pub/Sub-Optionen

| Flag                    | Standard                | Beschreibung                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | (keiner)                 | GCP-Projekt-ID (Eigentümer des OAuth-Clients). Verwendet ersatzweise zunächst die Projekt-ID des Themas und danach das aus den Anmeldedaten von `gog` ermittelte Projekt. |
| `--topic <name>`        | `gog-gmail-watch`      | Name des Pub/Sub-Themas.                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Name des Pub/Sub-Abonnements.                                                                                                              |
| `--label <label>`       | `INBOX`                | Zu überwachendes Gmail-Label.                                                                                                                   |
| `--push-endpoint <url>` | (keiner)                 | Expliziter Pub/Sub-Push-Endpunkt. Überschreibt Tailscale.                                                                                    |

### OpenClaw-Zustellungsoptionen

| Flag                   | Standard                                      | Beschreibung                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | Aus `hooks.path` und dem Gateway-Port erstellt | OpenClaw-Webhook-URL.                      |
| `--hook-token <token>` | `hooks.token` oder ein generiertes Token          | OpenClaw-Webhook-Token.                    |
| `--push-token <token>` | Generiertes Token                              | An `gog watch serve` weitergeleitetes Push-Token. |

### Optionen für `gog watch serve`

| Flag                  | Standard         | Beschreibung                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | Bind-Host von `gog watch serve`.                                                                                                                 |
| `--port <port>`       | `8788`          | Port von `gog watch serve`.                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | Pfad von `gog watch serve`. Wird auf `/` festgelegt, wenn Tailscale ohne explizites Ziel aktiviert ist, da Tailscale den Pfad vor der Proxy-Weiterleitung entfernt. |
| `--include-body`      | `true`          | Textausschnitte aus E-Mail-Inhalten einschließen. Es gibt kein CLI-Flag, um dies zu deaktivieren; legen Sie stattdessen `hooks.gmail.includeBody: false` in der Konfiguration fest.                  |
| `--max-bytes <n>`     | `20000`         | Maximale Byteanzahl pro Textausschnitt.                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | Gmail-Watch alle N Minuten verlängern.                                                                                                           |

### Bereitstellung über Tailscale

| Flag                      | Standard  | Beschreibung                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | Push-Endpunkt über Tailscale bereitstellen: `funnel`, `serve` oder `off`. |
| `--tailscale-path <path>` | (keiner)   | Pfad für Tailscale Serve/Funnel.                                 |
| `--tailscale-target <t>`  | (keiner)   | Ziel für Tailscale Serve/Funnel (Port, `host:port` oder URL).       |

### Ausgabe

| Flag     | Beschreibung                                       |
| -------- | ------------------------------------------------- |
| `--json` | Eine maschinenlesbare Zusammenfassung anstelle von Text ausgeben. |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

Führt `gog watch serve` zusammen mit der Schleife zur automatischen Verlängerung des Watch im Vordergrund aus und startet `gog watch serve` nach einer Verzögerung von 2s neu, wenn es unerwartet beendet wird.

`run` akzeptiert dieselben Flags für Pub/Sub, OpenClaw-Zustellung, `gog watch serve` und Tailscale wie `setup`, mit folgenden Ausnahmen:

- `--account` ist bei `run` **optional**; ersatzweise wird `hooks.gmail.account` verwendet.
- `run` akzeptiert `--project`, `--push-endpoint` oder `--json` **nicht**.
- Jedes Flag verwendet ersatzweise zunächst den entsprechenden `hooks.gmail.*`-Konfigurationswert (geschrieben von `setup`) und danach denselben integrierten Standardwert wie `setup`, mit einer Ausnahme: `--tailscale` verwendet bei `run` standardmäßig `off` (nicht `funnel`), wenn weder das Flag noch `hooks.gmail.tailscale.mode` festgelegt ist.

| Kategorie          | Flags                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`, `--topic`, `--subscription`, `--label`                              |
| OpenClaw-Zustellung | `--hook-url`, `--hook-token`, `--push-token`                                     |
| `gog watch serve` | `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes` |
| Tailscale         | `--tailscale`, `--tailscale-path`, `--tailscale-target`                          |

<Note>
Für `run` ist der Wert von `--topic` der vollständige Pub/Sub-Themenpfad (`projects/.../topics/...`), nicht nur der kurze Themenname.
</Note>

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Webhook-Automatisierung](/de/automation/cron-jobs)
- [Gmail-Pub/Sub-Integration](/de/automation/cron-jobs#gmail-pubsub-integration)
