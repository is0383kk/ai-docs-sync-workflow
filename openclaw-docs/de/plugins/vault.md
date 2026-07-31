---
read_when:
    - Sie möchten, dass OpenClaw API-Schlüssel aus HashiCorp Vault liest
    - Sie richten SecretRefs auf einem lokalen Computer oder Server ein
    - Sie müssen Vault-gestützte Zugangsdaten für den Modell-Provider konfigurieren
summary: Verwenden Sie das mitgelieferte Vault-Plugin, um SecretRefs aus HashiCorp Vault aufzulösen
title: Vault-SecretRefs
x-i18n:
    generated_at: "2026-07-26T18:04:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# Vault-SecretRefs

Das mitgelieferte Vault-Plugin ermöglicht OpenClaw, `exec`-SecretRefs beim Start des Gateways und bei Neuladevorgängen aus
HashiCorp Vault aufzulösen. OpenClaw speichert Vault-
Referenzen in der Konfiguration, hält aufgelöste Werte im In-Memory-Secrets-Snapshot
und schreibt die aufgelösten API-Schlüssel nicht zurück in `openclaw.json`.

Verwenden Sie dies, wenn Sie Vault bereits betreiben oder die Schlüssel der Modell-Provider außerhalb
der OpenClaw-Konfigurationsdateien speichern möchten. Informationen zum SecretRef-Laufzeitmodell finden Sie unter
[Secrets-Verwaltung](/de/gateway/secrets).

## Vorbereitungen

Sie benötigen:

- OpenClaw mit verfügbarem mitgeliefertem `vault`-Plugin
- einen erreichbaren Vault-Server
- eine Vault-Authentifizierung, die ein Client-Token mit Lesezugriff auf die Secret-
  Pfade erzeugen kann, die OpenClaw auflösen soll
- die Umgebung, die das Gateway startet, muss `VAULT_ADDR` und entweder
  `VAULT_TOKEN`, `OPENCLAW_VAULT_AUTH_METHOD=token_file` mit `VAULT_TOKEN_FILE`
  oder eine konfigurierte JWT-/Kubernetes-Anmeldung enthalten

Der Resolver kommuniziert von Node aus über HTTP mit Vault. Das Gateway benötigt die
Vault-CLI nicht, um SecretRefs aufzulösen.

Aktivieren Sie das mitgelieferte Plugin, bevor Sie die `openclaw vault`-Befehle ausführen:

```bash
openclaw plugins enable vault
```

## Einen Provider-Schlüssel in Vault speichern

OpenClaw verwendet standardmäßig KV v2, das unter `secret` eingehängt ist, entsprechend den
Beispielen für den Vault-Entwicklungsserver. Legen Sie für eine produktive Vault-Instanz `OPENCLAW_VAULT_KV_MOUNT` auf Ihren tatsächlichen KV-
Einhängepfad fest, bevor Sie SecretRef-IDs erstellen. Mit den OpenClaw-Standardwerten liest diese
SecretRef-ID:

```text
providers/openrouter/apiKey
```

dieses Vault-Feld:

```text
secret/data/providers/openrouter -> apiKey
```

Eine Möglichkeit, es mit der Vault-CLI zu erstellen:

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

Verwenden Sie für OpenClaw ein eingeschränktes Client-Token und kein Root-Token. Für das standardmäßige KV-v2-
Layout sieht eine minimale Richtlinie für Schlüssel von Modell-Providern wie folgt aus:

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## Vault für das Gateway sichtbar machen

Exportieren Sie bei einem lokalen Gateway ohne Container die Vault-Einstellungen in derselben Shell,
die OpenClaw startet. Die Standardauthentifizierungsmethode liest ein Vault-Client-Token aus
`VAULT_TOKEN`:

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

Wenn Vault Agent ein Token in eine Token-Sink-Datei schreibt, verwenden Sie die Token-Datei-Authentifizierung:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

Bei einem Vault-Server, der von einer privaten CA signiert wurde, installieren Sie diese CA entweder im
Vertrauensspeicher des Hosts und aktivieren Sie den Systemvertrauensspeicher von Node:

```bash
export NODE_USE_SYSTEM_CA=1
```

Oder stellen Sie direkt ein PEM-Bundle bereit:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

Diese Variablen müssen beim Start von OpenClaw vorhanden sein. Das Vault-Plugin leitet
sie an seinen Resolver-Prozess weiter.

Verwenden Sie für eine nicht interaktive JWT-Authentifizierung eine Workload-JWT-Datei und eine Vault-Rolle vom Typ
`jwt`:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

Die JWT-Datei sollte ein projiziertes Workload-Token sein, beispielsweise ein Kubernetes-Dienstkonto-
Token mit einer von der Vault-Rolle akzeptierten Zielgruppe.
Die interaktive OIDC-Browseranmeldung ist für Menschen nützlich, die Gateway-Laufzeit benötigt jedoch
eine nicht interaktive JWT-Anmeldung oder eine Token-Datei.

Verwenden Sie für die Kubernetes-Authentifizierungsmethode von Vault `kubernetes`. Sie ist für
Gateways vorgesehen, die als Pods ausgeführt werden; der Standardeinhängepunkt ist `kubernetes`, und die standardmäßige JWT-
Datei ist der übliche Pfad des Dienstkonto-Tokens:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

Legen Sie `OPENCLAW_VAULT_AUTH_MOUNT` nur fest, wenn die Kubernetes-Authentifizierung in Vault an einer anderen Stelle als
`auth/kubernetes` eingehängt wurde. Legen Sie `OPENCLAW_VAULT_JWT_FILE` nur fest, wenn das Dienstkonto-
Token unter einem benutzerdefinierten Pfad projiziert wird.

Optionale Einstellungen:

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

Prüfen Sie, was die aktuelle Shell erkennen kann:

```bash
openclaw vault status
```

Wenn mehr als ein Vault-gestützter Secret-Provider konfiguriert ist, wählen Sie einen anhand seines
Alias aus:

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status` gibt niemals `VAULT_TOKEN` aus; es meldet lediglich, ob das
Token, die Token-Datei und die JWT-Datei festgelegt sind.

<Warning>
Wenn das Gateway als Dienst, LaunchAgent, systemd-Unit, geplante Aufgabe oder
Container ausgeführt wird, muss diese Laufzeitumgebung dieselben Vault-Variablen erhalten.
Das Festlegen von Variablen in einer interaktiven Shell bestätigt nur diese Shell, nicht das
bereits ausgeführte Gateway.
</Warning>

## Einen SecretRef-Plan erstellen und anwenden

Erstellen Sie einen Plan, der den API-Schlüssel des OpenRouter-Modell-Providers Vault zuordnet:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

Wenden Sie den Plan an und überprüfen Sie ihn:

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

Verwenden Sie `--allow-exec`, da das Vault-Plugin die Auflösung über einen von OpenClaw verwalteten
Exec-SecretRef-Provider durchführt.

Wenn das Gateway noch nicht ausgeführt wird, starten Sie es nach dem Anwenden des Plans
wie gewohnt, anstatt `openclaw secrets reload` auszuführen.

## Weitere Provider-Schlüssel konfigurieren

Integrierte Kurzformen:

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

Mehrere Provider-Schlüssel in einem Plan:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

Für mitgelieferte Provider ohne Kurzformen sowie bereits konfigurierte OpenAI-kompatible und
benutzerdefinierte Modell-Provider verwenden Sie `--provider-key`:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

Jedes `--provider-key <provider=id>` schreibt eine SecretRef nach
`models.providers.<provider>.apiKey`. Bei benutzerdefinierten Providern werden die Einstellungen
`baseUrl`, `api` oder `models` des Providers nicht erstellt; konfigurieren Sie diese zuerst.

Verwenden Sie `--target <path=id>` für einen beliebigen bekannten SecretRef-Zielpfad:

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

Zielpfade ohne Präfix gelten für `openclaw.json`. Verwenden Sie
`auth-profiles:<agentId>:<path>` für vorhandene `auth-profiles.json`-Ziele.
Der Zielpfad muss ein registriertes OpenClaw-SecretRef-Ziel sein. Der Setup-
Befehl erstellt keine beliebigen benannten Secrets in OpenClaw; Vault bleibt der
Secret-Speicher, und OpenClaw speichert SecretRefs nur in unterstützten Konfigurationsfeldern.

## Format der SecretRef-ID

Vault-SecretRef-IDs verwenden diese Konvention:

```text
<vault-secret-path>/<field>
```

Beispiele:

| SecretRef-ID                  | Standardmäßiger KV-v2-Lesezugriff in Vault | Zurückgegebenes Feld |
| ----------------------------- | ---------------------------------- | -------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

Das zurückgegebene Vault-Feld muss eine Zeichenfolge sein.

Legen Sie für KV v1 Folgendes fest:

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

Dann liest `providers/openrouter/apiKey`:

```text
secret/providers/openrouter -> apiKey
```

## Was OpenClaw speichert

Durch das Anwenden eines Vault-Setup-Plans wird ein vom Plugin verwalteter Provider gespeichert:

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

Anmeldedatenfelder verweisen auf diesen Provider:

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

Der aufgelöste Wert befindet sich ausschließlich im aktiven Laufzeit-Secrets-Snapshot.

## Container und verwaltete Bereitstellungen

Containerisierte Gateways verwenden weiterhin dasselbe Plugin und dieselbe SecretRef-Konfiguration. Der
Container muss Folgendes erhalten:

- `VAULT_ADDR`
- eine Authentifizierungsquelle:
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` plus `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` plus `OPENCLAW_VAULT_AUTH_MOUNT`,
    `OPENCLAW_VAULT_AUTH_ROLE` und `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` plus `OPENCLAW_VAULT_AUTH_ROLE`; optional können
    `OPENCLAW_VAULT_AUTH_MOUNT` oder `OPENCLAW_VAULT_JWT_FILE` überschrieben werden
- optional `VAULT_NAMESPACE`, `OPENCLAW_VAULT_KV_MOUNT` und
  `OPENCLAW_VAULT_KV_VERSION`

Bevorzugen Sie bei der Verwendung von Kubernetes `OPENCLAW_VAULT_AUTH_METHOD=kubernetes`,
wenn in Vault die Kubernetes-Authentifizierung für den Cluster konfiguriert ist. Verwenden Sie
`OPENCLAW_VAULT_AUTH_METHOD=jwt` nur, wenn Vault so konfiguriert ist, dass der Cluster
als generischer JWT-/OIDC-Aussteller behandelt wird. Beide Optionen sind besser als ein langlebiges Vault-
Token in einem Kubernetes-Secret. Bereitstellungen mit Vault-Agent-Sidecar oder -Injector können
stattdessen `token_file` verwenden.

Belassen Sie bei mandantenfähigen Vault-Konfigurationen das Mandanten-Routing in der Vault-Richtlinie und
Bereitstellungskonfiguration. OpenClaw erfordert keinen festen Einhängepunkt, keine feste Rolle und keinen festen Pfad: Jede
Gateway-Umgebung kann eigene Werte für `OPENCLAW_VAULT_KV_MOUNT`,
`OPENCLAW_VAULT_AUTH_ROLE` und SecretRef-IDs festlegen. Wenn ein gemeinsames Gateway gleichzeitig Secrets
für verschiedene Vault-Benutzer auflösen muss, verwenden Sie manuell konfigurierte Exec-Provider,
die unterschiedliche Authentifizierungsumgebungen kapseln, oder verteilen Sie Mandanten auf Gateway-
Umgebungen mit separaten Vault-Umgebungsvariablen.

## Verwandte Themen

- [Secrets-Verwaltung](/de/gateway/secrets)
- [`openclaw secrets`](/de/cli/secrets)
- [Plugin-Inventar](/de/plugins/plugin-inventory)
