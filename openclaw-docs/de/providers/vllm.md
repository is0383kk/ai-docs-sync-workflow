---
read_when:
    - Sie möchten OpenClaw mit einem lokalen vLLM-Server ausführen
    - Sie möchten OpenAI-kompatible /v1-Endpunkte mit Ihren eigenen Modellen
summary: OpenClaw mit vLLM ausführen (OpenAI-kompatibler lokaler Server)
title: vLLM
x-i18n:
    generated_at: "2026-07-26T18:04:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM stellt Open-Source-Modelle (und einige benutzerdefinierte Modelle) über eine **OpenAI-kompatible** HTTP-API bereit. OpenClaw stellt die Verbindung über die `openai-completions`-API her und kann Modelle **automatisch erkennen**, wenn Sie dies mit `VLLM_API_KEY` aktivieren.

| Eigenschaft      | Wert                                       |
| ---------------- | ------------------------------------------ |
| Provider-ID      | `vllm`                         |
| API              | `openai-completions` (OpenAI-kompatibel)     |
| Authentifizierung | Umgebungsvariable `VLLM_API_KEY`      |
| Standard-Basis-URL | `http://127.0.0.1:8000/v1`                       |
| Streaming-Nutzung | Unterstützt (`stream_options.include_usage`)          |

## Erste Schritte

<Steps>
  <Step title="vLLM mit einem OpenAI-kompatiblen Server starten">
    Ihre Basis-URL muss `/v1`-Endpunkte bereitstellen (`/v1/models`, `/v1/chat/completions`). vLLM wird üblicherweise unter folgender Adresse ausgeführt:

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="Umgebungsvariable für den API-Schlüssel festlegen">
    Jeder nicht leere Wert funktioniert, wenn Ihr Server keine Authentifizierung erzwingt:

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="Modell auswählen">
    Ersetzen Sie die Angabe durch eine Ihrer vLLM-Modell-IDs:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="Verfügbarkeit des Modells überprüfen">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
Übergeben Sie für die nicht interaktive Einrichtung (CI, Skripterstellung) die Basis-URL, den Schlüssel und das Modell direkt:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## Modellerkennung (impliziter Provider)

Wenn `VLLM_API_KEY` festgelegt ist (oder ein Authentifizierungsprofil vorhanden ist) und `models.providers.vllm` **nicht** definiert ist, fragt OpenClaw `GET http://127.0.0.1:8000/v1/models` ab und wandelt die zurückgegebenen IDs in Modelleinträge um.

<Note>
Wenn Sie `models.providers.vllm` ausdrücklich festlegen, verwendet OpenClaw nur die von Ihnen deklarierten Modelle. Fügen Sie `"vllm/*": {}` zu `agents.defaults.models` hinzu, damit OpenClaw außerdem den `/models`-Endpunkt dieses konfigurierten Providers abfragt und alle angebotenen vLLM-Modelle einschließt.
</Note>

## Explizite Konfiguration

Konfigurieren Sie den Provider ausdrücklich, wenn vLLM auf einem anderen Host oder Port ausgeführt wird, Sie `contextWindow`/`maxTokens` fest vorgeben möchten, Ihr Server einen echten API-Schlüssel erfordert oder Sie eine Verbindung zu einem vertrauenswürdigen Loopback-, LAN- oder Tailscale-Endpunkt herstellen:

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // Optional: Anforderungszeitüberschreitung für langsame lokale Modelle verlängern
        models: [
          {
            id: "your-model-id",
            name: "Lokales vLLM-Modell",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Fügen Sie dem sichtbaren Modellkatalog einen Platzhalter hinzu, um den Provider dynamisch zu halten, ohne jedes Modell aufzuführen:

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Proxyähnliches Verhalten">
    vLLM wird als proxyähnliches OpenAI-kompatibles `/v1`-Backend und nicht als nativer OpenAI-Endpunkt behandelt:

    | Verhalten                              | Angewendet?                                   |
    | -------------------------------------- | --------------------------------------------- |
    | Native OpenAI-Anforderungsaufbereitung | Nein                                          |
    | `service_tier`                     | Wird nicht gesendet                           |
    | Responses `store`           | Wird nicht gesendet                           |
    | Hinweise zum Prompt-Cache              | Werden nicht gesendet                         |
    | OpenAI-kompatible Aufbereitung der Reasoning-Nutzlast | Wird nicht angewendet             |
    | Verborgene OpenClaw-Zuordnungs-Header  | Werden bei benutzerdefinierten Basis-URLs nicht eingefügt |

  </Accordion>

  <Accordion title="Qwen-Steuerung für Denkprozesse">
    Legen Sie bei Qwen-Modellen `compat.thinkingFormat: "qwen-chat-template"` in der Modellzeile fest, wenn der Server Schlüsselwortargumente für die Qwen-Chatvorlage erwartet. Diese Modelle stellen ein binäres `/think`-Profil (`off`, `on`) bereit, da Denkprozesse in Qwen-Chatvorlagen ein Ein/Aus-Schalter und keine OpenAI-ähnliche Aufwandsabstufung sind.

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw ordnet `/think off` Folgendem zu:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    Denkstufen außerhalb von `off` senden `enable_thinking: true`. Wenn Ihr Endpunkt stattdessen Flags auf oberster Ebene im DashScope-Stil erwartet, verwenden Sie `compat.thinkingFormat: "qwen"`, um `enable_thinking` auf der Wurzelebene der Anfrage zu senden.

  </Accordion>

  <Accordion title="Nemotron-3-Steuerung für Denkprozesse">
    Für `vllm/nemotron-3-*`-Modelle mit deaktiviertem Denkprozess sendet das gebündelte Plugin:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    Um diese Werte anzupassen, legen Sie `chat_template_kwargs` unter den Modellparametern fest. Wenn Sie außerdem `params.extra_body.chat_template_kwargs` festlegen, hat dieser Wert Vorrang, da `extra_body` die letzte Überschreibung des Anfragekörpers ist.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen-Toolaufrufe werden als Text angezeigt">
    Vergewissern Sie sich zunächst, dass vLLM mit dem richtigen Toolaufruf-Parser und der richtigen Chatvorlage für das Modell gestartet wurde. vLLM dokumentiert `hermes` für Qwen2.5-Modelle und `qwen3_xml` für Qwen3-Coder-Modelle.

    Symptome: Skills/Tools werden nie ausgeführt, der Assistent gibt unaufbereitetes JSON/XML wie `{"name":"read","arguments":...}` aus oder vLLM gibt ein leeres `tool_calls`-Array zurück, wenn OpenClaw `tool_choice: "auto"` sendet.

    Einige Qwen/vLLM-Kombinationen geben strukturierte Toolaufrufe nur zurück, wenn die Anfrage `tool_choice: "required"` verwendet. Erzwingen Sie dies mit `params.extra_body` für jedes Modell einzeln:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    Ersetzen Sie die Modell-ID durch die genaue ID aus `openclaw models list --provider vllm` oder wenden Sie dieselbe Überschreibung über die CLI an:

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    Dies ist eine optionale Problemumgehung: Sie erzwingt bei jeder Interaktion mit Tools einen Toolaufruf. Verwenden Sie sie daher nur für einen dedizierten Modelleintrag, bei dem dies vertretbar ist. Legen Sie sie nicht als globalen Standard für alle vLLM-Modelle fest und kombinieren Sie sie nicht mit einem Proxy, der beliebigen Assistententext in ausführbare Toolaufrufe umwandelt.

  </Accordion>

  <Accordion title="Benutzerdefinierte Basis-URL">
    Wenn Ihr vLLM-Server auf einem vom Standard abweichenden Host oder Port ausgeführt wird, legen Sie `baseUrl` in der expliziten Provider-Konfiguration fest:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "Entferntes vLLM-Modell",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Langsame erste Antwort oder Zeitüberschreitung des entfernten Servers">
    Legen Sie für große lokale Modelle, entfernte LAN-Hosts oder Tailnet-Verbindungen eine Provider-spezifische Anforderungszeitüberschreitung fest:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "Lokales vLLM-Modell" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` gilt ausschließlich für HTTP-Anfragen an vLLM-Modelle: Verbindungsaufbau, Antwort-Header, Streaming des Antwortkörpers und den gesamten geschützten Fetch-Abbruch. Außerdem wird dadurch die Obergrenze des LLM-Inaktivitäts-/Streaming-Watchdogs über den impliziten Standardwert von ~120s für diesen Provider angehoben. Ziehen Sie dies einer Erhöhung von `agents.defaults.timeoutSeconds` vor, da diese Einstellung den gesamten Agentenlauf steuert.

  </Accordion>

  <Accordion title="Server nicht erreichbar">
    Prüfen Sie, ob der vLLM-Server ausgeführt wird und erreichbar ist:

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    Wenn ein Verbindungsfehler angezeigt wird, überprüfen Sie den Host, den Port und ob vLLM im OpenAI-kompatiblen Servermodus gestartet wurde. OpenClaw vertraut dem exakt konfigurierten Ursprung `models.providers.vllm.baseUrl` für geschützte Modellanfragen an Loopback-, LAN- und Tailscale-Endpunkte. Metadaten- und Link-Local-Ursprünge bleiben ohne ausdrückliche Aktivierung gesperrt. Legen Sie `models.providers.vllm.request.allowPrivateNetwork: true` nur fest, wenn vLLM-Anfragen einen anderen privaten Ursprung erreichen müssen, oder `false`, um das Vertrauen in den exakten Ursprung zu deaktivieren.

  </Accordion>

  <Accordion title="Authentifizierungsfehler bei Anfragen">
    Wenn Anfragen aufgrund von Authentifizierungsfehlern fehlschlagen, legen Sie einen echten `VLLM_API_KEY` fest, der Ihrer Serverkonfiguration entspricht, oder konfigurieren Sie den Provider ausdrücklich unter `models.providers.vllm`.

    <Tip>
    Wenn Ihr vLLM-Server keine Authentifizierung erzwingt, funktioniert jeder nicht leere Wert für `VLLM_API_KEY` als Aktivierungssignal für OpenClaw.
    </Tip>

  </Accordion>

  <Accordion title="Keine Modelle erkannt">
    Für die automatische Erkennung muss `VLLM_API_KEY` festgelegt sein. Wenn Sie `models.providers.vllm` definiert haben, verwendet OpenClaw nur die von Ihnen deklarierten Modelle, sofern `agents.defaults.models` nicht `"vllm/*": {}` enthält.
  </Accordion>

  <Accordion title="Tools werden als unaufbereiteter Text dargestellt">
    Wenn ein Qwen-Modell JSON/XML-Toolsyntax ausgibt, statt einen Skill auszuführen:

    - Starten Sie vLLM mit dem richtigen Parser/der richtigen Vorlage für dieses Modell.
    - Bestätigen Sie die genaue Modell-ID mit `openclaw models list --provider vllm`.
    - Fügen Sie nur dann eine dedizierte modellspezifische Überschreibung mit `params.extra_body.tool_choice: "required"` hinzu, wenn `tool_choice: "auto"` weiterhin leere oder ausschließlich textbasierte Toolaufrufe zurückgibt.

  </Accordion>
</AccordionGroup>

<Warning>
Weitere Hilfe: [Fehlerbehebung](/de/help/troubleshooting) und [FAQ](/de/help/faq).
</Warning>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="OpenAI" href="/de/providers/openai" icon="bolt">
    Nativer OpenAI-Provider und Verhalten OpenAI-kompatibler Routen.
  </Card>
  <Card title="OAuth und Authentifizierung" href="/de/gateway/authentication" icon="key">
    Details zur Authentifizierung und Regeln für die Wiederverwendung von Anmeldedaten.
  </Card>
  <Card title="Fehlerbehebung" href="/de/help/troubleshooting" icon="wrench">
    Häufige Probleme und deren Behebung.
  </Card>
</CardGroup>
