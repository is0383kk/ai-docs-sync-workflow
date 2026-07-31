---
read_when:
    - Sie möchten von Bedrock Mantle gehostete OSS-Modelle mit OpenClaw verwenden
    - Sie benötigen den OpenAI-kompatiblen Mantle-Endpunkt für GPT-OSS, Qwen, Kimi oder GLM
    - Sie möchten Claude Opus 5, Sonnet 5 oder Mythos 5 über Amazon Bedrock Mantle verwenden
summary: Verwenden Sie die OpenAI-kompatiblen und Claude-Messages-Modelle von Amazon Bedrock Mantle mit OpenClaw
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-07-26T18:02:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d2b49120560c4466aff217c3101fab057dd87c1c501f1b8eb94d74f62bd1037
    source_path: providers/bedrock-mantle.md
    workflow: 16
---

OpenClaw enthält einen gebündelten **Amazon Bedrock Mantle**-Provider, der eine Verbindung zum
OpenAI-kompatiblen Mantle-Endpunkt herstellt. Mantle stellt Open-Source- und
Drittanbietermodelle (GPT-OSS, Qwen, Kimi, GLM und ähnliche) über eine standardmäßige
`/v1/chat/completions`-Schnittstelle bereit, die auf Bedrock-Infrastruktur basiert. Mantle stellt außerdem
Anthropic-Claude-Modelle über eine Anthropic-Messages-Route bereit.

| Eigenschaft    | Wert                                                                                   |
| -------------- | -------------------------------------------------------------------------------------- |
| Provider-ID    | `amazon-bedrock-mantle`                                                                |
| API            | `openai-completions` für erkannte OSS-Modelle, `anthropic-messages` für Claude-Modelle |
| Authentifizierung | Explizites `AWS_BEARER_TOKEN_BEDROCK` oder Bearer-Token-Generierung über die IAM-Anmeldedatenkette |
| Standardregion | `us-east-1` (überschreibbar mit `AWS_REGION` oder `AWS_DEFAULT_REGION`)                       |

## Erste Schritte

Wählen Sie Ihre bevorzugte Authentifizierungsmethode und führen Sie die Einrichtungsschritte aus.

<Tabs>
  <Tab title="Explizites Bearer-Token">
    **Am besten geeignet für:** Umgebungen, in denen bereits ein Mantle-Bearer-Token vorhanden ist.

    <Steps>
      <Step title="Bearer-Token auf dem Gateway-Host festlegen">
        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        Legen Sie optional eine Region fest (Standardwert: `us-east-1`):

        ```bash
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Erkennung der Modelle überprüfen">
        ```bash
        openclaw models list
        ```

        Erkannte Modelle werden unter dem Provider `amazon-bedrock-mantle` angezeigt. Es ist keine
        zusätzliche Konfiguration erforderlich, sofern Sie die Standardwerte nicht überschreiben möchten.
      </Step>
    </Steps>

  </Tab>

  <Tab title="IAM-Anmeldedaten">
    **Am besten geeignet für:** die Verwendung AWS-SDK-kompatibler Anmeldedaten (gemeinsame Konfiguration, SSO, Webidentität, Instanz- oder Aufgabenrollen).

    <Steps>
      <Step title="AWS-Anmeldedaten auf dem Gateway-Host konfigurieren">
        Jede AWS-SDK-kompatible Authentifizierungsquelle wird unterstützt:

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="Erkennung der Modelle überprüfen">
        ```bash
        openclaw models list
        ```

        OpenClaw generiert automatisch ein Mantle-Bearer-Token aus der Anmeldedatenkette.
      </Step>
    </Steps>

    <Tip>
    Wenn `AWS_BEARER_TOKEN_BEDROCK` nicht festgelegt ist, erstellt OpenClaw das Bearer-Token für Sie aus der standardmäßigen AWS-Anmeldedatenkette, einschließlich gemeinsamer Anmeldedaten-/Konfigurationsprofile, SSO, Webidentität sowie Instanz- oder Aufgabenrollen.
    </Tip>

  </Tab>
</Tabs>

## Automatische Modellerkennung

Wenn `AWS_BEARER_TOKEN_BEDROCK` festgelegt ist, verwendet OpenClaw es direkt. Andernfalls
versucht OpenClaw, ein Mantle-Bearer-Token aus der standardmäßigen
AWS-Anmeldedatenkette zu generieren. Anschließend erkennt es verfügbare Mantle-Modelle durch Abfragen des
regionalen Endpunkts `/v1/models`.

| Verhalten         | Details                                                                              |
| ----------------- | ------------------------------------------------------------------------------------ |
| Erkennungs-Cache  | Ergebnisse werden pro Region 1 Stunde zwischengespeichert; bei einem Abruffehler wird das letzte zwischengespeicherte Ergebnis zurückgegeben |
| IAM-Token-Aktualisierung | Alle 2 Stunden, pro Region zwischengespeichert                                  |

Um das Mantle-Plugin aktiviert zu lassen, aber die automatische Erkennung und die
IAM-Bearer-Token-Generierung zu unterdrücken, deaktivieren Sie den Plugin-eigenen Erkennungsschalter:

```bash
openclaw config set plugins.entries.amazon-bedrock-mantle.config.discovery.enabled false
```

<Note>
Das Bearer-Token ist dasselbe `AWS_BEARER_TOKEN_BEDROCK`, das vom standardmäßigen [Amazon Bedrock](/de/providers/bedrock)-Provider verwendet wird.
</Note>

### Unterstützte Regionen

`us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## Manuelle Konfiguration

Wenn Sie eine explizite Konfiguration anstelle der automatischen Erkennung bevorzugen:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

Eine explizite, nicht leere `models`-Liste ist maßgeblich und ersetzt jede
erkannte Zeile, einschließlich der nachfolgenden Claude-Zeilen. Lassen Sie `models` weg, um den
automatischen Mantle-Katalog beizubehalten, oder geben Sie die vollständigen Einträge der Claude-Modelle an, die Sie
verwenden möchten.

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Reasoning-Unterstützung">
    Die Reasoning-Unterstützung wird aus Modell-IDs abgeleitet, die Muster wie
    `thinking`, `reasoner`, `reasoning`, `deepseek.r`, `gpt-oss-120b` oder
    `gpt-oss-safeguard-120b` enthalten. OpenClaw setzt während der Erkennung für
    übereinstimmende Modelle automatisch `reasoning: true`.
  </Accordion>

  <Accordion title="Nichtverfügbarkeit des Endpunkts">
    Wenn der Mantle-Endpunkt nicht verfügbar ist, keine Modelle zurückgibt oder die
    Auflösung des Bearer-Tokens fehlschlägt, liefert die Erkennung ein leeres Ergebnis zurück und der implizite
    Provider wird übersprungen. OpenClaw gibt keinen Fehler aus; andere konfigurierte Provider
    funktionieren weiterhin normal.
  </Accordion>

  <Accordion title="Claude über die Anthropic-Messages-Route">
    Wenn die automatische Erkennung die Modellliste verwaltet, hängt OpenClaw nach einer erfolgreichen Abfrage fünf Claude-
    Modelle an, unabhängig davon, was `/v1/models` zurückgibt:
    `amazon-bedrock-mantle/anthropic.claude-opus-5` (Claude Opus 5),
    `amazon-bedrock-mantle/anthropic.claude-sonnet-5` (Claude Sonnet 5),
    `amazon-bedrock-mantle/anthropic.claude-opus-4-7` (Claude Opus 4.7) und
    `amazon-bedrock-mantle/anthropic.claude-mythos-5` (Claude Mythos 5) sowie
    `amazon-bedrock-mantle/anthropic.claude-mythos-preview` (Claude Mythos
    Preview). Sie verwenden die API-Schnittstelle `anthropic-messages` und streamen über
    denselben per Bearer-Token authentifizierten Anthropic-kompatiblen Endpunkt
    (`<mantle-base>/anthropic`), sodass das AWS-Bearer-Token nicht wie ein
    Anthropic-API-Schlüssel behandelt wird.

    Claude Opus 5 weist ein Kontextfenster von 1.000.000 Token, ein
    Ausgabelimit von 128.000 Token, Bildeingabe und `$5/$25`-Preise für Ein- und Ausgabe aus. Adaptives
    Denken verwendet standardmäßig `high`; `/think off` deaktiviert das Denken und
    `/think xhigh|max` verwendet die nativen Aufwandsstufen des Modells. OpenClaw lässt
    vom Aufrufer ausgewählte Sampling-Parameter weg.

    Claude Sonnet 5 verwendet immer adaptives Denken und standardmäßig den Aufwand `high`.
    `/think off` und `/think minimal` werden auf `low` abgebildet, da die Mantle-
    Route das Denken nicht deaktivieren kann. OpenClaw lässt außerdem benutzerdefinierte Temperaturwerte bei
    Sonnet-5-Anfragen weg.

    Claude Mythos 5 ist nur eingeschränkt zugänglich. Es weist ein Kontextfenster von 1.000.000 Token
    und ein Ausgabelimit von 128.000 Token aus, verwendet immer adaptives Denken, bildet
    `/think off` und `/think minimal` auf `low` ab und lässt vom Aufrufer ausgewählte
    Sampling-Parameter weg.

    Claude Mythos Preview fordert immer Reasoning an und verwendet standardmäßig den Aufwand `high`,
    wenn keine `/think`-Stufe festgelegt ist (`xhigh`/`max` werden nach unten auf
    `high` und `minimal` nach oben auf `low` abgebildet). Opus 4.7 streamt auf Mantle ohne
    vom Modell bereitgestelltes Reasoning, und OpenClaw lässt seinen Parameter `temperature`
    weg, da Opus 4.7 auf dieser Route keine Sampling-Überschreibungen akzeptiert; Mythos
    Preview akzeptiert eine `temperature`-Überschreibung wie gewohnt.

    Eine nicht leere, explizite `models.providers["amazon-bedrock-mantle"].models`-
    Liste ersetzt den vollständigen erkannten Katalog. Lassen Sie diese Liste weg, wenn Sie
    diese integrierten Claude-Zeilen verwenden möchten.

  </Accordion>

  <Accordion title="Beziehung zum Amazon-Bedrock-Provider">
    Bedrock Mantle ist ein vom standardmäßigen
    [Amazon Bedrock](/de/providers/bedrock)-Provider separater Provider. Mantle verwendet für seinen
    OSS-Katalog eine OpenAI-kompatible `/v1`-Schnittstelle, während der standardmäßige
    Bedrock-Provider die native Bedrock-Converse-API verwendet.

    Beide Provider nutzen dieselben `AWS_BEARER_TOKEN_BEDROCK`-Anmeldedaten, sofern
    vorhanden.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Amazon Bedrock" href="/de/providers/bedrock" icon="cloud">
    Nativer Bedrock-Provider für Anthropic Claude, Titan und andere Modelle.
  </Card>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="OAuth und Authentifizierung" href="/de/gateway/authentication" icon="key">
    Details zur Authentifizierung und Regeln für die Wiederverwendung von Anmeldedaten.
  </Card>
  <Card title="Fehlerbehebung" href="/de/help/troubleshooting" icon="wrench">
    Häufige Probleme und deren Behebung.
  </Card>
</CardGroup>
