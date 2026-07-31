---
read_when:
    - Depuración de la autenticación del modelo o la caducidad de OAuth
    - Documentación del almacenamiento de autenticación o credenciales
summary: 'Autenticación de modelos: OAuth, claves de API, reutilización de la CLI de Claude y token de configuración de Anthropic'
title: Autenticación
x-i18n:
    generated_at: "2026-07-26T05:12:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
Esta página abarca la autenticación de **proveedores de modelos** (claves de API, OAuth, reutilización de la CLI de Claude, token de configuración de Anthropic). Para la autenticación de **conexión al Gateway** (token, contraseña, proxy de confianza), consulte [Configuración](/es/gateway/configuration) y [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).
</Note>

OpenClaw admite OAuth y claves de API para proveedores de modelos. Para un host de Gateway siempre activo, una clave de API es la opción más predecible; los flujos de suscripción/OAuth también funcionan cuando se ajustan al modelo de cuenta del proveedor.

- Flujo completo de OAuth y disposición del almacenamiento: [/concepts/oauth](/es/concepts/oauth)
- Autenticación basada en SecretRef (proveedores `env`/`file`/`exec`): [Gestión de secretos](/es/gateway/secrets)
- Códigos de elegibilidad/motivo de credenciales utilizados por `models status --probe`: [Semántica de las credenciales de autenticación](/es/auth-credential-semantics)

## Configuración recomendada: clave de API (cualquier proveedor)

1. Cree una clave de API en la consola del proveedor.
2. Colóquela en el **host de Gateway** (la máquina que ejecuta `openclaw gateway`):

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Si el Gateway se ejecuta mediante systemd/launchd, coloque la clave en `~/.openclaw/.env` para que el daemon pueda leerla:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Reinicie el proceso de Gateway (o el daemon) y vuelva a comprobarlo:

```bash
openclaw models status
openclaw doctor
```

`openclaw onboard` también puede almacenar claves de API para que las utilice el daemon si no desea gestionar personalmente las variables de entorno. Consulte [Variables de entorno](/es/help/environment) para conocer la precedencia completa de carga del entorno (`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd).

## Anthropic: reutilización de la CLI de Claude

La autenticación mediante token de configuración de Anthropic sigue siendo una vía admitida. La reutilización de la CLI de Claude (uso del tipo `claude -p`) también está autorizada para esta integración; cuando hay disponible un inicio de sesión de la CLI de Claude en el host, esa es la vía preferida para el uso local o de escritorio. Para hosts de Gateway de larga duración, una clave de API de Anthropic sigue siendo la opción más predecible, con control explícito de facturación del lado del servidor.

Configuración del host para reutilizar la CLI de Claude:

```bash
# Ejecutar en el host de Gateway
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Este proceso consta de dos pasos: iniciar sesión en Anthropic con Claude Code en el host y, después, indicar a OpenClaw que enrute la selección de modelos de Anthropic a través del backend local `claude-cli` y almacene el perfil de autenticación correspondiente de OpenClaw.

El servicio de Gateway debe poder resolver `claude` en `PATH`. Si un despliegue necesita una
ruta de ejecutable no estándar, registre un contenedor mediante un
[Plugin de backend de CLI](/es/plugins/cli-backend-plugins).

## Introducción manual del token

Funciona con cualquier proveedor; escribe en el almacén de autenticación SQLite por agente y actualiza la configuración:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw lee los perfiles de autenticación desde el `openclaw-agent.sqlite` de cada agente. Los detalles del endpoint (`baseUrl`, `api`, identificadores de modelos, encabezados y tiempos de espera) deben estar en `models.providers.<id>` dentro de `openclaw.json` o `models.json`, no en los perfiles de autenticación.

Si una instalación anterior todavía tiene `auth-profiles.json`, `auth-state.json` o una estructura plana como `{ "openrouter": { "apiKey": "..." } }`, ejecute `openclaw doctor --fix` para importarla a SQLite; doctor conserva copias de seguridad con marca de tiempo junto a los archivos JSON originales.

Las rutas de autenticación externas, como `auth: "aws-sdk"` de Bedrock, no son credenciales. Para una ruta de Bedrock con nombre, establezca `auth.profiles.<id>.mode: "aws-sdk"` en `openclaw.json`; no escriba `type: "aws-sdk"` en el almacén de perfiles de autenticación. `openclaw doctor --fix` migra los marcadores heredados del SDK de AWS desde el almacén de credenciales a los metadatos de configuración.

### Credenciales respaldadas por SecretRef

- Las credenciales `api_key` pueden usar `keyRef: { source, provider, id }`
- Las credenciales `token` pueden usar `tokenRef: { source, provider, id }`
- Los perfiles en modo OAuth rechazan las credenciales SecretRef: si `auth.profiles.<id>.mode` es `"oauth"`, se rechaza un `keyRef`/`tokenRef` respaldado por SecretRef para ese perfil.

## Comprobación del estado de autenticación de los modelos

```bash
openclaw models status
openclaw doctor
```

Comprobación apta para automatización: código de salida `1` si ha caducado o falta y `2` si está próximo a caducar:

```bash
openclaw models status --check
```

Sondeos de autenticación en vivo (añada `--probe-provider`, `--probe-profile`, `--probe-timeout`, `--probe-concurrency` o `--probe-max-tokens` para limitar el ámbito):

```bash
openclaw models status --probe
```

Notas:

- Las filas del sondeo pueden proceder de perfiles de autenticación, credenciales del entorno o `models.json`.
- Si `auth.order.<provider>` omite un perfil almacenado, el sondeo informa de `excluded_by_auth_order` para ese perfil en lugar de intentar utilizarlo.
- Si existe autenticación, pero OpenClaw no puede resolver un modelo sondeable para ese proveedor, el sondeo informa de `status: no_model`.
- Los periodos de espera por límite de velocidad pueden limitarse a un modelo: un perfil que esté en espera para un modelo aún puede servir a otro modelo del mismo proveedor.

Scripts operativos opcionales (systemd/Termux): [Scripts de supervisión de autenticación](/es/help/scripts#auth-monitoring-scripts).

## Rotación de claves de API (Gateway)

Algunos proveedores vuelven a intentar una solicitud con otra clave configurada cuando una llamada alcanza un límite de velocidad del proveedor.

Orden de prioridad de las claves por proveedor:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (anulación única, fija una clave)
2. `<PROVIDER>_API_KEYS` (lista separada por comas, espacios o puntos y comas)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (cualquier variable de entorno con este prefijo)

Los proveedores de Google (`google`, `google-vertex`) también recurren a `GOOGLE_API_KEY` como alternativa. La lista combinada se deduplica antes de utilizarse.

OpenClaw solo rota a la siguiente clave cuando el mensaje de error coincide con: `rate_limit`, `rate limit`, `429`, `quota exceeded`/`quota_exceeded`, `resource exhausted`/`resource_exhausted` o `too many requests`. Los demás errores no se vuelven a intentar con claves alternativas. Si todas las claves fallan, se devuelve el error final del último intento.

<Note>
Las frases específicas del proveedor, como `ThrottlingException`, `concurrency limit reached` o `workers_ai ... quota limit exceeded`, determinan la **clasificación de conmutación por error/reintento** (cambio de modelos o proveedores ante fallos repetidos), un mecanismo distinto de la rotación de claves de API descrita anteriormente.
</Note>

Eliminar la autenticación guardada no revoca la clave en el proveedor; rótela o revóquela en el panel del proveedor cuando necesite invalidarla del lado del proveedor.

## Eliminación de la autenticación del proveedor mientras el Gateway está en ejecución

Cuando se elimina la autenticación de un proveedor mediante el plano de control del Gateway, OpenClaw elimina los perfiles de autenticación guardados de ese proveedor y cancela las ejecuciones activas de chat/agente cuyo proveedor de modelo seleccionado coincida con el eliminado. Las ejecuciones canceladas emiten los eventos normales de cancelación/ciclo de vida con `stopReason: "auth-revoked"`, para que los clientes conectados puedan mostrar que la ejecución se detuvo porque se eliminaron las credenciales.

## Control de la credencial utilizada

### OpenAI e identificadores heredados `openai-codex`

Tanto los perfiles de clave de API de OpenAI como los perfiles OAuth de ChatGPT/Codex utilizan el identificador de proveedor canónico `openai`. Utilice identificadores de perfil `openai:*` y `auth.order.openai` para las configuraciones nuevas.

Si encuentra `openai-codex` en configuraciones antiguas, identificadores de perfiles de autenticación o `auth.order.openai-codex`, trátelo como entrada de migración heredada; no cree perfiles `openai-codex` nuevos. Ejecute:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor reescribe los identificadores de perfil heredados `openai-codex:*` y las entradas `auth.order.openai-codex` para utilizar la ruta canónica `openai`. Para el enrutamiento de modelos/entorno de ejecución específico de OpenAI, consulte [OpenAI](/es/providers/openai).

### Durante el inicio de sesión (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` mantiene separados varios inicios de sesión OAuth del mismo proveedor dentro de un agente.

`--force` elimina los perfiles de autenticación guardados de ese proveedor en el directorio del agente seleccionado y, a continuación, vuelve a ejecutar el mismo flujo de autenticación. Utilícelo cuando un perfil guardado esté bloqueado, haya caducado o esté vinculado a la cuenta equivocada. No revoca las credenciales en el proveedor.

```bash
openclaw models auth login --provider anthropic --force
```

### Por sesión (comando de chat)

- `/model <alias-or-id>@<profileId>` fija una credencial específica del proveedor para la sesión actual (ejemplos de identificadores de perfil: `anthropic:default`, `anthropic:work`).
- `/model` (o `/model list`) muestra un selector compacto; `/model status` muestra la vista completa (candidatos + siguiente perfil de autenticación, además de los detalles del endpoint del proveedor cuando estén configurados).

Si cambia el orden de autenticación o la fijación de perfiles de un chat que ya está en ejecución, envíe `/new` o `/reset` para iniciar una sesión nueva; las sesiones existentes conservan la selección actual de modelo/perfil hasta que se restablezcan.

### Por agente (anulación mediante CLI)

Las anulaciones del orden de autenticación se almacenan en el estado de autenticación SQLite de ese agente:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Utilice `--agent <id>` para especificar un agente concreto; omítalo para utilizar el agente predeterminado configurado. `openclaw models status --probe` muestra los perfiles almacenados omitidos como `excluded_by_auth_order` en lugar de omitirlos silenciosamente.

## Solución de problemas

### "No se encontraron credenciales"

Configure una clave de API de Anthropic en el **host de Gateway** o configure la vía del token de configuración de Anthropic y vuelva a comprobarlo:

```bash
openclaw models status
```

### Token próximo a caducar/caducado

Ejecute `openclaw models status` para ver qué perfil está próximo a caducar. Si falta un perfil de token de Anthropic o ha caducado, actualícelo mediante el token de configuración o migre a una clave de API de Anthropic.

## Relacionado

- [Gestión de secretos](/es/gateway/secrets)
- [Acceso remoto](/es/gateway/remote)
- [Almacenamiento de autenticación](/es/concepts/oauth)
