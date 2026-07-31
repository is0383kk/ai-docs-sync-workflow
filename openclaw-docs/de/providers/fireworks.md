---
read_when:
    - Sie möchten Fireworks mit OpenClaw verwenden
    - Sie benötigen die Umgebungsvariable für den Fireworks-API-Schlüssel oder die Standardmodell-ID
    - Sie debuggen das Verhalten von Kimi bei deaktiviertem Thinking auf Fireworks
summary: Fireworks-Einrichtung (Authentifizierung + Modellauswahl)
title: Fireworks
x-i18n:
    generated_at: "2026-07-26T18:06:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7720b23b69aa716d2e2903f5644bb74f81ca1c5e753f71d72d4d7a25c0747884
    source_path: providers/fireworks.md
    workflow: 16
---

[Fireworks](https://fireworks.ai) stellt Open-Weight- und geroutete Modelle über eine OpenAI-kompatible API bereit. Installieren Sie das offizielle Fireworks-Provider-Plugin, um zwei vorkatalogisierte Kimi-Modelle sowie jede beliebige Fireworks-Modell- oder Router-ID zur Laufzeit zu verwenden.

| Eigenschaft        | Wert                                                  |
| --------------- | ------------------------------------------------------ |
| Provider-ID     | `fireworks` (Alias: `fireworks-ai`)                    |
| Paket         | `@openclaw/fireworks-provider`                         |
| Umgebungsvariable für die Authentifizierung    | `FIREWORKS_API_KEY`                                    |
| Onboarding-Flag | `--auth-choice fireworks-api-key`                      |
| Direkter CLI-Flag | `--fireworks-api-key <key>`                            |
| API             | OpenAI-kompatibel (`openai-completions`)               |
| Basis-URL        | `https://api.fireworks.ai/inference/v1`                |
| Standardmodell   | `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` |
| Standardalias   | `Kimi K2.6 Turbo`                                      |

## Erste Schritte

<Steps>
  <Step title="Plugin installieren">
    ```bash
    openclaw plugins install @openclaw/fireworks-provider
    ```
  </Step>
  <Step title="Fireworks-API-Schlüssel festlegen">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice fireworks-api-key
```

```bash Direkter Flag
openclaw onboard --non-interactive \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY"
```

```bash Nur Umgebungsvariable
export FIREWORKS_API_KEY=fw-...
```

    </CodeGroup>

    Das Onboarding speichert den Schlüssel für den Provider `fireworks` in Ihren Authentifizierungsprofilen und legt den Kimi-K2.6-Turbo-Router **Fire Pass** als Standardmodell fest.

  </Step>
  <Step title="Verfügbarkeit des Modells überprüfen">
    ```bash
    openclaw models list --provider fireworks
    ```

    Die Liste sollte `Kimi K2.6` und `Kimi K2.6 Turbo (Fire Pass)` enthalten. Wenn `FIREWORKS_API_KEY` nicht aufgelöst wird, meldet `openclaw models status --json` die fehlenden Anmeldedaten unter `auth.unusableProfiles`.

  </Step>
</Steps>

## Nicht interaktive Einrichtung

Übergeben Sie bei skriptgesteuerten oder CI-Installationen alle Angaben über die Befehlszeile:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## Integrierter Katalog

| Modellreferenz                                              | Name                        | Eingabe        | Kontext | Maximale Ausgabe | Thinking             |
| ------------------------------------------------------ | --------------------------- | ------------ | ------- | ---------- | -------------------- |
| `fireworks/accounts/fireworks/models/kimi-k2p6`        | Kimi K2.6                   | Text + Bild | 262,144 | 262,144    | Erzwungen deaktiviert           |
| `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` | Kimi K2.6 Turbo (Fire Pass) | Text + Bild | 256,000 | 256,000    | Erzwungen deaktiviert (Standard) |

<Note>
  OpenClaw setzt alle Fireworks-Kimi-Modelle fest auf `thinking: off`, da Kimi auf Fireworks Gedankengänge in der sichtbaren Antwort offenlegen kann, sofern die Anfrage Thinking nicht ausdrücklich deaktiviert. Wird dasselbe Modell direkt über [Moonshot](/de/providers/moonshot) geroutet, bleibt die Reasoning-Ausgabe von Kimi erhalten. Informationen zum Wechsel zwischen Providern finden Sie unter [Thinking-Modi](/de/tools/thinking).
</Note>

## Benutzerdefinierte Fireworks-Modell-IDs

OpenClaw akzeptiert zur Laufzeit jede beliebige Fireworks-Modell- oder Router-ID. Verwenden Sie die von Fireworks angezeigte exakte ID und stellen Sie ihr `fireworks/` voran. Bei der dynamischen Auflösung wird die Fire-Pass-Vorlage geklont (Text- und Bildeingabe, OpenAI-kompatible API, Standardkosten null), und Thinking wird automatisch deaktiviert, wenn die ID dem Kimi-Muster entspricht. Dynamische GLM-IDs werden als reine Textmodelle gekennzeichnet, sofern Sie keinen benutzerdefinierten Modelleintrag mit Bildeingabe konfigurieren.

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/models/<your-model-id>",
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Funktionsweise der Modell-ID-Präfixe">
    Jede Fireworks-Modellreferenz in OpenClaw beginnt mit `fireworks/`, gefolgt von der exakten ID oder dem Router-Pfad der Fireworks-Plattform. Beispiel:

    - Router-Modell: `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo`
    - Direktes Modell: `fireworks/accounts/fireworks/models/<model-name>`

    OpenClaw entfernt beim Erstellen der API-Anfrage das Präfix `fireworks/` und sendet den verbleibenden Pfad als OpenAI-kompatibles Feld `model` an den Fireworks-Endpunkt.

  </Accordion>

  <Accordion title="Warum Thinking für Kimi erzwungen deaktiviert wird">
    Fireworks stellt Kimi ohne separaten Reasoning-Kanal bereit, sodass Gedankengänge im sichtbaren `content`-Stream erscheinen können. Bei jeder Fireworks-Kimi-Anfrage sendet OpenClaw `thinking: { type: "disabled" }` und entfernt `reasoning`, `reasoning_effort` sowie `reasoningEffort` aus der Nutzlast (`extensions/fireworks/stream.ts`). Die Provider-Richtlinie (`extensions/fireworks/thinking-policy.ts`) gibt für Kimi-Modell-IDs ausschließlich die Thinking-Stufe `off` an, sodass manuelle `/think`-Umschaltungen und Provider-Richtlinienoberflächen mit dem Laufzeitvertrag übereinstimmen.

    Um Kimi-Reasoning durchgängig zu verwenden, konfigurieren Sie den [Moonshot-Provider](/de/providers/moonshot) und routen Sie dasselbe Modell darüber.

  </Accordion>

  <Accordion title="Verfügbarkeit der Umgebung für den Daemon">
    Wenn der Gateway als verwalteter Dienst ausgeführt wird (launchd, systemd, Docker), muss der Fireworks-Schlüssel für diesen Prozess sichtbar sein – nicht nur für Ihre interaktive Shell.

    <Warning>
      Ein Schlüssel, der nur in einer interaktiven Shell exportiert wurde, hilft einem launchd- oder systemd-Daemon nicht, sofern diese Umgebung dort nicht ebenfalls importiert wird. Legen Sie den Schlüssel in `~/.openclaw/.env` oder über `env.shellEnv` fest, damit er vom Gateway-Prozess gelesen werden kann.
    </Warning>

    OpenClaw lädt `~/.openclaw/.env` beim Laden der Konfiguration, sodass dort gespeicherte Schlüssel auf jeder Plattform die verwalteten Gateway-Dienste erreichen. Starten Sie den Gateway neu (oder führen Sie `openclaw doctor --fix` erneut aus), nachdem Sie den Schlüssel rotiert haben.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modell-Provider" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="Thinking-Modi" href="/de/tools/thinking" icon="brain">
    `/think`-Stufen, Provider-Richtlinien und das Routing Reasoning-fähiger Modelle.
  </Card>
  <Card title="Moonshot" href="/de/providers/moonshot" icon="moon">
    Führen Sie Kimi über die eigene API von Moonshot mit nativer Thinking-Ausgabe aus.
  </Card>
  <Card title="Fehlerbehebung" href="/de/help/troubleshooting" icon="wrench">
    Allgemeine Fehlerbehebung und häufig gestellte Fragen.
  </Card>
</CardGroup>
