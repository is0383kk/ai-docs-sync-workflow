---
read_when:
    - Trabajo en las funciones del canal de Microsoft Teams
summary: Estado, capacidades y configuración de la compatibilidad con bots de Microsoft Teams
title: Microsoft Teams
x-i18n:
    generated_at: "2026-07-26T05:31:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a4cf686da27e28b58f7afaad8cc837dbddb93219cde0c37285f9f6895f6fb8c
    source_path: channels/msteams.md
    workflow: 16
---

Estado: se admiten texto y archivos adjuntos en mensajes directos; el envío de archivos a canales/grupos requiere `sharePointSiteId` y permisos de Graph (consulte [Envío de archivos en chats grupales](#sending-files-in-group-chats)). Las encuestas se envían mediante tarjetas adaptables. Las acciones de mensajes exponen explícitamente `upload-file` para los envíos en los que el archivo va primero.

## Plugin incluido

Microsoft Teams se distribuye como un Plugin incluido en las versiones actuales de OpenClaw; no se requiere una instalación independiente en la compilación empaquetada normal.

En una compilación anterior o una instalación personalizada que excluya el Teams incluido, instale directamente el paquete npm:

```bash
openclaw plugins install @openclaw/msteams
```

Use el paquete sin especificar versión para seguir la etiqueta de la versión oficial actual. Fije una versión exacta solo cuando necesite una instalación reproducible.

Checkout local (ejecución desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

Detalles: [Plugins](/es/tools/plugin)

## Configuración rápida

[`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) gestiona el registro del bot, la creación del manifiesto y la generación de credenciales con un solo comando.

**1. Instalar e iniciar sesión**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # verifique que ha iniciado sesión y consulte la información del inquilino
```

<Note>
La CLI de Teams se encuentra actualmente en versión preliminar. Los comandos y las opciones pueden cambiar entre versiones.
</Note>

**2. Iniciar un túnel** (Teams no puede acceder a localhost)

Instale y autentique la CLI de devtunnel si es necesario ([guía de introducción](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# Configuración única (URL persistente entre sesiones):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# En cada sesión de desarrollo:
devtunnel host my-openclaw-bot
# Su endpoint: https://<tunnel-id>.devtunnels.ms/api/messages
```

<Note>
`--allow-anonymous` es obligatorio porque Teams no puede autenticarse con devtunnels. El SDK de Teams sigue validando cada solicitud entrante del bot.
</Note>

Alternativas: `ngrok http 3978` o `tailscale funnel 3978` (las URL pueden cambiar en cada sesión).

**3. Crear la aplicación**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

Esto crea una aplicación de Entra ID (Azure AD), genera un secreto de cliente, crea y carga un manifiesto de aplicación de Teams (con iconos) y registra un bot administrado por Teams (no se necesita una suscripción de Azure). La salida incluye `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID` y un **Teams App ID**; también ofrece instalar directamente la aplicación en Teams.

**4. Configurar OpenClaw** con las credenciales de la salida:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

También puede usar directamente las variables de entorno: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

**5. Instalar la aplicación en Teams**

`teams app create` le solicita que instale la aplicación; seleccione "Install in Teams". Para obtener el enlace de instalación más adelante:

```bash
teams app get <teamsAppId> --install-link
```

**6. Verificar que todo funciona**

```bash
teams app doctor <teamsAppId>
```

Ejecuta diagnósticos del registro del bot, la configuración de la aplicación AAD, la validez del manifiesto y la configuración de SSO.

Para producción, considere la [autenticación federada](#federated-authentication-certificate-plus-managed-identity) (certificado o identidad administrada) en lugar de secretos de cliente.

<Note>
Los chats grupales están bloqueados de forma predeterminada (`channels.msteams.groupPolicy: "allowlist"`). Para permitir respuestas grupales, establezca `channels.msteams.groupAllowFrom` o use `groupPolicy: "open"` para permitir a cualquier miembro (con la condición de que se mencione al bot).
</Note>

## Objetivos

- Comunicarse con OpenClaw mediante mensajes directos, chats grupales o canales de Teams.
- Mantener un enrutamiento determinista: las respuestas siempre vuelven al canal del que proceden.
- Aplicar de forma predeterminada un comportamiento seguro en los canales (se requieren menciones, salvo que se configure lo contrario).

## Escrituras de configuración

De forma predeterminada, Microsoft Teams puede escribir actualizaciones de configuración activadas por `/config set|unset` (requiere `commands.config: true`).

Para desactivarlo:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## Control de acceso (mensajes directos y grupos)

**Acceso a mensajes directos**

- Valor predeterminado: `channels.msteams.dmPolicy = "pairing"`. Los remitentes desconocidos se ignoran hasta que sean aprobados.
- `channels.msteams.allowFrom` debe usar identificadores de objeto de AAD estables o grupos estáticos de acceso de remitentes como `accessGroup:core-team`.
- No dependa de la coincidencia de UPN/nombre para mostrar en las listas de permitidos, ya que pueden cambiar. OpenClaw desactiva de forma predeterminada la coincidencia directa de nombres; actívela con `channels.msteams.dangerouslyAllowNameMatching: true`.
- El asistente puede resolver nombres a identificadores mediante Microsoft Graph cuando las credenciales lo permiten.

**Acceso de grupos**

- Valor predeterminado: `channels.msteams.groupPolicy = "allowlist"` (bloqueado a menos que añada `groupAllowFrom`). `channels.defaults.groupPolicy` puede reemplazar el valor predeterminado compartido cuando `channels.msteams.groupPolicy` no está definido.
- `channels.msteams.groupAllowFrom` controla qué remitentes o grupos estáticos de acceso de remitentes pueden activar el bot en chats grupales/canales (utiliza `channels.msteams.allowFrom` como alternativa).
- Establezca `groupPolicy: "open"` para permitir a cualquier miembro (de forma predeterminada, se sigue requiriendo una mención).
- Para bloquear **todos** los canales, establezca `channels.msteams.groupPolicy: "disabled"`.

Ejemplo:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**Lista de permitidos de equipos y canales**

- Limite las respuestas de grupos/canales enumerando los equipos y canales en `channels.msteams.teams`.
- Use como claves los identificadores estables de conversación de Teams obtenidos de los enlaces de Teams, no nombres para mostrar que puedan cambiar (consulte [Identificadores de equipo y canal](#team-and-channel-ids-common-gotcha)).
- Cuando `groupPolicy="allowlist"` y una lista de equipos permitidos están presentes, solo se aceptan los equipos/canales enumerados (con la condición de que se mencione al bot).
- El asistente de configuración acepta entradas `Team/Channel` y las almacena.
- Al iniciarse, OpenClaw resuelve los nombres de equipos/canales y de las listas de usuarios permitidos a identificadores (cuando los permisos de Graph lo permiten) y registra la correspondencia. Los nombres sin resolver se conservan tal como se escribieron, pero se ignoran para el enrutamiento, salvo que se establezca `channels.msteams.dangerouslyAllowNameMatching: true`.

Ejemplo:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>Configuración manual (sin la CLI de Teams)</strong></summary>

### Cómo funciona

1. Asegúrese de que el Plugin de Microsoft Teams esté disponible (incluido en las versiones actuales).
2. Cree un **Azure Bot** (identificador de aplicación + secreto + identificador de inquilino).
3. Cree un **paquete de aplicación de Teams** que haga referencia al bot e incluya los permisos RSC indicados a continuación.
4. Cargue/instale la aplicación de Teams en un equipo (o en el ámbito personal para mensajes directos).
5. Configure `msteams` en `~/.openclaw/openclaw.json` (o variables de entorno) e inicie el Gateway.
6. El Gateway escucha de forma predeterminada el tráfico de Webhook de Bot Framework en `/api/messages`.

### Paso 1: Crear Azure Bot

1. Vaya a [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot)
2. Complete la pestaña **Basics**:

   | Campo              | Valor                                                    |
   | ------------------ | -------------------------------------------------------- |
   | **Bot handle**     | El nombre del bot, p. ej., `openclaw-msteams` (debe ser único) |
   | **Subscription**   | Seleccione su suscripción de Azure                       |
   | **Resource group** | Cree uno nuevo o use uno existente                       |
   | **Pricing tier**   | **Free** para desarrollo/pruebas                         |
   | **Type of App**    | **Single Tenant** (recomendado; consulte la nota siguiente) |
   | **Creation type**  | **Create new Microsoft App ID**                          |

<Warning>
La creación de nuevos bots multiinquilino quedó obsoleta después del 2025-07-31. Use **Single Tenant** para los bots nuevos.
</Warning>

3. Haga clic en **Review + create** y, a continuación, en **Create** (~1-2 minutos).

### Paso 2: Obtener las credenciales

1. Recurso de Azure Bot → **Configuration** → copie **Microsoft App ID** (su `appId`).
2. **Manage Password** → App Registration → **Certificates & secrets** → **New client secret** → copie **Value** (su `appPassword`).
3. **Overview** → copie **Directory (tenant) ID** (su `tenantId`).

### Paso 3: Configurar el endpoint de mensajería

1. Azure Bot → **Configuration**.
2. Establezca **Messaging endpoint**:
   - Producción: `https://your-domain.com/api/messages`
   - Desarrollo local: use un túnel (consulte [Desarrollo local](#local-development-tunneling))

### Paso 4: Habilitar el canal de Teams

1. Azure Bot → **Channels**.
2. Haga clic en **Microsoft Teams** → Configure → Save.
3. Acepte los Términos de servicio.

### Paso 5: Crear el manifiesto de la aplicación de Teams

- Incluya una entrada `bot` con `botId = <App ID>`.
- Ámbitos: `personal`, `team`, `groupChat`.
- `supportsFiles: true` (obligatorio para gestionar archivos en el ámbito personal).
- Añada permisos RSC (consulte [Permisos RSC](#current-teams-rsc-permissions-manifest)).
- Cree los iconos: `outline.png` (32x32) y `color.png` (192x192).
- Comprima juntos `manifest.json`, `outline.png` y `color.png`.

### Paso 6: Configurar OpenClaw

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

Variables de entorno: `MSTEAMS_APP_ID`, `MSTEAMS_APP_PASSWORD`, `MSTEAMS_TENANT_ID`.

### Paso 7: Ejecutar el Gateway

El canal de Teams se inicia automáticamente cuando el Plugin está disponible y la configuración `msteams` contiene credenciales.

</details>

## Autenticación federada (certificado más identidad administrada)

Para producción, OpenClaw admite la **autenticación federada** como alternativa a los secretos de cliente mediante `channels.msteams.authType: "federated"`. Hay dos métodos:

### Opción A: Autenticación basada en certificados

Use un certificado PEM registrado con el registro de la aplicación de Entra ID.

**Configuración:**

1. Genere u obtenga un certificado (formato PEM con clave privada).
2. Entra ID → App Registration → **Certificates & secrets** → **Certificates** → cargue el certificado público.

**Configuración:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Variables de entorno:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### Opción B: Azure Managed Identity

Use Azure Managed Identity para la autenticación sin contraseña en la infraestructura de Azure (AKS, App Service y máquinas virtuales de Azure).

**Cómo funciona:**

1. El pod o la máquina virtual del bot tiene una identidad administrada (asignada por el sistema o por el usuario).
2. Una credencial de identidad federada vincula la identidad administrada con el registro de la aplicación de Entra ID.
3. Durante la ejecución, OpenClaw usa `@azure/identity` para obtener tokens del endpoint IMDS de Azure.
4. El token se pasa al SDK de Teams para autenticar el bot.

**Requisitos previos:**

- Infraestructura de Azure con identidad administrada habilitada (identidad de carga de trabajo de AKS, App Service, VM).
- Credencial de identidad federada creada en el registro de aplicación de Entra ID.
- Acceso de red a IMDS (`169.254.169.254:80`) desde el pod o la VM.

**Configuración (identidad administrada asignada por el sistema):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**Configuración (identidad administrada asignada por el usuario):** añada `managedIdentityClientId: "<MI_CLIENT_ID>"` al bloque anterior.

**Variables de entorno:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>` (solo para la asignada por el usuario)

### Configuración de la identidad de carga de trabajo de AKS

Para implementaciones de AKS que usan identidad de carga de trabajo:

1. **Habilite la identidad de carga de trabajo** en el clúster de AKS.
2. **Cree una credencial de identidad federada** en el registro de aplicación de Entra ID:

   ```bash
   az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "<AKS_OIDC_ISSUER_URL>",
     "subject": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. **Anote la cuenta de servicio de Kubernetes** con el identificador de cliente de la aplicación:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: "<APP_CLIENT_ID>"
   ```

4. **Etiquete el pod** para la inyección de la identidad de carga de trabajo:

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. **Permita el acceso de red** a IMDS (`169.254.169.254`): si se usa NetworkPolicy, añada una regla de salida para `169.254.169.254/32` en el puerto 80.

### Comparación de tipos de autenticación

| Método                   | Configuración                                  | Ventajas                                | Desventajas                                            |
| ------------------------ | ---------------------------------------------- | --------------------------------------- | ------------------------------------------------------ |
| **Secreto de cliente**   | `appPassword`                             | Configuración sencilla                  | Requiere rotar el secreto; es menos seguro              |
| **Certificado**          | `authType: "federated"` + `certificatePath`       | Sin secretos compartidos por la red     | Sobrecarga de administración de certificados           |
| **Identidad administrada** | `authType: "federated"` + `useManagedIdentity`     | Sin contraseñas ni secretos que gestionar | Requiere infraestructura de Azure                    |

`certificateThumbprint` se puede establecer junto con `certificatePath`, pero actualmente la ruta de autenticación no lo lee; solo se acepta para compatibilidad futura.

**Valor predeterminado:** cuando `authType` no está establecido, OpenClaw usa autenticación mediante secreto de cliente (`appPassword`). Las configuraciones existentes siguen funcionando sin cambios.

## Desarrollo local (túneles)

Teams no puede acceder a `localhost`. Use un túnel de desarrollo persistente para que la URL permanezca estable entre sesiones:

```bash
# Configuración inicial:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# En cada sesión de desarrollo:
devtunnel host my-openclaw-bot
```

Alternativas: `ngrok http 3978` o `tailscale funnel 3978` (las URL pueden cambiar en cada sesión).

Si cambia la URL del túnel, actualice el punto de conexión:

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## Prueba del bot

**Ejecute los diagnósticos:**

```bash
teams app doctor <teamsAppId>
```

Comprueba en una sola pasada el registro del bot, la aplicación de AAD, el manifiesto y la configuración de SSO.

**Envíe un mensaje de prueba:**

1. Instale la aplicación de Teams (enlace de instalación de `teams app get <id> --install-link`).
2. Busque el bot en Teams y envíele un mensaje directo.
3. Revise los registros del Gateway para comprobar la actividad entrante.

## Variables de entorno

Estas claves de configuración relacionadas con la autenticación se pueden establecer mediante variables de entorno en lugar de `openclaw.json` (otras claves de configuración, como `groupPolicy` o `historyLimit`, solo pueden establecerse en la configuración):

| Variable de entorno                  | Clave de configuración     | Notas                                      |
| ------------------------------------ | -------------------------- | ------------------------------------------ |
| `MSTEAMS_APP_ID`                  | `appId`         |                                            |
| `MSTEAMS_APP_PASSWORD`                  | `appPassword`         |                                            |
| `MSTEAMS_TENANT_ID`                  | `tenantId`         |                                            |
| `MSTEAMS_AUTH_TYPE`                  | `authType`         | `"secret"` o `"federated"`   |
| `MSTEAMS_CERTIFICATE_PATH`                  | `certificatePath`         | federada + certificado                     |
| `MSTEAMS_CERTIFICATE_THUMBPRINT`                  | `certificateThumbprint`         | se acepta, pero no se requiere para autenticar |
| `MSTEAMS_USE_MANAGED_IDENTITY`                  | `useManagedIdentity`         | federada + identidad administrada          |
| `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID`                  | `managedIdentityClientId`         | solo identidad administrada asignada por el usuario |

## Acción de información de miembros

OpenClaw expone una acción `member-info` respaldada por Graph para Microsoft Teams, de modo que los agentes y las automatizaciones puedan obtener detalles verificados de los participantes de una conversación configurada.

Requisitos:

- Permisos RSC `ChannelSettings.Read.Group` y `TeamMember.Read.Group` (ya incluidos en el manifiesto recomendado).

La acción está disponible siempre que se hayan configurado las credenciales de Graph; no existe un conmutador `channels.msteams.actions.memberInfo` independiente.
Las consultas de canales estándar devuelven la identidad coincidente de la lista de miembros del equipo, el nombre para mostrar, el correo electrónico y los roles.
En el mensaje directo o chat grupal actual, la acción puede devolver el identificador de usuario estable del remitente de confianza.
Las consultas de miembros de canales privados o compartidos y de chats que no sean el actual requieren permisos adicionales para acceder a la lista de miembros
y se rechazan con la configuración de permisos predeterminada.

## Contexto del historial

- `channels.msteams.historyLimit` controla cuántos mensajes recientes del canal o grupo se incluyen en el prompt. Recurre a `messages.groupChat.historyLimit` y después usa 50 de forma predeterminada. Establezca `0` para deshabilitarlo.
- El historial de hilos obtenido se filtra según las listas de remitentes permitidos (`allowFrom` / `groupAllowFrom`), por lo que la inicialización del contexto de hilos solo incluye mensajes de remitentes permitidos.
- El contexto de archivos adjuntos citados (analizado a partir del HTML del esquema Skype Reply en los archivos adjuntos de la propia respuesta) se transmite sin filtrar; actualmente, el filtro de la lista de remitentes permitidos solo se aplica a la inicialización con el historial de hilos.
- El historial de mensajes directos se puede limitar con `channels.msteams.dmHistoryLimit` (turnos del usuario). Anulaciones por usuario: `channels.msteams.dms["<user_id>"].historyLimit`.

## Permisos RSC actuales de Teams (manifiesto)

Estos son los **permisos resourceSpecific existentes** en el manifiesto de nuestra aplicación de Teams. Solo se aplican dentro del equipo o chat donde está instalada la aplicación.

**Para canales (ámbito del equipo):**

- `ChannelMessage.Read.Group` (Application) - recibir todos los mensajes del canal sin @mención
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**Para chats grupales:**

- `ChatMessage.Read.Chat` (Application) - recibir todos los mensajes del chat grupal sin @mención

Añada permisos RSC mediante la CLI de Teams:

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## Ejemplo de manifiesto de Teams (censurado)

Ejemplo mínimo y válido con los campos obligatorios. Sustituya los identificadores y las URL.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "Su organización",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw en Teams", full: "OpenClaw en Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### Consideraciones del manifiesto (campos obligatorios)

- `bots[].botId` **debe** coincidir con el identificador de la aplicación de Azure Bot.
- `webApplicationInfo.id` **debe** coincidir con el identificador de la aplicación de Azure Bot.
- `bots[].scopes` debe incluir las superficies que se prevé usar (`personal`, `team`, `groupChat`).
- `bots[].supportsFiles: true` es obligatorio para gestionar archivos en el ámbito personal.
- `authorization.permissions.resourceSpecific` debe incluir la lectura y el envío en canales para el tráfico de canales.

### Actualización de una aplicación existente

```bash
# Descargar, editar y volver a cargar el manifiesto
teams app manifest download <teamsAppId> manifest.json
# Editar manifest.json localmente...
teams app manifest upload manifest.json <teamsAppId>
# La versión se incrementa automáticamente si cambia el contenido
```

Después de actualizar, vuelva a instalar la aplicación en cada equipo y **cierre Teams por completo y vuelva a iniciarlo** (no se limite a cerrar la ventana) para borrar los metadatos de la aplicación almacenados en caché.

<details>
<summary>Actualización manual del manifiesto (sin CLI)</summary>

1. Actualice `manifest.json` con la nueva configuración.
2. **Incremente el campo `version`** (p. ej., `1.0.0` → `1.1.0`).
3. **Vuelva a comprimir** el manifiesto con los iconos (`manifest.json`, `outline.png`, `color.png`).
4. Cargue el nuevo archivo zip:
   - **Teams Admin Center:** Teams apps → Manage apps → busque la aplicación → Upload new version.
   - **Transferencia local:** Teams → Apps → Manage your apps → Upload a custom app.

</details>

## Capacidades: solo RSC frente a Graph

### Con **solo RSC de Teams** (aplicación instalada, sin permisos de la API de Graph)

Funciona:

- Leer el contenido de **texto** de los mensajes del canal.
- Enviar contenido de **texto** en mensajes del canal.
- Recibir archivos adjuntos **personales (mensajes directos)**.

NO funciona:

- El contenido de **imágenes o archivos** de canales o grupos (la carga útil solo incluye un fragmento HTML).
- Descargar archivos adjuntos almacenados en SharePoint/OneDrive.
- Leer el historial de mensajes más allá del evento de Webhook en directo.

### Con **RSC de Teams + permisos de aplicación de Microsoft Graph**

Añade:

- Descargar contenido hospedado (imágenes pegadas en los mensajes).
- Descargar archivos adjuntos almacenados en SharePoint/OneDrive.
- Leer el historial de mensajes de canales o chats mediante Graph.

### RSC frente a la API de Graph

| Capacidad                 | Permisos RSC               | API de Graph                                       |
| ------------------------- | -------------------------- | -------------------------------------------------- |
| **Mensajes en tiempo real** | Sí (mediante webhook)    | No (solo sondeo)                                   |
| **Mensajes históricos**   | No                         | Sí (permite consultar el historial)                |
| **Complejidad de configuración** | Solo el manifiesto de la aplicación | Requiere consentimiento del administrador + flujo de tokens |
| **Funciona sin conexión** | No (debe estar en ejecución) | Sí (permite consultar en cualquier momento)      |

**En resumen:** RSC sirve para escuchar en tiempo real; la API de Graph sirve para el acceso histórico. Para recuperar los mensajes perdidos mientras se estaba sin conexión, se necesita la API de Graph con `ChannelMessage.Read.All` (requiere consentimiento del administrador).

## Contenido multimedia e historial mediante Graph

Habilite únicamente los permisos de aplicación de Microsoft Graph necesarios para los ámbitos de Teams y los datos que utilice:

1. Entra ID (Azure AD) **App Registration** → añada **Application permissions** de Graph:
   - `ChannelMessage.Read.All` para los archivos adjuntos y el historial de los canales.
   - `Chat.Read.All` para los archivos adjuntos y el historial de los chats grupales.
   - `Files.Read.All` cuando deban descargarse los bytes de los archivos adjuntos desde el almacenamiento de SharePoint/OneDrive; las configuraciones que solo utilizan el historial no lo necesitan.
2. Seleccione **Grant admin consent** para el inquilino.
3. Incremente la **manifest version** de la aplicación de Teams, vuelva a cargarla y **reinstale la aplicación en Teams**.
4. **Cierre Teams por completo y vuelva a iniciarlo** para borrar los metadatos almacenados en caché de la aplicación.

### Recuperación de archivos de canales y grupos (`graphMediaFallback`)

Teams puede eliminar los marcadores de archivos de la actividad HTML enviada a un bot. En ese caso, la actividad de Bot Framework no se puede distinguir de un mensaje HTML normal; la referencia completa del archivo adjunto solo existe en la copia del mensaje de Graph.

Habilite el mecanismo alternativo después de conceder los permisos anteriores:

```json5
{
  channels: {
    msteams: {
      graphMediaFallback: true,
    },
  },
}
```

Esto se aplica únicamente a canales y chats grupales. Añade una consulta de mensaje a Graph cada vez que una actividad HTML no produce contenido multimedia que se pueda descargar directamente, incluidos los mensajes normales o que solo contienen menciones. El valor predeterminado es `false`, por lo que las instalaciones existentes no generan automáticamente tráfico adicional de Graph ni errores de permisos.

**Menciones de usuarios:** las @menciones funcionan de forma inmediata para los usuarios que ya participan en la conversación. Para buscar y mencionar dinámicamente a usuarios que **no participan en la conversación actual**, añada el permiso `User.Read.All` (Application) y conceda el consentimiento del administrador.

## Limitaciones conocidas

### Tiempos de espera de los webhooks

Teams entrega los mensajes mediante un webhook HTTP. OpenClaw aplica tiempos de espera fijos del servidor HTTP
a ese receptor de webhooks: 30 s de inactividad, 30 s para la solicitud completa y 15 s
para recibir los encabezados. El contenido multimedia entrante opcional y el enriquecimiento del contexto comparten
un límite de 10 segundos. El SDK responde después de que la actividad sin procesar se haya añadido de forma duradera;
el turno del agente se procesa de manera independiente y responde proactivamente. Si el
procesamiento de la solicitud o la admisión duradera exceden la ventana del transporte, Teams puede reintentar la
actividad, y la marca de eliminación de entrada rechaza los eventos con un ID repetido.

### Compatibilidad con la nube de Teams y la URL del servicio

Esta ruta de Teams basada en el SDK se valida en producción para la nube pública de Microsoft Teams.

Las respuestas entrantes utilizan el contexto del turno entrante del SDK de Teams. Las operaciones proactivas fuera de contexto —envíos, ediciones, eliminaciones, tarjetas, encuestas, mensajes de consentimiento para archivos y respuestas en cola de larga duración— utilizan la referencia de conversación almacenada `serviceUrl`. De forma predeterminada, la nube pública utiliza el entorno de nube pública del SDK de Teams y permite referencias almacenadas en el host público de Teams Connector: `https://smba.trafficmanager.net/`.

La nube pública es el valor predeterminado. No es necesario establecer `channels.msteams.cloud` ni `channels.msteams.serviceUrl` para los bots normales de la nube pública.

Para nubes de Teams que no sean públicas, establezca `cloud` y el límite proactivo correspondiente cuando Microsoft publique uno:

- `channels.msteams.cloud` selecciona el ajuste predefinido de nube del SDK de Teams para la autenticación, la validación de JWT, los servicios de tokens y el ámbito de Graph.
- `channels.msteams.serviceUrl` selecciona el límite del punto de conexión de Bot Connector utilizado para validar las referencias de conversación almacenadas antes de ejecutar envíos, ediciones, eliminaciones, tarjetas, encuestas, mensajes de consentimiento para archivos y respuestas en cola de larga duración. Es obligatorio para las nubes USGov y DoD del SDK. Para China/21Vianet, OpenClaw utiliza el ajuste predefinido `China` del SDK y solo acepta URL de servicio almacenadas o configuradas en hosts de canales de Azure China Bot Framework.

Microsoft publica los puntos de conexión proactivos globales de Bot Connector en la sección [Crear la conversación](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages?tabs=dotnet#create-the-conversation) de la documentación sobre mensajería proactiva de Teams. Utilice el `serviceUrl` de la actividad entrante cuando esté disponible; de lo contrario, utilice la tabla de Microsoft que aparece a continuación.

| Entorno de Teams | Configuración de OpenClaw                                  | `serviceUrl` proactivo                          |
| ---------------- | ----------------------------------------------------------- | ----------------------------------------------------- |
| Público          | no se necesita configuración de nube/serviceUrl             | `https://smba.trafficmanager.net/teams`                                    |
| GCC              | establezca `serviceUrl`; no existe un ajuste predefinido independiente de nube del SDK de Teams | `https://smba.infra.gcc.teams.microsoft.com/teams` |
| GCC High         | `cloud: "USGov"` + `serviceUrl`                    | `https://smba.infra.gov.teams.microsoft.us/teams`                                    |
| DoD              | `cloud: "USGovDoD"` + `serviceUrl`                    | `https://smba.infra.dod.teams.microsoft.us/teams`                                    |
| China/21Vianet   | `cloud: "China"`                                          | utilice el `serviceUrl` de la actividad entrante |

Ejemplo para GCC, donde Microsoft documenta una URL de servicio proactivo independiente, pero el SDK de Teams no ofrece un ajuste predefinido independiente para la nube GCC:

```json
{
  "channels": {
    "msteams": {
      "serviceUrl": "https://smba.infra.gcc.teams.microsoft.com/teams"
    }
  }
}
```

Ejemplo para GCC High:

```json
{
  "channels": {
    "msteams": {
      "cloud": "USGov",
      "serviceUrl": "https://smba.infra.gov.teams.microsoft.us/teams"
    }
  }
}
```

`channels.msteams.serviceUrl` está restringido a los hosts compatibles de Microsoft Teams Bot Connector. Cuando se configura una URL de servicio, OpenClaw comprueba que el `serviceUrl` de la conversación almacenada utilice el mismo host antes de ejecutar envíos, ediciones, eliminaciones, tarjetas, encuestas o respuestas en cola de larga duración. Con la configuración predeterminada de la nube pública, OpenClaw aplica un cierre seguro si una conversación almacenada apunta fuera del host público de Teams Connector. Después de cambiar la configuración de la nube o la URL del servicio, reciba un mensaje nuevo de la conversación para que la referencia de conversación almacenada esté actualizada.

China/21Vianet no tiene una URL global independiente de `smba` proactivo en la tabla de puntos de conexión proactivos de Teams de Microsoft. Configure `cloud: "China"` para que el SDK de Teams utilice los puntos de conexión de autenticación, tokens y JWT de Azure China. Los envíos proactivos requieren entonces una referencia de conversación almacenada procedente de una actividad entrante de Teams en China, o una URL de servicio configurada explícitamente, dentro del límite del canal de Azure China Bot Framework (`*.botframework.azure.cn`). Los asistentes de Teams respaldados por Graph están deshabilitados para `cloud: "China"` hasta que OpenClaw enrute las solicitudes de Graph a través del punto de conexión de Graph de Azure China.

### Formato

El Markdown de Teams es más limitado que el de Slack o Discord:

- El formato básico funciona: **negrita**, _cursiva_, `code`, enlaces.
- Es posible que el Markdown complejo (tablas y listas anidadas) no se represente correctamente.
- Las tarjetas adaptables son compatibles con las encuestas y los envíos de presentaciones semánticas (véase más adelante).

## Configuración

Opciones principales (consulte [/gateway/configuration](/es/gateway/configuration) para conocer los patrones compartidos de los canales):

- `channels.msteams.enabled`: habilita/deshabilita el canal.
- `channels.msteams.appId`, `channels.msteams.appPassword`, `channels.msteams.tenantId`: credenciales del bot.
- `channels.msteams.cloud`: entorno de nube del SDK de Teams (`Public`, `USGov`, `USGovDoD` o `China`; valor predeterminado: `Public`). Se configura con `serviceUrl` para las nubes del SDK USGov/DoD; China utiliza el ajuste preestablecido del SDK y las referencias de conversación almacenadas de Azure China Bot Framework, con los auxiliares respaldados por Graph deshabilitados hasta que se publique el enrutamiento de Graph para Azure China.
- `channels.msteams.serviceUrl`: límite de la URL del servicio Bot Connector para operaciones proactivas del SDK. La nube pública utiliza el valor predeterminado del SDK; configúrelo para GCC (`https://smba.infra.gcc.teams.microsoft.com/teams`), GCC High o DoD. China acepta hosts de canal de Azure China Bot Framework cuando la referencia de conversación almacenada procede de Teams operado por 21Vianet.
- `channels.msteams.webhook.port` (valor predeterminado: `3978`).
- `channels.msteams.webhook.path` (valor predeterminado: `/api/messages`).
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (valor predeterminado: `pairing`).
- `channels.msteams.allowFrom`: lista de permitidos para mensajes directos (se recomiendan los identificadores de objeto de AAD). El asistente resuelve los nombres en identificadores durante la configuración cuando el acceso a Graph está disponible.
- `channels.msteams.dangerouslyAllowNameMatching`: conmutador de emergencia para volver a habilitar la coincidencia mutable de UPN/nombre para mostrar y el enrutamiento directo por nombre de equipo/canal.
- `channels.msteams.textChunkLimit`: tamaño de los fragmentos de texto saliente en caracteres (valor predeterminado: `4000`, con un límite máximo estricto de `4000` independientemente de que se configure un valor superior).
- `channels.msteams.streaming.chunkMode`: `length` (valor predeterminado) o `newline` para dividir por líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.msteams.mediaAllowHosts`: lista de permitidos para hosts de archivos adjuntos entrantes (de forma predeterminada, dominios de Microsoft/Teams: Graph, SharePoint/OneDrive, CDN de Teams, Bot Framework y Azure Media Services).
- `channels.msteams.mediaAuthAllowHosts`: lista de permitidos para adjuntar encabezados Authorization en reintentos de contenido multimedia (de forma predeterminada, hosts de Graph y Bot Framework).
- `channels.msteams.graphMediaFallback`: habilita las consultas de mensajes mediante Graph cuando el HTML del canal/grupo omite los marcadores de archivo (valor predeterminado: `false`; consulte [Recuperación de archivos de canales/grupos](#channelgroup-file-recovery-graphmediafallback)).
- `channels.msteams.mediaMaxMb`: anulación por canal del límite de tamaño del contenido multimedia en MB. Si no se configura, se utiliza `agents.defaults.mediaMaxMb`.
- `channels.msteams.requireMention`: exige una @mención en canales/grupos (valor predeterminado: `true`).
- `channels.msteams.replyStyle`: `thread | top-level` (consulte [Estilo de respuesta](#reply-style-threads-vs-posts)).
- `channels.msteams.teams.<teamId>.replyStyle`: anulación por equipo.
- `channels.msteams.teams.<teamId>.requireMention`: anulación por equipo.
- `channels.msteams.teams.<teamId>.tools`: anulaciones predeterminadas por equipo de la política de herramientas (`allow`/`deny`/`alsoAllow`) utilizadas cuando no existe una anulación para el canal.
- `channels.msteams.teams.<teamId>.toolsBySender`: anulaciones predeterminadas por equipo y remitente de la política de herramientas (se admite el comodín `"*"`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: anulación por canal.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: anulación por canal.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: anulaciones por canal de la política de herramientas (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: anulaciones por canal y remitente de la política de herramientas (se admite el comodín `"*"`).
- Las claves de `toolsBySender` deben usar prefijos explícitos: `channel:`, `id:`, `e164:`, `username:`, `name:` (las claves heredadas sin prefijo siguen asignándose únicamente a `id:`).
- `channels.msteams.authType`: tipo de autenticación: `"secret"` (valor predeterminado) o `"federated"`.
- `channels.msteams.certificatePath`: ruta al archivo de certificado PEM (autenticación federada y mediante certificado).
- `channels.msteams.certificateThumbprint`: huella digital del certificado; se acepta, pero no es obligatoria para la autenticación.
- `channels.msteams.useManagedIdentity`: habilita la autenticación mediante identidad administrada (modo federado).
- `channels.msteams.managedIdentityClientId`: identificador de cliente de la identidad administrada asignada por el usuario.
- `channels.msteams.sharePointSiteId`: identificador del sitio de SharePoint para cargar archivos en chats grupales/canales (consulte [Envío de archivos en chats grupales](#sending-files-in-group-chats)).
- `channels.msteams.welcomeCard`, `channels.msteams.groupWelcomeCard`, `channels.msteams.promptStarters`: tarjeta adaptable de bienvenida que se muestra en el primer contacto por mensaje directo/grupo y sus botones de instrucciones sugeridas.
- `channels.msteams.responsePrefix`: texto que se antepone a las respuestas salientes.
- `channels.msteams.feedbackEnabled` (valor predeterminado: `true`), `channels.msteams.feedbackReflection` (valor predeterminado: `true`), `channels.msteams.feedbackReflectionCooldownMs`: comentarios de aprobación/rechazo sobre las respuestas y seguimiento de reflexión ante comentarios negativos.
- `channels.msteams.sso`, `channels.msteams.delegatedAuth`: conexión OAuth de Bot Framework y ámbitos delegados de Graph para flujos respaldados por SSO; `sso.enabled: true` requiere `sso.connectionName`.

## Enrutamiento y sesiones

- Las claves de sesión siguen el formato estándar del agente (consulte [/conceptos/sesión](/es/concepts/session)):
  - Los mensajes directos comparten la sesión principal (`agent:<agentId>:<mainKey>`).
  - Los mensajes de canal/grupo utilizan el identificador de conversación:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## Estilo de respuesta: hilos frente a publicaciones

Teams tiene dos estilos de interfaz de canal sobre el mismo modelo de datos subyacente:

| Estilo                    | Descripción                                               | `replyStyle` recomendado |
| ------------------------ | --------------------------------------------------------- | ------------------------ |
| **Publicaciones** (clásico)      | Los mensajes aparecen como tarjetas con respuestas encadenadas debajo | `thread` (valor predeterminado)       |
| **Hilos** (similar a Slack) | Los mensajes fluyen linealmente, de forma más parecida a Slack                   | `top-level`              |

**El problema:** la API de Teams no indica qué estilo de interfaz utiliza un canal. Si se utiliza el valor de `replyStyle` incorrecto:

- `thread` en un canal con estilo de hilos → las respuestas aparecen anidadas de forma poco natural.
- `top-level` en un canal con estilo de publicaciones → las respuestas aparecen como publicaciones independientes de nivel superior en lugar de hacerlo dentro del hilo.

**Solución:** configure `replyStyle` por canal según cómo esté configurado el canal:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

### Precedencia de resolución

Cuando el bot envía una respuesta a un canal, `replyStyle` se resuelve desde la anulación más específica hasta el valor predeterminado. Se utiliza el primer valor que no sea `undefined`:

1. **Por canal**: `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`
2. **Por equipo**: `channels.msteams.teams.<teamId>.replyStyle`
3. **Global**: `channels.msteams.replyStyle`
4. **Valor predeterminado implícito**: derivado de `requireMention`:
   - `requireMention: true` → `thread`
   - `requireMention: false` → `top-level`

Si se configura `requireMention: false` globalmente sin un valor explícito de `replyStyle`, las menciones en canales con estilo de publicaciones aparecen como publicaciones de nivel superior incluso cuando la entrada era una respuesta dentro de un hilo. Fije `replyStyle: "thread"` en el nivel global, de equipo o de canal para evitar sorpresas.

Para los envíos proactivos a una conversación de canal almacenada (respuestas en cola a llamadas de herramientas, agentes de larga duración), se aplica la misma resolución de equipo/canal; los chats grupales y las conversaciones personales (mensajes directos) siempre se resuelven como `top-level` para los envíos proactivos, independientemente de `replyStyle`.

### Conservación del contexto del hilo

Cuando `replyStyle: "thread"` está activo y se ha @mencionado al bot desde dentro de un hilo del canal, OpenClaw vuelve a adjuntar la raíz original del hilo a la referencia de la conversación saliente (`19:...@thread.tacv2;messageid=<root>`) para que la respuesta llegue al mismo hilo. Esto se aplica tanto a los envíos en directo (dentro del turno) como a los envíos proactivos realizados después de que haya caducado el contexto del turno de Bot Framework (por ejemplo, agentes de larga duración y respuestas en cola a llamadas de herramientas mediante `mcp__openclaw__message`).

La raíz del hilo se obtiene de `threadId` almacenado en la referencia de conversación. Las referencias almacenadas más antiguas, anteriores a `threadId`, recurren a `activityId` (la actividad entrante que haya inicializado la conversación más recientemente), por lo que las implementaciones existentes siguen funcionando sin tener que volver a inicializarse.

Cuando `replyStyle: "top-level"` está activo, las entradas de hilos de canal se responden intencionadamente como nuevas publicaciones de nivel superior; no se adjunta ningún sufijo de hilo. Este comportamiento es correcto para los canales con estilo de hilos; si aparecen publicaciones de nivel superior donde se esperaban respuestas encadenadas, `replyStyle` está configurado incorrectamente para ese canal.

## Archivos adjuntos e imágenes

**Limitaciones actuales:**

- **Mensajes directos:** las imágenes y los archivos adjuntos funcionan mediante las API de archivos para bots de Teams.
- **Canales/grupos:** los archivos adjuntos se encuentran en el almacenamiento de M365 (SharePoint/OneDrive). La carga útil del Webhook solo incluye un fragmento HTML, no los bytes reales del archivo. **Se requieren permisos de la API de Graph** para descargar archivos adjuntos de canales.
- Para envíos explícitos que priorizan el archivo, utilice `action=upload-file` con `media` / `filePath` / `path`; el valor opcional `message` se convierte en el texto/comentario adjunto y `filename` (o `title`) anula el nombre del archivo cargado.

Sin permisos de Graph, los mensajes de canal con imágenes llegan únicamente como texto (el bot no puede acceder al contenido de la imagen).
De forma predeterminada, OpenClaw solo descarga contenido multimedia de nombres de host de Microsoft/Teams. Anule este comportamiento con `channels.msteams.mediaAllowHosts` (utilice `["*"]` para permitir cualquier host).
Los encabezados Authorization solo se adjuntan para los hosts incluidos en `channels.msteams.mediaAuthAllowHosts` (de forma predeterminada, hosts de Graph y Bot Framework). Mantenga esta lista estricta (evite los sufijos multiinquilino).

## Envío de archivos en chats grupales

Los bots pueden enviar archivos en mensajes directos mediante el flujo FileConsentCard integrado. **El envío de archivos en chats grupales/canales** requiere una configuración adicional:

| Contexto                  | Cómo se envían los archivos                           | Configuración necesaria                                    |
| ------------------------ | -------------------------------------------- | ----------------------------------------------- |
| **Mensajes directos**                  | FileConsentCard → el usuario acepta → el bot carga el archivo | Funciona sin configuración adicional                            |
| **Chats grupales/canales** | Carga en SharePoint → tarjeta de archivo nativa      | Requiere `sharePointSiteId` y permisos de Graph |
| **Imágenes (cualquier contexto)** | Incorporadas en Base64                        | Funciona sin configuración adicional                            |

### Por qué los chats grupales necesitan SharePoint

Los bots utilizan una identidad de aplicación, mientras que el recurso `/me` de Microsoft Graph [requiere un usuario que haya iniciado sesión](https://learn.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0). Para enviar archivos en chats grupales/canales, el bot los carga en un **sitio de SharePoint** y crea un enlace para compartir.

### Configuración

1. **Añada permisos de la API de Graph** en Entra ID (Azure AD) → App Registration:
   - `Sites.ReadWrite.All` (Aplicación): permite cargar archivos en SharePoint.
   - `ChatMember.Read.All` (Aplicación): permiso con privilegios mínimos para todo el inquilino destinado al envío de archivos en chats grupales. `Chat.Read.All` también funciona y ya lo cubre cuando está habilitado el historial de chats grupales. Como alternativa por chat, utilice el [permiso de consentimiento específico del recurso](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent) `ChatMember.Read.Chat`.
2. **Conceda el consentimiento del administrador** para el inquilino.
3. **Obtenga el identificador del sitio de SharePoint:**

   ```bash
   # Mediante Graph Explorer o curl con un token válido:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # Ejemplo: para un sitio en "contoso.sharepoint.com/sites/BotFiles"
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # La respuesta incluye: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **Configurar OpenClaw:**

   ```json5
   {
     channels: {
       msteams: {
         // ... otra configuración ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### Comportamiento del uso compartido

| Contexto y permiso                                                     | Comportamiento del uso compartido                                      |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Canal + `Sites.ReadWrite.All`                                             | Enlace para toda la organización (cualquier miembro puede acceder)      |
| Chat grupal + `Sites.ReadWrite.All` + un permiso de lectura compatible para miembros del chat | Enlace por usuario (solo pueden acceder los miembros del chat) |
| Chat grupal sin un permiso de lectura compatible para miembros del chat | El envío falla de forma segura                                          |

El uso compartido por usuario es más seguro, ya que solo los participantes del chat pueden acceder al archivo. OpenClaw requiere una búsqueda correcta de miembros para los chats grupales; los tiempos de espera agotados, los fallos de transporte, los resultados vacíos y las denegaciones de la API de Graph hacen que el envío falle en lugar de ampliar el acceso a toda la organización.

### Comportamiento alternativo

| Situación                                                        | Resultado                                                 |
| ---------------------------------------------------------------- | --------------------------------------------------------- |
| Chat grupal + archivo + permisos de SharePoint y de miembros configurados | Se carga en SharePoint y se envía una tarjeta de archivo nativa |
| Chat grupal + archivo + faltan permisos de SharePoint o de miembros | Falla con un error de configuración que indica cómo actuar |
| Canal + archivo + `sharePointSiteId` configurado                 | Se carga en SharePoint y se envía una tarjeta de archivo nativa |
| Chat personal + archivo                                          | Flujo FileConsentCard (funciona sin SharePoint)           |
| Cualquier contexto + imagen                                      | Codificación Base64 insertada (funciona sin SharePoint)   |

### Ubicación de los archivos almacenados

Los archivos cargados se almacenan en una carpeta `/OpenClawShared/` de la biblioteca de documentos predeterminada del sitio de SharePoint configurado.

## Encuestas (tarjetas adaptables)

OpenClaw envía las encuestas de Teams como tarjetas adaptables (Teams no dispone de una API nativa para encuestas).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> --poll-question "..." --poll-option "..." --poll-option "..."`.
- El Gateway registra los votos en la base de datos SQLite del estado del Plugin de OpenClaw, en `state/openclaw.sqlite`.
- Los archivos `msteams-polls.json` existentes se importan mediante `openclaw doctor --fix`, no mediante el Plugin en ejecución.
- El Gateway debe permanecer en línea para registrar los votos.
- Las encuestas no publican automáticamente resúmenes de resultados y todavía no existe una CLI para consultar sus resultados.

## Tarjetas de presentación

Envíe cargas útiles de presentación semánticas a usuarios o conversaciones de Teams mediante la herramienta `message`, la CLI o la entrega normal de respuestas. OpenClaw las representa como tarjetas adaptables de Teams a partir del contrato de presentación genérico.

El parámetro `presentation` acepta bloques semánticos. Cuando se proporciona `presentation`, el texto del mensaje es opcional. Los botones se representan como acciones de envío o de URL de las tarjetas adaptables. Los menús de selección no son nativos del representador de Teams, por lo que OpenClaw los convierte en texto legible antes de la entrega.

**Herramienta del agente:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  presentation: {
    title: "Hola",
    blocks: [{ type: "text", text: "¡Hola!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Hola","blocks":[{"type":"text","text":"¡Hola!"}]}'
```

Para obtener detalles sobre el formato de los destinos, consulte [Formatos de destino](#target-formats) más adelante.

## Formatos de destino

Los destinos de MSTeams usan prefijos para distinguir entre usuarios y conversaciones:

| Tipo de destino       | Formato                          | Ejemplo                                                                                                |
| --------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Usuario (por ID)      | `user:<aad-object-id>`               | `user:40a1a0ed-4ff2-4164-a219-55518990c197`                                                                                     |
| Usuario (por nombre)  | `user:<display-name>`               | `user:John Smith` (requiere la API de Graph)                                                          |
| Grupo/canal           | `conversation:<conversation-id>`               | `conversation:19:abc123...@thread.tacv2`                                                                                     |
| Grupo/canal (sin procesar) | `<conversation-id>`          | `19:abc123...@thread.tacv2`, `19:...@unq.gbl.spaces` o un identificador de Bot Framework `a:`/`8:orgid:`/`29:` sin prefijo |

**Ejemplos de la CLI:**

```bash
# Enviar a un usuario por ID
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "Hola"

# Enviar a un usuario por nombre para mostrar (activa una búsqueda en la API de Graph)
openclaw message send --channel msteams --target "user:John Smith" --message "Hola"

# Enviar a un chat grupal o canal
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "Hola"

# Enviar una tarjeta de presentación a una conversación
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"Hola","blocks":[{"type":"text","text":"Hola"}]}'
```

**Ejemplos de la herramienta del agente:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "¡Hola!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  presentation: {
    title: "Hola",
    blocks: [{ type: "text", text: "Hola" }],
  },
}
```

<Note>
Sin el prefijo `user:`, los nombres se resuelven de forma predeterminada como grupos o equipos. Use siempre `user:` al seleccionar personas por su nombre para mostrar.
</Note>

## Mensajería proactiva

- Los mensajes proactivos solo son posibles **después** de que un usuario haya interactuado, porque OpenClaw almacena las referencias de conversación en ese momento.
- Consulte [/gateway/configuration](/es/gateway/configuration) para obtener información sobre `dmPolicy` y el control mediante listas de permitidos.

## Identificadores de equipos y canales (error habitual)

El parámetro de consulta `groupId` de las URL de Teams **NO** es el identificador de equipo utilizado para la configuración. En su lugar, extraiga los identificadores de la ruta de la URL:

**URL del equipo:**

```text
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    Identificador de conversación del equipo (descodifique la URL)
```

**URL del canal:**

```text
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      Identificador del canal (descodifique la URL)
```

**Para la configuración:**

- Clave del equipo = segmento de la ruta posterior a `/team/` (descodificado de la URL; por ejemplo, `19:Bk4j...@thread.tacv2`; los inquilinos más antiguos pueden mostrar `@thread.skype`, que también es válido).
- Clave del canal = segmento de la ruta posterior a `/channel/` (descodificado de la URL).
- **Ignore** el parámetro de consulta `groupId` para el enrutamiento de OpenClaw. Es el identificador de grupo de Microsoft Entra, no el identificador de conversación de Bot Framework utilizado en las actividades entrantes de Teams.

## Canales privados

Los bots tienen compatibilidad limitada con los canales privados:

| Función                      | Canales estándar | Canales privados              |
| ---------------------------- | ---------------- | ----------------------------- |
| Instalación del bot          | Sí               | Limitada                      |
| Mensajes en tiempo real (Webhook) | Sí          | Es posible que no funcione    |
| Permisos RSC                 | Sí               | Pueden comportarse de otro modo |
| @menciones                   | Sí               | Si el bot está accesible      |
| Historial de la API de Graph | Sí               | Sí (con permisos)             |

**Alternativas si los canales privados no funcionan:**

1. Use canales estándar para las interacciones con el bot.
2. Use mensajes directos; los usuarios siempre pueden enviar mensajes directamente al bot.
3. Use la API de Graph para acceder al historial (requiere `ChannelMessage.Read.All`).

## Solución de problemas

### Problemas habituales

- **Las imágenes no aparecen en los canales:** faltan permisos de Graph o el consentimiento del administrador. Reinstale la aplicación de Teams, ciérrela por completo y vuelva a abrirla.
- **No hay respuestas en el canal:** las menciones son obligatorias de forma predeterminada; establezca `channels.msteams.requireMention=false` o configure cada equipo/canal.
- **Discrepancia de versión (Teams sigue mostrando el manifiesto antiguo):** elimine y vuelva a agregar la aplicación, y cierre Teams por completo para actualizarla.
- **Respuesta 401 Unauthorized del Webhook:** es lo esperado al realizar pruebas manuales sin un JWT de Azure; significa que se puede acceder al punto de conexión, pero la autenticación ha fallado. Use Azure Web Chat para realizar la prueba correctamente.

### Errores de carga del manifiesto

- **"Icon file cannot be empty":** el manifiesto hace referencia a archivos de iconos de 0 bytes. Cree iconos PNG válidos (32x32 para `outline.png`, 192x192 para `color.png`).
- **"webApplicationInfo.Id already in use":** la aplicación sigue instalada en otro equipo/chat. Localícela y desinstálela primero, o espere entre 5 y 10 minutos para que se propaguen los cambios.
- **"Something went wrong" durante la carga:** en su lugar, realice la carga mediante [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com), abra las herramientas para desarrolladores del navegador (F12) → pestaña Network y compruebe el cuerpo de la respuesta para ver el error real.
- **Error de instalación local:** pruebe "Upload an app to your org's app catalog" en lugar de "Upload a custom app"; esto suele evitar las restricciones de instalación local.

### Los permisos RSC no funcionan

1. Compruebe que `webApplicationInfo.id` coincida exactamente con el identificador de aplicación de su bot.
2. Vuelva a cargar la aplicación y reinstálela en el equipo/chat.
3. Compruebe si el administrador de su organización ha bloqueado los permisos RSC.
4. Confirme que utiliza el ámbito correcto: `ChannelMessage.Read.Group` para equipos y `ChatMessage.Read.Chat` para chats grupales.

## Referencias

- [Crear un bot de Azure](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - guía de configuración de Azure Bot
- [Portal para desarrolladores de Teams](https://dev.teams.microsoft.com/apps) - creación y administración de aplicaciones de Teams
- [Esquema del manifiesto de aplicaciones de Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [Recibir mensajes de canal con RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [Referencia de permisos RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [Gestión de archivos mediante bots de Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) (los canales/grupos requieren Graph)
- [Mensajería proactiva](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [@microsoft/teams.cli](https://www.npmjs.com/package/@microsoft/teams.cli) - CLI de Teams para administrar bots

## Contenido relacionado

- [Descripción general de los canales](/es/channels) - todos los canales compatibles
- [Vinculación](/es/channels/pairing) - autenticación por mensaje directo y flujo de vinculación
- [Grupos](/es/channels/groups) - comportamiento del chat grupal y restricción mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) - enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) - modelo de acceso y refuerzo de la seguridad
