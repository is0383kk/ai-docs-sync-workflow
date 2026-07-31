---
read_when:
    - Trabajo en la resolución de perfiles de autenticación o el enrutamiento de credenciales
    - Depuración de fallos de autenticación del modelo o del orden de los perfiles
summary: Semántica canónica de elegibilidad y resolución de credenciales para perfiles de autenticación
title: Semántica de las credenciales de autenticación
x-i18n:
    generated_at: "2026-07-26T05:00:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0516b1bb23f400d5ac5fd39a628736034440216ac22823eef061b38564dff0
    source_path: auth-credential-semantics.md
    workflow: 16
---

Estas semánticas mantienen alineado el comportamiento de autenticación durante la selección y en tiempo de ejecución. Las comparten:

- `resolveAuthProfileOrder` (orden de perfiles)
- `resolveApiKeyForProfile` (resolución de credenciales en tiempo de ejecución)
- `openclaw models status --probe`
- comprobaciones de autenticación de `openclaw doctor` (`doctor-auth`)

## Códigos estables de motivo de sondeo

Los resultados del sondeo incluyen una categoría `status` (`ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`) y un `reasonCode` estable cuando el sondeo nunca llegó a realizar una llamada al modelo:

| `reasonCode`             | Significado                                                                      |
| ------------------------ | ---------------------------------------------------------------------------- |
| `excluded_by_auth_order` | Perfil omitido del orden de autenticación explícito de su proveedor.               |
| `missing_credential`     | No hay ninguna credencial en línea ni SecretRef configurada.                             |
| `expired`                | El token `expires` está en el pasado.                                              |
| `invalid_expires`        | `expires` no es una marca de tiempo Unix válida y positiva en ms.                         |
| `unresolved_ref`         | No se pudo resolver la SecretRef configurada.                                  |
| `ineligible_profile`     | El perfil es incompatible con la configuración del proveedor (incluye una entrada de clave con formato incorrecto). |
| `no_model`               | Existen credenciales, pero no se resolvió ningún modelo candidato que se pueda sondear.                 |

Las comprobaciones de idoneidad indican `ok` como código de motivo para las credenciales utilizables.

## Credenciales de token

Las credenciales de token (`type: "token"`) admiten `token` en línea y/o `tokenRef`.

### Reglas de idoneidad

1. Un perfil de token no es apto cuando faltan tanto `token` como `tokenRef` (`missing_credential`).
2. `expires` es opcional. Cuando está presente, debe ser un número finito de milisegundos desde la época Unix mayor que `0` y no superior a la marca de tiempo `Date` máxima de JavaScript (8640000000000000).
3. Si `expires` no es válido (tipo incorrecto, `NaN`, `0`, negativo, no finito o superior a ese máximo), el perfil no es apto y se indica `invalid_expires`.
4. Si `expires` está en el pasado, el perfil no es apto y se indica `expired`.
5. `tokenRef` no omite la validación de `expires`.

### Reglas de resolución

1. Las semánticas del resolutor coinciden con las de idoneidad para `expires`.
2. Para los perfiles aptos, el material del token puede resolverse a partir del valor en línea o de `tokenRef`.
3. Las referencias que no se puedan resolver generan `unresolved_ref` en la salida de `models status --probe`.

## Portabilidad de copias entre agentes

La herencia de autenticación de agentes se realiza mediante lectura directa. Cuando un agente no tiene un perfil local, resuelve los perfiles desde el almacén del agente predeterminado/principal en tiempo de ejecución sin copiar material secreto en su propio almacén de credenciales (`agents/<agentId>/agent/openclaw-agent.sqlite`).

Los flujos de copia explícitos, como `openclaw agents add`, utilizan esta política de portabilidad:

- Los perfiles `api_key` y `token` son portátiles, salvo que `copyToAgents: false`.
- Los perfiles `oauth` no son portátiles de forma predeterminada porque los tokens de actualización pueden ser de un solo uso o sensibles a la rotación.
- Los flujos de OAuth gestionados por el proveedor pueden habilitarlo mediante `copyToAgents: true` solo cuando se sabe que es seguro copiar el material de actualización entre agentes; la habilitación solo se aplica cuando el perfil contiene material de acceso/actualización en línea.

Los perfiles no portátiles siguen disponibles mediante herencia de lectura directa, salvo que el agente de destino inicie sesión por separado y cree su propio perfil local.

## Rutas de autenticación solo de configuración

Las entradas `auth.profiles` con `mode: "aws-sdk"` son metadatos de enrutamiento, no credenciales almacenadas. Son válidas cuando el proveedor de destino utiliza `models.providers.<id>.auth: "aws-sdk"`, la ruta que escribe la configuración de Amazon Bedrock gestionada por el Plugin. Estos identificadores de perfil pueden aparecer en `auth.order` y en las anulaciones de sesión incluso cuando no existe ninguna entrada coincidente en el almacén de credenciales.

No se debe escribir `type: "aws-sdk"` en el almacén de credenciales; las credenciales almacenadas solo pueden ser `api_key`, `token` o `oauth`. Si un `auth-profiles.json` heredado contiene dicho marcador, `openclaw doctor --fix` lo mueve a `auth.profiles` y elimina el marcador del almacén.

## Filtrado explícito del orden de autenticación

- Cuando se establece `auth.order.<provider>` o la anulación del orden del almacén de autenticación para un proveedor, `models status --probe` solo sondea los identificadores de perfil que permanecen en el orden de autenticación resuelto para ese proveedor. La anulación almacenada tiene prioridad sobre la configuración de `auth.order`.
- Un perfil almacenado para ese proveedor que se omita del orden explícito no se prueba silenciosamente más adelante. La salida del sondeo lo indica con `reasonCode: excluded_by_auth_order` y el detalle `Excluded by auth.order for this provider.`

## Resolución del destino del sondeo

- Los destinos del sondeo pueden proceder de perfiles de autenticación, credenciales de entorno o `models.json` (resultado `source`: `profile`, `env`, `models.json`).
- Si un proveedor tiene credenciales, pero OpenClaw no puede resolver un modelo candidato que se pueda sondear, `models status --probe` indica `status: no_model` con `reasonCode: no_model`.

## Detección de credenciales de CLI externas

- Las credenciales exclusivas del tiempo de ejecución gestionadas por CLI externas (Claude CLI para `claude-cli`, Codex CLI para `openai`, MiniMax CLI para `minimax-portal`) solo se detectan cuando el proveedor, el entorno de ejecución o el perfil de autenticación están dentro del ámbito de la operación actual, o cuando ya existe un perfil local almacenado para esa fuente externa.
- Los invocadores del almacén de autenticación eligen un modo explícito de detección de CLI externas: `none` únicamente para autenticación persistida/del Plugin, `existing` para actualizar perfiles de CLI externas ya almacenados o `scoped` para un conjunto concreto de proveedores/perfiles.
- Las rutas de solo lectura/estado pasan `allowKeychainPrompt: false`; utilizan únicamente credenciales de CLI externas respaldadas por archivos y no leen ni reutilizan los resultados del llavero de macOS.

## Protección de la política de SecretRef para OAuth

La entrada SecretRef es únicamente para credenciales estáticas. Las credenciales de OAuth son mutables en tiempo de ejecución (los flujos de actualización conservan los tokens rotados), por lo que el material de OAuth respaldado por SecretRef dividiría el estado mutable entre almacenes.

- Si la credencial de un perfil es `type: "oauth"`, se rechazan los objetos SecretRef para cualquier campo de material de credenciales de ese perfil.
- Si `auth.profiles.<id>.mode` es `"oauth"`, se rechaza la entrada `keyRef`/`tokenRef` respaldada por SecretRef para ese perfil.
- Las infracciones son fallos definitivos (errores lanzados) en las rutas de preparación de secretos durante el inicio o la recarga y en las rutas de resolución de perfiles.

## Mensajería compatible con versiones heredadas

Para mantener la compatibilidad con scripts, los errores del sondeo conservan esta primera línea sin cambios:

`Auth profile credentials are missing or expired.`

A continuación, en líneas posteriores, aparecen detalles fáciles de entender y el código de motivo estable con el formato `↳ Auth reason [code]: ...`.

## Relacionado

- [Gestión de secretos](/es/gateway/secrets)
- [Almacenamiento de autenticación](/es/concepts/oauth)
