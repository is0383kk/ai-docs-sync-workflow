---
read_when:
    - Quieres que OpenClaw lea las claves de API desde HashiCorp Vault
    - Está configurando SecretRefs en una máquina local o un servidor
    - Necesita configurar las credenciales del proveedor de modelos respaldadas por Vault
summary: Usa el plugin de Vault incluido para resolver SecretRefs desde HashiCorp Vault
title: SecretRefs del almacén de secretos
x-i18n:
    generated_at: "2026-07-26T04:53:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# SecretRefs de Vault

El plugin de Vault incluido permite que OpenClaw resuelva SecretRefs `exec` desde
HashiCorp Vault al iniciar el Gateway y durante las recargas. OpenClaw almacena las
referencias de Vault en la configuración, conserva los valores resueltos en la instantánea
de secretos en memoria y no vuelve a escribir las claves de API resueltas en `openclaw.json`.

Úselo si ya ejecuta Vault o desea que las claves de proveedores de modelos residan fuera de
los archivos de configuración de OpenClaw. Para conocer el modelo de ejecución de SecretRef, consulte
[Gestión de secretos](/es/gateway/secrets).

## Antes de comenzar

Se necesita:

- OpenClaw con el plugin `vault` incluido disponible
- un servidor Vault accesible
- autenticación de Vault que pueda generar un token de cliente con acceso de lectura a las rutas
  de secretos que OpenClaw debe resolver
- el entorno que inicia el Gateway debe incluir `VAULT_ADDR` y, además,
  `VAULT_TOKEN`, o `OPENCLAW_VAULT_AUTH_METHOD=token_file` con `VAULT_TOKEN_FILE`,
  o un inicio de sesión JWT/Kubernetes configurado

El resolutor se comunica con Vault mediante HTTP desde Node. El Gateway no necesita la
CLI de Vault para resolver SecretRefs.

Active el plugin incluido antes de ejecutar los comandos `openclaw vault`:

```bash
openclaw plugins enable vault
```

## Almacenar una clave de proveedor en Vault

De forma predeterminada, OpenClaw utiliza KV v2 montado en `secret`, lo que coincide con los
ejemplos del servidor de desarrollo de Vault. Para un Vault de producción, establezca `OPENCLAW_VAULT_KV_MOUNT` en la ruta
de montaje de KV real antes de crear identificadores de SecretRef. Con los valores predeterminados de OpenClaw, este
identificador de SecretRef:

```text
providers/openrouter/apiKey
```

lee este campo de Vault:

```text
secret/data/providers/openrouter -> apiKey
```

Una forma de crearlo con la CLI de Vault es:

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

Utilice un token de cliente con ámbito limitado para OpenClaw, no un token raíz. Para la disposición predeterminada
de KV v2, una política mínima para las claves de proveedores de modelos tiene este aspecto:

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## Hacer que Vault sea visible para el Gateway

Para un Gateway local sin contenedores, exporte la configuración de Vault en el mismo shell
que inicia OpenClaw. El método de autenticación predeterminado lee un token de cliente de Vault desde
`VAULT_TOKEN`:

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

Si Vault Agent escribe un archivo receptor de tokens, utilice la autenticación mediante archivo de token:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

Para un servidor Vault firmado por una CA privada, instale esa CA en el almacén
de confianza del host y active la confianza del sistema de Node:

```bash
export NODE_USE_SYSTEM_CA=1
```

O proporcione directamente un paquete PEM:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

Estas variables deben estar presentes cuando OpenClaw se inicia. El plugin de Vault las reenvía
a su proceso resolutor.

Para la autenticación JWT no interactiva, utilice un archivo JWT de carga de trabajo y un rol de Vault de tipo
`jwt`:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

El archivo JWT debe ser un token de carga de trabajo proyectado, como un token de cuenta de servicio de Kubernetes
con una audiencia aceptada por el rol de Vault.
El inicio de sesión interactivo mediante navegador OIDC es útil para las personas, pero la ejecución del Gateway necesita
un inicio de sesión JWT no interactivo o un archivo de token.

Para el método de autenticación de Kubernetes de Vault, utilice `kubernetes`. Está pensado para
Gateways que se ejecutan como Pods; el montaje predeterminado es `kubernetes` y el archivo JWT
predeterminado es la ruta estándar del token de la cuenta de servicio:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

Establezca `OPENCLAW_VAULT_AUTH_MOUNT` únicamente cuando la autenticación de Kubernetes de Vault esté montada en un lugar
distinto de `auth/kubernetes`. Establezca `OPENCLAW_VAULT_JWT_FILE` únicamente cuando el token de la cuenta
de servicio se proyecte en una ruta personalizada.

Configuración opcional:

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

Compruebe qué puede ver el shell actual:

```bash
openclaw vault status
```

Cuando haya configurado más de un proveedor de secretos respaldado por Vault, seleccione uno mediante
su alias:

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status` nunca muestra `VAULT_TOKEN`; solo indica si están configurados
el token, el archivo de token y el archivo JWT.

<Warning>
Si el Gateway se ejecuta como servicio, LaunchAgent, unidad systemd, tarea programada o
contenedor, ese entorno de ejecución debe recibir las mismas variables de Vault.
Configurar las variables en un shell interactivo solo valida ese shell, no el
Gateway que ya se está ejecutando.
</Warning>

## Generar y aplicar un plan de SecretRef

Cree un plan que asigne la clave de API del proveedor de modelos OpenRouter a Vault:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

Aplique y verifique el plan:

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

Utilice `--allow-exec` porque el plugin de Vault resuelve mediante un proveedor
de SecretRef exec gestionado por OpenClaw.

Si el Gateway aún no está en ejecución, inícielo con normalidad después de aplicar el plan
en lugar de ejecutar `openclaw secrets reload`.

## Configurar más claves de proveedores

Atajos integrados:

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

Varias claves de proveedores en un solo plan:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

Los proveedores incluidos sin atajos, o los proveedores de modelos personalizados y compatibles con OpenAI
que ya estén configurados, utilizan `--provider-key`:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

Cada `--provider-key <provider=id>` escribe una SecretRef en
`models.providers.<provider>.apiKey`. Para los proveedores personalizados, no crea
la configuración `baseUrl`, `api` ni `models` del proveedor; configúrela primero.

Utilice `--target <path=id>` para cualquier ruta de destino de SecretRef conocida:

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

Las rutas de destino sin prefijo se aplican a `openclaw.json`. Utilice
`auth-profiles:<agentId>:<path>` para los destinos `auth-profiles.json` existentes.
La ruta de destino debe ser un destino de SecretRef registrado en OpenClaw. El comando
de configuración no crea secretos con nombres arbitrarios en OpenClaw; Vault sigue siendo el
almacén de secretos y OpenClaw solo almacena SecretRefs en campos de configuración compatibles.

## Formato del identificador de SecretRef

Los identificadores de SecretRef de Vault utilizan esta convención:

```text
<vault-secret-path>/<field>
```

Ejemplos:

| Identificador de SecretRef    | Lectura predeterminada de Vault KV v2 | Campo devuelto |
| ----------------------------- | ------------------------------------- | -------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

El campo devuelto por Vault debe ser una cadena.

Para KV v1, establezca:

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

Entonces, `providers/openrouter/apiKey` lee:

```text
secret/providers/openrouter -> apiKey
```

## Qué almacena OpenClaw

Al aplicar un plan de configuración de Vault se almacena un proveedor gestionado por el plugin:

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

Los campos de credenciales apuntan a ese proveedor:

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

El valor resuelto solo reside en la instantánea de secretos activa de la ejecución.

## Contenedores e implementaciones gestionadas

Los Gateways en contenedores siguen utilizando el mismo plugin y la misma configuración de SecretRef. El
contenedor debe recibir:

- `VAULT_ADDR`
- una fuente de autenticación:
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` junto con `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` junto con `OPENCLAW_VAULT_AUTH_MOUNT`,
    `OPENCLAW_VAULT_AUTH_ROLE` y `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` junto con `OPENCLAW_VAULT_AUTH_ROLE`; opcionalmente,
    sustituya `OPENCLAW_VAULT_AUTH_MOUNT` o `OPENCLAW_VAULT_JWT_FILE`
- los valores opcionales `VAULT_NAMESPACE`, `OPENCLAW_VAULT_KV_MOUNT` y
  `OPENCLAW_VAULT_KV_VERSION`

Al utilizar Kubernetes, se recomienda `OPENCLAW_VAULT_AUTH_METHOD=kubernetes`
cuando Vault tenga configurada la autenticación de Kubernetes para el clúster. Utilice
`OPENCLAW_VAULT_AUTH_METHOD=jwt` únicamente cuando Vault esté configurado para tratar el clúster
como un emisor JWT/OIDC genérico. Cualquiera de las dos opciones es mejor que un token de Vault
de larga duración en un Secret de Kubernetes. Las implementaciones con un contenedor auxiliar o inyector de Vault Agent pueden
utilizar `token_file` en su lugar.

Para configuraciones de Vault multiinquilino, mantenga el enrutamiento de inquilinos en la política de Vault y en
la configuración de la implementación. OpenClaw no requiere un montaje, rol o ruta fijos: cada
entorno de Gateway puede configurar sus propios `OPENCLAW_VAULT_KV_MOUNT`,
`OPENCLAW_VAULT_AUTH_ROLE` e identificadores de SecretRef. Si un único Gateway compartido debe resolver
distintos usuarios de Vault al mismo tiempo, utilice proveedores exec configurados manualmente
que encapsulen entornos de autenticación distintos, o separe los inquilinos entre entornos de Gateway
con entornos de Vault independientes.

## Contenido relacionado

- [Gestión de secretos](/es/gateway/secrets)
- [`openclaw secrets`](/es/cli/secrets)
- [Inventario de plugins](/es/plugins/plugin-inventory)
