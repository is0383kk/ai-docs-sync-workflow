---
read_when:
    - Sie möchten Zugriff auf von OpenCode gehostete Modelle
    - Sie möchten zwischen den Katalogen Zen und Go wählen
summary: OpenCode-Zen- und Go-Kataloge mit OpenClaw verwenden
title: OpenCode
x-i18n:
    generated_at: "2026-07-26T19:12:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de287eb8a349f26c265f95b8b1de3af4035aa2bdc3501c7279f714d297bb8b9b
    source_path: providers/opencode.md
    workflow: 16
---

OpenCode stellt in OpenClaw zwei gehostete Kataloge bereit:

| Katalog | Präfix            | Runtime-Provider |
| ------- | ----------------- | ---------------- |
| **Zen** | `opencode/...`    | `opencode`       |
| **Go**  | `opencode-go/...` | `opencode-go`    |

Beide Kataloge verwenden denselben OpenCode-API-Schlüssel (`OPENCODE_API_KEY`, Alias
`OPENCODE_ZEN_API_KEY`). OpenClaw hält die IDs der Runtime-Provider getrennt, damit
das Upstream-Routing pro Modell korrekt bleibt; Onboarding und Dokumentation behandeln sie jedoch als
eine gemeinsame OpenCode-Konfiguration.

## Erste Schritte

<Tabs>
  <Tab title="Zen-Katalog">
    **Am besten geeignet für:** den kuratierten OpenCode-Multi-Modell-Proxy (Claude, GPT, Gemini, GLM,
    DeepSeek, Kimi, MiniMax, Qwen).

    <Steps>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice opencode-zen
        ```

        Oder übergeben Sie den Schlüssel direkt:

        ```bash
        openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Ein Zen-Modell als Standard festlegen">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode/claude-opus-4-6"
        ```
      </Step>
      <Step title="Verfügbarkeit der Modelle überprüfen">
        ```bash
        openclaw models list --provider opencode
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Go-Katalog">
    **Am besten geeignet für:** die von OpenCode gehostete Auswahl an Kimi, GLM, MiniMax, Qwen und DeepSeek.

    <Steps>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        Oder übergeben Sie den Schlüssel direkt:

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Ein Go-Modell als Standard festlegen">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```
      </Step>
      <Step title="Verfügbarkeit der Modelle überprüfen">
        ```bash
        openclaw models list --provider opencode-go
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Konfigurationsbeispiel

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## Integrierte Kataloge

### Zen

| Eigenschaft      | Wert                                                                                          |
| ---------------- | --------------------------------------------------------------------------------------------- |
| Runtime-Provider | `opencode`                                                                                    |
| Beispielmodelle  | `opencode/claude-opus-4-6`, `opencode/gpt-5.5`, `opencode/gemini-3.1-pro`, `opencode/glm-5.2` |

Führen Sie `openclaw models list --provider opencode` aus, um die vollständige aktuelle Liste anzuzeigen, die
auch Einträge des kostenlosen Tarifs wie `opencode/big-pickle` und
`opencode/deepseek-v4-flash-free` enthält.

### Go

| Eigenschaft      | Wert                                                                     |
| ---------------- | ------------------------------------------------------------------------ |
| Runtime-Provider | `opencode-go`                                                            |
| Beispielmodelle  | `opencode-go/kimi-k2.6`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5` |

Die vollständige Go-Modelltabelle finden Sie unter [OpenCode Go](/de/providers/opencode-go).

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Aliasse für API-Schlüssel">
    `OPENCODE_ZEN_API_KEY` wird ebenfalls als Alias für `OPENCODE_API_KEY` akzeptiert.
  </Accordion>

  <Accordion title="Gemeinsam genutzte Anmeldedaten">
    Wenn Sie während der Einrichtung einen OpenCode-Schlüssel eingeben, werden die Anmeldedaten für beide Runtime-
    Provider gespeichert. Sie müssen das Onboarding nicht für jeden Katalog separat durchführen.
  </Accordion>

  <Accordion title="API-Schlüssel beziehen">
    Erstellen Sie ein OpenCode-Konto und generieren Sie unter
    [opencode.ai/auth](https://opencode.ai/auth) einen API-Schlüssel. Abrechnung und Katalog-
    verfügbarkeit werden über das OpenCode-Dashboard verwaltet.
  </Accordion>

  <Accordion title="Replay-Verhalten von Gemini">
    Gemini-basierte OpenCode-Referenzen verbleiben im Proxy-Gemini-Pfad, sodass OpenClaw dort
    weiterhin die Bereinigung von Gemini-Denksignaturen durchführt, ohne die native Gemini-
    Replay-Validierung oder Bootstrap-Umschreibungen zu aktivieren.
  </Accordion>

  <Accordion title="Replay-Verhalten für Nicht-Gemini-Modelle">
    OpenCode-Referenzen, die nicht auf Gemini basieren, behalten die minimale OpenAI-kompatible Replay-Richtlinie bei.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="OpenCode Go" href="/de/providers/opencode-go" icon="server">
    Vollständige Referenz zum Go-Katalog.
  </Card>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/configuration-reference" icon="gear">
    Vollständige Konfigurationsreferenz für Agenten, Modelle und Provider.
  </Card>
</CardGroup>
