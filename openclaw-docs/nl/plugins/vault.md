---
read_when:
    - Je wilt dat OpenClaw API-sleutels uit HashiCorp Vault leest
    - Je stelt SecretRefs in op een lokale machine of server
    - Je moet modelproviderreferenties configureren die door Vault worden beheerd
summary: Gebruik de meegeleverde Vault-plugin om SecretRefs uit HashiCorp Vault op te lossen
title: Vault SecretRefs
x-i18n:
    generated_at: "2026-07-27T05:28:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# Vault SecretRefs

Met de gebundelde Vault-Plugin kan OpenClaw tijdens het opstarten en opnieuw laden van de Gateway `exec`-SecretRefs uit
HashiCorp Vault omzetten. OpenClaw slaat Vault-
verwijzingen op in de configuratie, bewaart omgezette waarden in de secrets-snapshot in het geheugen
en schrijft de omgezette API-sleutels niet terug naar `openclaw.json`.

Gebruik dit als je Vault al gebruikt of als je sleutels van modelproviders buiten
de OpenClaw-configuratiebestanden wilt bewaren. Zie
[Secrets beheren](/nl/gateway/secrets) voor het runtime-model van SecretRef.

## Voordat je begint

Je hebt het volgende nodig:

- OpenClaw met de gebundelde `vault`-Plugin beschikbaar
- een bereikbare Vault-server
- Vault-authenticatie die een clienttoken kan leveren met leestoegang tot de geheime
  paden die OpenClaw moet omzetten
- de omgeving die de Gateway start, moet `VAULT_ADDR` bevatten en daarnaast
  `VAULT_TOKEN`, of `OPENCLAW_VAULT_AUTH_METHOD=token_file` met `VAULT_TOKEN_FILE`,
  of een geconfigureerde JWT-/Kubernetes-aanmelding

De resolver communiceert vanuit Node via HTTP met Vault. De Gateway heeft de
Vault-CLI niet nodig om SecretRefs om te zetten.

Schakel de gebundelde Plugin in voordat je de `openclaw vault`-opdrachten uitvoert:

```bash
openclaw plugins enable vault
```

## Een providersleutel opslaan in Vault

OpenClaw gebruikt standaard KV v2, gekoppeld aan `secret`, overeenkomstig de
voorbeelden voor de Vault-ontwikkelserver. Stel voor Vault in productie `OPENCLAW_VAULT_KV_MOUNT` in op het daadwerkelijke
KV-koppelingspad voordat je SecretRef-id's maakt. Met de standaardinstellingen van OpenClaw leest deze
SecretRef-id:

```text
providers/openrouter/apiKey
```

dit Vault-veld:

```text
secret/data/providers/openrouter -> apiKey
```

Je kunt dit onder andere met de Vault-CLI maken:

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

Gebruik voor OpenClaw een clienttoken met een beperkt bereik, geen roottoken. Voor de standaardindeling van KV v2
ziet een minimaal beleid voor sleutels van modelproviders er als volgt uit:

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## Vault zichtbaar maken voor de Gateway

Exporteer voor een lokale Gateway zonder container de Vault-instellingen in dezelfde shell
waarin OpenClaw wordt gestart. De standaardauthenticatiemethode leest een Vault-clienttoken uit
`VAULT_TOKEN`:

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

Gebruik authenticatie via een tokenbestand als Vault Agent een token-sinkbestand schrijft:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

Voor een Vault-server die door een privé-CA is ondertekend, kun je die CA in het
vertrouwensarchief van de host installeren en het systeemvertrouwen van Node inschakelen:

```bash
export NODE_USE_SYSTEM_CA=1
```

Of geef rechtstreeks een PEM-bundel op:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

Deze variabelen moeten aanwezig zijn wanneer OpenClaw wordt gestart. De Vault-Plugin geeft
ze door aan het resolverproces.

Gebruik voor niet-interactieve JWT-authenticatie een JWT-bestand voor de workload en een Vault-rol van het type
`jwt`:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

Het JWT-bestand moet een geprojecteerd workloadtoken zijn, zoals een token voor een Kubernetes-serviceaccount
met een doelgroep die door de Vault-rol wordt geaccepteerd.
Interactief aanmelden via een OIDC-browser is nuttig voor mensen, maar de Gateway-runtime vereist
niet-interactief aanmelden via JWT of een tokenbestand.

Gebruik `kubernetes` voor de Kubernetes-authenticatiemethode van Vault. Dit is bedoeld voor
Gateways die als pods worden uitgevoerd; de standaardkoppeling is `kubernetes` en het standaard-JWT-
bestand is het standaardpad voor het serviceaccounttoken:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

Stel `OPENCLAW_VAULT_AUTH_MOUNT` alleen in wanneer Kubernetes-authenticatie in Vault ergens anders is gekoppeld
dan aan `auth/kubernetes`. Stel `OPENCLAW_VAULT_JWT_FILE` alleen in wanneer het
serviceaccounttoken naar een aangepast pad wordt geprojecteerd.

Optionele instellingen:

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

Controleer wat voor de huidige shell zichtbaar is:

```bash
openclaw vault status
```

Wanneer meer dan één door Vault ondersteunde secretprovider is geconfigureerd, selecteer je er een via
de alias:

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status` geeft `VAULT_TOKEN` nooit weer; er wordt alleen gemeld of het
token, het tokenbestand en het JWT-bestand zijn ingesteld.

<Warning>
Als de Gateway wordt uitgevoerd als service, LaunchAgent, systemd-eenheid, geplande taak of
container, moet die runtime-omgeving dezelfde Vault-variabelen ontvangen.
Het instellen van variabelen in een interactieve shell bewijst alleen dat ze in die shell werken, niet in de
al actieve Gateway.
</Warning>

## Een SecretRef-plan genereren en toepassen

Maak een plan dat de API-sleutel van de OpenRouter-modelprovider aan Vault koppelt:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

Pas het plan toe en verifieer het:

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

Gebruik `--allow-exec`, omdat de Vault-Plugin omzetten uitvoert via een door OpenClaw beheerde
exec-SecretRef-provider.

Als de Gateway nog niet actief is, start je deze na het toepassen van het plan op de gebruikelijke manier
in plaats van `openclaw secrets reload` uit te voeren.

## Meer providersleutels configureren

Ingebouwde snelkoppelingen:

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

Meerdere providersleutels in één plan:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

Gebruik `--provider-key` voor gebundelde providers zonder snelkoppeling, of voor reeds geconfigureerde OpenAI-compatibele en
aangepaste modelproviders:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

Elke `--provider-key <provider=id>` schrijft een SecretRef naar
`models.providers.<provider>.apiKey`. Voor aangepaste providers worden hiermee niet
de instellingen `baseUrl`, `api` of `models` van de provider gemaakt; configureer deze eerst.

Gebruik `--target <path=id>` voor elk bekend doelpad van SecretRef:

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

Doelpaden zonder voorvoegsel zijn van toepassing op `openclaw.json`. Gebruik
`auth-profiles:<agentId>:<path>` voor bestaande `auth-profiles.json`-doelen.
Het doelpad moet een geregistreerd OpenClaw SecretRef-doel zijn. De setup-
opdracht maakt geen willekeurige benoemde secrets in OpenClaw; Vault blijft de
secretopslag en OpenClaw slaat SecretRefs alleen op in ondersteunde configuratievelden.

## Indeling van SecretRef-id's

Vault SecretRef-id's gebruiken deze conventie:

```text
<vault-secret-path>/<field>
```

Voorbeelden:

| SecretRef-id                  | Standaard KV v2-leesbewerking in Vault | Geretourneerd veld |
| ----------------------------- | ---------------------------------- | -------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

Het geretourneerde Vault-veld moet een tekenreeks zijn.

Stel voor KV v1 het volgende in:

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

Vervolgens leest `providers/openrouter/apiKey`:

```text
secret/providers/openrouter -> apiKey
```

## Wat OpenClaw opslaat

Bij het toepassen van een Vault-setupplan wordt een door een Plugin beheerde provider opgeslagen:

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

Referentievelden verwijzen naar die provider:

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

De omgezette waarde bevindt zich alleen in de secrets-snapshot van de actieve runtime.

## Containers en beheerde implementaties

Gateways in containers gebruiken nog steeds dezelfde Plugin- en SecretRef-configuratie. De
container moet het volgende ontvangen:

- `VAULT_ADDR`
- één authenticatiebron:
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` plus `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` plus `OPENCLAW_VAULT_AUTH_MOUNT`,
    `OPENCLAW_VAULT_AUTH_ROLE` en `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` plus `OPENCLAW_VAULT_AUTH_ROLE`; overschrijf eventueel
    `OPENCLAW_VAULT_AUTH_MOUNT` of `OPENCLAW_VAULT_JWT_FILE`
- optioneel `VAULT_NAMESPACE`, `OPENCLAW_VAULT_KV_MOUNT` en
  `OPENCLAW_VAULT_KV_VERSION`

Geef bij gebruik van Kubernetes de voorkeur aan `OPENCLAW_VAULT_AUTH_METHOD=kubernetes`
wanneer Kubernetes-authenticatie in Vault voor het cluster is geconfigureerd. Gebruik
`OPENCLAW_VAULT_AUTH_METHOD=jwt` alleen wanneer Vault is geconfigureerd om het cluster
als een algemene JWT-/OIDC-uitgever te behandelen. Beide opties zijn beter dan een langlevend Vault-
token in een Kubernetes-secret. Implementaties met een Vault Agent-sidecar of -injector kunnen in plaats daarvan
`token_file` gebruiken.

Houd bij Vault-configuraties voor meerdere tenants de tenantroutering in het Vault-beleid en de
implementatieconfiguratie. OpenClaw vereist geen vaste koppeling, rol of pad: elke
Gateway-omgeving kan eigen waarden instellen voor `OPENCLAW_VAULT_KV_MOUNT`,
`OPENCLAW_VAULT_AUTH_ROLE` en SecretRef-id's. Als één gedeelde Gateway tegelijkertijd secrets van
verschillende Vault-gebruikers moet omzetten, gebruik dan handmatig geconfigureerde exec-providers
die afzonderlijke authenticatieomgevingen omwikkelen, of verdeel tenants over Gateway-
omgevingen met afzonderlijke Vault-omgevingsvariabelen.

## Gerelateerd

- [Secrets beheren](/nl/gateway/secrets)
- [`openclaw secrets`](/nl/cli/secrets)
- [Pluginoverzicht](/nl/plugins/plugin-inventory)
