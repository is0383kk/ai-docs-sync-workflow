---
read_when:
    - Sie möchten `code_execution` aktivieren oder konfigurieren
    - Sie möchten eine Remote-Analyse ohne lokalen Shell-Zugriff.
    - Sie möchten x_search oder web_search mit einer Remote-Python-Analyse kombinieren
summary: 'code_execution: Sandbox-geschützte Remote-Python-Analyse mit xAI ausführen'
title: Codeausführung
x-i18n:
    generated_at: "2026-07-26T18:07:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution` führt eine isolierte Remote-Python-Analyse über die Responses API von xAI aus
(`https://api.x.ai/v1/responses`, derselbe Endpunkt, den `x_search` verwendet). Das Tool wird
vom gebündelten `xai`-Plugin unter dem `tools`-Vertrag registriert.

<Warning>
  `code_execution` wird auf den Servern von xAI ausgeführt. xAI berechnet $5 pro 1.000 Tool-Aufrufe
  zuzüglich der Ein- und Ausgabe-Token des Modells.
</Warning>

| Eigenschaft        | Wert                                                                              |
| ------------------ | --------------------------------------------------------------------------------- |
| Tool-Name          | `code_execution`                                                                  |
| Provider-Plugin    | `xai` (gebündelt, `enabledByDefault: true`)                                         |
| Authentifizierung  | xAI-Authentifizierungsprofil, `XAI_API_KEY` oder `plugins.entries.xai.config.webSearch.apiKey` |
| Standardmodell     | `grok-4.3`                                                                        |
| Standard-Timeout   | 30 Sekunden                                                                        |
| Standardwert für `maxTurns` | nicht festgelegt (xAI wendet ein eigenes internes Limit an)                      |

Verwenden Sie das Tool für Berechnungen, Tabellierungen, schnelle Statistiken und diagrammartige
Analysen, einschließlich Daten, die von `x_search` oder `web_search` zurückgegeben werden. Es hat keinen
Zugriff auf lokale Dateien, Ihre Shell, Ihr Repository oder gekoppelte Geräte und speichert
zwischen Aufrufen keinen Zustand. Behandeln Sie daher jeden Aufruf als temporäre Analyse und nicht
als Notebook-Sitzung. Führen Sie für aktuelle X-Daten zuerst [`x_search`](/de/tools/web#x_search)
aus und leiten Sie das Ergebnis weiter.

Verwenden Sie für die lokale Ausführung stattdessen [`exec`](/de/tools/exec).

## Einrichtung

<Steps>
  <Step title="xAI-Anmeldedaten bereitstellen">
    OAuth erfordert ein berechtigtes SuperGrok- oder X-Premium-Abonnement
    (Verifizierung per Gerätecode, sodass es auf Remote-Hosts ohne
    localhost-Callback funktioniert):

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Bei einer Neuinstallation ist dieselbe Auswahl beim Onboarding verfügbar:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    Alternativ mit einem API-Schlüssel:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    Oder über die Konfiguration:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    Jede dieser drei Optionen stellt außerdem die Funktionalität für `x_search` und Grok `web_search` bereit.

  </Step>

  <Step title="code_execution aktivieren und abstimmen">
    Wenn `enabled` nicht angegeben ist, wird `code_execution` nur bereitgestellt, wenn der Provider
    des aktiven Modells `xai` ist und die xAI-Anmeldedaten aufgelöst werden können. Legen Sie bei einem aktiven Modell
    mit einem bekannten Nicht-xAI-Provider
    `plugins.entries.xai.config.codeExecution.enabled` auf `true` fest, um die
    providerübergreifende Verwendung zu aktivieren. Wenn der Provider des aktiven Modells fehlt oder nicht aufgelöst werden kann,
    bleibt das Tool ausgeblendet. Legen Sie `enabled` auf `false` fest, um es für jeden
    Provider zu deaktivieren. xAI-Anmeldedaten sind immer erforderlich.

    Verwenden Sie denselben Block, um das Modell, das Rundenlimit oder den Timeout zu überschreiben:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // für einen bekannten Nicht-xAI-Modell-Provider erforderlich
                model: "grok-4.3", // überschreibt das standardmäßige xAI-Modell für die Codeausführung
                maxTurns: 2,            // optionales Limit für interne Tool-Runden
                timeoutSeconds: 30,     // Anfrage-Timeout (Standard: 30)
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Gateway neu starten">
    ```bash
    openclaw gateway restart
    ```

    `code_execution` erscheint in der Tool-Liste des Agenten, sobald sich das xAI-Plugin
    erneut registriert und die obigen Prüfungen für Provider, Aktivierung und Authentifizierung erfolgreich sind.

  </Step>
</Steps>

## Verwendung

Formulieren Sie die beabsichtigte Analyse ausdrücklich. Das Tool akzeptiert einen einzelnen Parameter `task`.
Senden Sie daher die vollständige Anfrage und alle eingebetteten Daten in einem Prompt:

```text
Verwende code_execution, um den gleitenden 7-Tage-Durchschnitt für diese Zahlen zu berechnen: ...
```

```text
Verwende x_search, um Beiträge zu finden, in denen OpenClaw diese Woche erwähnt wird, und anschließend code_execution, um sie nach Tagen zu zählen.
```

```text
Verwende web_search, um die neuesten KI-Benchmarkwerte zusammenzutragen, und anschließend code_execution, um die prozentualen Änderungen zu vergleichen.
```

## Fehler

Ohne Authentifizierung gibt das Tool einen strukturierten JSON-Fehler zurück (und löst keine
Ausnahme aus), sodass der Agent den Fehler selbst korrigieren kann:

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution benötigt xAI-Anmeldedaten. Führen Sie `openclaw onboard --auth-choice xai-oauth` aus, um sich bei Grok anzumelden, führen Sie `openclaw onboard --auth-choice xai-api-key` aus, legen Sie `XAI_API_KEY` in der Gateway-Umgebung fest oder konfigurieren Sie `plugins.entries.xai.config.webSearch.apiKey`.",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Exec-Tool" href="/de/tools/exec" icon="terminal">
    Lokale Shell-Ausführung auf Ihrem Computer oder einer gekoppelten Node.
  </Card>
  <Card title="Exec-Genehmigungen" href="/de/tools/exec-approvals" icon="shield">
    Richtlinie zum Zulassen oder Ablehnen der Shell-Ausführung.
  </Card>
  <Card title="Web-Tools" href="/de/tools/web" icon="globe">
    `web_search`, `x_search` und `web_fetch`.
  </Card>
  <Card title="xAI-Provider" href="/de/providers/xai" icon="microchip">
    Grok-Modelle, Web-/X-Suche und Konfiguration der Codeausführung.
  </Card>
</CardGroup>
