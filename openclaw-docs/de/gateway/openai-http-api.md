---
read_when:
    - Integration von Tools, die OpenAI Chat Completions erwarten
summary: Einen OpenAI-kompatiblen HTTP-Endpunkt `/v1/chat/completions` über das Gateway bereitstellen
title: OpenAI-Chat-Vervollständigungen
x-i18n:
    generated_at: "2026-07-26T18:27:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

Der Gateway kann eine kleine, mit OpenAI kompatible Chat-Completions-Oberfläche bereitstellen. Sie ist **standardmäßig deaktiviert**.

Nach der Aktivierung stellt sie all diese Endpunkte auf demselben Port wie der Gateway bereit (WS- und HTTP-Multiplexing):

| Methode | Pfad                   |
| ------ | ---------------------- |
| POST   | `/v1/chat/completions` |
| GET    | `/v1/models`           |
| GET    | `/v1/models/{id}`      |
| POST   | `/v1/embeddings`       |
| POST   | `/v1/responses`        |

Anfragen werden als normale Gateway-Agentenausführung verarbeitet (derselbe Codepfad wie `openclaw agent`), sodass Routing, Berechtigungen und Konfiguration Ihrem Gateway entsprechen.

## Endpunkt aktivieren

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

Legen Sie `enabled: false` fest (oder lassen Sie es weg), um ihn zu deaktivieren.

## Sicherheitsgrenze (wichtig)

Behandeln Sie diesen Endpunkt als **vollständigen Operatorzugriff** auf die Gateway-Instanz:

- Ein gültiges Gateway-Token/-Passwort für diesen Endpunkt entspricht einer Zugangsinformation für Eigentümer/Operatoren und nicht einem eingeschränkten benutzerspezifischen Geltungsbereich.
- Anfragen durchlaufen denselben Agentenpfad der Steuerungsebene wie vertrauenswürdige Operatoraktionen. Wenn die Richtlinie des Zielagenten sensible Werkzeuge zulässt, kann dieser Endpunkt sie daher verwenden.
- Beschränken Sie ihn auf Loopback, Tailnet oder privaten Eingang. Stellen Sie ihn nicht im öffentlichen Internet bereit.

Authentifizierungsmatrix:

| Authentifizierungspfad                                                                                            | Verhalten                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` oder `"password"` + `Authorization: Bearer ...`                            | Belegt den Besitz des gemeinsamen Gateway-Geheimnisses. Ignoriert jeden `x-openclaw-scopes`-Header und stellt den vollständigen Standardsatz der Operator-Geltungsbereiche wieder her: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Behandelt Chat-Beiträge als Beiträge des Eigentümer-Absenders. |
| Vertrauenswürdiges identitätstragendes HTTP (Trusted-Proxy-Authentifizierung oder `gateway.auth.mode="none"` bei privatem Eingang) | Berücksichtigt `x-openclaw-scopes`, wenn vorhanden; greift andernfalls auf den Standardsatz der Operator-Geltungsbereiche zurück. Verliert die Eigentümersemantik nur, wenn der Aufrufer die Geltungsbereiche ausdrücklich einschränkt und `operator.admin` auslässt. Erfordert `operator.admin` für Steuerungsmöglichkeiten auf Eigentümerebene wie `x-openclaw-model`.                        |

Siehe [Operator-Geltungsbereiche](/de/gateway/operator-scopes), [Sicherheit](/de/gateway/security) und [Fernzugriff](/de/gateway/remote).

## Authentifizierung

Verwendet die Gateway-Authentifizierungskonfiguration (Einzelheiten zu diesem Modus finden Sie unter [Trusted-Proxy-Authentifizierung](/de/gateway/trusted-proxy-auth)):

| Modus                                | Authentifizierung                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`. Festgelegt über `gateway.auth.token` oder `OPENCLAW_GATEWAY_TOKEN`.                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`. Festgelegt über `gateway.auth.password` oder `OPENCLAW_GATEWAY_PASSWORD`.                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | Leiten Sie die Anfrage über den konfigurierten identitätsbewussten Proxy weiter; dieser fügt die erforderlichen Identitäts-Header ein. Loopback-Proxys auf demselben Host benötigen ausdrücklich `gateway.auth.trustedProxy.allowLoopback = true`. |
| `gateway.auth.mode="none"`          | Kein Authentifizierungs-Header erforderlich (nur privater Eingang).                                                                                                                                         |

Hinweise:

- Aufrufer auf demselben Host, die den Proxy eines `trusted-proxy`-Gateways umgehen, können direkt auf `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` zurückgreifen. Nachweise durch einen `Forwarded`-, `X-Forwarded-*`- oder `X-Real-IP`-Header belassen die Anfrage stattdessen auf dem Trusted-Proxy-Pfad.
- Wenn `gateway.auth.rateLimit` konfiguriert ist und zu viele Authentifizierungsversuche fehlschlagen, gibt der Endpunkt `429` mit einem `Retry-After`-Header zurück.

## Wann dieser Endpunkt verwendet werden sollte

- Bevorzugen Sie diesen Endpunkt gegenüber dem Hinzufügen eines neuen integrierten Kanals, wenn Ihre Integration lediglich eine weitere Operator-/Client-Oberfläche für denselben Gateway ist.
- Für native mobile Clients, die sich direkt mit einem entfernten Gateway verbinden, sollten Sie [WebChat](/de/web/webchat) oder das [Gateway-Protokoll](/de/gateway/protocol) mit dem Bootstrap-/Geräte-Token-Ablauf für gekoppelte Geräte bevorzugen, damit das Gerät kein gemeinsames HTTP-Token/-Passwort benötigt.
- Erstellen Sie stattdessen ein Kanal-Plugin, wenn Sie ein externes Messaging-Netzwerk mit eigenen Benutzern, Räumen, Webhook-Zustellung oder ausgehendem Transport integrieren. Siehe [Plugins erstellen](/de/plugins/building-plugins).

## Agentenorientierter Modellvertrag

OpenClaw behandelt das OpenAI-Feld `model` als **Agentenziel** und nicht als rohe Provider-Modell-ID.

| Wert von `model`                                | Weiterleitung an                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | Konfigurierter Standardagent                                                                                                 |
| `openclaw/default`                           | Konfigurierter Standardagent (stabiler Alias; kann sicher fest codiert werden, selbst wenn sich die tatsächliche ID des Standardagenten zwischen Umgebungen ändert) |
| `openclaw/<agentId>` oder `openclaw:<agentId>` | Bestimmter Agent                                                                                                           |
| `agent:<agentId>`                            | Bestimmter Agent (Kompatibilitätsalias)                                                                                     |

Optionale Anfrage-Header:

| Header                                          | Wirkung                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | Überschreibt das Backend-Modell für den ausgewählten Agenten. Bearer-Aufrufer mit gemeinsamem Geheimnis können dies direkt verwenden; identitätstragende Aufrufer (Trusted Proxy oder privater Eingang ohne Authentifizierung mit `x-openclaw-scopes`) benötigen `operator.admin`, andernfalls `403 missing scope: operator.admin`. |
| `x-openclaw-agent-id: <agentId>`                | Kompatibilitätsüberschreibung für die Agentenauswahl.                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | Explizites Sitzungsrouting. Wird mit `400 invalid_request_error` abgelehnt, wenn ein reservierter interner Namensraum verwendet wird (`subagent:`, `cron:`, `acp:`).                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | Legt den synthetischen Eingangskanalkontext für kanalbezogene Prompts/Richtlinien fest.                                                                                                                                                                                              |

`/v1/models` listet Agentenziele der obersten Ebene auf (`openclaw`, `openclaw/default`, `openclaw/<agentId>`), nicht Backend-Provider-Modelle oder Sub-Agenten; Sub-Agenten bleiben Teil der internen Ausführungstopologie. Wenn Sie `x-openclaw-model` auslassen, wird der ausgewählte Agent mit seinem regulär konfigurierten Modell ausgeführt.

`/v1/embeddings` verwendet dieselben Agentenziel-IDs aus `model`. Senden Sie `x-openclaw-model` (von einem Aufrufer mit gemeinsamem Geheimnis oder einem identitätstragenden Aufrufer mit `operator.admin`), um ein bestimmtes Einbettungsmodell auszuwählen; andernfalls verwendet die Anfrage die normale Einbettungskonfiguration des ausgewählten Agenten.

## Sitzungsverhalten

Standardmäßig ist der Endpunkt **pro Anfrage zustandslos** (bei jedem Aufruf wird ein neuer Sitzungsschlüssel erzeugt).

Wenn die Anfrage eine OpenAI-Zeichenfolge `user` enthält, leitet der Gateway daraus einen stabilen Sitzungsschlüssel ab, sodass wiederholte Aufrufe dieselbe Agentensitzung verwenden können. Verwenden Sie bei benutzerdefinierten Apps denselben Wert für `user` pro Konversationsthread erneut; vermeiden Sie Kennungen auf Kontoebene, sofern nicht mehrere Konversationen/Geräte dieselbe OpenClaw-Sitzung verwenden sollen. Verwenden Sie `x-openclaw-session-key` nur, wenn Sie eine explizite Routingsteuerung über mehrere Clients/Threads hinweg benötigen, und nutzen Sie dabei anwendungseigene Schlüssel, die die oben genannten reservierten Namensräume vermeiden.

## Anfragebeschränkungen

Der Endpunkt verwendet integrierte Grenzwerte von 20 MB pro Anfrageinhalt, 8 `image_url`-Teilen
aus der neuesten Benutzernachricht und 20 MB kumulativer decodierter
Bilddaten. Die Richtlinie für Bildquellen bleibt unter
`gateway.http.endpoints.chatCompletions.images` konfigurierbar:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

Standardeinstellungen für Bilder:

| Schlüssel                   | Standard                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false` (aus URLs stammende `image_url`-Teile werden abgelehnt, sofern sie nicht aktiviert sind) |
| `images.maxBytes`     | 10 MB pro Bild                                                      |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10 s                                                                 |

HEIC/HEIF-Quellen für `image_url` werden akzeptiert und vor der Übermittlung an den Provider durch den gemeinsamen OpenClaw-Bildprozessor (Rastermill) in JPEG normalisiert. Dieser greift bei Formaten, die externe Codec-Unterstützung benötigen, auf einen Systemkonverter zurück (`sips`, ImageMagick, GraphicsMagick oder ffmpeg).

Sicherheitshinweis: Das Zulassen eines Hostnamens setzt die Blockierung privater/interner IP-Adressen nicht außer Kraft. Wenden Sie bei Gateways, die im Internet erreichbar sind, zusätzlich zu Schutzmaßnahmen auf Anwendungsebene Kontrollen für ausgehenden Netzwerkverkehr an. Siehe [Sicherheit](/de/gateway/security).

## Vertrag für Chat-Werkzeuge

`/v1/chat/completions` unterstützt eine Teilmenge von Funktionswerkzeugen, die mit gängigen OpenAI-Chat-Clients kompatibel ist.

### Unterstützte Anfragefelder

| Feld                       | Hinweise                                                                                                                                      |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | Array von `{ "type": "function", "function": { ... } }`                                                                                     |
| `tool_choice`              | `"auto"`, `"none"`, `"required"` oder `{ "type": "function", "function": { "name": "..." } }`                                                |
| `messages[*].role: "tool"` | Nachfolgende Interaktionen                                                                                                                    |
| `messages[*].tool_call_id` | Ordnet ein Werkzeugergebnis einem vorherigen Werkzeugaufruf zu                                                                                |
| `max_completion_tokens`    | Zahl; Obergrenze pro Aufruf für die Gesamtzahl der Vervollständigungstoken (einschließlich Reasoning-Token). Aktueller Feldname; wird verwendet, wenn sowohl dieses Feld als auch `max_tokens` gesendet werden. |
| `max_tokens`               | Zahl; veralteter Alias, wird ignoriert, wenn auch `max_completion_tokens` vorhanden ist.                                                      |
| `temperature`              | Zahl 0–2; nach bestem Bemühen an den vorgelagerten Provider weitergeleitet. `400 invalid_request_error` bei Überschreitung des Wertebereichs.   |
| `top_p`                    | Zahl 0–1; nach bestem Bemühen. `400 invalid_request_error` bei Überschreitung des Wertebereichs.                                               |
| `frequency_penalty`        | Zahl -2.0 bis 2.0; nach bestem Bemühen. `400 invalid_request_error` bei Überschreitung des Wertebereichs.                                      |
| `presence_penalty`         | Zahl -2.0 bis 2.0; nach bestem Bemühen. `400 invalid_request_error` bei Überschreitung des Wertebereichs.                                      |
| `seed`                     | Ganzzahl; nach bestem Bemühen. `400 invalid_request_error` für Werte, die keine Ganzzahlen sind.                                               |
| `stop`                     | Zeichenfolge oder Array mit bis zu 4 Zeichenfolgen; nach bestem Bemühen. `400 invalid_request_error` bei mehr als 4 Sequenzen oder Einträgen, die keine Zeichenfolgen oder leer sind. |

Alle Sampling- und Tokenbegrenzungsfelder verwenden denselben Stream-Parameter-Kanal des Agenten und werden nach bestem Bemühen weitergeleitet:

- Tokenbegrenzung: Der Feldname im Übertragungsformat wird vom Provider-Transport gewählt: `max_completion_tokens` für Endpunkte der OpenAI-Familie, `max_tokens` für Provider, die nur den veralteten Namen akzeptieren (Mistral, Chutes).
- `stop` wird dem Stop-Feld des Transports zugeordnet: `stop` für Chat-Completions-Backends, `stop_sequences` für Anthropic. Die OpenAI Responses API besitzt keinen Stop-Parameter, daher wird `stop` bei Responses-basierten Modellen nicht angewendet.
- Das ChatGPT-basierte Codex-Responses-Backend verwendet serverseitig festgelegtes Sampling und entfernt `temperature`/`top_p` (zusammen mit `max_output_tokens`, `metadata`, `prompt_cache_retention`, `service_tier`), bevor die Anfrage dieses Backend erreicht.

### Nicht unterstützte Varianten

Gibt `400 invalid_request_error` zurück für:

- `tools`, die keine Arrays sind, Werkzeugeinträge, die keine Funktionen sind, oder fehlendes `tool.function.name`
- `tool_choice`-Varianten wie `allowed_tools` und `custom`
- `tool_choice.function.name`-Werte, die keinem bereitgestellten Werkzeug entsprechen

Für `tool_choice: "required"` und funktionsgebundenes `tool_choice` schränkt der Endpunkt die offengelegte Menge der Client-Funktionswerkzeuge ein, weist die Laufzeit an, vor der Antwort ein Client-Werkzeug aufzurufen, und gibt einen Fehler aus, wenn die Agentenantwort keinen passenden strukturierten Client-Werkzeugaufruf enthält. Dies gilt für die vom Aufrufer bereitgestellte HTTP-Liste `tools`, nicht für jedes interne OpenClaw-Agentenwerkzeug.

### Struktur der nicht gestreamten Werkzeugantwort

Wenn der Agent Werkzeuge aufruft, verwendet die Antwort:

- `choices[0].finish_reason = "tool_calls"`
- `choices[0].message.tool_calls[]`-Einträge mit `id`, `type: "function"`, `function.name`, `function.arguments` (JSON-Zeichenfolge)
- Assistentenkommentar vor dem Werkzeugaufruf in `choices[0].message.content` (möglicherweise leer)

### Struktur der gestreamten Werkzeugantwort

Bei `stream: true` treffen Werkzeugaufrufe als inkrementelle SSE-Chunks ein: zunächst ein Delta mit der Assistentenrolle, optionale Deltas mit Assistentenkommentaren, ein oder mehrere `delta.tool_calls`-Chunks mit Werkzeugidentität und Argumentfragmenten und anschließend ein abschließender Chunk mit `finish_reason: "tool_calls"` und `data: [DONE]`.

Bei `stream_options.include_usage=true` wird vor `[DONE]` ein abschließender Nutzungs-Chunk ausgegeben.

### Nachfolgeschleife für Werkzeuge

Führen Sie nach dem Empfang von `tool_calls` die angeforderte(n) Funktion(en) aus und senden Sie eine Folgeanfrage, die die vorherige Assistentennachricht mit dem Werkzeugaufruf sowie eine oder mehrere `role: "tool"`-Nachrichten mit übereinstimmendem `tool_call_id` enthält. Dadurch wird dieselbe Reasoning-Schleife des Agenten fortgesetzt, um die endgültige Antwort zu erzeugen.

## Streaming (SSE)

Legen Sie `stream: true` fest, um Server-Sent Events zu empfangen:

- `Content-Type: text/event-stream`
- Jede Ereigniszeile ist `data: <json>`
- Der Stream endet mit `data: [DONE]`

## Open WebUI-Schnelleinrichtung

- Basis-URL: `http://127.0.0.1:18789/v1`
- Basis-URL für Docker unter macOS: `http://host.docker.internal:18789/v1`
- API-Schlüssel: Ihr Gateway-Bearer-Token
- Modell: `openclaw/default`

Erwartetes Verhalten: `GET /v1/models` listet `openclaw/default` auf und Open WebUI verwendet es als Chatmodell-ID. Legen Sie für einen bestimmten Backend-Provider bzw. ein bestimmtes Backend-Modell das normale Standardmodell des Agenten fest oder senden Sie `x-openclaw-model` (Aufrufer mit gemeinsamem Geheimnis oder identitätstragender Aufrufer mit `operator.admin`).

Kurzer Funktionstest:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Wenn dies `openclaw/default` zurückgibt, können die meisten Open-WebUI-Einrichtungen mit derselben Basis-URL und demselben Token eine Verbindung herstellen.

## Beispiele

Stabile Sitzung für eine App-Unterhaltung:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"Fasse meine Aufgaben für heute zusammen"}]
  }'
```

Verwenden Sie bei späteren Aufrufen für diese Unterhaltung denselben `user`-Wert erneut, um dieselbe Agentensitzung fortzusetzen.

Nicht gestreamt:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"Hallo"}]
  }'
```

Gestreamt:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"Hallo"}]
  }'
```

Modelle auflisten:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Ein Modell abrufen:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Einbettungen erstellen:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings` unterstützt `input` als Zeichenfolge oder Array von Zeichenfolgen.

## Verwandte Themen

- [Konfigurationsreferenz](/de/gateway/configuration-reference)
- [Operator-Berechtigungsbereiche](/de/gateway/operator-scopes)
- [OpenAI](/de/providers/openai)
